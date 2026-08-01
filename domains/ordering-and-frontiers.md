# Ordering, partitioning and progress frontiers

## The problem

One total order per plane makes unrelated updates contend, and lets every consumer describe its
progress with one integer. Partitioned order exposes parallelism, and replaces that integer with
a set of per-partition positions. This is the central semantic choice in front of Ablo, and it
is client-visible: the cursor is part of the public contract, so changing it is a protocol
change, not a refactor.

Why the scalar order caps throughput is in
[05-why-it-is-hard.md](../05-why-it-is-hard.md#ordering-creates-a-serial-fraction).

## What the field knows

**Kafka made the tradeoff explicit and shipped it.** Ordering and a scalar offset exist within a
partition, never across partitions. Keys map to partitions by a Murmur2 hash of the serialised
key modulo the partition count, so the same key always lands in the same partition. The offset
is a permanent identifier: compaction never changes it, and never reorders retained records.

Compaction is also the cleanest existing statement of a distinction Ablo needs. Kafka guarantees
that a consumer staying caught up to the log head sees *every* message, while a consumer
starting from the beginning of a compacted log sees the *final state* of every key plus delete
markers within the retention window. Those are two different products served from one log:
event-complete for processors that need every transition, state-convergent for observers that
need current state. The retention and correctness contracts differ, and conflating them is a
common source of silent data loss in downstream systems.

**Frontiers are a shipped idea, not only a paper one.** Materialize's `SUBSCRIBE` emits
`mz_timestamp` and `mz_diff` per row, and with `PROGRESS` it emits `mz_progressed`, an explicit
statement that no further records will arrive at times strictly less than that timestamp. That
is a frontier in a production SQL interface. Its durable-subscription pattern pairs it with
`RETAIN HISTORY`, so a disconnected client can resume from its last timestamp instead of
restarting from a snapshot, which is precisely the reconnect problem regime B creates.

Naiad and timely dataflow are the formal treatment of the same idea, and Calvin is the opposite
corner of the design space: order transactions deterministically up front and avoid distributed
commit during execution.

## The candidate models for Ablo

Entity or key-range lanes, capability-scope-derived lanes, claim-domain lanes, static shape
partitions, deterministic multi-lane barriers, interval-tree or compressed vector frontiers, and
epoch-based split and merge of hot lanes. The regime B sketch is in
[06-scale-regimes.md](../06-scale-regimes.md#regime-b-one-hot-plane-at-one-million).

Conflict rate is the hidden variable that decides whether any of it works. Sparse, well-declared
dependencies favour partitioned progress. One hot key, or frequent cross-partition transactions,
collapses the design back into global coordination while keeping all the added complexity.

## Questions this raises

- Which Ablo operations genuinely require a total order? Write the list. If it is short, the
  frontier work is justified; if it is long, it is not.
- Claims already define exclusive domains. Could a claim scope serve directly as a lane
  assignment, and what breaks when a transaction spans two claims?
- Can a mutation declare its coordination keys at the schema level, the way a partition key is
  declared, rather than being inferred from a read set at runtime?
- What is the measured conflict rate in a realistic agent workload? Nobody should design lanes
  before knowing it.
- Could a scalar-cursor client be served from a merged view of lanes without losing exactness,
  and what would that merge cost per delta?
- What is the equivalent of `RETAIN HISTORY` in Ablo, and how long is it? That number decides
  whether a reconnecting client resumes or resnapshots.
- If a frontier becomes the cursor, what is the largest it can get, and how is it persisted
  atomically alongside client state?
