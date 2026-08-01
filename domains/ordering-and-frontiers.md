# Ordering, partitioning and progress frontiers

The cursor is part of the public contract, so the choice between one integer and a set of
positions is a protocol change rather than a refactor. Why a scalar order caps throughput:
[Amdahl and the serial fraction](../05-why-it-is-hard.md#ordering-creates-a-serial-fraction).

| Model | Consumer position | Parallelism | Cost |
| --- | --- | --- | --- |
| One total order per plane | one integer | none across the plane | unrelated updates contend |
| Partitioned lanes | one position per lane | per lane | cross-lane transactions need a barrier |
| Frontier | a set, or a compressed vector | per lane | clients and operators must understand it |

## What the field already shipped

| System | Mechanism | Detail |
| --- | --- | --- |
| Kafka | order and offset **within** a partition, never across | key routed by Murmur2 hash of the serialised key, modulo partition count, so one key always lands in one partition ([design](https://kafka.apache.org/documentation/#design)) |
| Kafka | offsets are permanent identifiers | compaction never changes an offset and never reorders retained records |
| Kafka | two products from one log | a consumer at the head sees **every** message; a consumer reading a compacted log from the start sees the **final state** per key plus delete markers in the retention window |
| Materialize | a frontier in a SQL interface | `SUBSCRIBE` emits `mz_timestamp` and `mz_diff`; with `PROGRESS`, `mz_progressed` asserts no further records below that timestamp ([SUBSCRIBE](https://materialize.com/docs/sql/subscribe/)) |
| Materialize | resume instead of resnapshot | `RETAIN HISTORY` lets a disconnected client resume from its last timestamp ([durable subscriptions](https://materialize.com/docs/transform-data/patterns/durable-subscriptions/)) |
| Naiad | frontiers, formally | logical timestamps and progress tracking ([paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/11/naiad_sosp2013.pdf)) |
| Calvin | the opposite corner | order transactions deterministically up front, avoid distributed commit during execution ([paper](https://www.cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)) |

Kafka's compaction contract is the cleanest existing statement of the event-complete against
state-convergent split that [fan-out](fanout-and-incremental-views.md) needs.

## Candidate lane assignments for Ablo

Entity or key range, capability scope, claim domain, static shape partition, deterministic
multi-lane barrier, interval-tree or compressed vector frontier, epoch-based split and merge.
The regime B sketch is in
[06-scale-regimes.md](../06-scale-regimes.md#regime-b-one-hot-plane-at-1m).

Conflict rate is the hidden variable. Sparse declared dependencies favour partitioned progress.
One hot key, or frequent cross-partition transactions, collapses the design back to global
coordination while keeping the added complexity.

## In the code

The cursor is [wire/feedCursor.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/feedCursor.ts) on the wire and
[local/logPosition.ts](https://github.com/Abloatai/ablo/blob/main/packages/humans/src/local/logPosition.ts) on the client, which is where a
scalar-to-frontier change would land. Conflict detection between two writes is
[coordination/targetConflict.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/coordination/targetConflict.ts), and what a
claim covers is [footprint.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/footprint.ts).

## See it yourself

Kafka shows per-partition lag as separate numbers, which is a frontier in operational clothing:

```sh
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group
```

One `CURRENT-OFFSET` and one `LAG` per partition, never one number for the topic. Any system
with independent lanes ends up printing this same table.

## Go deeper

| Read | For |
| --- | --- |
| [Kafka design](https://kafka.apache.org/documentation/#design) | partition-local order and compaction, the tradeoff made explicit and shipped |
| [Naiad](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/11/naiad_sosp2013.pdf) | what a frontier is, stated precisely |
| [Calvin](https://www.cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) | determinism as an alternative to coordination |
| [Jepsen consistency models](https://jepsen.io/consistency/models) | which guarantee you are actually claiming, and what it costs |

## Still open

- Which Ablo operations genuinely require a total order. The list decides whether frontier work
  is justified at all, and nobody has written it down.
- Whether a mutation can declare coordination keys at the schema level, the way a partition key
  is declared, instead of inferring a read set at runtime.
- Whether a scalar-cursor client can be served from a deterministic merge of lanes without
  losing exactness, and what that merge costs per delta.
- What the measured conflict rate is in a realistic agent workload. Lane design before that
  number is guesswork.
