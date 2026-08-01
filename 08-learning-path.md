# 08. Learning path

A route through the space for someone who wants to reason about it independently. Six stages,
each with what to read, something to run, and what you should be able to explain afterwards.

## 1. One transaction, end to end

Read [03-the-system-today.md](03-the-system-today.md), then follow a single write through the
stages in [05](05-why-it-is-hard.md#an-update-is-not-one-operation).

| | |
| --- | --- |
| **Run** | the two-line [queued against confirmed](02-the-contract.md#see-it-yourself) comparison, and time the gap |
| **Explain after** | why `queued` and `confirmed` are different states, and what sits between them |

Everything else in this pack is a magnification of one stage in that path.

## 2. The database underneath

| Read | For |
| --- | --- |
| [MVCC](https://www.postgresql.org/docs/current/mvcc.html) | snapshots and tuple versions |
| [Heap-only tuples](https://www.postgresql.org/docs/current/storage-hot.html) | why an indexed UPDATE costs what it costs |
| [Routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html) | the bill that arrives after the benchmark ends |
| [Logical decoding](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html) | how WAL becomes an application-level change |
| [pg_replication_slots](https://www.postgresql.org/docs/current/view-pg-replication-slots.html) | how a consumer becomes a disk-space risk |

**Run:** the two queries in
[postgres-under-hot-updates](domains/postgres-under-hot-updates.md#see-it-yourself) and
[logical-decoding-and-cdc](domains/logical-decoding-and-cdc.md#see-it-yourself). Add an index on
a hot column and watch `n_tup_hot_upd` stop tracking `n_tup_upd`.

**Explain after:** the two conditions for a HOT update, and what `safe_wal_size` falling to zero
does to a customer.

## 3. Ordering and consistency vocabulary

| Read | For |
| --- | --- |
| [Jepsen consistency models](https://jepsen.io/consistency/models) | naming a guarantee precisely before claiming it |
| [Kafka design](https://kafka.apache.org/documentation/#design) | order inside a partition, never across |
| [Spanner](https://research.google.com/archive/spanner-osdi2012.pdf) | what global consistency actually costs |
| [Calvin](https://www.cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) | determinism instead of coordination |
| [FoundationDB](https://apple.github.io/foundationdb/architecture.html) | separated roles under one transaction contract |

**Run:** `kafka-consumer-groups.sh --describe` on a multi-partition topic and read the per
partition lag. That table is a frontier in operational clothing
([ordering](domains/ordering-and-frontiers.md#see-it-yourself)).

**Explain after:** why a scalar cursor caps throughput, and what replaces it.

## 4. Streaming, progress and incremental views

| Read | For |
| --- | --- |
| [Naiad](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/11/naiad_sosp2013.pdf) | frontiers, stated formally |
| [Materialize SUBSCRIBE](https://materialize.com/docs/sql/subscribe/) | a frontier in a shipped SQL interface |
| [Differential Dataflow](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/01/differentialdataflow.pdf) | incremental maintenance instead of recomputation |
| [Flink guarantees](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/connectors/datastream/guarantees/) | where an end-to-end guarantee terminates |
| [Electric HTTP API](https://electric.ax/docs/sync/api/http) | a read path built as a cacheable log |

**Run:** curl an Electric shape and follow the offsets
([fan-out](domains/fanout-and-incremental-views.md#see-it-yourself)).

**Explain after:** the difference between a consumer that needs every transition and one that
needs current state, and why one log can serve both.

## 5. Performance engineering

| Read | For |
| --- | --- |
| [Little's law](https://pubsonline.informs.org/doi/10.1287/opre.9.3.383) | why drain, not throughput, is the health signal |
| [How Not to Measure Latency](https://qconsf.com/sf2012/dl/qcon-sanfran-2012/slides/GilTene_HowNotToMeasureLatency.pdf) | coordinated omission |
| [TigerBeetle requests](https://docs.tigerbeetle.com/coding/requests) | how a headline number is constructed |
| [07-measuring-it.md](07-measuring-it.md) | the standard applied to Ablo's own runs |

**Run:** the `structuredClone` timing in
[representation-and-memory](domains/representation-and-memory.md#see-it-yourself). Halve the
payload, halve the time. That is the entire mechanism behind the last retained result.

**Explain after:** why runs 124 to 128 were rejected despite matching throughput
([04](04-the-evidence.md#the-rejected-experiment-that-taught-the-most)).

## 6. Agent coordination, the current frontier

Start with [research/](research/README.md), which sorts the literature into three layers, then
read in the order given in
[shared-state-concurrency](research/shared-state-concurrency.md#go-deeper).

**Run:** reproduce stale-generation in two terminals
([shared-state-concurrency](research/shared-state-concurrency.md#see-it-yourself)), then extend it
to the cross-object case that most systems fail.

**Explain after:** why several 2026 groups independently concluded that a tool returning is not
settlement, and what that implies for anything calling itself an agent framework.

## Questions worth asking of any claim

Applies to this pack too. The full standard is [07-measuring-it.md](07-measuring-it.md).

| Ask | Because |
| --- | --- |
| What is the unit, and how many operations per transaction? | batching moves a headline number by an order of magnitude |
| Open loop or closed loop? | a closed-loop number says nothing past the knee |
| Was drain measured, or only throughput? | a stable rate with growing backlog is a failing system that reports as healthy |
| How many repetitions, decided in advance? | otherwise the best of N is being reported |
| What was protected from regressing? | a claim with no protected metric traded something away silently |
| What did it cost to run? | capacity without cost is not a service |

Then [measure the noise floor](research/evaluation-and-failure.md#see-it-yourself) of whatever
harness produced the claim. Most reported coordination gains do not clear it.

## Where the open problems are

Each file ends with a **Still open** section. The largest ones, if you want to go straight to
the frontier:

| Problem | File |
| --- | --- |
| Which operations genuinely need a total order | [ordering-and-frontiers](domains/ordering-and-frontiers.md) |
| Sustained provider outage envelopes for logical replication | [logical-decoding-and-cdc](domains/logical-decoding-and-cdc.md) |
| Whether a vertical split survives the read contract | [postgres-under-hot-updates](domains/postgres-under-hot-updates.md) |
| Premises attached to the read object rather than the write target | [shared-state-concurrency](research/shared-state-concurrency.md) |
| What a guarantee means in a workspace that never quiesces | [shared-state-concurrency](research/shared-state-concurrency.md) |
| The noise floor of any coordination benchmark | [evaluation-and-failure](research/evaluation-and-failure.md) |
