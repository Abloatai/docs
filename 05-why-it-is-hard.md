# 05. Why this is a hard systems problem

## An update is not one operation

One logical update causes work in fourteen places:

| | Stage | |
| --- | --- | --- |
| 1 | authentication and schema validation | 8 | logical decoding |
| 2 | capability and tenant scope | 9 | row-to-domain projection |
| 3 | claim or fence lookup | 10 | subscription and group lookup |
| 4 | application-row mutation | 11 | worker to host transfer |
| 5 | secondary-index maintenance | 12 | serialisation and socket enqueue |
| 6 | sync-log append | 13 | client queue and local persistence |
| 7 | audit append, commit, WAL | 14 | materialisation and cursor advance |

At rate, the arithmetic is unforgiving.

| At 100,000 deltas/sec | Cost |
| --- | --- |
| 1 µs of CPU per delta | 10% of one core, permanently |
| 300 bytes copied per delta | 30 MB/sec of memory traffic |
| one global lock or lookup per delta | a serialisation point, not an overhead |

Which is why a change in bytes per delta is a first-class result
([run 136](04-the-evidence.md#the-most-recent-retained-result)). At this rate, representation is
architecture.

## Ordering creates a serial fraction

Something must decide the order that observers replay. Amdahl bounds what parallelism can then
do:

```text
speedup <= 1 / s        s = the serial fraction
```

| Serial fraction | Ceiling on speedup |
| --- | --- |
| 1% | 100x |
| 5% | 20x |
| 10% | 10x |
| 20% | 5x |

Worse, `s` grows under load, as workers contend for the same lock, queue, cache line, credit
ledger, checkpoint or socket dispatcher. The question is not how to parallelise the code. It is
which ordering is semantically necessary
([ordering and frontiers](domains/ordering-and-frontiers.md)).

## Exactly once is built, not switched on

No transport prevents a client seeing a retry after a crash. The guarantee is assembled from
idempotency keys, durable transaction identifiers, deterministic delta identity, replay from a
durable log, monotonic cursor advance, consumer-side deduplication, and acknowledgement
persisted only after application.

The neighbours admit the same thing in their own vocabulary, which is why they are worth reading:

| System | The admission |
| --- | --- |
| Flink | exactly-once operator state comes from checkpoints, but end-to-end delivery needs the **sink** to participate: two-phase commit, or a JDBC XA sink with retries disabled ([guarantees](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/connectors/datastream/guarantees/)) |
| Temporal | no exactly-once claim at all. A worker can complete an activity and crash before reporting it, so it is retried, and idempotency is the application's job ([activity definition](https://docs.temporal.io/activity-definition)) |

Optimising one stage without modelling the crash boundary opens a narrow data-loss window that
steady-state tests never show.

## Backpressure crosses administrative boundaries

A database commits faster than one subscriber consumes. One client has a bad network. One plane
runs hot enough to starve another. A provider retains WAL until its disk is at risk.

The policy has to be explicit: bounded queues, durable spill or replay, per-plane and
per-subscriber credits, lag limits, admission control at the commit boundary, priority and
fairness, and recovery after overload. Unlimited queues produce delayed failure, not
reliability, and they hide it behind a healthy average
([backpressure](domains/backpressure-and-overload.md)).

## Physical effects are not database transactions

A database update rolls back; a robot has already moved. That needs command intent separated
from acknowledgement, idempotent actuator commands, leases and fences on control ownership,
compensating actions instead of fictional rollback, and safety policy that stays local when the
cloud is unreachable ([physical state](domains/physical-world-state.md)).

## Cost sits in the same table as latency

A design reaching one million per second on ten times the compute can be commercially worse than
a 300,000 per second design with predictable horizontal scale, particularly where
[fan-out is not separately billable](01-the-space.md#the-commercial-shape-of-the-neighbourhood).

## Go deeper

| Read | For |
| --- | --- |
| [Flink connector guarantees](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/connectors/datastream/guarantees/) | where an end-to-end guarantee actually terminates |
| [Temporal activity definition](https://docs.temporal.io/activity-definition) | at-least-once stated plainly by a system people assume is stronger |
| [Jepsen consistency models](https://jepsen.io/consistency/models) | naming the guarantee you are claiming before defending it |
| [Little's law](https://pubsonline.informs.org/doi/10.1287/opre.9.3.383) | why the queue, not the rate, is the health signal |

## Still open

- The serial fraction, measured rather than estimated, and which stage owns it.
- Which stages are single-threaded by necessity against by history.
- The largest window in which a crash produces a duplicate the client must deduplicate.
- What the system does when the customer's database, rather than Ablo, is the slow component.
