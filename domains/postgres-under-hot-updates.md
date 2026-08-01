# Postgres under a high update rate

| | |
| --- | --- |
| **Measured cost** | task UPDATE 321 ms p50, 400 ms p95, of 338/416 ms total application SQL ([run 118](../04-the-evidence.md#where-update-cost-actually-goes)) |
| **Dominant wait** | `LWLock:BufferContent` |
| **Causal proof** | drop every secondary index: 100,384/sec (run 121), breaks the read contract |
| **Proposal** | vertical split of identity, mutable state and projections. Untested |
| **Ceiling it sets** | write amplification on indexed mutable columns |

## What decides the cost

| Mechanism | Rule | Source |
| --- | --- | --- |
| HOT eligibility | an update skips index writes only if no indexed column changed **and** the page has room for the new version | [storage-hot](https://www.postgresql.org/docs/current/storage-hot.html) |
| HOT benefit | indexes keep pointing at the original item id, which becomes a redirect; intermediate versions are reclaimed during reads instead of vacuum | [storage-hot](https://www.postgresql.org/docs/current/storage-hot.html) |
| Index cost | indexes speed reads, add DML overhead, and can prevent heap-only tuples | [indexes-intro](https://www.postgresql.org/docs/current/indexes-intro.html) |
| Summarising indexes | BRIN and other `amsummarizing` methods keep HOT eligible, provided the column is not in an index predicate | [index-api](https://www.postgresql.org/docs/current/index-api.html) |
| Vacuum debt | dead tuples persist until vacuumed, so a miss is paid twice: index writes now, reclamation later | [routine-vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html) |

A 20-second measured interval flatters the second payment, because vacuum arrives under
different load.

## The proposed split

```text
task_identity_and_routing    stable identity, ownership, tenant, routing keys. Indexed
task_mutable_state           hot fields. Minimal or no secondary indexing
task_query_projection        measured query dimensions only. Freshness policy stated
```

Trades write amplification in one wide indexed row for joins, projection maintenance, and an
explicit freshness policy.

## See it yourself

The HOT ratio is one query. Run an update workload, add an index on the column being updated,
run it again, and watch the second number stop tracking the first.

```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / nullif(n_tup_upd, 0), 1) AS hot_pct,
       n_dead_tup
FROM pg_stat_user_tables ORDER BY n_tup_upd DESC LIMIT 10;
```

`EXPLAIN (ANALYZE, BUFFERS)` on the same update shows the buffer writes the index maintenance
added, and `pg_stat_statements` shows where the latency went.

## Go deeper

| Read | For |
| --- | --- |
| [Heap-only tuples](https://www.postgresql.org/docs/current/storage-hot.html) | one short page, and the whole cost model is in it |
| [Routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html) | why a write-heavy benchmark looks better than the system behaves |
| [MVCC](https://www.postgresql.org/docs/current/mvcc.html) | tuple versions and snapshots, the layer HOT is an optimisation of |
| [TigerBeetle on batching](https://docs.tigerbeetle.com/coding/requests) | what a storage engine achieves when it owns the schema and can batch 8,189 events per request |

## Still open

- Whether a vertical split preserves the read paths that justify the current indexes, and what
  it does to p99 read latency once a logical entity spans three relations.
- Whether index choice rather than index count is the lever: BRIN or covering indexes could
  restore HOT eligibility for part of the workload.
- How a split is offered at all to a customer whose schema Ablo does not own and never runs DDL
  against.
