# 01. The space Ablo is trying to take

The longer bet is [00-the-vision.md](00-the-vision.md). This file is the near-term version:
what Ablo claims today, and who else already occupies part of it.

## The shift that creates the problem

Software is moving from passive tools operated by humans toward active software actors. One
business process may now involve a human changing a plan, an agent scheduling work against it,
a service applying a policy, a robot moving an object, a sensor contradicting all of them, and
a second organisation confirming or rejecting the result.

Each actor holds a partial and possibly stale view. Each may retry. Several act concurrently.
Authority gets delegated and narrowed along the way, and the process routinely crosses an
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
rather than collapsing into one mutable boolean.

That is the next direction rather than a distant one. An agent acting on a warehouse, a vehicle
or a building needs to know what is true about a physical object right now, how stale that
reading is, whether it is allowed to change it, and whether its last command landed. Today that
state lives inside device protocols and fleet controllers, which move messages reliably and say
nothing about authority or settlement. The full model, the standards that already exist, and
what they leave open is [domains/physical-world-state.md](domains/physical-world-state.md).

## Who else is in this space, and where they stop

None of these systems is a competitor across the whole surface. Each owns one part of the
problem well, and the boundary of what it declines to own is the interesting part. Sources for
every claim below are in [09-reading-list.md](09-reading-list.md).

| System | Owns | Explicitly does not own |
| --- | --- | --- |
| **[PostgreSQL](https://www.postgresql.org/docs/current/mvcc.html)** | atomic row changes, constraints, isolation, durable WAL | who was allowed to ask, what a subscriber has applied |
| **[ElectricSQL](https://electric.ax/docs/sync/api/http)** | the read path out of Postgres into clients, as cacheable HTTP shape logs | the write path. Electric states plainly that syncing data back into Postgres is the application's problem |
| **[Debezium](https://debezium.io/documentation/reference/3.6/connectors/postgresql.html)** | extracting committed changes as an event stream, with slot and failover handling | authorisation, conflict resolution, receipts. CDC observes a write that already happened |
| **[Kafka](https://kafka.apache.org/documentation/#design)** | durable replay, partition-local order, consumer offsets, compaction | that a broker record corresponds to a database transaction or an authorised action |
| **[Materialize](https://materialize.com/docs/sql/subscribe/)** | incrementally maintained SQL views over a CDC stream, with a progress-carrying subscription | the write path and the authority model |
| **[Temporal](https://docs.temporal.io/activity-definition)** | durable execution of *your* code across failures, with replay and retries | shared state between independent parties. Activities are at-least-once, so idempotency is yours to build |
| **[TigerBeetle](https://docs.tigerbeetle.com/coding/requests)** | a purpose-built transaction state machine at very high rates | generality. It owns its storage and its schema, which is exactly why it is fast |
| **[FoundationDB](https://apple.github.io/foundationdb/architecture.html), [Spanner](https://research.google.com/archive/spanner-osdi2012.pdf)** | strict transactional guarantees, with the machinery on display | working against a database the customer already owns and operates |
| **[CRDTs](https://people.eecs.berkeley.edu/~kubitron/courses/cs262a-F21/handouts/papers/Shapiro-CRDT.pdf), [local-first](https://www.inkandswitch.com/essay/local-first/)** | offline edits that deterministically converge | non-commutative invariants such as "one holder at a time" or "balance may not go negative" |
| **[MQTT](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html), [ROS 2](https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Quality-of-Service-Settings.html)** | device transport with real delivery, deadline and liveliness policies | organisational authority. A QoS acknowledgement is not a completed physical action |

The pattern is consistent. The read-path products stop before the write. The write-path
products own the storage. The transport products deliver bytes and make no claim about
authority. The convergent products converge without deciding who was allowed to act.

One thing every actor has to share before any of this works: a vocabulary. Ablo coordinates
changes to declared things, a document, an order, a button, an aircraft, and it takes that
vocabulary from the organisation rather than imposing one, because a task in a hospital is not a
task in a factory. [Palantir's Ontology](https://www.palantir.com/docs/foundry/architecture-center/ontology-system)
reached the same conclusion from the enterprise direction. See
[domains/ontology-and-schema.md](domains/ontology-and-schema.md).

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

[Electric Cloud's pricing](https://electric.ax/blog/2026/04/02/electric-cloud-pricing) is the
useful external anchor, because it prices the exact read path Ablo also runs.

| Unit | Price |
| --- | --- |
| Writes | $1 per million, chunked at 10 KB |
| Retention | $0.10 per GB-month |
| Writes emitted to the shape log | $2 per million |
| Reads, fan-out, concurrent users, connections | free |

Two things follow. The market has already decided fan-out is not separately billable, which puts
its cost on the vendor's architecture rather than the customer's invoice, and that is the
argument for [subscription-aware fan-out](domains/fanout-and-incremental-views.md). And the
billable unit is the write, where Ablo's unit is a mediated, authorised, confirmed write:
strictly more work per unit than an observed one. Whether that is defensible at the same order
of price is open.

## Still open

- Whether the mediated write is the part a customer pays a premium for, when the read path next
  door is priced near zero and fan-out is free.
- Which neighbouring system extends into this position fastest, and which part of the
  composition is hardest to copy.
- Which guarantee becomes load-bearing only when the actors are software rather than people.
  Agents retry more, hold context longer, and act concurrently more often.
