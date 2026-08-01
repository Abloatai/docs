# 08. The question bank

Each domain file and each research file ends with its own questions. This file holds the
cross-cutting ones, grouped by who can answer them. Ask the first group of a person; answer the second group with a
measurement; treat the third as unresolved design; treat the fourth as unresolved strategy.

## A. Questions answerable from the code today

Ask these of the team or of the repository. Every one has a definite answer right now, and each
answer constrains what is worth proposing next.

**Write path**

- What runs between the socket read and the Postgres commit, in order, and what is the cost of
  each step at the measured rate?
- Where is the idempotency key derived, stored, and expired?
- What exactly does a client receive when a claim guard rejects a commit, and what does a
  retrying agent do with it?
- Which parts of a commit are set-based today, and which still iterate per row?

**Read path**

- What decides a publication group?
- How is a subscription matched against a delta, and what does that cost per subscriber?
- What is persisted on the client alongside the cursor, and is that write atomic with the state
  it describes?

**Recovery**

- Which failure injections exist in the test suite, and which failure modes have only ever been
  reasoned about?
- What is the exact sequence when a decoder loses its lease mid-transaction?
- What is recorded durably before an acknowledgement is sent back to Postgres?

**Boundaries**

- Which types are parsed at a boundary and which are re-parsed inside? The repository treats a
  second parse as a defect, so any that remain are worth knowing about.
- Where does the wire contract live, and what enforces that documentation derives from it rather
  than describing it?

## B. Questions to ask of any performance claim

Including every number in [04-the-evidence.md](04-the-evidence.md). The full standard is in
[07-measuring-it.md](07-measuring-it.md).

- What is the unit, and how many operations are in one transaction?
- Open loop or closed loop? If closed, the throughput number is a lower bound on capacity and
  says nothing about behaviour past the knee.
- Was drain measured, or only throughput? A stable rate with growing backlog is a failing system
  that reports as healthy.
- How many repetitions, decided in advance, and what was the variance?
- What did not regress? A claim with no protected metric is a claim that something was traded
  away silently.
- Did the correctness gates run in the same execution, or afterwards on a clean run?
- What did it cost to run, and what does the improvement cost per million delivered deltas?

## C. Open design decisions

Unresolved, and legitimately so: different customer workloads would justify different answers.
The detailed sub-questions live in the domain files.

| Decision | Depends on | Detail |
| --- | --- | --- |
| Scalar cursor or frontier | which operations truly need total order, and the real conflict rate | [ordering-and-frontiers](domains/ordering-and-frontiers.md) |
| One decoder per source, or decode per server | subscriber count, transform cost, and the acknowledgement boundary | [logical-decoding-and-cdc](domains/logical-decoding-and-cdc.md) |
| One stream or two, event-complete and state-convergent | whether auditors and observers are actually the same consumer | [fanout-and-incremental-views](domains/fanout-and-incremental-views.md) |
| Vertical split of the hot relation | whether read regression is acceptable, and whether Ablo can propose schema to customers | [postgres-under-hot-updates](domains/postgres-under-hot-updates.md) |
| Typed binary internal record | where the remaining stage cost actually is | [representation-and-memory](domains/representation-and-memory.md) |
| Overload policy per tenant | who absorbs delay, stated explicitly rather than by accident | [backpressure-and-overload](domains/backpressure-and-overload.md) |
| Signed receipts and cross-organisation proof | whether a real counterparty needs it yet | [capabilities-and-evidence](domains/capabilities-and-evidence.md) |
| Physical actuation as a first-class transaction | whether there is a customer or only a direction | [physical-world-state](domains/physical-world-state.md) |
| Declared footprints or inferred read sets | whether agents can be required to declare anything | [shared-state-concurrency](research/shared-state-concurrency.md#s-bus-read-sets-without-agent-cooperation) |
| Mediated writes or branch and merge | whether the customer's effects are live or forkable | [shared-state-concurrency](research/shared-state-concurrency.md#storm-mediated-writes-against-branch-and-merge) |

## D. Questions about the space

- What does the first customer with a genuinely hot plane look like, and does that customer
  exist yet?
- Is the mediated write the part customers pay for, when the read path next door is priced near
  zero and fan-out is free? See
  [01-the-space.md](01-the-space.md#the-commercial-shape-of-the-neighbourhood).
- Which capability would make a customer choose Ablo over assembling Debezium, Kafka and their
  own authorisation layer? Name it in one sentence, then check whether the current roadmap
  builds it.
- Agents retry more, hold context longer, and act concurrently more often than humans. Which
  guarantee becomes load-bearing only when the actors are software, and is that the wedge?
- What would have to be true for the physical-world direction to be a product rather than a
  thesis, and what is the cheapest experiment that tests it?
- Where is the line between what Ablo builds and what it delegates to the neighbours in
  [01-the-space.md](01-the-space.md#who-else-is-in-this-space-and-where-they-stop)? Every
  additional responsibility taken on is one more thing to be world-class at.
