<p align="center">
  <a href="https://abloatai.com"><img src="assets/banner.png" alt="Ablo" width="480" /></a>
</p>

<p align="center">
  <strong>The system, and the space around it.</strong>
</p>

<p align="center">
  <a href="01-the-space.md">Start here</a> &nbsp;|&nbsp;
  <a href="02-the-contract.md">The contract</a> &nbsp;|&nbsp;
  <a href="04-the-evidence.md">Evidence</a> &nbsp;|&nbsp;
  <a href="08-learning-path.md">Learning path</a> &nbsp;|&nbsp;
  <a href="https://github.com/Abloatai/ablo">Ablo on GitHub</a>
</p>

<p align="center">
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

Holding those answers at a high rate has a price in latency, throughput and cost. This pack
states that price plainly, including the experiments that failed and the parts that remain
unproven. Every file ends with what is still open.

## Start

An hour, to get oriented:

1. [01-the-space.md](01-the-space.md) what Ablo is claiming, and who else is in the space
2. [02-the-contract.md](02-the-contract.md) the vocabulary and the non-negotiable guarantees
3. [03-the-system-today.md](03-the-system-today.md) the path a write actually takes
4. [04-the-evidence.md](04-the-evidence.md) what has been measured, including what failed

Then, to go deep:

5. [05-why-it-is-hard.md](05-why-it-is-hard.md) the systems difficulty, stated precisely
6. [06-scale-regimes.md](06-scale-regimes.md) why one million per second is three different targets
7. [domains/](domains/README.md) the eight technical fields, one file each
8. [research/](research/README.md) the agent-coordination literature, through July 2026
9. [07-measuring-it.md](07-measuring-it.md) how to read a performance claim, including ours

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
