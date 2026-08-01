# Subscription-aware fan-out and incremental views

## The problem

Full-stream fan-out grows as event rate multiplied by subscriber count, and becomes impossible
well before world scale. The arithmetic is in
[06-scale-regimes.md](../06-scale-regimes.md#target-3-delivered-and-applied-throughput): one
hundred full-stream observers at a million deltas per second is roughly 27 GB/sec of payload
before any framing.

The commercial version of the same problem is sharper. The market price for fan-out is
[currently zero](../01-the-space.md#the-commercial-shape-of-the-neighbourhood), so every byte
sent to a subscriber who did not need it is margin, not revenue.

## What the field knows

**Two stream products, not one.** Kafka's compaction contract is the cleanest existing statement
of the split: a consumer caught up to the log head sees every message, while a consumer reading
a compacted log from the beginning sees only the final state per key. Ablo needs the same two
products with the difference made explicit rather than emergent:

- an **event-complete stream** for processors and auditors that require every committed
  transition
- a **state-convergent stream** for observers that require current state and may skip superseded
  intermediate values

They have different throughput, retention and correctness contracts. An auditor served from a
coalescing stream is being quietly lied to.

**Incremental view maintenance is a shipped product category.** Materialize maintains SQL views
over a Postgres CDC stream and pushes changes as diffs rather than recomputing, and its
subscription carries progress information alongside the data. Differential Dataflow is the
underlying idea: treat changing collections as inputs to an incrementally maintained
computation. The vocabulary it provides is worth borrowing even where the machinery is not:
data change, logical time, maintained result, and progress are four separate things.

**Fan-out can be pushed out of the server entirely.** ElectricSQL serves shapes as plain HTTP
with offsets, which makes the read path cacheable by ordinary CDN infrastructure and is why its
pricing can treat reads, fan-out, concurrent users and connections as free. Any architecture
that holds a stateful socket per observer is choosing to pay a cost that a log-plus-offset
design can hand to a cache.

## The design dimensions

Compiled dependency indexes can decide which materialised shapes a delta may affect without
re-evaluating every query. The dimensions that decide whether that works: safe dependency
signatures for queries, indexing of aliases, joins, ranges and large membership sets, accepting
false-positive routing to avoid false negatives, coalescing of superseded state, distinguishing
event-complete from current-state consumers, and ordering subscription-index changes correctly
against data changes.

Likely required mechanisms at scale: decode once per source rather than once per server, route
only to subscribers whose materialised shape may change, publish compact field masks where
semantics permit, retain the audit stream separately from state-materialisation feeds, push
regional fan-out toward the edge, and serve new or badly lagged consumers a snapshot plus a
recent delta tail.

## Questions this raises

- Does Ablo have the two stream products explicitly today, or does one stream serve both
  audiences by accident?
- What is the current routing cost per delta per subscriber, and how does it scale with
  subscriber count?
- Which parts of the read path could be served over cacheable HTTP rather than a live socket,
  and what is lost by doing so?
- How is a subscription index kept correct while the schema changes underneath it?
- What is the false-positive rate of subscription matching today, and is it measured?
- If an agent subscribes to a broad shape it barely reads, who pays: the customer, the
  architecture, or the margin?
- What is the cost of the audit stream if it must retain every transition while the state stream
  coalesces?
