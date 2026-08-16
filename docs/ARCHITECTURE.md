# B2B Pharmacy Platform — Backend Architecture & Development Plan

**Prepared by:** Curlware Digital Agency
**Date:** August 16, 2026
**Version:** 1.1
**Scope:** Backend only — API, data model, jobs, infrastructure, delivery plan

---

## 0. Confirmed Decisions

These were settled before design. Everything downstream follows from them; changing one changes the architecture, not just the code.

| Decision | Answer | Architectural consequence |
| :--- | :--- | :--- |
| Scale at 12 months | ~2,000 pharmacies, ~1,000 orders/day, ~20,000 products | Single-instance modular monolith is correct. No microservices, no dedicated search engine. |
| Stock deduction | **Reserved at order placement**, auto-release after 3 hours | Requires a reservation ledger, row-level locking, and a scheduled release mechanism. This is the hardest part of the system. |
| Partial approval | Admin may drop or reduce individual lines | Order totals are recalculated post-decision; line items need their own status and an original-vs-approved snapshot. |
| Delivery status | No agent app. Admin/warehouse marks status **and** courier APIs push status back | Two write paths into one delivery state. Courier adapter layer + signed webhook ingestion. |
| Couriers | Pathao, Steadfast, RedX | Provider-agnostic interface with three implementations. |
| Digital payment | Captured **after approval**, on the final approved amount | Introduces an `AWAITING_PAYMENT` state between approval and confirmation, and extends reservation life through the payment window. |
| Returns / damaged goods | Out of scope at launch | Stock ledger supports the movement type; no API or client flow built. |
| Order money model | **Delivery charge + VAT**, both on every order | Order total is no longer subtotal-minus-discount. Needs zone table, settings table, and a fixed calculation order. |
| Delivery charge | By district/zone, admin-configurable | `delivery_zones` table; zone resolved from pharmacy district and **snapshotted** on the order. |
| VAT | Single rate on the whole order, admin-configurable | Lives on the order, not the product. No per-item exemption handling. |
| Minimum order | **Both** order value and per-product minimums | Enforced at cart and re-validated at placement. Products gain `min_order_qty` and `order_multiple`. |
| Invoice / challan | System-generated PDF | New `invoices` module — immutable snapshot, gapless numbering, async generation. |
| Reservation expiry | 3 hours, plain clock time | Simple. Accepts that overnight orders expire unreviewed — mitigated by the expiry UX below. |
| Expired reservation | Auto-cancel + push with one-tap reorder | Notification payload carries the original lines for re-pricing. |
| Catalogue seeding | Client supplies Excel/CSV; we build the importer | Staged importer with dry-run validation is a real deliverable, not a script. |
| Dispatch mode | Admin picks internal vs courier per order | No routing rules engine. `deliveries.mode` chosen at dispatch. |
| Store credit | Removed from scope | No credit tables, no exposure checks. |
| Runtime & framework | **NestJS 11 on Node.js 24 LTS, TypeScript 6.0** | Changed from the earlier Bun + Hono direction. Drizzle retained as the data layer. |

> **Note on the stack change.** The quotation and earlier planning assumed Bun + Hono. NestJS is a Node framework — Bun support exists but is not a supported production target for the full Nest ecosystem (Bull, Terminus, microservice transports). Drizzle carries over unchanged.

---

## 1. Design Principles

Five rules that this plan is measured against.

1. **Simple by default, strong where it counts.** Complexity is spent only where the domain demands it — stock reservation, order approval, payment sequencing. Everything else is boring CRUD, deliberately.
2. **One deployable.** A modular monolith with hard internal boundaries. At 1,000 orders/day, microservices would add distributed-transaction problems to a system whose core difficulty is already a transaction.
3. **The database is the source of truth, not the cache.** Availability is never served from cache. Money and stock are never computed outside a transaction.
4. **Everything that changes stock or money is an append-only ledger entry.** Balances are derived and cached in columns; the ledger is the record. Pharma requires this for audit and it makes reconciliation possible.
5. **Idempotency everywhere at the edges.** Mobile clients retry. Couriers redeliver webhooks. Payment gateways send duplicate callbacks. Every external write path is safe to replay.

### Load reality check

1,000 orders/day is roughly 40/hour, peaking maybe 150/hour. Read traffic from 2,000 pharmacies browsing a 20,000-product catalogue peaks in the low hundreds of requests per second. **A single well-indexed Postgres instance and two application containers handle this with a large margin.** This plan is sized for that reality, with a documented scaling path — not for imagined load.

---

## 2. System Context

```
┌──────────────────┐     ┌──────────────────┐     ┌────────────────────┐
│  Pharmacy App    │     │   Admin Panel    │     │  Courier Partners  │
│  (React Native)  │     │    (Next.js)     │     │ Pathao/Steadfast/  │
│                  │     │                  │     │       RedX         │
└────────┬─────────┘     └────────┬─────────┘     └─────────┬──────────┘
         │ REST/JSON              │ REST/JSON               │ webhooks
         │ Bearer JWT             │ Cookie + JWT            │ HMAC-signed
         └────────────────────────┴─────────────────────────┘
                                  │
                        ┌─────────▼──────────┐
                        │   Caddy (TLS/LB)   │
                        └─────────┬──────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
      ┌───────▼────────┐  ┌───────▼────────┐  ┌───────▼────────┐
      │  API container │  │  API container │  │ Worker container│
      │   (NestJS)     │  │   (NestJS)     │  │ (same image,    │
      │                │  │                │  │  BullMQ only)   │
      └───────┬────────┘  └───────┬────────┘  └───────┬────────┘
              └───────────────────┼───────────────────┘
                    ┌─────────────┼─────────────┐
             ┌──────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐
             │ PostgreSQL  │ │  Redis   │ │ S3 Object  │
             │  (managed)  │ │ cache+   │ │  Storage   │
             │             │ │ queues   │ │  + CDN     │
             └─────────────┘ └──────────┘ └────────────┘
```

**Three consumers, one API.** The pharmacy app and admin panel share the same REST surface, separated by role and route prefix (`/api/v1/app/*`, `/api/v1/admin/*`), not by separate services. Courier webhooks land on a dedicated unauthenticated-but-signature-verified prefix (`/api/v1/webhooks/*`).

**The worker runs the same image as the API** with a different entrypoint, so there is one build artifact and no drift between what the API and the jobs believe about the domain.

---

## 3. Technology Stack

| Layer | Choice | Why this and not the alternative |
| :--- | :--- | :--- |
| Runtime | Node.js 24 LTS | Active LTS since Oct 2025, EOL Apr 2028. Node 22 is Maintenance LTS (EOL Apr 2027) — a migration target for existing services, not a greenfield choice. Node 26 is Current until Oct 2026. |
| Framework | NestJS 11.2.x | Module boundaries, DI, guards/interceptors/pipes as first-class. The structure is the point — it's what keeps a monolith from rotting. |
| Language | TypeScript 6.0 | Last JavaScript-based release (GA Mar 2026). `strict` is on by default. See the TS 7 note below. |
| Compiler | SWC (`nest build -b swc`) | ~20× faster builds than tsc; `tsc --noEmit` still runs in CI for type safety. |
| ORM | Drizzle ORM | SQL-shaped, no query-engine binary, trivial to drop to raw SQL for `FOR UPDATE` locking — which this domain needs constantly. Prisma's abstraction fights you exactly where reservations live. |
| Database | PostgreSQL 17 | Relational integrity across order → line → reservation → movement. Also provides full-text search, so no separate search engine. |
| Cache & queues | Redis 7 + BullMQ | Same infrastructure serves cache, rate limiting, and delayed jobs (reservation expiry). |
| Validation | Zod + `nestjs-zod` | One schema drives DTO type, runtime validation, and OpenAPI. |
| Auth | `@nestjs/jwt` + argon2 | Access/refresh with rotation and reuse detection. |
| Docs | `@nestjs/swagger`, generated from Zod | Mobile team consumes OpenAPI directly. |
| Logging | Pino (`nestjs-pino`) | Structured JSON, request-scoped correlation IDs. |
| Testing | Vitest + Testcontainers + Supertest | Vitest aligns with where Nest 12 is heading. Testcontainers means integration tests hit real Postgres. |
| Health | `@nestjs/terminus` | `/health/live`, `/health/ready` for orchestration. |
| Errors | Sentry | Release-tagged, source-mapped. |
| Container | Docker, multi-stage, distroless base | Small image, non-root user. |

### On TypeScript 7

TypeScript 7.0 went GA in July 2026 — the Go port, roughly 10× faster type-checking. **Do not adopt it on this project yet.**

It ships without a public compiler API until 7.1, which breaks `typescript-eslint`, `ts-morph`, and any custom transformer. This codebase uses `typescript-eslint`. Because the build runs through SWC and `tsc` is only doing `--noEmit`, the speed gain applies to the check step alone — not worth losing linting for. Revisit when 7.1 ships the stable API.

**TS 6.0 configuration notes.** 6.0 was a defaults-breaking release. For NestJS 11, set explicitly:

```jsonc
{
  "module": "commonjs",          // 6.0 defaults to esnext; Nest 11 is CJS
  "moduleResolution": "bundler", // "node" was removed in 6.0
  "target": "es2022",            // 6.0 defaults to es2025
  "strict": true,                // now the default, but be explicit
  "noUncheckedIndexedAccess": true,
  "experimentalDecorators": true,
  "emitDecoratorMetadata": true
}
```

`baseUrl` was removed — declare path aliases under `paths` without it.

### On NestJS 12

v12 targets early Q3 2026: full ESM, Vitest as default, Rspack, and native Standard Schema validation in route decorators. **Build on v11 now.** The choices above — Vitest, Zod, SWC — are deliberately the ones that make the v12 upgrade a configuration change rather than a rewrite. Revisit at the 6-month maintenance review.

---

## 4. Module Architecture

Twelve modules. Each owns its tables and exposes a service; **no module reaches into another module's tables directly** — cross-module reads go through the owning service. This is what makes a future extraction possible without making it necessary now.

```
src/
├─ main.ts                     # bootstrap, global pipes/filters/interceptors
├─ worker.ts                   # BullMQ-only entrypoint (same image)
├─ app.module.ts
│
├─ common/                     # framework-level, no domain logic
│  ├─ decorators/              # @CurrentUser, @Roles, @Public, @Idempotent
│  ├─ guards/                  # JwtAuthGuard, RolesGuard, PharmacyScopeGuard
│  ├─ interceptors/            # LoggingInterceptor, TransformInterceptor
│  ├─ filters/                 # AllExceptionsFilter → RFC 7807 problem+json
│  ├─ pipes/                   # ZodValidationPipe
│  └─ pagination/              # keyset cursor helpers
│
├─ infra/
│  ├─ database/                # Drizzle provider, schema, migrations, tx helper
│  ├─ redis/                   # cache service, key registry
│  ├─ queue/                   # BullMQ registration, base processor
│  ├─ storage/                 # S3 client, presigned upload/download
│  └─ config/                  # Zod-validated env, fails fast at boot
│
└─ modules/
   ├─ auth/                    # login, refresh rotation, password reset, OTP
   ├─ users/                   # staff + pharmacy users, roles
   ├─ pharmacies/              # registration, documents, approval queue
   ├─ catalog/                 # manufacturers, generics, products, price tiers
   ├─ search/                  # tsvector + trigram query layer
   ├─ inventory/               # stock ledger, reservations, adjustments  ★
   ├─ orders/                  # placement, approval, partial approval  ★
   ├─ pricing/                 # tier resolution, VAT, delivery zones, minimums
   ├─ payments/                # gateway adapter, post-approval capture
   ├─ invoices/                # PDF generation, numbering, immutable snapshots
   ├─ settings/                # VAT rate, minimums, zones, TTLs — DB-backed config
   ├─ fulfilment/              # picking groups, dispatch, delivery state
   ├─ couriers/                # provider adapters + webhook ingestion
   ├─ requests/                # medicine requests
   ├─ notifications/           # push, SMS, in-app
   └─ reports/                 # dashboard aggregates, Excel export
```

★ = the two modules carrying real domain complexity. Everything else is conventional.

---

## 5. Data Model

### 5.1 Entity overview

```
pharmacies ──< pharmacy_documents
     │
     └──< users (role: PHARMACY_OWNER)
     │
     └──< orders ──< order_items >── products
              │            │
              │            └──< stock_reservations
              │
              ├──< order_status_history
              ├──< payments
              └──< fulfilment_groups ──< deliveries ──< delivery_events

manufacturers ──< products >── generics        (generics = active ingredient)
                    │
                    ├──< product_price_tiers
                    └──< stock_movements       (append-only ledger)
```

### 5.2 Core tables

**`pharmacies`** — the tenant.
`id`, `name`, `owner_name`, `phone`, `email`, `address`, `district`, `status` (`PENDING | APPROVED | REJECTED | INFO_REQUIRED | SUSPENDED`), `rejection_reason`, `reviewed_by`, `reviewed_at`, `created_at`.
Indexes: `(status, created_at DESC)` for the approval queue; unique on `phone`.

**`pharmacy_documents`**
`id`, `pharmacy_id`, `type` (`DRUG_LICENSE | TRADE_LICENSE | NID | OTHER`), `file_key`, `original_name`, `mime`, `size_bytes`, `uploaded_at`.
Files live in object storage; only the key is stored. Access via short-lived presigned URLs — never public.

**`users`**
`id`, `pharmacy_id` (null for staff), `phone`, `email`, `password_hash` (argon2id), `role` (`PHARMACY_OWNER | ADMIN | WAREHOUSE_MANAGER | DELIVERY_AGENT`), `status`, `last_login_at`.
Unique on `phone`. Index `(pharmacy_id)`.

**`refresh_tokens`**
`id`, `user_id`, `token_hash`, `family_id`, `expires_at`, `revoked_at`, `replaced_by`, `user_agent`, `ip`.
Rotation with family-based reuse detection — a replayed token revokes the whole family.

**`manufacturers`** — `id`, `name`, `slug`, `logo_key`, `is_active`, `sort_order`.

**`generics`** — `id`, `name`, `strength`, `form`. This is the active-ingredient table that makes alternate brands work.

**`products`**
`id`, `name`, `manufacturer_id`, `generic_id`, `pack_size`, `mrp`, `base_price`, `is_active`, `image_key`, `low_stock_threshold`, **`min_order_qty`**, **`order_multiple`**, **`stock_on_hand`**, **`stock_reserved`**, `search_doc` (tsvector), `created_at`, `updated_at`.

`min_order_qty` is the smallest orderable quantity; `order_multiple` forces carton/pack multiples (order 12, 24, 36 — not 15). Both default to 1, so products without minimums need no special handling.

Two derived columns, both maintained inside the same transaction as the ledger entry that changes them:
- `stock_on_hand` — physical stock, sum of all `stock_movements`
- `stock_reserved` — sum of `ACTIVE` reservations
- **`available = stock_on_hand - stock_reserved`** — what a pharmacy can actually order

Indexes:
```sql
CREATE INDEX products_search_idx      ON products USING GIN (search_doc);
CREATE INDEX products_name_trgm_idx   ON products USING GIN (name gin_trgm_ops);
CREATE INDEX products_manufacturer_idx ON products (manufacturer_id) WHERE is_active;
CREATE INDEX products_generic_idx     ON products (generic_id) WHERE is_active;
CREATE INDEX products_low_stock_idx   ON products (stock_on_hand)
  WHERE is_active AND stock_on_hand <= low_stock_threshold;
```

**`product_price_tiers`** — `id`, `product_id`, `min_qty`, `unit_price`.
Unique `(product_id, min_qty)`. Tier resolution: highest `min_qty` ≤ ordered quantity; falls back to `base_price`.

**`orders`**
`id`, `order_no` (human-readable, e.g. `ORD-260816-0042`), `pharmacy_id`, `status`, `payment_method` (`COD | DIGITAL`), `payment_status` (`NOT_REQUIRED | AWAITING_PAYMENT | PAID | FAILED | EXPIRED`), `idempotency_key`, `placed_at`, `decided_by`, `decided_at`, `decision_note`, `confirmed_at`, `cancelled_at`, `cancel_reason`.

**Money columns, held in placed / approved pairs:**
`placed_subtotal`, `placed_discount`, `placed_vat`, `placed_delivery_charge`, `placed_total`
`approved_subtotal`, `approved_discount`, `approved_vat`, `approved_delivery_charge`, `approved_total`

**Snapshotted rate columns:** `vat_rate` (e.g. `0.0500`), `delivery_zone_id`, `delivery_zone_name`, `vat_applies_to_delivery` (bool).

Rates are snapshotted per order. Changing the VAT rate in settings tomorrow must not alter an order placed today — and it must not alter the invoice already issued against it.

Keeping **both** placed and approved totals is what makes "here's what changed and why" possible on the pharmacy side without reconstructing history.
Indexes: `(pharmacy_id, placed_at DESC)`, `(status, placed_at)` for the approval queue, unique `(pharmacy_id, idempotency_key)`.

**`order_items`**
`id`, `order_id`, `product_id`, `manufacturer_id` *(snapshot)*, `product_name` *(snapshot)*, `pack_size` *(snapshot)*, `requested_qty`, `approved_qty`, `unit_price` *(snapshot, tier-resolved)*, `line_total`, `status` (`PENDING | APPROVED | REDUCED | REMOVED`), `removal_reason`.

**Every price and name is snapshotted at placement.** A historical order must never change because someone edited a product. This is a common and expensive mistake — an invoice from three months ago must render exactly as it did then.

**`stock_reservations`** — the heart of the system.
`id`, `order_id`, `order_item_id`, `product_id`, `qty`, `status` (`ACTIVE | CONSUMED | RELEASED`), `expires_at`, `created_at`, `resolved_at`, `release_reason`.
Indexes: `(status, expires_at)` for the sweeper, `(order_id)`, `(product_id) WHERE status='ACTIVE'`.

**`stock_movements`** — append-only ledger. Never updated, never deleted.
`id`, `product_id`, `type` (`RECEIPT | ADJUSTMENT | ALLOCATION | RETURN`), `qty_delta` (signed), `balance_after`, `reference_type`, `reference_id`, `actor_id`, `reason`, `created_at`.
`RETURN` exists in the enum but has no client flow at launch, per scope.

**`fulfilment_groups`** — the multi-manufacturer split.
`id`, `order_id`, `manufacturer_id`, `status` (`PENDING | PICKING | PACKED`), `item_count`.
Created at order confirmation. One order remains one order to the pharmacy; the warehouse sees one picking set per manufacturer.

**`deliveries`**
`id`, `order_id`, `mode` (`INTERNAL | COURIER`), `agent_id` (internal), `courier_provider`, `consignment_id`, `tracking_code`, `status`, `assigned_at`, `dispatched_at`, `delivered_at`, `failure_reason`.

**`delivery_events`** — append-only, every status transition from either write path.
`id`, `delivery_id`, `source` (`ADMIN | COURIER_WEBHOOK | SYSTEM`), `raw_status`, `mapped_status`, `payload` (jsonb), `event_id` *(provider's, for dedup)*, `occurred_at`, `received_at`.
Unique `(courier_provider, event_id)` — the idempotency guard against redelivered webhooks.

**`medicine_requests`** — `id`, `pharmacy_id`, `product_text`, `manufacturer_text`, `qty_estimate`, `note`, `status` (`OPEN | CATALOGUED | UNAVAILABLE`), `resolution_note`, `resolved_product_id`, `resolved_by`, `resolved_at`.

**`delivery_zones`**
`id`, `name`, `districts` (text array or child table), `charge`, `is_active`, `sort_order`.
A pharmacy's district maps to exactly one active zone. Unmapped districts fall back to a configurable default charge and raise an admin warning rather than blocking the order — an unmappable address must never silently break checkout.

**`platform_settings`** — single-row, typed, cached in Redis with explicit invalidation on write.
`vat_rate`, `vat_applies_to_delivery`, `min_order_value`, `default_delivery_charge`, `reservation_ttl_hours`, `payment_window_hours`, `invoice_prefix`, `company_name`, `company_address`, `company_bin` *(VAT registration number, required on a BD invoice)*.

Every one of these was a hardcoded constant in v1.0. They are business rules, and business rules change without a deploy.

**`invoices`**
`id`, `order_id`, `invoice_no` *(gapless, from a Postgres sequence)*, `issued_at`, `pdf_key`, `snapshot` (jsonb), `total`, `void_reason`, `voided_at`.

**The `snapshot` column is the invoice.** It holds the full rendered payload — line items, prices, VAT rate, delivery charge, company details, buyer details — frozen at issue. The PDF is a rendering of the snapshot, regenerable byte-identical at any time. An invoice is a financial document: it is never edited, only voided and reissued.
Unique on `invoice_no` and on `order_id`.

**`favourites`** — `pharmacy_id`, `product_id`, `created_at`. PK `(pharmacy_id, product_id)`. Powers "My Products".

**`audit_logs`** — `id`, `actor_id`, `actor_role`, `action`, `entity_type`, `entity_id`, `before` (jsonb), `after` (jsonb), `ip`, `created_at`.
Written by an interceptor on every admin mutation. Partitioned monthly once it grows.

**`notifications`** / **`device_tokens`** — standard fan-out storage and push registration.

### 5.3 Conventions

- **UUID v7 primary keys** — globally unique, time-sortable, index-friendly. Avoids the enumeration exposure of sequential IDs on a B2B platform.
- **`timestamptz` everywhere**, UTC in the database, formatted at the edge.
- **Money as `numeric(12,2)`**, never floating point. Serialised as string in JSON.
- **Soft delete only where the domain needs it** (products, users). Orders and ledgers are never deleted.

---

## 6. Order Lifecycle — The Core Flow

```
                    ┌─────────────┐
                    │   PLACED    │  stock RESERVED, 3h TTL
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           │               │               │
    (3h elapsed)      (admin acts)   (pharmacy cancels)
           │               │               │
    ┌──────▼──────┐        │        ┌──────▼──────┐
    │   EXPIRED   │        │        │  CANCELLED  │
    │ reservations│        │        │ reservations│
    │  RELEASED   │        │        │  RELEASED   │
    └─────────────┘        │        └─────────────┘
                    ┌──────┴──────┐
                    │             │
             ┌──────▼─────┐  ┌────▼──────┐
             │  APPROVED  │  │ REJECTED  │
             │ (full or   │  │reservations│
             │  partial)  │  │ RELEASED   │
             └──────┬─────┘  └───────────┘
                    │
        ┌───────────┴────────────┐
        │ COD                    │ DIGITAL
        │                        │
        │              ┌─────────▼──────────┐
        │              │  AWAITING_PAYMENT  │ reservation TTL
        │              │                    │ extended to 24h
        │              └─────────┬──────────┘
        │                   paid │  │ window expires
        │                        │  └──────► CANCELLED (released)
        └───────────┬────────────┘
                    │
            ┌───────▼────────┐
            │   CONFIRMED    │  reservations CONSUMED → ledger ALLOCATION
            │                │  fulfilment_groups created
            └───────┬────────┘
                    │
            ┌───────▼────────┐
            │   PREPARING    │  warehouse picking
            └───────┬────────┘
            ┌───────▼────────┐
            │ OUT_FOR_DELIVERY│ internal agent OR courier consignment
            └───────┬────────┘
            ┌───────▼────────┐
            │   DELIVERED    │
            └────────────────┘
```

**The critical insight:** reservation is *held* from placement all the way through payment, and only *consumed* at confirmation. Approving a digital-payment order does not deduct stock — if the pharmacy never pays, the stock returns to the pool automatically. This is what makes "charge after approval" safe.

### 6.1 Order placement — the locking transaction

The one piece of code that must be exactly right.

```ts
async placeOrder(pharmacyId: string, dto: PlaceOrderDto, idempotencyKey: string) {
  return this.db.transaction(async (tx) => {
    // 1. Idempotency — replayed request returns the original order
    const existing = await tx.query.orders.findFirst({
      where: and(eq(orders.pharmacyId, pharmacyId),
                 eq(orders.idempotencyKey, idempotencyKey)),
    });
    if (existing) return existing;

    // 2. Lock product rows in a DETERMINISTIC ORDER — prevents deadlock
    //    when two pharmacies order overlapping products simultaneously.
    const productIds = [...new Set(dto.items.map(i => i.productId))].sort();
    const locked = await tx.execute(sql`
      SELECT id, stock_on_hand, stock_reserved, base_price, is_active,
             manufacturer_id, name, pack_size
      FROM products
      WHERE id = ANY(${productIds})
      ORDER BY id
      FOR UPDATE
    `);

    // 3. Validate availability against the LOCKED rows
    for (const item of dto.items) {
      const p = locked.find(r => r.id === item.productId);
      if (!p?.is_active) throw new ProductUnavailableError(item.productId);
      const available = p.stock_on_hand - p.stock_reserved;
      if (available < item.qty) {
        throw new InsufficientStockError(item.productId, available);
      }
    }

    // 4. Resolve tier price per line, snapshot everything
    const lines = dto.items.map(item => {
      const p = locked.find(r => r.id === item.productId)!;
      const unitPrice = this.pricing.resolveTier(p, item.qty);
      return { ...item, unitPrice, lineTotal: unitPrice * item.qty,
               productName: p.name, packSize: p.pack_size,
               manufacturerId: p.manufacturer_id };
    });

    // 5. Insert order + items
    const order = await tx.insert(orders).values({ ... }).returning();
    const items = await tx.insert(orderItems).values(lines).returning();

    // 6. Create reservations and bump stock_reserved — same transaction
    const expiresAt = addHours(new Date(), this.config.RESERVATION_TTL_HOURS);
    await tx.insert(stockReservations).values(
      items.map(i => ({ orderId: order.id, orderItemId: i.id,
                        productId: i.productId, qty: i.requestedQty,
                        status: 'ACTIVE', expiresAt }))
    );
    for (const i of items) {
      await tx.execute(sql`
        UPDATE products SET stock_reserved = stock_reserved + ${i.requestedQty}
        WHERE id = ${i.productId}
      `);
    }

    // 7. Schedule expiry AFTER commit (see note below)
    this.afterCommit(() =>
      this.reservationQueue.add('expire', { orderId: order.id },
        { delay: ms(this.config.RESERVATION_TTL_HOURS + 'h'),
          jobId: `expire:${order.id}` })  // jobId = dedup guarantee
    );

    return order;
  });
}
```

**Four details that matter:**

1. **Deterministic lock ordering** (`ORDER BY id`) is the difference between a system that works and one that deadlocks intermittently under concurrent ordering.
2. **Availability is checked against locked rows**, not a cached or pre-read value. Between read and lock, anything can change.
3. **The queue job is scheduled after commit**, never inside the transaction — otherwise a rolled-back transaction leaves a live job referencing a non-existent order.
4. **`jobId` is deterministic**, so a retried enqueue cannot create two expiry jobs for one order.

### 6.2 Reservation expiry — belt and braces

Two independent mechanisms, because a lost job means stock stranded forever:

- **Primary:** a BullMQ delayed job per order, fired at `expires_at`.
- **Safety net:** a cron sweep every 5 minutes — `SELECT ... WHERE status='ACTIVE' AND expires_at < now() FOR UPDATE SKIP LOCKED` — catching anything Redis dropped during a restart or failover.

Release is **idempotent**: it moves `ACTIVE → RELEASED` and decrements `stock_reserved` in one transaction, and does nothing at all if the reservation is no longer `ACTIVE`. Running it twice is harmless. `SKIP LOCKED` means multiple workers can sweep concurrently without contention.

### 6.3 Partial approval

When admin approves 8 of 10 lines:

1. Lock the order and its reservations.
2. For each line: `APPROVED` (unchanged), `REDUCED` (`approved_qty < requested_qty`), or `REMOVED` (`approved_qty = 0`, with a reason).
3. **Release the delta immediately** — a reduced line releases `requested - approved`; a removed line releases the full quantity.
4. Recalculate `approved_subtotal` / `approved_total` from surviving lines. **Re-resolve the price tier on reduced lines** — dropping from 100 units to 40 may cross a tier boundary, and silently charging the 100-unit price would be wrong.
5. Write `order_status_history` with the decision and actor.
6. Notify the pharmacy with an explicit changed-lines payload, not just a status flip.

Step 4 is easy to miss and produces incorrect invoices.

### 6.4 Confirmation

Triggered by approval (COD) or payment success (digital). In one transaction:

1. Reservations `ACTIVE → CONSUMED`.
2. `stock_movements` rows of type `ALLOCATION` with negative `qty_delta`.
3. `products.stock_on_hand` decremented; `stock_reserved` decremented by the same amount.
4. `fulfilment_groups` created — one per distinct `manufacturer_id` across approved lines.
5. Order → `CONFIRMED`.
6. After commit: notify pharmacy, alert warehouse, evaluate low-stock thresholds, **enqueue invoice generation**.

### 6.5 Money model

One calculation, one place — `PricingService`. Never duplicated in a controller, a report, or a PDF template.

```
line_total        = tier_unit_price × qty            (per approved line)
subtotal          = Σ line_total
discount          = order-level discount, if any
taxable_base      = subtotal − discount
vat               = round(taxable_base × vat_rate, 2)
delivery_charge   = zone.charge                       (snapshotted at placement)
vat_on_delivery   = vat_applies_to_delivery
                      ? round(delivery_charge × vat_rate, 2)
                      : 0
grand_total       = taxable_base + vat + delivery_charge + vat_on_delivery
```

**Rules:**
- **Round once, at the end of each component**, using banker's-safe `numeric` arithmetic in Postgres. Rounding per line and summing produces totals that disagree with the invoice by a few poisha — small enough to ship, large enough to become an accounting complaint.
- **VAT is recalculated on partial approval**, because the taxable base changed. Delivery charge is *not* — the van still drives the same distance for 8 items as for 10.
- **Delivery zone and VAT rate are snapshotted at placement.** Settings changes never reach back into existing orders.
- If every line is removed, the order is **rejected**, not approved with a delivery-charge-only total.

### 6.6 Minimum order enforcement

Two independent minimums, both checked at cart preview **and** re-validated inside the placement transaction — a client-side check is a convenience, never a control.

| Rule | Source | Applies to |
| :--- | :--- | :--- |
| Minimum order value | `platform_settings.min_order_value` | `subtotal` (goods only — before VAT and delivery, so the charge can't be used to reach the minimum) |
| Minimum quantity | `products.min_order_qty` | Each line |
| Order multiple | `products.order_multiple` | Each line — qty must be an exact multiple |

Violations return `422` with a per-line breakdown, so the app can highlight the offending items rather than showing one unhelpful banner.

**Admin partial approval is exempt.** If reducing a line takes it below its MOQ, or drops the order under the minimum value, the approval still stands — the minimums exist to shape what pharmacies *submit*, not to override your team's judgement about what to ship. This asymmetry is deliberate.

### 6.7 Invoice generation

Triggered on order confirmation, produced asynchronously in the worker.

1. Build the snapshot — lines, rates, company details from `platform_settings`, buyer details from `pharmacies`.
2. Draw `invoice_no` from a Postgres sequence with a yearly prefix (`INV-2026-000418`). **Gapless numbering is a compliance expectation** — a sequence, not a `COUNT(*) + 1`, which produces duplicates under concurrency.
3. Render an HTML template → PDF via **headless Chromium (Playwright)** in the worker container. Chosen over pdfkit/pdfmake specifically for **Bangla font rendering and complex-script shaping**, which the programmatic libraries handle poorly.
4. Upload to object storage; store `pdf_key`.
5. Notify the pharmacy; the PDF is served through a short-lived presigned URL.

**Delivery challan** uses the same pipeline with a different template and no pricing — one renderer, two templates.

**Regeneration** re-renders from `snapshot` and always yields the same document. **Correction** voids the invoice and issues a new one referencing it; the original is never mutated or deleted.

### 6.8 Catalogue import

The 20,000-product seed is a staged pipeline, not a one-off script:

1. **Upload** — Excel/CSV to object storage.
2. **Parse & validate** — every row against a Zod schema: manufacturer and generic must resolve (by name, case-insensitive, with fuzzy suggestions for near-misses), prices numeric and positive, MOQ consistent with `order_multiple`.
3. **Dry-run report** — downloadable, row-numbered: what will be created, updated, skipped, and rejected with reasons. **Nothing is written yet.**
4. **Commit** — batched insert/update in a transaction, with an import batch ID recorded on every affected row.
5. **Rollback** — by batch ID, available until the first order references an imported product.

The dry-run is what makes a 20,000-row import survivable. Committing directly and discovering 400 bad manufacturer names afterwards means a manual cleanup nobody has budgeted for. The same importer serves ongoing price-list updates, so it earns its cost well past launch.

---

## 7. Cross-Cutting Concerns

### 7.1 Authentication

- **Access token:** JWT, 15 minutes, carries `sub`, `role`, `pharmacyId`, `jti`.
- **Refresh token:** 30 days, opaque, stored hashed, **rotated on every use**.
- **Reuse detection:** tokens carry a `family_id`. Presenting an already-rotated token revokes the entire family and forces re-login — the standard defence against token theft.
- **Mobile:** tokens in secure device storage (Keychain / Keystore).
- **Admin panel:** `httpOnly`, `Secure`, `SameSite=Strict` cookies. Never `localStorage`.
- **Passwords:** argon2id.
- **OTP** for pharmacy login/reset: 6 digits, 5-minute TTL in Redis, max 5 attempts, rate-limited per phone and per IP.

### 7.2 Authorisation — three layers

```ts
@Controller('admin/orders')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('ADMIN', 'WAREHOUSE_MANAGER')
export class AdminOrdersController { ... }
```

1. **`JwtAuthGuard`** — authenticated, globally applied, opted out with `@Public()`.
2. **`RolesGuard`** — role membership via `@Roles()`.
3. **`PharmacyScopeGuard`** — *the important one.* Any route resolving a pharmacy-owned resource asserts `resource.pharmacyId === user.pharmacyId`. **Enforced in a guard, not in each service**, because the one service that forgets is the one that leaks Pharmacy A's orders to Pharmacy B. On a B2B platform where competitors share the system, that is the incident that ends the contract.

### 7.3 Validation

Zod schemas, one definition serving DTO type, runtime validation, and OpenAPI:

```ts
export const PlaceOrderSchema = z.object({
  paymentMethod: z.enum(['COD', 'DIGITAL']),
  items: z.array(z.object({
    productId: z.string().uuid(),
    qty: z.number().int().positive().max(10_000),
  })).min(1).max(200),
});
export class PlaceOrderDto extends createZodDto(PlaceOrderSchema) {}
```

Global `ZodValidationPipe`, `whitelist: true`. Unknown fields are rejected, not silently dropped.

### 7.4 Error handling

Global exception filter emitting RFC 7807 `application/problem+json`:

```json
{
  "type": "https://api.example.com/errors/insufficient-stock",
  "title": "Insufficient stock",
  "status": 409,
  "detail": "Napa Extra 500mg — 12 units available, 50 requested",
  "instance": "/api/v1/app/orders",
  "requestId": "01J8X...",
  "context": { "productId": "...", "available": 12 }
}
```

Domain errors are typed classes mapped to status codes. Unexpected errors log the stack, report to Sentry, and return a generic 500 with the request ID — **internals never reach the client**.

### 7.5 Idempotency

`Idempotency-Key` header required on order placement and payment initiation. Key + pharmacy scope is stored; a replay within 24 hours returns the original response. Courier webhooks dedupe on `(provider, event_id)`.

### 7.6 Rate limiting

`@nestjs/throttler` on Redis: 100 req/min per authenticated user globally; 5/min on login and OTP request; 10/min on order placement. Per-IP limits on all unauthenticated routes.

### 7.7 File uploads

Client requests a presigned PUT URL, uploads directly to object storage, then submits the key. **Files never pass through the API.** Server-side validation of declared MIME and size; final content-type verified from magic bytes after upload. Licence documents are private — served only through short-lived presigned GETs, scoped to admin reviewers.

---

## 8. Background Jobs

| Queue | Trigger | Behaviour |
| :--- | :--- | :--- |
| `reservation.expire` | Delayed, per order | Idempotent release at TTL |
| `reservation.sweep` | Cron, every 5 min | Safety net for missed jobs |
| `notifications.dispatch` | Event | Push/SMS/in-app fan-out, retries with backoff |
| `courier.create-consignment` | Order dispatched | Calls provider API, stores tracking code |
| `courier.reconcile` | Cron, every 30 min | Polls status for consignments with no recent webhook — webhooks get lost |
| `invoices.generate` | Order confirmed | Snapshot → Chromium render → storage; idempotent on `order_id` |
| `catalog.import` | Admin commit | Batched write with progress reporting |
| `reports.export` | On demand | Excel generation, uploaded to storage, link pushed to user |
| `inventory.low-stock` | After confirmation | Threshold evaluation, admin alert |
| `search.reindex` | After product write | Refreshes `search_doc` tsvector |

**Standards:** exponential backoff (3 attempts), dead-letter queue with alerting, every processor idempotent, deterministic `jobId` for anything schedulable twice. Workers run in a separate container so a job spike never degrades API latency.

---

## 9. Search & Caching

### Search — Postgres, not Meilisearch

At 20,000 products, a dedicated search engine is unnecessary infrastructure. Postgres delivers this well:

```sql
-- weighted tsvector: product name > generic > manufacturer
UPDATE products SET search_doc =
  setweight(to_tsvector('simple', name), 'A') ||
  setweight(to_tsvector('simple', coalesce(generic_name, '')), 'B') ||
  setweight(to_tsvector('simple', coalesce(manufacturer_name, '')), 'C');
```

Full-text for structured queries, `pg_trgm` similarity for typo tolerance on brand names (which pharmacy staff mistype constantly). Combined query ranks FTS first, falls back to trigram above a similarity threshold. Sub-30ms at this catalogue size with GIN indexes.

**Revisit at ~200k products or if multi-language search is added** — then Meilisearch, behind the existing `SearchService` interface so nothing else changes.

### Caching

| Data | TTL | Invalidation |
| :--- | :--- | :--- |
| Product detail | 5 min | Explicit delete on write |
| Category/manufacturer listings | 5 min | Version-stamped keys |
| Home banners, top companies | 15 min | Explicit |
| Search results (popular queries) | 60 s | TTL only |
| **Stock availability** | **never cached** | — |
| Session/permissions | 15 min | Delete on role change |

**Stock is never served from cache.** A pharmacy seeing stale availability places an order that fails at reservation — the exact frustration this design exists to prevent. Availability is always a live read on an indexed column.

Version-stamped keys (`products:v{n}:list:...`) mean a catalogue-wide invalidation is a single counter increment rather than a scan-and-delete across thousands of keys.

---

## 10. Performance

### Targets

| Metric | Target |
| :--- | :--- |
| p95 catalogue read (cached) | < 80 ms |
| p95 search | < 150 ms |
| p95 order placement (locking transaction) | < 400 ms |
| p95 admin order list | < 200 ms |
| Sustained throughput, single instance | 300+ rps |
| Availability | 99.5% |

### Techniques

- **Keyset pagination everywhere.** `WHERE (created_at, id) < (:cursor_ts, :cursor_id) ORDER BY created_at DESC, id DESC LIMIT 20`. `OFFSET` degrades linearly and the admin order archive will be the first place it hurts.
- **No N+1.** Aggregate queries or explicit joins; Drizzle's relational queries with `with` batch correctly. Enforced by a dev-mode query-count assertion in integration tests.
- **Connection pooling** — PgBouncer in transaction mode, pool sized to `4 × vCPU`. Note: transaction mode forbids session-level prepared statements; Drizzle is configured accordingly.
- **Partial indexes** over full ones wherever a predicate is fixed (`WHERE is_active`, `WHERE status='ACTIVE'`) — smaller, faster, cheaper to maintain.
- **`EXPLAIN ANALYZE` on every query touching orders or products**, as part of code review, not after the complaint.
- **Compression and HTTP caching** at Caddy; `ETag` on catalogue responses so mobile clients revalidate cheaply on poor connections.

### Scaling path

No rewrite required at any step:
1. Vertical — larger Postgres instance (comfortable well past the 12-month target).
2. Horizontal API — add containers behind Caddy; the app is stateless.
3. Read replicas — route reporting and catalogue reads away from the primary.
4. Table partitioning — `audit_logs`, `delivery_events`, `stock_movements` by month.
5. Extract a module to its own service — the boundaries already exist.

---

## 11. Security

**Application:** RBAC + tenant scoping in guards; Zod validation on every input; parameterised queries throughout (no string-built SQL); Helmet headers with CSP; strict CORS allowlist; rate limiting on auth and ordering; argon2id passwords; no secrets in source, ever.

**Data:** licence documents private with presigned access only; PII (phone, address) encrypted at rest via disk encryption; audit log on every admin mutation; least-privilege database roles — the application user cannot `DROP`.

**Infrastructure:** TLS enforced end to end with automatic renewal; Postgres and Redis on a private network, never publicly bound; firewall limited to 80/443/SSH; SSH keys only; Cloudflare for DDoS and edge filtering; dependency scanning in CI with prompt CVE patching.

**Payments:** no card data ever touches the server — gateway-hosted. bKash/Nagad callbacks verified server-side against the provider API, never trusted from the client. Webhook signatures verified before the payload is parsed.

**Pharma-specific:** the immutable stock ledger and audit trail exist because controlled-substance traceability is a regulatory expectation. Even though returns are out of scope, every unit that entered or left is reconstructable.

---

## 12. Observability

- **Structured logging** (Pino) — JSON, request-scoped correlation ID propagated through jobs. Passwords, tokens, and OTPs redacted at the serialiser.
- **Health checks** — `/health/live` (process), `/health/ready` (Postgres + Redis + storage reachable).
- **Metrics** — Prometheus endpoint: request rate/latency/errors by route, queue depth and job duration, pool utilisation, **active reservation count and expiry rate** (the domain-specific signal that tells you the approval workflow is keeping up).
- **Error tracking** — Sentry, release-tagged with source maps.
- **Alerts** — dead-letter queue non-empty; reservation sweep finding more than N expired (means admins aren't reviewing); courier webhook silence beyond 2 hours; error rate above 1%; disk above 80%.

---

## 13. Testing

| Layer | Tool | Coverage |
| :--- | :--- | :--- |
| Unit | Vitest | Pricing tiers, state transitions, courier status mapping |
| Integration | Vitest + Testcontainers | Real Postgres. Every repository and transactional service. |
| E2E | Supertest | Full flows against a live app instance |
| Concurrency | Custom | **Parallel order placement on the same low-stock product — asserts no oversell** |
| Load | k6 | Catalogue browse + order placement at 3× projected peak |

**Mandatory E2E scenarios:**
1. Place → reserve → approve fully → COD → confirm → dispatch → deliver
2. Place → reserve → **partial approve** → verify tier re-resolution and released delta
3. Place → reserve → **let TTL expire** → verify stock returns and order cancels
4. Place → approve → digital payment → **payment window expires unpaid** → verify release
5. Two pharmacies race the last 10 units → exactly one succeeds
6. Courier webhook delivered **three times** → exactly one state transition
7. Pharmacy A attempts to read Pharmacy B's order → 404, not 403 *(no existence leak)*
8. Partial approval → **VAT recalculated, delivery charge unchanged**, invoice total matches the sum of approved lines to the poisha
9. VAT rate changed in settings → **existing orders and issued invoices unchanged**
10. Order below minimum value → rejected at placement; same order reduced *by admin* below minimum → still approves
11. Invoice regenerated from snapshot → byte-identical PDF; void and reissue → new number, original intact

Coverage floor: 80% overall, **100% on `inventory` and `orders`**. CI blocks merge on failure.

---

## 14. Environments & Deployment

| Environment | Purpose |
| :--- | :--- |
| Local | Docker Compose — Postgres, Redis, MinIO, seeded data |
| Staging | Production mirror, sanitised data, courier and payment sandboxes |
| Production | Live |

### CI/CD (GitHub Actions)

```
PR opened
 ├─ lint + prettier
 ├─ tsc --noEmit
 ├─ unit tests
 ├─ integration tests (Testcontainers Postgres)
 ├─ dependency audit
 └─ docker build (verify only)

merge → main
 ├─ full pipeline
 ├─ build + push image (tagged with SHA)
 ├─ deploy to staging
 ├─ run migrations
 └─ smoke tests

manual approval → production
 ├─ database backup
 ├─ run migrations (expand-only)
 ├─ rolling container replace, zero downtime
 └─ health check; automatic rollback on failure
```

### Migrations

Drizzle Kit, versioned in source, **forward-only**. Expand-and-contract for breaking changes: add column → backfill → dual-write → switch reads → drop old, across separate deploys. Every migration reviewed for lock duration — an `ALTER TABLE` that takes an `ACCESS EXCLUSIVE` lock on `products` during business hours is an outage.

### Backups

Managed Postgres daily snapshot + PITR (7-day window). **Monthly restore drill into a scratch instance** — an untested backup is not a backup. Object storage versioning enabled. Documented RTO 4h / RPO 1h.

---

## 15. API Surface

### Pharmacy app — `/api/v1/app`

| Method | Path | Purpose |
| :--- | :--- | :--- |
| POST | `/auth/register` | Registration + document keys |
| POST | `/auth/login` | Phone/email + password |
| POST | `/auth/otp/request` · `/otp/verify` | OTP login |
| POST | `/auth/refresh` · `/logout` | Token rotation |
| GET | `/me` · PATCH `/me` | Profile, account status |
| GET | `/home` | Banners, top companies, recently purchased |
| GET | `/products` | Search, filter, keyset paginate |
| GET | `/products/:id` | Detail with live availability |
| GET | `/products/:id/alternates` | Same generic, in stock |
| GET | `/manufacturers` · `/:id/products` | Company pages |
| GET/POST/DELETE | `/favourites` | My Products |
| POST | `/cart/preview` | Live totals — tiers, VAT, delivery by zone, minimum-order violations |
| POST | `/orders` | **Place order** (idempotent) |
| GET | `/orders/:id/invoice` | Presigned PDF URL |
| GET | `/orders` · `/orders/:id` | List, detail with status history |
| POST | `/orders/:id/cancel` | While PLACED only |
| POST | `/orders/:id/reorder` | Re-price against current stock |
| POST | `/orders/:id/pay` | Initiate capture (post-approval) |
| GET/POST | `/medicine-requests` | Request an uncatalogued item |
| GET | `/notifications` · POST `/device-tokens` | Push |

### Admin panel — `/api/v1/admin`

| Method | Path | Purpose |
| :--- | :--- | :--- |
| GET | `/pharmacies?status=PENDING` | Approval queue |
| GET | `/pharmacies/:id` | Profile + presigned document URLs |
| POST | `/pharmacies/:id/approve` · `/reject` | Decision with reason |
| CRUD | `/products`, `/manufacturers`, `/generics`, `/price-tiers` | Catalogue |
| POST | `/products/import` | Upload → dry-run report → commit → rollback by batch |
| CRUD | `/settings` | VAT rate, minimum order value, TTLs, company/BIN details |
| CRUD | `/delivery-zones` | District mapping and charges |
| GET | `/invoices` · `/invoices/:id` | List, download, void and reissue |
| GET | `/orders?status=PLACED` | Approval queue |
| POST | `/orders/:id/approve` | **Full or partial**, per-line decisions |
| POST | `/orders/:id/reject` | With reason |
| POST | `/inventory/receipts` | Stock received |
| POST | `/inventory/adjustments` | With mandatory reason |
| GET | `/inventory/movements` | Ledger, filterable |
| GET | `/fulfilment/ready-to-ship` | Picking board by manufacturer group |
| POST | `/fulfilment/:id/dispatch` | Internal agent **or** courier booking |
| PATCH | `/deliveries/:id/status` | Manual status write path |
| GET | `/medicine-requests` · POST `/:id/resolve` | Request queue |
| GET | `/dashboard` | Live metrics |
| POST | `/reports/export` | Async Excel generation |
| CRUD | `/staff` | Users and roles |
| GET | `/audit-logs` | Filterable trail |

### Webhooks — `/api/v1/webhooks`

`POST /couriers/pathao` · `/steadfast` · `/redx` — signature-verified, deduped on `(provider, event_id)`, raw payload persisted before processing.
`POST /payments/:provider` — signature-verified, idempotent capture confirmation.

---

## 16. Delivery Roadmap

Execution is organised as **17 sequential phases**, specified in `BUILD_PLAN.md` — each a complete, curl-verifiable slice with its own definition of done. Phases replace calendar weeks: a phase is finished when its acceptance criteria are green, not when a date arrives.

| Phase | Focus |
| :--- | :--- |
| 0 | Repository & toolchain |
| 1 | Database schema & migrations |
| 2 | Authentication & RBAC |
| 3 | Pharmacy onboarding |
| 4 | Catalogue |
| 5 | Catalogue importer |
| 6 | Search & browse |
| 7 | Settings, delivery zones & pricing |
| 8 | **Inventory ledger & reservations** ★ |
| 9 | **Order placement** ★ |
| 10 | **Approval, partial approval & confirmation** ★ |
| 11 | Payments |
| 12 | Invoicing |
| 13 | Fulfilment & couriers |
| 14 | Notifications & medicine requests |
| 15 | Dashboard & reports |
| 16 | Hardening & deployment |

★ = the three phases carrying real risk. They are strictly sequential — reservations must be provably correct before orders are built on them. If schedule pressure appears, move Phase 15 scope; never compress Phase 8.

For commercial planning, the phase set maps to roughly 14 weeks of backend effort. Frontend and mobile run in parallel against the OpenAPI contract, published from Phase 3.

<details>
<summary>Original week-based schedule (for reference)</summary>

| Week | Focus | Exit criteria |
| :--- | :--- | :--- |
| **1** | Foundation | Nest scaffold, Docker Compose, config validation, logging, error filter, health checks, CI green |
| **2** | Schema & auth | Full Drizzle schema, migrations, seeds, JWT with rotation, guards, RBAC |
| **3** | Pharmacy onboarding | Registration, presigned document upload, approval queue, decisions. **OpenAPI v1 published** |
| **4** | Catalogue & importer | Manufacturers, generics, products, price tiers, MOQ fields, alternate-brand linking, **staged importer with dry-run** |
| **5** | Search, browse & settings | tsvector + trigram, filters, keyset pagination, caching, home endpoints, `platform_settings`, `delivery_zones` |
| **6–7** | **Inventory & reservations** | Ledger, receipts, adjustments, reservation transaction, 3h expiry job + sweeper, auto-cancel notification. **Concurrency tests pass** |
| **8–9** | **Orders & pricing** | Cart preview, minimum-order enforcement, VAT + delivery calculation, placement with idempotency, approval queue, partial approval with tier and VAT re-resolution, confirmation, fulfilment groups |
| **10** | Payments & invoicing | Gateway adapter, post-approval capture, payment window expiry, **invoice/challan PDF pipeline with Bangla rendering**, gapless numbering |
| **11** | Fulfilment & couriers | Picking board, admin-selected dispatch mode, three courier adapters, webhook ingestion, reconcile job |
| **12** | Notifications & reports | Push/SMS fan-out, dashboard aggregates, Excel export, medicine requests |
| **13** | Hardening | Load testing, index tuning, security review, penetration checks, E2E suite complete |
| **14** | Launch | Production provisioning, deployment, restore drill, runbook, admin training, handover |

</details>

---

## 17. Handover Checklist

- [ ] Source code in a private repository, full history, ownership transferred
- [ ] OpenAPI spec published and versioned
- [ ] `README` — local setup working from clone in under 15 minutes
- [ ] Architecture Decision Records for the five decisions in Section 0
- [ ] `.env.example` with every variable documented
- [ ] Database schema diagram and migration guide
- [ ] Runbook — deploy, rollback, restore, common incidents
- [ ] Monitoring dashboards and alert routing configured
- [ ] Verified restore from backup, witnessed
- [ ] Load test report against agreed targets
- [ ] Admin training session recorded
- [ ] 12-month warranty terms confirmed in writing

---

## 18. Deliberately Excluded

Named so nobody assumes otherwise:

| Excluded | Why | Cost to add later |
| :--- | :--- | :--- |
| Store credit / BNPL | Removed from scope | Moderate — new tables, exposure checks at approval |
| Returns & damaged goods | Out of scope; ledger type reserved | Low — the ledger already models it |
| Multi-warehouse | Single warehouse at launch | Moderate — stock becomes per-location |
| Delivery agent app | Admin + courier webhooks cover it | Moderate — new client surface, agent auth |
| GPS live tracking | Not required | Moderate |
| ERP / accounting integration | Scoped separately | Depends on target system |
| Per-product VAT exemption | Single order-level rate confirmed | Moderate — tax moves to the product, totals become per-line |
| Free-delivery threshold | Zone-based charge only | Low — one rule in `PricingService` |
| Rule-based dispatch routing | Admin picks per order | Low |
| Dedicated search engine | Postgres sufficient at 20k products | Low — `SearchService` interface already isolates it |
| Microservices | Unjustified at this scale | Module boundaries make extraction possible |
| GraphQL | REST fits three known clients | Low |

Each was excluded on scale or scope grounds, not capability — and the architecture leaves the door open on every one.

---

## 19. Assumptions Closed Without Confirmation

Small enough to decide, large enough to write down. Overturn any of these cheaply now; each costs more once built against.

| # | Assumption | Reversal cost |
| :--- | :--- | :--- |
| 1 | **Reorder re-prices at current tiers and current availability**, and flags changed or unavailable lines rather than silently substituting | Low |
| 2 | **Pharmacy self-cancellation only while `PLACED`** — after approval, cancellation is an admin action | Low |
| 3 | **VAT applies to goods only by default**; `vat_applies_to_delivery` exists as a settings flag, defaulted off | Low |
| 4 | **Invoice issued at confirmation**, not at approval — so an unpaid digital order never produces one | Low |
| 5 | **Unmappable district falls back to a default charge + admin warning**, never blocks checkout | Low |
| 6 | **Minimum order value measured on goods subtotal**, excluding VAT and delivery | Low |
| 7 | **Order-level discount field exists but has no admin UI at launch** — schema-ready, unused | Low |
| 8 | **One active delivery zone per district**; overlaps rejected at configuration time | Moderate |
| 9 | **Invoice numbering resets yearly** with a year prefix | Low |
| 10 | **Notifications are push + in-app; SMS reserved for OTP and confirmation only** — SMS is a per-message cost and full parity gets expensive fast | Low |

---

**Curlware Digital Agency**
curlware.com
