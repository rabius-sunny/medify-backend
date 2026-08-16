# B2B Pharmacy Platform — Backend Documentation Bundle

Drop these at the root of the backend repository.

```
.
├── CLAUDE.md                  # agent working rules — read first
├── ARCHITECTURE.md            # system design and rationale
├── BUILD_PLAN.md              # 17 sequential phases, each curl-verifiable
└── .claude/skills/
    ├── pricing-rules/         # money: tiers, VAT, delivery, minimums
    ├── inventory-invariants/  # stock: ledger, locking, reservations
    └── order-lifecycle/       # states, snapshots, idempotency, tenancy
```

## Order of use

1. Read `CLAUDE.md` — working rules, commands, invariants.
2. Read `ARCHITECTURE.md` §0 (confirmed decisions) and §5 (data model).
3. Execute `BUILD_PLAN.md` from Phase 0. One phase at a time.
4. Consult the relevant skill *before* writing code in its area.

## Non-negotiables

- pnpm/pnpx only — never edit `package.json` by hand, always `pnpm add`
- `pnpm typecheck` after every completed todo
- curl verification after every phase, against a running server
- unit tests only

Phases 8, 9 and 10 (inventory, placement, approval) carry the real risk and are strictly sequential.
