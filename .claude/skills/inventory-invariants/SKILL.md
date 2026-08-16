---
name: inventory-invariants
description: The stock and reservation rules for this pharmacy platform — the append-only ledger, row locking, reserve/release/consume semantics, reservation expiry, and after-commit job scheduling. Consult this skill whenever writing or reviewing code that reads or changes stock levels, creates or resolves reservations, handles order placement or confirmation, schedules background jobs from inside a transaction, or investigates an overselling or stuck-stock bug. This is the highest-risk area in the codebase — concurrency bugs here oversell medicine or strand it permanently, and neither failure is visible until a pharmacy complains. Consult this skill rather than reasoning about locking from scratch.
---

# Inventory Invariants

All stock reads and mutations go through **`InventoryService`**. Nothing else in the codebase touches `products.stock_on_hand` or `products.stock_reserved`, and nothing else inserts into `stock_movements` or `stock_reservations`.

Stock is the one resource in this system that two pharmacies can genuinely contend for at the same instant. Every rule below exists because of that.

## The three quantities

```
stock_on_hand   physical units in the warehouse   (= SUM of stock_movements.qty_delta)
stock_reserved  units held by ACTIVE reservations (= SUM of active reservation qty)
available       stock_on_hand − stock_reserved    ← what a pharmacy can order
```

`stock_on_hand` and `stock_reserved` are **derived columns maintained in the same transaction as the ledger entry or reservation that changes them**. The ledger is the record; the columns are a cache for fast reads. If they ever disagree with the ledger, the ledger is right and there is a bug in whatever wrote them.

`stock_movements` and `delivery_events` are **append-only**. Never `UPDATE`, never `DELETE`. Corrections are new rows with opposite signs. This is a pharmaceutical supply chain — every unit that entered or left must be reconstructable.

## Reservation lifecycle

```
placement          →  ACTIVE     (stock_reserved += qty)
3h TTL elapses     →  RELEASED   (stock_reserved −= qty)
order rejected     →  RELEASED
line reduced       →  partial release of the delta
order cancelled    →  RELEASED
order CONFIRMED    →  CONSUMED   (stock_on_hand −= qty, stock_reserved −= qty,
                                  ALLOCATION ledger row written)
```

**Consumption happens at confirmation, never at approval.** For a digital-payment order, approval only moves it to `AWAITING_PAYMENT` and extends the reservation to the payment window. If the pharmacy never pays, the stock must return to the pool. Consuming at approval strands stock behind orders that will never be paid.

## The placement transaction

The single most important piece of code in the system. Four rules, each of which has bitten real systems:

**1. Lock in deterministic order.**

```sql
SELECT id, stock_on_hand, stock_reserved, base_price, is_active
FROM products
WHERE id = ANY($1)
ORDER BY id          -- ← this line prevents deadlock
FOR UPDATE
```

Two pharmacies ordering products A and B in opposite orders will deadlock intermittently without `ORDER BY id`. Intermittently is the problem: it passes every test, then fails in production under load, and the failure looks random.

**2. Check availability against the locked rows.** Not against a value read before the lock, and never against a cached value. Between any read and any lock, another transaction can change everything.

**3. Reserve inside the same transaction** that inserts the order. An order that exists without its reservations is an oversell waiting to happen.

**4. Schedule the expiry job after commit, never inside the transaction.** Use the after-commit hook registry in the database module. A job enqueued inside a transaction that then rolls back will fire against an order that does not exist. Give the job a deterministic `jobId` (`expire:${orderId}`) so a retried enqueue cannot produce two expiry jobs for one order.

## Expiry — two mechanisms, deliberately

Reservations expire 3 hours after placement, on plain clock time.

- **Primary:** a BullMQ delayed job per order.
- **Safety net:** a cron sweep every 5 minutes —
  `WHERE status='ACTIVE' AND expires_at < now() FOR UPDATE SKIP LOCKED`

Both exist because a Redis restart or failover loses delayed jobs, and a lost job means stock stranded forever with no error anywhere. `SKIP LOCKED` lets multiple workers sweep concurrently without contending.

An expired reservation auto-cancels its order and pushes a notification carrying the original lines, so the pharmacy can reorder in one tap.

## Idempotency

`release()` and `consume()` are both idempotent: they act only on `ACTIVE` reservations and are silent no-ops otherwise. Running either twice is harmless, and both will run twice eventually — the delayed job and the sweeper will race, webhooks will redeliver, workers will restart mid-batch.

Write them so that concurrent execution is boring rather than guarding against it with locks in application code.

## Testing this area

A passing unit test does not demonstrate correctness here. The check that matters is concurrent:

```bash
# 10 units in stock, 20 parallel single-unit orders
seq 20 | xargs -P 20 -I{} curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST localhost:3000/api/v1/app/orders \
  -H "authorization: Bearer $TOKEN" -H "idempotency-key: race-{}" \
  -d '{"items":[{"productId":"...","qty":1}],"paymentMethod":"COD"}' \
  | sort | uniq -c
```

Expected: exactly ten `201` and ten `409`. Any other split is an oversell or a lost update, and it is a stop-everything result — not something to note and move past.

Also assert after any inventory work:

```sql
-- must always hold
SELECT p.id FROM products p
WHERE p.stock_on_hand <> (
  SELECT COALESCE(SUM(qty_delta), 0) FROM stock_movements WHERE product_id = p.id
);
-- expect zero rows
```

## Before you finish

- Is every product lock ordered by id?
- Is availability checked against locked rows, inside the transaction?
- Are all queue jobs scheduled after commit, with deterministic job IDs?
- Are `release()` and `consume()` safe to run twice?
- Does the ledger sum match `stock_on_hand` for every product?
- Did the 20-parallel-order check produce exactly 10 successes?
