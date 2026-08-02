# 00. Why this matters

The engineering in this pack is in service of one bet: that within a decade the majority of
consequential state changes in the economy will be made by software actors rather than people,
that those actors will need to act on the same things at the same time, and that the record of
who was allowed to act and what actually settled becomes infrastructure rather than a feature.

## What changes when the actors are software

| | People coordinating | Software actors coordinating |
| --- | --- | --- |
| Rate of change | a few edits an hour | continuous |
| Reaction to a conflict | notice, ask, adjust | retry immediately, often harder |
| View of the world | fresh enough, because they just looked | read once, reason for thirty seconds, write into a world that moved |
| Number of concurrent actors | a team | unbounded |
| Crossing a company boundary | email, a call, a contract | nothing standard exists |

Humans absorb coordination failure invisibly. Somebody notices the document changed, asks a
colleague, waits their turn. Software actors do not notice, do not ask, and do not wait, so what
was social becomes a systems problem the moment the actor is a program.

## The scale that arrives with it

The arithmetic is more sobering than it first looks, and it is worth doing before designing
anything.

| Population | Acting once every 10 seconds | Deltas per second |
| --- | ---: | ---: |
| 10,000 agents | | 1,000 |
| 100,000 agents | | 10,000 |
| 1,000,000 agents | | **100,000** |

One million software actors, each making one consequential change every ten seconds, is exactly
the [100,000 deltas per second gate](04-the-evidence.md#the-gate) the engine is measured
against today. That is not a coincidence anyone should find comforting. It means the current
benchmark buys one million agents at a leisurely pace, and nothing more.

The physical side already exists at that order. The IFR recorded a
[record 4 million industrial robots operational in factories worldwide](https://ifr.org/ifr-press-releases/news/record-of-4-million-robots-working-in-factories-worldwide),
with [542,000 installed in 2024 and roughly 575,000 expected in 2025](https://ifr.org/ifr-press-releases/news/global-robot-demand-in-factories-doubles-over-10-years),
double the annual rate of a decade ago. Those machines are not waiting for a coordination layer
to be invented, and they already report state continuously.

A distinction the arithmetic forces: not every physical reading is a transaction. Four million
robots emitting position at 1 Hz is four million messages per second, and almost none of it is
consequential. The coordination layer takes the transitions that carry authority and
consequence, and leaves telemetry to the transports built for it. Deciding which is which is a
design problem nobody has solved cleanly
([physical-world-state](domains/physical-world-state.md)).

## State becomes the primary artifact

The current generation of agent infrastructure treats the **message** as primary: a conversation,
a tool call, a trace. That is a reasonable place to start and it does not survive scale, for one
reason.

Messages are derivable from state. State is not derivable from messages.

Given the ordered record of what changed, who was allowed to change it, and what the
authoritative source confirmed, you can reconstruct any narrative a reader needs. Given a
transcript of a thousand agents talking, you cannot reconstruct what is true, because nothing in
it distinguishes a proposal from a commitment, or an intent from an outcome. Systems that
recover by replaying chat history are rebuilding the world from its shadow.

So the artifact worth being the system of record for is the state and its provenance. Everything
else is a view over it.

## Across organisations

The interesting jump is not one company's agents. It is two companies whose agents transact
without sharing a database, a vendor, or a trust root.

The layer that does this for humans today is email: universal and neutral, which is why it won,
and completely unstructured, which is why every consequential exchange gets re-keyed into
somebody's system afterwards. It carries no authority, no receipt, no confirmation, no shared
record of what was agreed.

Agent-executed work needs the same neutrality with structure attached. A counterparty has to be
able to answer four questions without trusting the other side's log:

- who was allowed to act
- what changed
- whether their system confirmed it
- who is responsible now

That is also why the neutral position cannot be held by a model vendor or an agent framework.
Coordination only works if every actor passes through it, and in a real company one team is on
one stack, another wrote its own, and a human is in the loop. Neither lab will accept the other
as the authority over what its agent did. Neutrality here is not positioning, it is the product.

The network effect follows from the mechanics rather than from a growth strategy. Once two
organisations already hold their authority grants and their logs with the same operator,
connecting them is configuration rather than an integration project, and every organisation that
joins is reachable by everyone already there.

What is missing before that is real: signed receipts that bind to database settlement, disclosure
narrow enough that proving an outcome does not leak the rows behind it, and a way to map one
organisation's vocabulary onto another's without either adopting the other's
([ontology](domains/ontology-and-schema.md), [evidence](domains/capabilities-and-evidence.md)).
None of it is built.

## Why now rather than five years ago

| Signal | Evidence |
| --- | --- |
| Agents already act through declared schemas | every MCP tool carries an `inputSchema`, and clients validate results against `outputSchema` ([spec](https://modelcontextprotocol.io/docs/concepts/tools)) |
| The read path has been commoditised | reads, fan-out and connections are priced at zero next door ([Electric](https://electric.ax/blog/2026/04/02/electric-cloud-pricing)) |
| The research converged in one year | independent 2026 groups reached the same conclusions about shared-state concurrency for agents ([research](research/shared-state-concurrency.md)) |
| Structure beats model capability, measured | deadlock 86.7% at one round of pre-commitment, 0.0% at three, dominating the choice of model ([DPBench](https://arxiv.org/abs/2602.13255)) |
| Physical actors are already deployed in millions | [IFR World Robotics](https://ifr.org/ifr-press-releases/news/record-of-4-million-robots-working-in-factories-worldwide) |

The DPBench row is the argument in miniature. A better model does not fix a coordination
failure, because the failure is structural. A protocol does.

## What it looks like if it works

**A warehouse.** Four hundred robots, thirty agents planning and replanning, twelve people
overriding when something looks wrong. A robot holds a fenced lease on a zone and moves at local
latency inside it. An agent that reserved a corridor learns its premise expired before it acts
rather than after. A person's override is not silently reversed by an agent that never saw it.
Every one of those transitions is attributable a week later.

**A supply chain.** A buyer's agent changes an order. The supplier's agent accepts part of it and
declines the rest, and both companies hold the same receipt for what settled without sharing a
database or trusting the other's system of record. Nobody re-keys anything from an email.

**A company's own work.** The boring one, and the one that has to work first. Agents and people
writing the same rows all day, where the failure mode today is silent overwriting and the fix
today is telling everyone to be careful.

Today the first of those is the one that is demonstrable. The cross-organisation case is a
protocol direction with nothing built, and the physical case is modelled rather than shipped
([what is shipped](03-the-system-today.md#what-is-shipped-what-is-not)).
