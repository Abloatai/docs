# When one participant is a person

Agent-only evaluations quietly assume every participant consumes every message before acting
again, reacts at machine speed, and can have its actions mechanically reversed. A person
violates all three, and most of the shared-state protocols in
[shared-state-concurrency.md](shared-state-concurrency.md) depend on at least one of them.

## What the human-agent literature establishes

- [A Framework for Engineering Human/Agent Teaming Systems](https://ojs.aaai.org/index.php/AAAI/article/view/5629)
  treats teaming as a systems-engineering problem rather than an interface problem.
- [Beyond Accuracy: The Role of Mental Models in Human-AI Team Performance](https://aaai.org/papers/00002-5285-beyond-accuracy-the-role-of-mental-models-in-human-ai-team-performance/)
  shows that individual model accuracy does not predict team performance. What the human
  believes the system will do is a variable in the outcome.
- [Complementarity in Human-AI Collaboration](https://arxiv.org/abs/2404.00029) examines when a
  combined system outperforms either participant alone, which is a narrower condition than it is
  usually assumed to be.

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

## Questions this raises

- What does a person see today between `queued` and `confirmed`, and how long is that window in
  practice?
- When an agent's write conflicts with a human's in-progress edit, who yields, and who is told?
- Which agent actions are reversible in the current system, and is that classification explicit
  or implied?
- What is the escalation path when an effect cannot be undone? Is there one?
- Can a human hold a claim the way an agent does, and does the lease expiry make sense at human
  timescales?
- If the mental-model result holds, what is the single thing a user most needs to predict
  correctly about Ablo's behaviour, and does the current interface teach it?
