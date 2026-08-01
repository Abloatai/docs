# Capability security and cross-organisation evidence

## The problem

Two organisations can disagree about a transition, and neither should have to trust one vendor's
mutable log to settle it. At the same time, a globally verifiable record must not become a global
leak of customer data. Those two requirements pull in opposite directions, and the resolution is
to prove authority and outcome while disclosing as little of the underlying row as possible.

The near-term version of the same problem is agent containment. An agent acts with delegated
authority, retries more than a human, and can be compromised. The question "what is the blast
radius of this token" is answered by the capability model, not by the application.

## What the field knows

**Attenuation without a round trip is a solved primitive.** Macaroons demonstrated bearer
credentials that any holder can restrict further by appending contextual caveats, with
verification requiring no call back to the issuer. Biscuit takes the same shape into public-key
verification with Datalog policies and append-only attenuation blocks, so a holder can narrow
its authority but never widen it.

**Revocation stays stateful, and that is the boundary.** Offline verification and offline
attenuation do not remove the need for revocation state somewhere, which means the fast path can
be local while the safety property depends on something global. Every design that gives agents
long-lived attenuated tokens inherits that tension, and the interesting question is how quickly
a compromised delegation can be stopped, not whether it can be verified cheaply.

## What Ablo adds around the token

A capability answers whether an actor may act. Ablo layers on the operational half: claim
ownership so two authorised actors do not both act, fencing so a stale holder is detectably
stale, idempotency so a retry is not a second action, a transaction outcome so the actor learns
what happened, and customer-database scoping so authority is bounded by the tenant boundary that
already exists. Definitions in
[02-the-contract.md](../02-the-contract.md#vocabulary).

Candidate mechanisms for the cross-organisation layer: signed receipts, capability delegation
proofs, content-addressed audit evidence, hash-chained or transparency-log settlement records,
selective disclosure, verifiable organisation and actor attribution, and explicit revocation and
key-rotation epochs. None of this is built.

## Questions this raises

- How fast can authority be revoked today, end to end, and what is the worst case for an agent
  holding a token mid-operation?
- Is a capability verified once at connection time or on each action? The answer determines the
  revocation latency, and it is a real tradeoff rather than an oversight.
- What exactly would a signed receipt commit to: the request, the resulting delta, the database
  LSN, or all three? Only the third ties the signature to settlement.
- How would a counterparty verify a receipt without being shown row data they have no right to?
- What contains a compromised agent besides the token: rate, claim, scope, time? Which of those
  are enforced today?
- Device identity over a ten-year hardware lifetime needs key rotation. Does anything in the
  current model survive rotation?
- Attribution for a write that bypassed Ablo is weaker by construction. What is the strongest
  honest statement Ablo can make about such a write, and does the audit trail say it in those
  words?
