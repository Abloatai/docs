# Ablo: the system and the space around it

A technical orientation pack. It describes Ablo as a systems problem, states what exists
today, shows what has actually been measured, and maps the neighbouring fields whose
guarantees Ablo either inherits or deliberately declines.

**Date:** 2026-08-01. **Status:** orientation, not specification. Where this pack and the
source code disagree, the source code wins and the discrepancy is worth reporting.

## The claim in one paragraph

Ablo is not a fast synchronisation server. It is an attempt to be the coordination boundary
that humans, services, AI agents and devices write through, so that a shared, continuously
changing world has one answer to who was allowed to act, what committed, in what order, what
each observer may safely believe, and how the system recovers after partial failure. Customer
data stays in the customer's own Postgres. Ablo mediates the write, observes the authoritative
WAL echo, and publishes ordered, replayable deltas. The performance problem is therefore not
"move JSON faster", it is sustaining a high rate of consequential, attributable, replayable
state transitions without weakening any of those answers.

## Who this is for

A reader who wants to reason independently about the system: to challenge the architecture,
find the load-bearing assumptions, and ask better questions than the ones already written
down. Roughly a third of this pack is questions, and most of them are open.

## Reading order

**One hour, to get oriented:**

1. [01-the-space.md](01-the-space.md) what Ablo is claiming, and who else is in the space
2. [02-the-contract.md](02-the-contract.md) the vocabulary and the non-negotiable guarantees
3. [03-the-system-today.md](03-the-system-today.md) the path a write actually takes
4. [04-the-evidence.md](04-the-evidence.md) what has been measured, including what failed

**Then, to go deep:**

5. [05-why-it-is-hard.md](05-why-it-is-hard.md) the systems difficulty, stated precisely
6. [06-scale-regimes.md](06-scale-regimes.md) why one million per second is three different targets
7. [domains/](domains/README.md) the eight technical domains, one file each
8. [research/](research/README.md) the agent-coordination literature, including work from the last six months
9. [07-measuring-it.md](07-measuring-it.md) how to read a performance claim, including ours

**Always:**

- [08-questions.md](08-questions.md) the question bank, grouped by who can answer
- [09-reading-list.md](09-reading-list.md) every external source, with one line on why it matters
- [10-repo-map.md](10-repo-map.md) where each concept lives in the codebase

## How this pack is organised

Every fact has exactly one home. The rule comes from the codebase itself, where a
hand-maintained second copy of a type is treated as a defect because nothing fails when the
copies drift. Prose drifts the same way.

| Kind of fact | Lives in | Everywhere else |
| --- | --- | --- |
| A defined term | [02-the-contract.md](02-the-contract.md) | links to it |
| A measured number | [04-the-evidence.md](04-the-evidence.md) | links to it |
| An external source and why it matters | [09-reading-list.md](09-reading-list.md) | cites the specific fact, links the source, does not restate the blurb |
| A technical field Ablo touches | one file in [domains/](domains/README.md) | links to it |
| A research paper on agent coordination | one file in [research/](research/README.md) | links to it |
| A file path in the repo | [10-repo-map.md](10-repo-map.md) | links to it |

If you find the same idea explained twice, that is a bug in this pack. The previous version of
this material was a single 1,300-line document that explained the neighbouring systems four
separate times, which is what prompted the split.

## What this pack does not do

It does not assign work, propose a paper topic, or claim that any mechanism here is novel.
Sections describing possible architectures are maps of a design space, not recommendations.
Where something is unproven, it says so; the honest capability assessment is in
[04-the-evidence.md](04-the-evidence.md#can-ablo-serve-customers-today).
