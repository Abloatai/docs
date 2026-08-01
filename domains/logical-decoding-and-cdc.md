# Logical decoding and change data capture

The WAL echo promotes a receipt from accepted to confirmed
([settlement rule](../02-the-contract.md#the-settlement-rule)), so the decoder is a correctness
component and an availability dependency on infrastructure Ablo does not own.

## What the database gives you

| Concern | Mechanism | Source |
| --- | --- | --- |
| Retention floor | `wal_keep_size` | [runtime-config-replication](https://www.postgresql.org/docs/current/runtime-config-replication.html) |
| Disk-exhaustion cap | `max_slot_wal_keep_size`. Past it, segments are removed, the connection terminates, the slot is invalidated | [runtime-config-replication](https://www.postgresql.org/docs/current/runtime-config-replication.html) |
| Headroom, in bytes | `pg_replication_slots.safe_wal_size` | [view-pg-replication-slots](https://www.postgresql.org/docs/current/view-pg-replication-slots.html) |
| Why a slot died | `invalidation_reason`: WAL removed, rows removed, `wal_level` too low, idle timeout | [view-pg-replication-slots](https://www.postgresql.org/docs/current/view-pg-replication-slots.html) |
| Failover, PG17+ | `pg_create_logical_replication_slot(..., failover => true)` plus `synchronized_standby_slots` | [logical-replication-failover](https://www.postgresql.org/docs/current/logical-replication-failover.html) |
| Failover readiness | `synced AND NOT temporary AND invalidation_reason IS NULL`, verified rather than assumed | [logical-replication-failover](https://www.postgresql.org/docs/current/logical-replication-failover.html) |
| Version boundary | Debezium `slot.failover` defaults to false and is **ignored** on PostgreSQL 16 and earlier | [Debezium PG connector](https://debezium.io/documentation/reference/3.6/connectors/postgresql.html) |

Ablo supports Aurora, RDS, Neon, Supabase and self-managed Postgres, so failover behaviour is
per-provider and per-version rather than one guarantee.

## What the read path next door looks like

ElectricSQL serves shapes over plain HTTP: request a shape with `offset`, use `-1` for an
initial sync, then follow an append-only log carrying `up-to-date`, `snapshot-end` and
`must-refetch` ([HTTP API](https://electric.ax/docs/sync/api/http)). `must-refetch` is the same
admission Ablo's bounded-resnapshot path makes, stated in the protocol instead of raised as an
error.

## The decode-once choice

| | Decode per server | One fenced decoder per source |
| --- | --- | --- |
| CPU and memory bandwidth | multiplied by server count | paid once |
| Subscription lookup | duplicated per consumer | done once, routed compact |
| Recovery boundaries | one | two |
| Safe acknowledgement point | that server's position | highest **contiguous** LSN across all downstream partitions |

The invariant either way: never acknowledge source WAL past data that cannot be reconstructed
after the relevant failure.

## See it yourself

Slot health is one query, and it answers "how close is this customer to a disk incident".

```sql
SELECT slot_name, active, wal_status, safe_wal_size,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS behind,
       invalidation_reason
FROM pg_replication_slots;
```

Stop the consumer and watch `behind` grow and `safe_wal_size` shrink. That is the entire
operational risk of the read path, visible in one row.

## Go deeper

| Read | For |
| --- | --- |
| [Logical decoding concepts](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html) | how WAL becomes application-level change, and what a slot retains |
| [Debezium PostgreSQL connector](https://debezium.io/documentation/reference/3.6/connectors/postgresql.html) | the same problem operated at scale for a decade, including the failure modes |
| [Logical replication failover](https://www.postgresql.org/docs/current/logical-replication-failover.html) | why failover slots are newer and more conditional than they look |
| [Electric HTTP API](https://electric.ax/docs/sync/api/http) | a read path built as a cacheable log with offsets, and the control messages it needs |

## Still open

- The sustained outage envelope per provider: how Aurora, RDS, Neon, Supabase and self-managed
  Postgres behave as retained WAL and recovery time grow. Nobody has characterised this.
- The crossover between replaying from a retained slot and resnapshotting, as a function of lag
  and table size.
- Whether one decoding pass can safely feed all publication work, and how the highest contiguous
  safe LSN is computed cheaply when partitions progress independently.
