---
name: order-lifecycle
description: The order state machine and its surrounding rules for this pharmacy platform — valid transitions, snapshot-on-placement, idempotency keys, webhook deduplication, payment sequencing, and cross-tenant access behaviour. Consult this skill whenever building or reviewing order placement, approval, rejection, cancellation, confirmation, payment callbacks, courier webhooks, reorder, or any endpoint that returns an order to a pharmacy. Also use it when adding a new order status, a new external write path, or any endpoint where one pharmacy could conceivably see another's data. Getting a transition or an idempotency guard wrong here produces duplicate orders and cross-tenant leaks, so consult this rather than inferring the flow from the code.
---

# Order Lifecycle

An order moves through a fixed state machine. Every transition is recorded in `order_status_history` with the actor. There are no undocumented shortcuts between states, and adding one requires updating this skill first.

## States and transitions

```
PLACED ──(3h TTL)────────→ CANCELLED        stock released, push sent
       ──(pharmacy)──────→ CANCELLED        only from PLACED
       ──(admin reject)──→ REJECTED         stock released
       ──(admin approve)─→ APPROVED         full or partial

APPROVED ─(COD)──────────→ CONFIRMED        stock consumed
         ─(DIGITAL)──────→ AWAITING_PAYMENT reservation extended to payment window

AWAITING_PAYMENT ─(paid)─→ CONFIRMED        stock consumed
                 ─(expiry)→ CANCELLED        stock released, nothing charged

CONFIRMED → PREPARING → OUT_FOR_DELIVERY → DELIVERED
```

Two rules that are easy to get subtly wrong:

- **A pharmacy can self-cancel only from `PLACED`.** After approval, cancellation is an admin action — your team may already have picked the order.
- **Stock is consumed at `CONFIRMED`, not at `APPROVED`.** An approved-but-unpaid digital order must be able to give its stock back. See the `inventory-invariants` skill.

## Snapshot on placement

Every order line copies, at placement:

`product_name` · `pack_size` · `manufacturer_id` · `unit_price` (tier-resolved)

And every order copies: `vat_rate` · `delivery_zone_id` · `delivery_zone_name` · `vat_applies_to_delivery`.

**A historical order must never change because someone edited a product or a setting.** Rendering an order by joining live to `products` looks correct in development, where nothing has changed yet, and silently rewrites history in production. An invoice from three months ago must render exactly as it did then.

Corollary: never hard-delete a product, manufacturer, or generic that an order references. Soft delete only.

## Partial approval

Admin submits per-line decisions: `APPROVED`, `REDUCED` (with `approvedQty`), or `REMOVED` (with a reason).

The orchestration order matters:

1. Release the reservation delta on reduced and removed lines — immediately, in the same transaction.
2. Re-resolve price tiers on reduced lines and recalculate VAT (see the `pricing-rules` skill).
3. Write `approved_*` totals alongside the preserved `placed_*` totals.
4. Record the decision, actor, and per-line reasons.
5. After commit: notify the pharmacy with an explicit **changed-lines payload** — not just a status flip. "Your order was approved" when two items were dropped is a support ticket.

All lines removed → the order is `REJECTED`, not approved.

## Idempotency

Three external write paths, all of which will be replayed:

| Path | Key | Behaviour on replay |
| :--- | :--- | :--- |
| Order placement | `Idempotency-Key` header, scoped per pharmacy | Return the original order, unchanged, within 24h |
| Payment webhook | Provider event ID | Exactly one confirmation, ever |
| Courier webhook | `(provider, event_id)` unique constraint | Exactly one state transition |

Mobile clients retry on flaky connections. Payment gateways send duplicate callbacks routinely. Couriers redeliver webhooks for days. **Assume every external call arrives more than once** and make the second arrival a no-op rather than a second order.

For webhooks: verify the signature **before parsing the body**, and persist the raw payload before processing it. When a provider changes their format without telling you, the stored raw payload is the only way to find out what actually happened.

An unrecognised courier status is stored raw and flagged for review — never silently dropped, and never mapped to a guess.

## Payment sequencing

Capture happens **after approval**, on the **approved** total — never the placed total. A pharmacy who ordered 50,000 taka of stock and had 8,000 removed pays 42,000, and is never charged the larger amount even momentarily.

Between approval and payment the reservation stays `ACTIVE` with an extended TTL (payment window, default 24h from settings). Window expires unpaid → order cancelled, stock released, nothing charged, no invoice issued.

An invoice is issued at **confirmation**, so an unpaid order never produces one.

## Reorder

Re-price against **current** tiers and **current** availability. Return a diff of changed and unavailable lines rather than silently substituting or dropping them. A reorder that quietly ships different quantities at different prices than the pharmacy expected is worse than one that asks them to confirm.

## Cross-tenant access

Any endpoint returning a pharmacy-owned resource enforces ownership in `PharmacyScopeGuard`, not in the service.

**Return 404, not 403, for another tenant's resource.** A 403 confirms the resource exists — on a platform where competing pharmacies share the system, that is itself a leak. Enumerable IDs plus 403 responses is a directory of your customers' order volumes.

Enforcement lives in a guard because the one service that forgets is the one that leaks, and services get added faster than they get audited.

## Before you finish

- Is every transition in the diagram above, with history written and an actor recorded?
- Are all line prices and names snapshots, never live joins?
- Does replaying the same idempotency key or webhook produce exactly one effect?
- Is the capture amount the approved total?
- Does another pharmacy's order ID return 404?
- Does the approval notification tell the pharmacy which lines changed?
