# When one participant is a person

Agent-only evaluations quietly assume every participant consumes every message before acting
again, reacts at machine speed, and can have its actions mechanically reversed. A person
violates all three, and most of the shared-state protocols in
[shared-state-concurrency.md](shared-state-concurrency.md) depend on at least one of them.

## What the literature establishes

| Work | Finding |
| --- | --- |
| [Engineering Human/Agent Teaming Systems](https://ojs.aaai.org/index.php/AAAI/article/view/5629) | teaming is a systems-engineering problem, not an interface problem |
| [Beyond Accuracy](https://aaai.org/papers/00002-5285-beyond-accuracy-the-role-of-mental-models-in-human-ai-team-performance/) | model accuracy does not predict team performance. What the human expects the system to do is a variable in the outcome |
| [Complementarity in Human-AI Collaboration](https://arxiv.org/abs/2404.00029) | a combined system beats either participant alone under narrower conditions than usually assumed |

## Where agent-only protocols break

| Assumption | Broken by a person |
| --- | --- |
| every participant consumes every message before acting | somebody is still typing |
| actions can be mechanically reversed | the edit was theirs, and reversing it is not a technical decision |
| the system reaches quiescence | a live workspace may never quiesce |
| reaction times are uniform | they differ by orders of magnitude |

## What that requires of the protocol

A human is not a slow LLM endpoint. Supporting one means the coordination layer provides
understandable provenance, visible pending and settled states, interruptibility, bounded
automation, reversible authority where possible, and explicit escalation where reversal is
unsafe. It also means the protocol tolerates participants with very different reaction times and
different notification guarantees, instead of assuming that everyone has consumed every message
before the next action.

This is where several otherwise strong designs stop being applicable. A guarantee stated at
quiescence assumes a moment when everyone is done. A saga-style inverse assumes the action can
be undone. Neither survives contact with a person who is still typing, and Ablo's target is a
surface where that person is a first-class participant rather than an exception.

The distinction between pending and settled is the one that shows up in the interface first. It
is the same `queued` and `confirmed` split the write path already carries
([02-the-contract.md](../02-the-contract.md#the-settlement-rule)), which means the protocol
already has the information a good interface needs. Whether it is exposed usefully is a
different question.

## Still open

- What a person sees between `queued` and `confirmed`, and how long that window lasts in
  practice.
- Who yields when an agent's write meets a human's in-progress edit, and who gets told.
- What the escalation path is when an effect cannot be undone, and whether one exists.
