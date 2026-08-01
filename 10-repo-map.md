# 10. Where this lives in the codebase

Useful only with repository access. Paths verified on 2026-08-01. When the source and any
document disagree, the source wins and the discrepancy is worth reporting.

## Follow one transaction

The most useful first mental model is a single write moving through the real system. The stages
map to files in this order:

| Stage | Where |
| --- | --- |
| commit coordination, the chokepoint every write passes | `apps/sync-server/src/mutators/commit.ts` and `apps/sync-server/src/mutators/commit/` |
| claim, lease and fence validation | `apps/sync-server/src/presence/ClaimLeaseStore.ts`, `apps/sync-server/src/mutators/claim-guard.ts` |
| delta persistence and emission | `apps/sync-server/src/mutators/commit/deltaSink.ts` and the sync-log modules beside it |
| source registration, mapping, preflight, decoding, recovery | `apps/sync-server/src/replication/postgres/` |
| headless transaction contracts and wire types | `packages/transaction/` |
| local reactive application and client state | `packages/humans/src/local/` |
| load driver and disposable cloud topology | `apps/sync-server/scripts/bench-commit-throughput.cts`, `infra/terraform/environments/bench-ondemand/` |

## Documents worth reading alongside

| Document | What it covers |
| --- | --- |
| `docs/ablo-vision.md` | the long-horizon collaboration thesis |
| `docs/whitepaper.md` | protocol concepts: commits, receipts, capabilities, deltas |
| `docs/write-path-and-roles-explained.md` | the mediated write boundary and the database privilege model |
| `docs/wal-robustness-explained.md` | WAL failure and recovery semantics in detail |
| `docs/investor-technical-thesis-memo.md` | product and system framing with the same honest gaps |
| `docs/plans/throughput-next-performance-handoff-2026-07-30.md` | the source of every number in [04](04-the-evidence.md), run by run |
| `docs/plans/throughput-10k-handbook-and-100k-roadmap.md` | the performance contract, historical architecture, and benchmark discipline |
| `docs/decisions/` | the architecture decision records, which carry the reasoning this pack summarises |

## Conventions that explain the shape of the code

`packages/transaction/CONVENTIONS.md` states the rule the whole repository is organised around:
one definition site per shape, everything else derived. Schemas define, types are inferred,
documentation is generated. A hand-written second copy of a contract is treated as a defect
because nothing fails when the copies drift.

This pack follows the same rule for prose, which is why a fact appears in exactly one file here
and everything else links to it.
