# 03. The system as it exists today

Terms used here are defined in [02-the-contract.md](02-the-contract.md). File paths are in
[10-repo-map.md](10-repo-map.md).

## The path a write actually takes

```text
human / service / agent / device
            |
            v
   HTTP or WebSocket commit
            |
            v
validation + capability + tenant scope
            |
            v
   claim / lease / fence guard
            |
            v
 set-based mutation preparation
            |
            v
  customer Postgres transaction
   |         |            |
   |         |            +-- audit evidence / outbox
   |         +--------------- sync-log / marker data
   +------------------------- application rows
            |
            v
   Postgres commit and WAL
            |
            v
logical decoder with lease and fencing
            |
            v
decode -> projection -> publication grouping
            |
            v
worker/host transfer -> serialisation -> socket enqueue
            |
            v
bounded client queue -> local apply -> durable cursor
```

The property that matters: the commit path and the publication path are related but not
identical. Database settlement does not wait for every subscriber to consume an update, yet the
system retains enough durable ordering and replay information to recover a slow or disconnected
observer exactly. Everything hard about the architecture lives in that gap.

## What the implementation already does

The code is well past a naive row-at-a-time design. Retained mechanisms include set-based
writes, constant-shape JSONB recordset operations, batched ID allocation, asynchronous
publication, sharded writers, an isolated observer, segmented logs, bounded delivery, cheaper
claim handling, index pruning, direct typed WAL hydration, and compact worker-to-host
publication objects. Each of those was retained against a measured control, and the ones that
were tried and rejected are listed in
[04-the-evidence.md](04-the-evidence.md#rejected-and-exhausted-directions).

## WAL robustness

These mechanisms are part of the performance model, not separate from it. Slowing the hot path
to preserve one of them is a normal outcome.

- publication-schema drift is detected and fails loudly
- unchanged TOAST placeholders are handled without inventing values
- restart positions are persisted before acknowledgement
- a lease and fencing model prevents two processors from silently owning one slot
- an invalid or recreated slot forces a bounded resnapshot rather than skipped history
- durable receipts make mediated but unconfirmed work enumerable across a gap
- per-source WAL lag and disk-retention risk are observable
- source disconnect and removal clean up ownership and slot state

The open operational edge is sustained provider outage behaviour. How Aurora, RDS, Neon,
Supabase and self-managed Postgres each behave as retained WAL and recovery time grow is not
yet characterised empirically. The mechanics that decide it are in
[domains/logical-decoding-and-cdc.md](domains/logical-decoding-and-cdc.md).

## What is shipped, what is not

Reading this pack without this table produces an inflated picture of the system.

| Area | State |
| --- | --- |
| Mediated write through one commit chokepoint, into customer Postgres | shipped, journey-proven |
| Claims, leases, fencing, idempotency, typed conflicts, receipts | shipped |
| Logical replication read path, per-source schema binding, drift detection | shipped |
| Ordered publication with exact cursors and replay | shipped |
| Signed HTTP endpoint fallback for databases that cannot grant replication | shipped, marked as the exception |
| Single-plane capacity near 100,000 deltas/sec | measured under one documented topology, see [04](04-the-evidence.md) |
| Multi-plane ownership, rebalancing, quotas, isolation | architectural boundary exists, production proof does not |
| Cross-organisation signed evidence, revocation, dispute handling | direction only |
| Physical-world actuation and observation semantics | model only, research and product work |

## Questions this raises

- The commit path and publication path diverge after the Postgres commit. What is the longest
  observed distance between them under load, and what bounds it?
- Publication grouping happens after projection. What decides a group, and what happens to
  ordering guarantees when one group falls behind another?
- The decoder holds a lease. What is the lease duration, what renews it, and what does a
  fenced-out decoder do with work already in flight?
- A bounded resnapshot is the recovery path for an invalidated slot. What is the cost of that
  resnapshot on a large customer table, and what do observers see while it runs?
- Which parts of this path have never been exercised under a real provider failover rather than
  an injected one?
- The client queue is bounded. What happens at the bound: disconnect, spill, or drop? Who
  learns about it?
