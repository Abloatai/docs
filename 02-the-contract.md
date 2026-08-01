# 02. The contract

Performance work only means something if the contract being optimised is written down. This
file is the single definition site for the vocabulary and the guarantees. Every other file in
this pack links here rather than restating them.

## Vocabulary

| Term | Meaning |
| --- | --- |
| **Actor** | a human, service, agent, device or organisation that reads or attempts to change state |
| **Plane** | an independently owned coordination domain, normally scoped by organisation, project, and branch or environment |
| **Capability** | a verifiable, attenuable grant: which actor may perform which action over which resource and scope |
| **Claim** | a time-bounded ownership or exclusion grant used to coordinate concurrent work |
| **Fence** | a monotonic token that makes a stale claim holder detectably stale |
| **Commit** | an idempotent request to apply one or more mutations under a defined authority and conflict policy |
| **Receipt** | the durable result of a commit, distinguishing accepted work from confirmed settlement |
| **Delta** | a typed, ordered description of a committed state transition |
| **Cursor** | a position or frontier stating exactly what an observer has applied |
| **Audit record** | an attributable record of the authority, intent, result and evidence behind an action |
| **Observation** | a report about external or physical state, distinct from a desired-state mutation |

## The settlement rule

For a mediated write into customer-owned Postgres:

1. authenticate and validate the actor
2. verify capability and tenant scope
3. validate claim, lease and fencing constraints
4. apply the requested mutations in the customer database
5. append sync and audit evidence in the same transactional boundary where required
6. commit the Postgres transaction
7. observe the authoritative logical-WAL echo
8. publish ordered deltas
9. advance observers only after exact application

Two consequences carry most of the weight. A successful application write is never silently
treated as globally observed: `queued` and `confirmed` are different receipt states, and only
the authoritative feed promotes one to the other. And publication is durable, ordered,
replayable, idempotent and observable, which is what makes recovery of a slow or disconnected
observer exact rather than approximate.

## Invariants

Any optimisation intended for production preserves all of these:

- zero silently rejected or exhausted operations
- exact committed operation and delta counts
- exact final cursors or frontiers
- no missing, duplicate or reordered replay
- authored order preserved where the contract requires it
- deterministic conflict and missing-row semantics
- idempotent retry behaviour
- capability and tenant isolation
- claim and fencing correctness
- audit ordering and recoverability
- safe checkpoint recovery after process restart
- single-owner or correctly fenced processing after failover
- bounded queues, memory, retained WAL and retry state
- no receipt semantics weakened to improve a graph
- no unmeasured handoff of work from the server to an overloaded client

This list is why several attractive-looking optimisations are not valid shortcuts. The rejected
experiments in [04-the-evidence.md](04-the-evidence.md#rejected-and-exhausted-directions) are
mostly cases where the throughput was real and one of these invariants was the price.

## The trust boundary

Customer application data stays in the customer's Postgres. Ablo's own database holds
coordination and operational metadata plus the transaction log, never a shadow copy of customer
rows.

The database integration separates privileges deliberately:

- a replication-only role reads the logical change stream
- a DML-only role performs scoped mediated writes
- a one-time administrative bootstrap role is discarded after setup
- the runtime writer is checked to have no superuser, `BYPASSRLS`, role creation, replication,
  table ownership or schema-creation powers

A write that bypasses Ablo and goes straight to the customer database still appears in the WAL,
so it remains observable. It was not governed before execution and its attribution is weaker.
That difference stays explicit rather than being smoothed over.

## Questions this raises

- Which invariants are enforced by code, and which currently survive on discipline? For each
  one, what test would fail if it were violated?
- `queued` and `confirmed` are distinct states. What is the maximum time a receipt can sit at
  `queued` before an operator should care, and what happens to a client waiting on it?
- Idempotency: what exactly is the key derived from, how long is it retained, and what happens
  to a retry that arrives after the retention window?
- Fencing makes a stale claim holder detectably stale. Detectable by whom, at which check, and
  what does the holder observe when it loses?
- The privilege check runs at connect time. What detects privilege drift afterwards, when a
  customer DBA grants the writer something new?
- "No unmeasured handoff of work to an overloaded client" is an invariant. How is client-side
  apply cost measured today, and on which device classes?
- Which of these invariants would a customer actually notice being violated, and which are
  internal hygiene? That ranking decides what to defend hardest under load.
