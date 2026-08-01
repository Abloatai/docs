# 01. The space Ablo is trying to take

## The shift that creates the problem

Software is moving from passive tools operated by humans toward active software actors. One
business process may now involve a human changing a plan, an agent scheduling work against it,
a service applying a policy, a robot moving an object, a sensor contradicting all of them, and
a second organisation confirming or rejecting the result.

Each actor holds a partial and possibly stale view. Each may retry. Several act concurrently.
Authority gets delegated and attenuated along the way, and the process routinely crosses an
administrative boundary. A database transaction is necessary for all of this and sufficient for
none of it. It does not answer:

- Was this actor authorised for this exact action at this exact time?
- Was this a retry, or a second intentional action?
- Did it conflict with a newer owner or lease?
- Which transition committed, and which was declined?
- Has the transition merely been accepted, or has the authoritative source confirmed it?
- Which observers have caught up to that confirmation?
- What caused a physical side effect that cannot be rolled back?
- Which organisation can prove what it saw and what it accepted?

Ablo's intended role is to answer those questions at a neutral boundary, without requiring every
participant to share an AI model, an agent framework, an application stack or a database vendor.

## What "state of the world" is allowed to mean

It cannot mean perfect instantaneous knowledge of physical reality. The defensible version is
narrower:

> Ablo maintains the best authorised, time-bounded, attributable and causally interpretable
> account of the state that participating systems have committed or observed.

For digital state, a committed row and its WAL record can be the settlement boundary. For
physical state, desired, commanded, acknowledged and observed have to stay four distinct facts
rather than collapsing into one mutable boolean. That model, and what it costs to maintain, is
in [domains/physical-world-state.md](domains/physical-world-state.md).

## Who else is in this space, and where they stop

None of these systems is a competitor across the whole surface. Each owns one part of the
problem well, and the boundary of what it declines to own is the interesting part. Sources for
every claim below are in [09-reading-list.md](09-reading-list.md).

| System | Owns | Explicitly does not own |
| --- | --- | --- |
| **PostgreSQL** | atomic row changes, constraints, isolation, durable WAL | who was allowed to ask, what a subscriber has applied |
| **ElectricSQL** | the read path out of Postgres into clients, as cacheable HTTP shape logs | the write path. Electric states plainly that syncing data back into Postgres is the application's problem |
| **Debezium** | extracting committed changes as an event stream, with slot and failover handling | authorisation, conflict resolution, receipts. CDC observes a write that already happened |
| **Kafka** | durable replay, partition-local order, consumer offsets, compaction | that a broker record corresponds to a database transaction or an authorised action |
| **Materialize** | incrementally maintained SQL views over a CDC stream, with a progress-carrying subscription | the write path and the authority model |
| **Temporal** | durable execution of *your* code across failures, with replay and retries | shared state between independent parties. Activities are at-least-once, so idempotency is yours to build |
| **TigerBeetle** | a purpose-built transaction state machine at very high rates | generality. It owns its storage and its schema, which is exactly why it is fast |
| **FoundationDB, Spanner** | strict transactional guarantees, with the machinery on display | working against a database the customer already owns and operates |
| **CRDTs, local-first** | offline edits that deterministically converge | non-commutative invariants such as "one holder at a time" or "balance may not go negative" |
| **MQTT, ROS 2 / DDS** | device transport with real delivery, deadline and liveliness policies | organisational authority. A QoS acknowledgement is not a completed physical action |

The pattern is consistent. The read-path products stop before the write. The write-path
products own the storage. The transport products deliver bytes and make no claim about
authority. The convergent products converge without deciding who was allowed to act.

Ablo's bet is that the composition is the product: customer-owned storage, mediated authority,
WAL-derived settlement, claims and fencing, replayable publication, and an explicit gap between
what was committed and what was observed.

The research side of the same space moved quickly in 2026. Several independent groups arrived
at Ablo's distinctions from the agent direction: that a tool returning is not settlement, that
progress belongs per resource, that irreversible effects need their own path, and that agents
mutating shared state are doing concurrency control whether or not anyone designed it. That work
is in [research/](research/README.md), and it is the part of this pack most likely to be out of
date first.

## The commercial shape of the neighbourhood

Electric Cloud's published pricing is a useful external anchor for what this space is worth
today, because it prices the exact read path Ablo also runs: one dollar per million writes
(chunked at 10KB), ten cents per GB-month of retention, two dollars per million writes emitted
into the shape log from the replication stream, and nothing at all for reads, fan-out,
concurrent users or connections.

Two things follow. First, the market has already decided that fan-out is not separately
billable, which pushes the cost of fan-out onto the vendor's architecture rather than the
customer's invoice. That is a direct argument for the subscription-aware fan-out work in
[domains/fanout-and-incremental-views.md](domains/fanout-and-incremental-views.md). Second,
the billable unit is the write. Ablo's unit is a mediated, authorised, confirmed write, which
is strictly more work per unit than an observed one. Whether that difference is defensible at
the same order of price is an open commercial question, not a settled one.

## Questions this raises

- If the read path is table stakes and priced near zero, is the mediated write the only part a
  customer would pay a premium for? What evidence exists either way?
- Electric declines the write path deliberately. Is that a scoping choice or a statement about
  where the hard problems are? What does Ablo know that makes taking it worthwhile?
- Which of the systems above would a well-resourced competitor extend into Ablo's position
  fastest, and what part of the composition is hardest to copy?
- Is "the composition is the product" a moat or a description of an integration burden? Which
  specific customer problem is impossible to solve by wiring three of these together by hand?
- Does an agent-heavy workload change the answer? Agents retry more, hold context longer, and
  act concurrently more often than humans do. Which of the guarantees above becomes load-bearing
  only when the actors are software?
