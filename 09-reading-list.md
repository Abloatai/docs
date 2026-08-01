# 09. Reading list

Every external source used in this pack, with one line on why it is worth the time. Other files
cite specific facts and link the source directly; this is the only place the sources are
described. Nothing here implies Ablo should reproduce the design.

Documentation links for PostgreSQL, Debezium, Kafka, Materialize, Flink, TigerBeetle, Temporal
and ElectricSQL were checked against current published docs on 2026-08-01. Papers and
specifications are stable and were not.

## PostgreSQL storage and change capture

- [MVCC](https://www.postgresql.org/docs/current/mvcc.html): transaction visibility, snapshots
  and isolation, the foundation everything else in this section sits on.
- [Heap-only tuple updates](https://www.postgresql.org/docs/current/storage-hot.html): the two
  conditions under which an update skips index maintenance, and why an index on a mutable column
  is expensive.
- [Introduction to indexes](https://www.postgresql.org/docs/current/indexes-intro.html): the
  read benefit and the write cost stated together, including the effect on heap-only tuples.
- [Routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html): dead
  tuple reclamation, statistics, and wraparound protection. The bill that arrives late.
- [Logical decoding concepts](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html):
  how WAL becomes application-level changes and what a replication slot retains.
- [pg_replication_slots](https://www.postgresql.org/docs/current/view-pg-replication-slots.html):
  `safe_wal_size` and `invalidation_reason`, the two columns that turn replication lag from a
  guess into a measurement.
- [Replication configuration](https://www.postgresql.org/docs/current/runtime-config-replication.html):
  `wal_keep_size` and `max_slot_wal_keep_size`, the retention floor and the disk-exhaustion cap.
- [Logical replication failover](https://www.postgresql.org/docs/current/logical-replication-failover.html):
  PostgreSQL 17 failover slots, and the readiness check that has to pass before a failover is
  actually safe.
- [Debezium PostgreSQL connector](https://debezium.io/documentation/reference/3.6/connectors/postgresql.html):
  the production CDC comparison, including `slot.failover` defaulting to false and being ignored
  on PostgreSQL 16 and earlier.

## Transactions and consistency

- [Spanner](https://research.google.com/archive/spanner-osdi2012.pdf): what external consistency
  actually costs, including synchronous replication and an API that exposes clock uncertainty. A
  warning against calling a regional stream global truth.
- [Calvin](https://www.cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf): deterministic
  ordering of distributed transactions, the opposite corner from optimistic execution.
- [FoundationDB architecture](https://apple.github.io/foundationdb/architecture.html): roles
  separated into read-version assignment, commit proxies, resolvers, logs and storage, with
  epoch-based recovery.
- [FoundationDB paper](https://www.foundationdb.org/files/fdb-paper.pdf): the same system plus
  deterministic simulation of disk, process, network and request failure.
- [TigerBeetle documentation](https://docs.tigerbeetle.com/coding/requests): batching up to 8,189
  events per request to amortise consensus and I/O, and shrinking batches under light load.
  The clearest statement anywhere of how a headline throughput number is constructed.
- [Jepsen consistency models](https://jepsen.io/consistency/models): a practical map of
  linearizability, serializability, causal consistency and what each implies for availability.

## Logs, streaming and incremental computation

- [Kafka design](https://kafka.apache.org/documentation/#design): order and offsets within a
  partition and not across, key-to-partition hashing, and log compaction. Also the cleanest
  existing statement of the difference between a consumer that sees every message and one that
  sees final state per key.
- [Flink checkpointing](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/fault-tolerance/checkpointing/):
  state snapshots, replay, and exactly-once operator state.
- [Flink connector guarantees](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/connectors/datastream/guarantees/):
  end-to-end exactly-once requires the sink to participate, via two-phase commit or XA. Read
  alongside the checkpointing docs, which are frequently over-read as an end-to-end guarantee.
- [Materialize SUBSCRIBE](https://materialize.com/docs/sql/subscribe/): incremental diffs with
  `mz_diff`, and `PROGRESS` emitting an explicit frontier. A frontier in a shipped SQL interface.
- [Materialize durable subscriptions](https://materialize.com/docs/transform-data/patterns/durable-subscriptions/):
  `RETAIN HISTORY` so a disconnected client resumes rather than resnapshots.
- [Naiad](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/11/naiad_sosp2013.pdf):
  logical timestamps and frontiers for distributed progress tracking.
- [Differential Dataflow](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/01/differentialdataflow.pdf):
  incremental maintenance of changing and iterative computations.

## Sync engines and local-first

- [ElectricSQL HTTP API](https://electric.ax/docs/sync/api/http): shapes, offsets, and the
  `up-to-date`, `snapshot-end` and `must-refetch` control messages. The closest published
  analogue to Ablo's read path.
- [Electric on the write path](https://electric.ax/blog/2024/11/21/local-first-with-your-existing-api):
  the explicit statement that Electric does the read path and leaves writes to the application.
- [Electric Cloud pricing](https://electric.ax/blog/2026/04/02/electric-cloud-pricing): writes
  and retention billed, reads and fan-out free. The market's current opinion on what this layer
  is worth.
- [CRDTs](https://people.eecs.berkeley.edu/~kubitron/courses/cs262a-F21/handouts/papers/Shapiro-CRDT.pdf):
  state-based and operation-based convergence conditions, and by omission, what convergence does
  not give you.
- [Local-First Software](https://www.inkandswitch.com/essay/local-first/): ownership, offline
  operation and collaboration framed as product properties rather than sync mechanics.

## Durable execution

- [Temporal activity definition](https://docs.temporal.io/activity-definition): activities are
  retried when completion is not reported, so idempotency is the application's job. The most
  honest short statement of at-least-once in a system that is often assumed to be exactly-once.
- [Temporal error handling](https://docs.temporal.io/best-practices/error-handling): idempotency
  keys derived from run and activity identity.

## Delegated authorisation

- [Macaroons](https://research.google/pubs/macaroons-cookies-with-contextual-caveats-for-decentralized-authorization-in-the-cloud/):
  decentralised delegation through contextual caveats appended by the holder.
- [Biscuit specification](https://doc.biscuitsec.org/reference/specifications.html): public-key
  verifiable tokens with append-only Datalog attenuation.
- [Biscuit revocation](https://www.biscuitsec.org/docs/guides/revocation/): why offline
  attenuation does not remove the need for revocation state.

## Devices and robotics

- [MQTT 5.0](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html): publish and subscribe
  with three QoS levels, retained messages, sessions and retransmission.
- [ROS 2 quality of service](https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Quality-of-Service-Settings.html):
  history, reliability, durability, deadline, lifespan and liveliness as separate knobs. A richer
  vocabulary for delivery expectations than most cloud systems have.

## Agent coordination research

The multi-agent literature is cited in [research/](research/README.md) rather than here,
because those papers are arguments Ablo has to answer rather than references it draws on. Each
one is listed there with its publication status, since most of the relevant work is from the
last six months and is still preprint.

## Measurement

- [Little's law](https://pubsonline.informs.org/doi/10.1287/opre.9.3.383): the relationship
  between arrival rate, in-system work and residence time. Explains drain.
- [How Not to Measure Latency](https://qconsf.com/sf2012/dl/qcon-sanfran-2012/slides/GilTene_HowNotToMeasureLatency.pdf):
  coordinated omission, and why a stalled system reports excellent percentiles.
- [OpenTelemetry messaging conventions](https://opentelemetry.io/docs/specs/semconv/messaging/):
  shared vocabulary for tracing producer, consumer and processing boundaries.
