---
name: pricing-rules
description: The authoritative money rules for this pharmacy platform — price tier resolution, VAT, delivery charge, minimum order enforcement, rounding, and rate snapshotting. Consult this skill whenever touching anything that produces or consumes a price, subtotal, discount, VAT amount, delivery charge, or order total — including cart preview, order placement, partial approval, invoice generation, reports, and PDF templates. Also use it when reviewing code for correctness in these areas, or when a total looks wrong by a few poisha. Money bugs in this system surface as incorrect invoices to real pharmacies, so err strongly toward consulting this skill rather than reasoning from first principles.
---

# Pricing Rules

Every calculation involving money in this codebase lives in **one place: `PricingService`**. If money arithmetic appears in a controller, a repository, a report query, or a PDF template, that is a defect regardless of whether the number happens to come out right.

The reason is not tidiness. There are four consumers of these numbers — cart preview, order placement, partial approval, and the invoice — and they must agree exactly. Duplicated arithmetic drifts, and the drift is discovered by a pharmacy owner comparing an invoice to what the app showed them.

## The calculation

Fixed order. Never reorder these steps; each depends on the one before it.

```
line_total       = tier_unit_price × qty          (per line)
subtotal         = Σ line_total
discount         = order-level discount, if any
taxable_base     = subtotal − discount
vat              = round(taxable_base × vat_rate, 2)
delivery_charge  = zone.charge                     (snapshotted at placement)
vat_on_delivery  = vat_applies_to_delivery
                     ? round(delivery_charge × vat_rate, 2)
                     : 0
grand_total      = taxable_base + vat + delivery_charge + vat_on_delivery
```

**Round once per component, at the end.** Rounding each line and summing produces totals that disagree with the invoice by a few poisha — small enough to ship, large enough to become an accounting dispute. Use Postgres `numeric` arithmetic. JavaScript floats are not acceptable for money anywhere in this codebase, including in tests.

## Price tier resolution

Products carry a `base_price` and zero or more `product_price_tiers` rows (`min_qty`, `unit_price`).

**Rule: the highest `min_qty` that is less than or equal to the ordered quantity wins.** No tier matches, or no tiers exist → `base_price`.

```
tiers: [{min_qty: 10, price: 95}, {min_qty: 50, price: 90}, {min_qty: 100, price: 85}]
base_price: 100

qty 5   → 100  (below every tier)
qty 10  → 95   (exactly on a boundary — boundaries are inclusive)
qty 49  → 95
qty 50  → 90
qty 250 → 85   (above the highest tier)
```

Boundary inclusiveness is the thing people get wrong. `qty 10` gets the tier-10 price, not the base price.

## Rate snapshotting

`vat_rate`, `delivery_zone_id`, `delivery_zone_name`, and `vat_applies_to_delivery` are **copied onto the order at placement**. They are never read live from `platform_settings` when displaying, recalculating, or invoicing an existing order.

An admin changing the VAT rate tomorrow must not alter an order placed today, and must never alter an invoice already issued. Reading settings live to recompute an old order is the single most damaging mistake available in this area — it silently rewrites financial history.

The same applies to order lines: `unit_price`, `product_name`, `pack_size`, and `manufacturer_id` are snapshotted at placement. Editing a product later changes nothing about existing orders.

## Partial approval

When admin approves fewer items than requested:

1. **Re-resolve the price tier on every reduced line.** Dropping a line from 100 units to 40 may cross a tier boundary downward — the pharmacy no longer qualifies for the 100-unit price. Keeping the original unit price overcharges nobody but undercharges you, and produces an invoice whose line total doesn't match its own unit price × quantity.
2. **Recalculate VAT** on the new taxable base. The base changed, so the tax changed.
3. **Do not change the delivery charge.** The van drives the same distance for 8 items as for 10. Delivery is a zone charge, not a per-item cost.
4. **If every line is removed, the order is rejected** — never approved with a delivery-charge-only total. An order consisting solely of a delivery fee is not a thing that should exist.

Store both `placed_*` and `approved_*` totals. The pharmacy needs to see what changed and why, and reconstructing that from history is far more work than storing it.

## Minimum order enforcement

Two independent rules, both checked at cart preview **and** re-validated inside the placement transaction. A client-side check is a convenience; the transaction check is the control.

| Rule | Source | Applied to |
| :--- | :--- | :--- |
| Minimum order value | `platform_settings.min_order_value` | `subtotal` — **goods only** |
| Minimum quantity | `products.min_order_qty` | Each line |
| Order multiple | `products.order_multiple` | Each line — exact multiples only |

Minimum order value is measured on the goods subtotal, **excluding VAT and delivery charge**. Otherwise a pharmacy reaches your minimum partly by paying for shipping, which defeats the point of having one.

`order_multiple` means carton or pack multiples: with `order_multiple: 12`, valid quantities are 12, 24, 36 — not 15. Both fields default to 1, so products without minimums need no special handling.

Violations return `422` with a **per-line breakdown**, so the app can highlight offending items instead of showing one unhelpful banner.

**Admin partial approval is exempt from all minimums.** If reducing a line takes it below its MOQ, or drops the order under the minimum value, the approval still stands. Minimums exist to shape what pharmacies submit, not to override your team's judgement about what to ship. This asymmetry is deliberate — do not "fix" it by enforcing minimums on the admin path.

## Delivery zones

A pharmacy's district maps to exactly one active zone; overlapping district mappings are rejected at configuration time, not at order time.

An **unmapped district falls back to `default_delivery_charge` and raises an admin warning — it never blocks checkout**. A pharmacy in a district nobody remembered to map should still be able to order.

## Before you finish

- Is every arithmetic operation inside `PricingService`? Grep for `*`, `+`, and `toFixed` near price-shaped variable names elsewhere.
- Are rates read from the order, not from settings, for any existing order?
- Does the invoice total equal the sum of approved lines plus VAT plus delivery, to the poisha?
- Do unit tests cover tier boundaries (below first, exactly on, above last, no tiers) and VAT-on-delivery both on and off?
