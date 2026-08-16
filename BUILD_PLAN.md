# B2B Pharmacy Platform — Backend Build Plan

**Audience:** coding agent. Execute phases in order.
**Companion docs:** `ARCHITECTURE.md` (why), `CLAUDE.md` (how to work in this repo).

## How to use this document

- Phases are **sequential**. Do not start a phase until the previous one's Definition of Done is fully green.
- Each phase is a complete, verifiable slice. Never leave a phase half-built to start the next.
- **Every phase ends with real curl verification against a running server.** Not a passing unit test — an actual HTTP round trip.
- If a phase's acceptance criteria can't be met, stop and report. Do not work around it silently.

**Base URL for all verification:** `http://localhost:3000/api/v1`

---

## Phase 0 — Repository & Toolchain

**Depends on:** nothing.

**Build**
- Scaffold NestJS 11 app with the Nest CLI. Package manager pnpm, SWC builder.
- Node 24 LTS, pinned in `.nvmrc` and `package.json` `engines`.
- TypeScript 6.0. It is a defaults-breaking release — set `module: commonjs`, `moduleResolution: bundler`, `target: es2022` explicitly. `baseUrl` and `moduleResolution: node` were removed in 6.0; declare path aliases (`@/*` → `src/*`) under `paths` alone. Also `noUncheckedIndexedAccess: true`.
- **Do not install TypeScript 7.** No public compiler API until 7.1; it breaks `typescript-eslint`.
- `docker-compose.yml` for local dev: Postgres 17, Redis 7, MinIO. Named volumes, health checks.
- Config module: Zod-validated env schema that **throws at boot** on any missing or malformed variable. No `process.env` access anywhere else in the codebase.
- Pino logger with request-scoped correlation ID (`x-request-id`, generated if absent).
- Global: `ZodValidationPipe`, `AllExceptionsFilter` (RFC 7807 `problem+json`), `LoggingInterceptor`.
- Terminus health module: `/health/live`, `/health/ready`.
- `.env.example` documenting every variable.

**Toolchain compatibility gate — do this before building anything else.**
NestJS 11 predates TypeScript 6.0, and 6.0 changed defaults. Scaffold, install the full dependency set (Nest CLI, `typescript-eslint`, `nestjs-zod`, `drizzle-kit`, Vitest), and run `pnpm typecheck` and `pnpm lint` on the bare skeleton. If any of them is not TS 6-ready, **pin TypeScript 5.9, record why in a comment on the pin, and continue** — do not spend the phase fighting it.

**Definition of Done**
- `node -v` reports 24.x; `engines` and `.nvmrc` agree.
- Toolchain gate passed, or TS pinned to 5.9 with a recorded reason.
- `pnpm dev` boots clean, no warnings.
- `pnpm typecheck` passes.
- Missing env var causes a startup failure with a readable message naming the variable.

**Verify**
```bash
curl -s localhost:3000/api/v1/health/live | jq
curl -s localhost:3000/api/v1/health/ready | jq   # postgres + redis + storage all "up"
curl -si localhost:3000/api/v1/does-not-exist     # 404 as problem+json, carries requestId
```

---

## Phase 1 — Database Schema & Migrations

**Depends on:** Phase 0.

**Build**
- Drizzle schema for every table in `ARCHITECTURE.md` §5, split by domain file under `src/infra/database/schema/`.
- UUID v7 primary keys, `timestamptz` throughout, money as `numeric(12,2)`.
- Enums as Postgres native enums.
- All indexes from §5.2, including the partial ones and the GIN indexes (`pg_trgm` extension required).
- Transaction helper providing a typed `tx` and an **after-commit hook registry** — Phase 8 depends on this for safe job scheduling.
- Seed script: 1 admin, 1 warehouse manager, 1 approved pharmacy, 3 manufacturers, 10 generics, 30 products with price tiers, 3 delivery zones, default `platform_settings`.

**Definition of Done**
- `pnpm db:migrate` applies cleanly to an empty database.
- `pnpm db:seed` is idempotent — running twice does not duplicate or error.
- Every FK has an index. Every enum column is a native enum, not text.

**Verify**
```bash
pnpm db:migrate && pnpm db:seed && pnpm db:seed
psql $DATABASE_URL -c "\d products"        # confirm columns, indexes, defaults
psql $DATABASE_URL -c "SELECT count(*) FROM products;"   # 30
```

---

## Phase 2 — Authentication & RBAC

**Depends on:** Phase 1.

**Build**
- argon2id password hashing.
- Access JWT (15m) carrying `sub`, `role`, `pharmacyId`, `jti`. Refresh token (30d), opaque, stored hashed, **rotated on every use**, with `family_id` reuse detection that revokes the whole family on replay.
- `JwtAuthGuard` applied globally; `@Public()` decorator to opt out.
- `RolesGuard` + `@Roles()`.
- **`PharmacyScopeGuard`** — asserts a requested resource belongs to the caller's pharmacy. Returns **404, not 403**, for another tenant's resource.
- OTP request/verify: 6 digits, Redis, 5-minute TTL, max 5 attempts.
- Throttler on Redis: 5/min on login and OTP, 100/min global authenticated.

**Definition of Done**
- Reusing a rotated refresh token invalidates the entire family.
- A pharmacy user cannot reach any `/admin/*` route.
- Tokens never appear in logs (verify the Pino redaction list).

**Verify**
```bash
# login
TOKEN=$(curl -s -X POST localhost:3000/api/v1/app/auth/login \
  -H 'content-type: application/json' \
  -d '{"phone":"01700000001","password":"seedpassword"}' | jq -r .accessToken)

curl -s localhost:3000/api/v1/app/me -H "authorization: Bearer $TOKEN" | jq
curl -si localhost:3000/api/v1/admin/orders -H "authorization: Bearer $TOKEN"   # 403
curl -si localhost:3000/api/v1/app/me                                            # 401

# refresh rotation + reuse detection
R=$(curl -s -X POST .../auth/refresh -d "{\"refreshToken\":\"$OLD\"}" | jq -r .refreshToken)
curl -si -X POST .../auth/refresh -d "{\"refreshToken\":\"$OLD\"}"   # 401, family revoked
```

---

## Phase 3 — Pharmacy Onboarding

**Depends on:** Phase 2.

**Build**
- Registration: business details + document keys. Creates `pharmacies` (`PENDING`) and a `PHARMACY_OWNER` user who can log in but reach nothing except `/me`.
- Presigned PUT upload flow — files go **client → storage directly**, never through the API. Validate declared MIME/size on request; verify magic bytes after upload.
- Admin: approval queue (keyset paginated), detail with **short-lived presigned GET** URLs for documents, approve / reject / request-info with a written reason.
- `AccountStatusGuard`: a non-`APPROVED` pharmacy gets `403` with a machine-readable status code on every commerce route.

**Definition of Done**
- Documents are not publicly reachable; a presigned URL expires and stops working.
- Rejection reason is visible to the pharmacy on `/me`.

**Verify**
```bash
curl -s -X POST .../app/auth/register -d @register.json | jq
curl -s .../admin/pharmacies?status=PENDING -H "authorization: Bearer $ADMIN" | jq
curl -s -X POST .../admin/pharmacies/$ID/approve -H "authorization: Bearer $ADMIN" | jq
curl -s .../app/me -H "authorization: Bearer $TOKEN" | jq .pharmacy.status   # APPROVED
curl -sI "$PRESIGNED_URL"          # 200 now
sleep 400 && curl -sI "$PRESIGNED_URL"   # 403 expired
```

---

## Phase 4 — Catalogue

**Depends on:** Phase 3.

**Build**
- CRUD: manufacturers, generics, products, price tiers.
- Product fields include `min_order_qty`, `order_multiple`, `low_stock_threshold`.
- Price tier rule: **highest `min_qty` ≤ ordered qty wins**, falling back to `base_price`. Implement in `PricingService.resolveTier()` — this function is used by Phases 7, 9, 10 and must exist in exactly one place.
- Alternate brands: same `generic_id`, active, in stock, excluding self.
- Soft delete for products; never hard-delete anything an order references.

**Definition of Done**
- Unit tests cover tier resolution boundaries: below first tier, exactly on a boundary, above the last tier, single tier, no tiers.
- Deactivating a product does not alter any existing order line.

**Verify**
```bash
curl -s -X POST .../admin/products -H "authorization: Bearer $ADMIN" -d @product.json | jq
curl -s .../app/products/$ID -H "authorization: Bearer $TOKEN" | jq '.price, .availableQty'
curl -s .../app/products/$ID/alternates -H "authorization: Bearer $TOKEN" | jq length
```

---

## Phase 5 — Catalogue Importer

**Depends on:** Phase 4.

**Build**
Four-stage pipeline — **the dry-run stage is not optional**:
1. `POST /admin/products/import` — upload Excel/CSV, returns `importId`.
2. Parse and validate every row against a Zod schema. Resolve manufacturer/generic by name, case-insensitive, with fuzzy suggestions for near-misses.
3. `GET /admin/products/import/:id/report` — row-numbered: will-create / will-update / will-skip / rejected-with-reason. **Nothing written yet.**
4. `POST /admin/products/import/:id/commit` — batched write in a transaction, stamping `import_batch_id` on every affected row.
5. `POST /admin/products/import/:id/rollback` — available until an order references an imported product.

**Definition of Done**
- A 20,000-row file completes validation in under 60 seconds.
- One bad row does not abort the batch — it appears in the report as rejected.
- Commit is idempotent: replaying the same `importId` does not double-write.

**Verify**
```bash
ID=$(curl -s -X POST .../admin/products/import -F file=@catalog-20k.xlsx -H "authorization: Bearer $ADMIN" | jq -r .importId)
curl -s .../admin/products/import/$ID/report -H "authorization: Bearer $ADMIN" | jq '.summary'
curl -s -X POST .../admin/products/import/$ID/commit -H "authorization: Bearer $ADMIN" | jq
curl -s -X POST .../admin/products/import/$ID/commit -H "authorization: Bearer $ADMIN" | jq  # no double-write
```

---

## Phase 6 — Search & Browse

**Depends on:** Phase 5.

**Build**
- `search_doc` tsvector, weighted: name (A) > generic (B) > manufacturer (C). Refreshed by a job on product write.
- Query layer: full-text first, `pg_trgm` similarity fallback above a threshold for typo tolerance.
- Filters: category, manufacturer, in-stock-only. **Keyset pagination**, never `OFFSET`.
- Home endpoint: banners, top companies, recently purchased.
- Redis caching per `ARCHITECTURE.md` §9 — version-stamped keys. **Stock availability is never cached.**

**Definition of Done**
- Search p95 under 150ms against the full 20k catalogue.
- A typo'd brand name still returns the product.
- `EXPLAIN ANALYZE` on the search query shows an index scan, not a sequential scan.

**Verify**
```bash
curl -s ".../app/products?q=napa&inStock=true" -H "authorization: Bearer $TOKEN" | jq '.items|length'
curl -s ".../app/products?q=napaa" -H "authorization: Bearer $TOKEN" | jq '.items[0].name'  # trigram catch
CUR=$(curl -s ".../app/products?limit=20" -H "authorization: Bearer $TOKEN" | jq -r .nextCursor)
curl -s ".../app/products?limit=20&cursor=$CUR" -H "authorization: Bearer $TOKEN" | jq '.items|length'
```

---

## Phase 7 — Settings, Delivery Zones & Pricing

**Depends on:** Phase 6.

**Build**
- `platform_settings` — single row, typed, Redis-cached with explicit invalidation on write.
- `delivery_zones` CRUD; district → zone mapping; overlapping districts rejected at configuration time.
- **`PricingService.quote()`** — the single money function. Implements §6.5 exactly:
  `subtotal → discount → taxable_base → vat → delivery_charge → (optional vat_on_delivery) → grand_total`.
  Round once per component. Use Postgres `numeric` arithmetic, never JS floats.
- Minimum-order enforcement (§6.6): order value on **goods subtotal only**; per-line `min_order_qty` and `order_multiple`.
- `POST /app/cart/preview` — returns full breakdown plus a per-line violation list.

**Definition of Done**
- Unit tests: VAT on/off for delivery, unmapped district falling back to default charge, minimum-value boundary, `order_multiple` violation.
- No money arithmetic exists anywhere outside `PricingService`. Grep to confirm.

**Verify**
```bash
curl -s -X POST .../app/cart/preview -H "authorization: Bearer $TOKEN" \
  -d '{"items":[{"productId":"...","qty":12}]}' | jq
# expect: subtotal, vat, vatRate, deliveryCharge, deliveryZone, grandTotal, violations[]

curl -s -X POST .../app/cart/preview -H "authorization: Bearer $TOKEN" \
  -d '{"items":[{"productId":"...","qty":5}]}' | jq '.violations'   # below MOQ
```

---

## Phase 8 — Inventory Ledger & Reservations ★

**Depends on:** Phase 7. **This is the highest-risk phase in the build. Do not rush it.**

**Build**
- `stock_movements` append-only ledger. `RECEIPT`, `ADJUSTMENT`, `ALLOCATION`, `RETURN` (enum only, no flow).
- Admin: stock receiving, stock adjustment with **mandatory reason**, ledger listing.
- `InventoryService.reserve()` — the transaction from `ARCHITECTURE.md` §6.1:
  - Lock product rows `FOR UPDATE` **ordered by id** (deadlock prevention — non-negotiable).
  - Check `available = stock_on_hand - stock_reserved` against the **locked** rows.
  - Insert reservations, increment `stock_reserved`, all in one transaction.
- `InventoryService.release()` and `.consume()` — both **idempotent**, both no-ops if the reservation is no longer `ACTIVE`.
- Expiry, two independent mechanisms:
  - BullMQ delayed job, deterministic `jobId`, scheduled **after commit**.
  - Cron sweep every 5 minutes: `WHERE status='ACTIVE' AND expires_at < now() FOR UPDATE SKIP LOCKED`.

**Definition of Done**
- **Concurrency test: 20 parallel reservations against 10 units → exactly 10 succeed, 10 fail with 409. Zero oversell.**
- Killing Redis mid-flight still releases the reservation via the sweeper.
- `stock_on_hand` always equals `SUM(qty_delta)` from the ledger. Assert this in a test.

**Verify**
```bash
curl -s -X POST .../admin/inventory/receipts -H "authorization: Bearer $ADMIN" \
  -d '{"productId":"...","qty":10,"reason":"initial"}' | jq

# concurrency: 20 parallel, expect exactly 10 × 201 and 10 × 409
seq 20 | xargs -P 20 -I{} curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST .../app/orders -H "authorization: Bearer $TOKEN" \
  -H "idempotency-key: race-{}" -d '{"items":[{"productId":"...","qty":1}],"paymentMethod":"COD"}' \
  | sort | uniq -c

psql $DATABASE_URL -c "SELECT stock_on_hand, stock_reserved FROM products WHERE id='...';"
```

---

## Phase 9 — Order Placement ★

**Depends on:** Phase 8.

**Build**
- `POST /app/orders` — requires `Idempotency-Key`. Replay within 24h returns the original order, not a duplicate.
- Inside one transaction: validate minimums → resolve tier prices → **snapshot** product name, pack size, manufacturer, unit price → insert order + items → reserve stock → compute and store placed totals with snapshotted `vat_rate` and `delivery_zone`.
- Order number generator: `ORD-YYMMDD-NNNN`, from a sequence.
- `GET /app/orders`, `/app/orders/:id` with status history. `POST /app/orders/:id/cancel` — **only while `PLACED`**.
- Reorder: re-price against **current** tiers and availability, returning a diff of changed and unavailable lines rather than silently substituting.
- Expiry consequence: auto-cancel + push notification carrying the original lines for one-tap reorder.

**Definition of Done**
- Same `Idempotency-Key` twice → one order, two identical responses.
- Every price and name on an order line is a snapshot. Editing the product afterwards changes nothing on the order.
- Cancelling releases every reservation.

**Verify**
```bash
K=$(uuidgen)
curl -s -X POST .../app/orders -H "authorization: Bearer $TOKEN" -H "idempotency-key: $K" \
  -d @order.json | jq '.orderNo, .placedTotal'
curl -s -X POST .../app/orders -H "authorization: Bearer $TOKEN" -H "idempotency-key: $K" \
  -d @order.json | jq '.orderNo'    # identical, no new order

psql $DATABASE_URL -c "SELECT status, qty, expires_at FROM stock_reservations WHERE order_id='...';"
```

---

## Phase 10 — Approval, Partial Approval & Confirmation ★

**Depends on:** Phase 9.

**Build**
- Admin approval queue, keyset paginated, filterable by status and date.
- `POST /admin/orders/:id/approve` accepting per-line decisions: `APPROVED` / `REDUCED` (with `approvedQty`) / `REMOVED` (with reason).
- On partial approval:
  1. Release the delta on reduced and removed lines **immediately**.
  2. **Re-resolve the price tier on reduced lines** — 100 → 40 units may cross a tier boundary. Missing this produces wrong invoices.
  3. Recalculate VAT on the new taxable base. **Delivery charge does not change.**
  4. All lines removed → the order is **rejected**, never approved with a delivery-only total.
  5. Admin approval bypasses minimum-order rules deliberately.
- Confirmation (COD immediately; digital after payment): reservations `ACTIVE → CONSUMED`, `ALLOCATION` ledger rows, `stock_on_hand` decremented, `fulfilment_groups` created one per manufacturer.
- After commit: notify pharmacy with a changed-lines payload, alert warehouse, evaluate low-stock, enqueue invoice.

**Definition of Done**
- Partial approval totals reconcile to the poisha against the sum of approved lines.
- Tier re-resolution proven by unit test at a boundary crossing.
- Order confirmed → ledger balance matches `stock_on_hand` exactly.

**Verify**
```bash
curl -s -X POST .../admin/orders/$ID/approve -H "authorization: Bearer $ADMIN" -d '{
  "lines":[{"itemId":"a","approvedQty":40},{"itemId":"b","approvedQty":0,"reason":"out of stock"}]
}' | jq '.approvedSubtotal, .approvedVat, .approvedDeliveryCharge, .approvedTotal'

curl -s .../app/orders/$ID -H "authorization: Bearer $TOKEN" | jq '.changedLines'
psql $DATABASE_URL -c "SELECT status, count(*) FROM stock_reservations WHERE order_id='...' GROUP BY 1;"
```

---

## Phase 11 — Payments

**Depends on:** Phase 10.

**Build**
- Gateway adapter interface; one concrete implementation plus a sandbox stub for tests.
- Digital flow: approval → `AWAITING_PAYMENT`, **reservation TTL extended to the payment window** (24h default, from settings). Payment success → confirm. Window expiry → auto-cancel and release.
- `POST /webhooks/payments/:provider` — signature verified **before parsing**, idempotent, raw payload persisted first.
- Capture amount is always the **approved** total. Never the placed total.

**Definition of Done**
- Duplicate webhook → exactly one confirmation.
- Unpaid at window expiry → order cancelled, stock released, nothing charged.
- No card data is stored anywhere. Grep to confirm.

**Verify**
```bash
curl -s -X POST .../app/orders/$ID/pay -H "authorization: Bearer $TOKEN" | jq '.paymentUrl, .amount'
curl -s -X POST .../webhooks/payments/sandbox -H "x-signature: $SIG" -d @payment-success.json
curl -s -X POST .../webhooks/payments/sandbox -H "x-signature: $SIG" -d @payment-success.json  # replay
curl -s .../app/orders/$ID -H "authorization: Bearer $TOKEN" | jq '.status, .paymentStatus'
```

---

## Phase 12 — Invoicing

**Depends on:** Phase 11.

**Build**
- Triggered on confirmation, generated **asynchronously in the worker**.
- Build `snapshot` jsonb: lines, rates, company details from settings (including BIN), buyer details. **The snapshot is the invoice; the PDF is a rendering of it.**
- `invoice_no` from a Postgres sequence with yearly prefix (`INV-2026-000418`). **Gapless — a sequence, never `COUNT(*) + 1`.**
- Render HTML template → PDF via **headless Chromium (Playwright)**. Chosen for Bangla complex-script shaping; pdfkit/pdfmake render it badly.
- Delivery challan: same pipeline, different template, no pricing.
- Void and reissue only. Never mutate or delete an issued invoice.

**Definition of Done**
- Regenerating from `snapshot` produces a byte-identical PDF.
- Bangla text renders correctly — **open the PDF and look at it**, do not assume.
- Concurrent confirmations produce no duplicate or skipped invoice numbers.

**Verify**
```bash
curl -s .../app/orders/$ID/invoice -H "authorization: Bearer $TOKEN" | jq -r .url | xargs curl -s -o inv.pdf
pdftotext inv.pdf - | head -40    # check totals, VAT line, BIN, Bangla glyphs
psql $DATABASE_URL -c "SELECT invoice_no FROM invoices ORDER BY issued_at DESC LIMIT 5;"
```

---

## Phase 13 — Fulfilment & Couriers

**Depends on:** Phase 12.

**Build**
- Ready-to-ship board grouped by `fulfilment_groups` (per manufacturer picking sets).
- Dispatch: **admin picks internal agent or courier per order**. No routing rules.
- `CourierProvider` interface — `createConsignment`, `cancel`, `parseWebhook`, `mapStatus`. Three implementations: Pathao, Steadfast, RedX.
- `POST /webhooks/couriers/:provider` — signature verified, **deduped on `(provider, event_id)`**, raw payload stored before processing.
- `delivery_events` append-only; both the admin write path and the webhook path land here.
- Reconcile job every 30 minutes for consignments with no recent webhook. Webhooks get lost; this is not optional.

**Definition of Done**
- The same webhook delivered three times produces exactly one state transition.
- An unknown courier status is stored raw and flagged, never silently dropped.
- Internal and courier deliveries converge on one delivery state model.

**Verify**
```bash
curl -s .../admin/fulfilment/ready-to-ship -H "authorization: Bearer $ADMIN" | jq '.groups'
curl -s -X POST .../admin/fulfilment/$GID/dispatch -H "authorization: Bearer $ADMIN" \
  -d '{"mode":"COURIER","provider":"steadfast"}' | jq '.trackingCode'
for i in 1 2 3; do curl -s -X POST .../webhooks/couriers/steadfast -H "x-signature: $SIG" -d @delivered.json; done
psql $DATABASE_URL -c "SELECT count(*) FROM delivery_events WHERE event_id='evt_123';"   # 1
```

---

## Phase 14 — Notifications & Medicine Requests

**Depends on:** Phase 13.

**Build**
- Device token registration; push via FCM. In-app notification store. **SMS reserved for OTP and order confirmation only** — it costs per message.
- Fan-out job with exponential backoff and a dead-letter queue.
- Notification events: account approved/rejected, order approved (with changed lines), order rejected, **order auto-cancelled on reservation expiry (with reorder payload)**, payment required, dispatched, delivered.
- Medicine requests: pharmacy submits; admin queue resolves as `CATALOGUED` (linking the created product) or `UNAVAILABLE`.

**Definition of Done**
- A failed push retries and lands in the DLQ after 3 attempts, with an alert.
- Every notification renders correctly in both Bangla and English templates.

**Verify**
```bash
curl -s -X POST .../app/device-tokens -H "authorization: Bearer $TOKEN" -d '{"token":"fcm_test"}'
curl -s -X POST .../admin/orders/$ID/approve -H "authorization: Bearer $ADMIN" -d @approve.json
curl -s .../app/notifications -H "authorization: Bearer $TOKEN" | jq '.items[0]'
```

---

## Phase 15 — Dashboard & Reports

**Depends on:** Phase 14.

**Build**
- Dashboard aggregates: today's orders, pending approvals, low stock, revenue. Single aggregate queries, not N queries in a loop.
- Reports: daily/monthly sales, product-wise, pharmacy-wise, order status breakdown.
- Excel export as an **async job** → object storage → link pushed to the user. Never block a request on generation.
- Audit log listing, filterable by actor, entity, date.

**Definition of Done**
- Dashboard responds under 200ms with 12 months of seeded data.
- A 50,000-row export completes without exhausting memory (stream, don't buffer).

**Verify**
```bash
curl -s .../admin/dashboard -H "authorization: Bearer $ADMIN" | jq
J=$(curl -s -X POST .../admin/reports/export -H "authorization: Bearer $ADMIN" -d '{"type":"sales","from":"2026-01-01","to":"2026-08-16"}' | jq -r .jobId)
curl -s .../admin/reports/export/$J -H "authorization: Bearer $ADMIN" | jq '.status, .url'
```

---

## Phase 16 — Hardening & Deployment

**Depends on:** all previous phases.

**Build**
- Helmet, strict CORS allowlist, final rate-limit tuning.
- `EXPLAIN ANALYZE` every query touching `orders` or `products`; add missing indexes.
- k6 load test at 3× projected peak: catalogue browse + order placement.
- Multi-stage Dockerfile, distroless base, non-root user. Separate API and worker entrypoints from **one image**.
- GitHub Actions: lint → typecheck → unit tests → build → image push → staging deploy → smoke tests.
- Caddy config, TLS, PgBouncer in transaction mode.
- Prometheus metrics including **active reservation count and expiry rate**.
- Sentry with source maps. Alerts: DLQ non-empty, expiry sweep above threshold, error rate above 1%, disk above 80%.
- Runbook: deploy, rollback, restore, common incidents. **Perform a real restore drill.**

**Definition of Done**
- Zero-downtime rolling deploy verified.
- Restore from backup verified into a scratch instance.
- Load test meets the targets in `ARCHITECTURE.md` §10.

**Verify**
```bash
k6 run load/browse-and-order.js
curl -s localhost:3000/metrics | grep -E 'reservation_active|http_request_duration'
docker compose up -d --no-deps --scale api=2 api    # rolling, health stays green
```

---

## Cross-Phase Rules

1. **Never** write money arithmetic outside `PricingService`.
2. **Never** read or mutate stock outside `InventoryService`.
3. **Never** schedule a queue job inside a transaction — use the after-commit hook.
4. **Never** trust a client-supplied price, total, or availability.
5. Every order line is a **snapshot**. Historical orders must never change.
6. Every external write path is **idempotent**: orders, payments, courier webhooks, imports.
7. Cross-tenant access returns **404, not 403**.
8. Every admin mutation writes an audit log entry.

---

## Global Definition of Done

A phase is complete only when all five are true:

- [ ] `pnpm typecheck` passes with zero errors
- [ ] `pnpm test` passes — **unit tests only**
- [ ] `pnpm lint` clean
- [ ] Curl verification for the phase executed against a running server, output checked
- [ ] Any new environment variable added to `.env.example` and the config schema
