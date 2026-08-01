# 02. The contract

Performance work only means something if the contract being optimised is written down. This
file is the single definition site for the vocabulary and the guarantees. Every other file in
this pack links here rather than restating them.

## Vocabulary

| Term | Meaning |
| --- | --- |
| **Actor** | a human, service, agent, device or organisation that reads or attempts to change state |
| **Plane** | an independently owned coordination domain, normally scoped by organisation, project, and branch or environment |
| **Capability** | a scoped grant carried by a minted key: which actor may perform which operations over which sync groups, with a TTL |
| **Claim** | a time-bounded ownership or exclusion grant used to coordinate concurrent work |
| **Fence** | a monotonic token that makes a stale claim holder detectably stale |
| **Commit** | an idempotent request to apply one or more mutations under a defined authority and conflict policy |
| **Receipt** | the durable result of a commit, distinguishing accepted work from confirmed settlement |
| **Delta** | a typed, ordered description of a committed state transition |
| **Cursor** | a position or frontier stating exactly what an observer has applied |
| **Audit record** | an attributable record of the authority, intent, result and evidence behind an action |
| **Observation** | a report about external or physical state, distinct from a desired-state mutation |

Each of these is a type in the public contract package, not a description:
[commit](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/commit.ts),
[delta](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/delta.ts),
[cursor](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/feedCursor.ts),
[claim](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/claims.ts),
[capability](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/capability.ts),
[receipt](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/transactions/settlement/pendingWrite.ts).
Full map in [10-repo-map.md](10-repo-map.md).

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

The idempotency identity that makes step 1 safe to retry is
[settlement/idempotencyKey.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/transactions/settlement/idempotencyKey.ts).

Two consequences carry most of the weight. A successful application write is never silently
treated as globally observed: `queued` and `confirmed` are different receipt states, and only
the authoritative feed promotes one to the other. And publication is durable, ordered,
replayable, idempotent and observable, which is what makes recovery of a slow or disconnected
observer exact rather than approximate.

## See it yourself

The two receipt states are one argument apart:

```ts
await ablo.orders.update({ id, data: { status: 'approved' } });
// returns on durable acceptance: queued

await ablo.orders.update({ id, data: { status: 'approved' }, wait: 'confirmed' });
// returns only after the authoritative database echoes the change back
```

Time both. The gap between them is the WAL round trip, and it is the difference between "we
accepted this" and "the source of truth agrees". Run the same call twice with the same
idempotency identity and the row changes once.

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

## Still open

- Which invariants are enforced by code, and which survive on discipline. Each one needs the
  test that would fail if it were violated.
- How long a receipt may sit at `queued` before an operator should care, and what a client
  waiting on it observes.
- What detects privilege drift after connect, when a customer DBA grants the writer something
  new.
