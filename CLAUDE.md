# CLAUDE.md

B2B pharmacy ordering platform — backend. Pharmacies order medicine in bulk; staff approve orders, manage stock, and dispatch deliveries.

**Stack:** NestJS 11 · Node 24 LTS · TypeScript 6.0 · Drizzle ORM · PostgreSQL 17 · Redis + BullMQ · Zod · pnpm

Pin Node 24 (Active LTS) and TypeScript 6.0. **Do not upgrade to TypeScript 7** — it has no public compiler API until 7.1 and breaks `typescript-eslint`.

**Read before building:** `ARCHITECTURE.md` (why the system is shaped this way) and `BUILD_PLAN.md` (phase-by-phase execution order). Build phases in order.

---

## Workflow rules

**Package manager is pnpm. Always.** `pnpm` / `pnpx` — never `npm`, `npx`, `yarn`, or `bun` in commands, scripts, docs, or CI.

**Never hand-write dependencies into `package.json`.** Do not type a package name or a version string into the file. Install through the CLI so the lockfile and version resolution stay honest:

```bash
pnpm add @nestjs/jwt          # runtime
pnpm add -D vitest            # dev
pnpm remove some-package
```

Hand-written entries produce versions that don't exist, lockfile drift, and installs that work on your machine and nowhere else.

**Typecheck after every completed todo:**

```bash
pnpm typecheck    # → pnpx tsc --noEmit
```

Not at the end of the phase. After each item. A type error found three todos later costs far more to unpick.

**Curl-verify after every finished phase or module.** Boot the server and exercise the real endpoints end to end — the exact commands are in each phase of `BUILD_PLAN.md`. A passing unit test is not verification. Read the response body; don't just check the status code.

**Unit tests only.** No integration suites, no e2e frameworks, no Testcontainers. Test pure logic — pricing, tier resolution, state transitions, validation, courier status mapping. Everything else is verified by curl.

---

## Commands

```bash
pnpm dev            # watch mode
pnpm build          # nest build -b swc
pnpm typecheck      # pnpx tsc --noEmit
pnpm test           # vitest run
pnpm lint           # eslint --fix
pnpm db:generate    # drizzle-kit generate
pnpm db:migrate     # apply migrations
pnpm db:seed        # idempotent seed
pnpm docker:up      # postgres + redis + minio
```

---

## Architecture rules

**Module boundaries are real.** A module owns its tables. Never query another module's tables directly — go through its service. This is what keeps the monolith from rotting.

**Two services hold the dangerous logic:**

- **`PricingService`** — every calculation involving money. Tier resolution, VAT, delivery charge, minimums, totals. If money arithmetic appears anywhere else, it is a bug, including in reports and PDF templates.
- **`InventoryService`** — every read or mutation of stock. Reservations, releases, consumption, ledger writes. Nothing else touches `stock_on_hand` or `stock_reserved`.

**Non-negotiable invariants:**

1. Lock product rows `FOR UPDATE` **ordered by id** — unordered locking deadlocks intermittently under concurrent orders.
2. Availability = `stock_on_hand - stock_reserved`, always read from **locked** rows inside the transaction. Never from cache.
3. Reservations are consumed at **confirmation**, not approval. An approved-but-unpaid order must be able to release its stock.
4. Order lines snapshot product name, pack size, manufacturer, and unit price at placement. A historical order never changes because a product was edited.
5. VAT rate and delivery zone are snapshotted per order. Settings changes never reach backwards.
6. Partial approval **re-resolves the price tier** on reduced lines (100 → 40 units may cross a tier boundary) and recalculates VAT. Delivery charge does not change.
7. Queue jobs are scheduled **after commit**, never inside a transaction.
8. Cross-tenant access returns **404, not 403** — a 403 confirms the resource exists.
9. Every external write path is idempotent: orders (`Idempotency-Key`), payments, courier webhooks, imports.
10. `stock_movements` and `delivery_events` are append-only. Never update, never delete.

---

## Conventions

- **IDs:** UUID v7. **Timestamps:** `timestamptz`, UTC in the database. **Money:** `numeric(12,2)`, never floats, serialised as strings.
- **Validation:** Zod schemas via `nestjs-zod`. One schema drives DTO type, runtime validation, and OpenAPI.
- **Errors:** typed domain error classes → RFC 7807 `problem+json`. Internals never reach the client.
- **Config:** Zod-validated env, throws at boot. `process.env` is read in exactly one file.
- **Logging:** Pino, structured, request-scoped correlation ID. Passwords, tokens, and OTPs redacted at the serialiser.
- **Pagination:** keyset cursors everywhere. `OFFSET` is never acceptable on a growing table.
- **Files:** presigned upload direct to storage. Files never pass through the API.
- **Auth:** guards enforce access, not services. `JwtAuthGuard` → `RolesGuard` → `PharmacyScopeGuard`.

---

## Skills

Three project skills live in `.claude/skills/`. They load automatically when relevant — consult them rather than reasoning from scratch:

- **`pricing-rules`** — any calculation involving money: tiers, VAT, delivery charge, minimums, partial-approval recalculation
- **`inventory-invariants`** — any read or mutation of stock: reservations, locking, release, consumption, expiry
- **`nest-module-conventions`** — scaffolding or extending a module: file layout, guards, validation, error mapping

---

## Do not

- Add a dependency by editing `package.json`
- Use `npm`, `npx`, `yarn`, or `bun`
- Cache stock availability
- Trust a client-supplied price, total, or availability
- Calculate money outside `PricingService`
- Touch stock outside `InventoryService`
- Write integration or e2e test suites
- Hard-delete anything an order references
- Move to the next phase with a red typecheck or unverified curl checks
