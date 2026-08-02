<p align="center">
  <a href="https://abloatai.com"><img src="assets/banner.png" alt="Ablo" width="480" /></a>
</p>

<p align="center">
  <strong>The system, and the space around it.</strong>
</p>

<p align="center">
  <a href="00-the-vision.md">The vision</a> &nbsp;|&nbsp;
  <a href="01-the-space.md">The space</a> &nbsp;|&nbsp;
  <a href="02-the-contract.md">The contract</a> &nbsp;|&nbsp;
  <a href="04-the-evidence.md">Evidence</a> &nbsp;|&nbsp;
  <a href="domains/ontology-and-schema.md">Ontology</a> &nbsp;|&nbsp;
  <a href="08-learning-path.md">Learning path</a> &nbsp;|&nbsp;
  <a href="https://github.com/Abloatai/ablo">Ablo on GitHub</a>
</p>

<p align="center">
  <a href="./LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-2563eb?style=flat-square" alt="license" /></a>
  <img src="https://img.shields.io/badge/status-orientation%20brief-2563eb?style=flat-square" alt="status" />
  <img src="https://img.shields.io/badge/updated-2026--08--01-22c55e?style=flat-square" alt="updated" />
</p>

---

Ablo is an authoritative transaction layer for shared application state. Every write goes
through one typed API where authority, idempotency, conflicts, ordering and confirmation are
enforced, and the customer's Postgres stays the source of truth.

This repository explains how that works as a systems problem: the guarantees it holds, what it
currently costs to run, where the evidence stops, and the neighbouring fields whose properties
Ablo inherits or declines.

## Why this exists

Software used to have one writer. AI applications now have humans, agents, workflows and
services acting concurrently, and coordinating them means answering questions a database
transaction leaves open: who was allowed to act, which transition committed, what each observer
may safely believe, and how the system recovers after partial failure.

The bet behind the engineering is that this keeps going: that software actors, and eventually
physical ones, come to make most consequential changes, that they will need to act on the same
things at the same time, and that the record of who was allowed to act and what settled becomes
infrastructure. One million agents each acting once every ten seconds is 100,000 changes per
second, which is exactly the gate the engine is measured against today.
[00-the-vision.md](00-the-vision.md) makes that case, and lists what would make it wrong.

Holding those answers at a high rate has a price in latency, throughput and cost. This pack
states that price plainly, including the experiments that failed and the parts that remain
unproven. Every file ends with what is still open.

## What it coordinates over

Ablo coordinates changes to a **declared thing**: a document, an order, a button, an aircraft.
The thing has a name, typed fields, relations, and rules about who may change it and what
happens when two actors try at once. Without that declaration nothing else is expressible. You
cannot detect a conflict on a value whose identity you cannot name, scope authority to a thing
that has no shape, or write an audit line a person can read.

The vocabulary belongs to the organisation, never to Ablo. A task in a hospital is not a task in
a factory, so there is no world ontology to adopt. Ablo takes the models a customer has already
declared and adds a coordination overlay.

| Layer | Owner | Varies by organisation |
| --- | --- | --- |
| The things and their fields | the customer | always |
| Identity, ownership, conflict disposition, tenancy, freshness | Ablo | never |

The nouns are never universal. The overlay always is. That is also what makes the physical
direction the same mechanism rather than a new one: a lamp, a vehicle and an aircraft are
declared things too, with two additions, a validity interval on every observation and commands
that cannot be undone.

[domains/ontology-and-schema.md](domains/ontology-and-schema.md) has the argument, the schema
doing the work, and where Palantir reached the same conclusion from the enterprise direction.

## Start

An hour, to get oriented:

1. [00-the-vision.md](00-the-vision.md) why any of this is worth building, and what would make it wrong
2. [01-the-space.md](01-the-space.md) what Ablo is claiming, and who else is in the space
3. [02-the-contract.md](02-the-contract.md) the vocabulary and the non-negotiable guarantees
4. [03-the-system-today.md](03-the-system-today.md) the path a write actually takes
5. [04-the-evidence.md](04-the-evidence.md) what has been measured, including what failed

Then, to go deep:

6. [05-why-it-is-hard.md](05-why-it-is-hard.md) the systems difficulty, stated precisely
7. [06-scale-regimes.md](06-scale-regimes.md) why one million per second is three different targets
8. [domains/](domains/README.md) the nine technical fields, one file each
9. [research/](research/README.md) the agent-coordination literature, through July 2026
10. [07-measuring-it.md](07-measuring-it.md) how to read a performance claim, including ours

Alongside any of it:

- [08-learning-path.md](08-learning-path.md) a six-stage route through the space, with things to run
- [09-reading-list.md](09-reading-list.md) every external source, with one line on why it matters
- [10-repo-map.md](10-repo-map.md) where each concept lives in the codebase

## How this is organised

Every fact has one home. The rule comes from the codebase, where a hand-maintained second copy
of a type is a defect because nothing fails when the copies drift. Prose drifts the same way.

| Kind of fact | Lives in | Everywhere else |
| --- | --- | --- |
| A defined term | [02-the-contract.md](02-the-contract.md) | links to it |
| A measured number | [04-the-evidence.md](04-the-evidence.md) | links to it |
| An external source and why it matters | [09-reading-list.md](09-reading-list.md) | cites the fact, links the source |
| A technical field Ablo touches | one file in [domains/](domains/README.md) | links to it |
| A research paper on agent coordination | one file in [research/](research/README.md) | links to it |
| An open problem | the **Still open** section of the file that owns it | [08-learning-path.md](08-learning-path.md) indexes them |
| A file path in the repo | [10-repo-map.md](10-repo-map.md) | links to it |

The same idea explained twice is a bug. This pack replaced a single 1,300-line document that
explained the neighbouring systems four separate times, which is what prompted the split.

## Status

Written 2026-08-01 as technical orientation rather than specification. Claims about neighbouring
systems were checked against their current published documentation on that date, and the
research files give publication status per item because most of that work is recent and still
preprint.

Where this pack and the source code disagree, the source code wins, and the discrepancy is worth
reporting.

## Ablo

The engine and the published SDKs live in [Abloatai/ablo](https://github.com/Abloatai/ablo).
Product documentation is at [docs.abloatai.com](https://docs.abloatai.com).

## License

Apache License 2.0. See [LICENSE](./LICENSE) and [NOTICE](./NOTICE).
