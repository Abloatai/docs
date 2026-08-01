# 05. Why this is a hard systems problem

## An update is not one operation

One logical update causes work in a dozen places:

authentication and schema validation, capability verification, tenant and branch routing, claim
or fence lookup, application-row mutation, secondary-index maintenance, sync-log append, audit
append or outbox work, transaction commit and WAL generation, logical decoding, row-to-domain
projection, subscription and group lookup, worker-host transfer, serialisation, socket enqueue,
then client queueing, local persistence, materialisation and cursor advancement.

The arithmetic is unforgiving at rate. At 100,000 deltas per second:

- one microsecond of CPU per delta is 10 percent of a core, permanently
- one copied 300-byte object per delta is 30 MB/sec of memory traffic
- one global lookup or lock per delta is a serialisation point, not an overhead

This is why [04-the-evidence.md](04-the-evidence.md#the-most-recent-retained-result) treats a
change in bytes-per-delta as a first-class result. At this rate, representation is architecture.

## Ordering creates a serial fraction

If every delta in a plane needs one total order and one scalar cursor, something must decide
that order. Database commits can execute concurrently and the observer still has to publish a
sequence that clients replay without ambiguity.

Amdahl's law applies directly. If fraction `s` of the work stays serial, the ceiling on speedup
from any number of parallel workers is:

```text
speedup <= 1 / s
```

A 5 percent serial fraction caps the ideal speedup at 20 times regardless of how many workers
are added. Worse, the serial fraction tends to grow under load, because workers start
contending for the same lock, queue, cache line, credit ledger, checkpoint or socket dispatcher.

The research question is not "how do we parallelise this code". It is which ordering is
semantically necessary, and how an observer expresses a complete applied position when
independent partitions progress at different rates. That is
[domains/ordering-and-frontiers.md](domains/ordering-and-frontiers.md).

## Exactly once is an end-to-end property

No transport prevents a client from seeing a retry after a crash. The guarantee has to be
constructed from idempotency keys at commit, durable transaction identifiers, deterministic
delta identity, replay from a durable log, monotonic cursor advancement, consumer-side
deduplication, and acknowledgement persisted only after application.

The neighbouring systems make the same admission in their own vocabulary, and they are worth
reading precisely because they do not pretend otherwise:

- **Flink** provides exactly-once operator state through checkpoints, but end-to-end delivery
  requires the *sink* to participate. Its JDBC exactly-once sink runs on XA transactions and
  requires retries to be disabled; the general pattern is a two-phase commit sink that only
  commits external effects when a checkpoint completes.
- **Temporal** does not claim exactly-once activity execution at all. A worker can complete an
  activity and crash before reporting it, so the activity is retried. The documented remedy is
  to design activities to be idempotent, using an idempotency key derived from the workflow run
  and activity identity.

Both say the same thing Ablo has to say: the guarantee lives at the boundary where an effect
becomes durable, and it is built, not switched on. Optimising one stage without modelling the
crash boundary can open a narrow data-loss window that steady-state tests never show.

## Backpressure crosses administrative boundaries

A database commits faster than one subscriber consumes. One client has a bad network. One plane
runs hot enough to starve another. An external provider retains WAL until its disk is at risk.

So the system needs explicit policy for bounded in-memory queues, durable spill or replay,
per-plane and per-subscriber credits, lag limits and disconnection, admission control at the
commit boundary, priority and fairness, recovery after overload, and preventing one customer's
slowness from becoming another customer's latency.

Unlimited queues do not produce reliability. They produce delayed failure, and they hide it
behind a healthy-looking average throughput number. See
[domains/backpressure-and-overload.md](domains/backpressure-and-overload.md), and runs 124 to
128 in [04](04-the-evidence.md#the-rejected-experiment-that-taught-the-most) for the local proof.

## Physical effects are not database transactions

A database update rolls back. A robot has already moved, a door has already unlocked, a message
has already reached a counterparty. This is the classic dual-write problem with safety and
authority attached, and it needs command intent separated from acknowledgement, transactional
outbox and idempotent actuator commands, leases and fences on control ownership, compensating
actions instead of fictional rollback, reconciliation between desired and observed state,
explicit uncertainty, and safety policy that stays local when the cloud is unreachable.

Detail in [domains/physical-world-state.md](domains/physical-world-state.md).

## Cost is a correctness-adjacent constraint

Throughput without cost is not a service. A design reaching one million per second on ten times
the compute may be commercially worse than a 300,000 per second design with predictable
horizontal scale, particularly in a market where
[fan-out is not separately billable](01-the-space.md#the-commercial-shape-of-the-neighbourhood).
Cost per useful delivered transition belongs in the same table as latency.

## Questions this raises

- What is the current serial fraction, measured rather than estimated? Which stage owns it?
- Which stages are single-threaded by necessity, and which are single-threaded by history?
- Where exactly is the crash boundary on the write path, and what is the largest window in
  which a crash produces a duplicate the client must deduplicate?
- If a subscriber is slow for ten minutes, what does the system do at minute one, minute five
  and minute ten? Is that policy stated anywhere a customer can read it?
- Which of the invariants in [02](02-the-contract.md#invariants) would break first if offered
  load doubled without warning?
- What does the system do when the customer's database, not Ablo, is the slow component?
