# 11. What is proven, and what is only claimed

A guarantee that nothing executes is a sentence. This file maps each invariant in
[02-the-contract.md](02-the-contract.md) onto the journey that runs against a real server and a
real Postgres, and names the ones with nothing behind them.

The suite is 60 journeys, roughly 180 assertions, run with `npm run test:journeys`. A journey
boots the actual server on an ephemeral port against a throwaway Postgres and drives it with the
real SDK, never a mock. That rule exists because a mock once emitted the frame the code under
test was supposed to emit, and hid the bug it was written to catch.

## The invariants, and what executes them

| Invariant | Journey | What it asserts |
| --- | --- | --- |
| Exact committed operation and delta counts | `reconnect`, `fanout`, `catchup-atomic` | a disconnected client replays exactly the missed deltas, and one store application per delivered frame |
| Exact final cursors | `node-deltas`, `reconnect` | a bare Node client's applied position advances on a peer's write, persisted never ahead of applied |
| No missing, duplicate or reordered replay | `post-commit-receipt`, `commit-dispatcher` | durable acknowledgement and replay recovery, dispatch in commit order against a real control-plane WAL |
| Deterministic conflict semantics | `hermitage-conformance`, `coordination-laws`, `functional-update-concurrent`, `connect-locate-conflict` | see the two sections below |
| Idempotent retry | `idempotent-create`, `post-commit-receipt` | the same create twice leaves one row, and a lost connection does not double-apply |
| Capability and tenant isolation | `org-isolation`, `syncgroups`, `auth-identity`, `dashboard`, `direct-write-rls-session-settings`, `postgres-replication-rls-snapshot` | members share, another organisation is blind, and the customer's own RLS governs the mediated write |
| Claim and fencing correctness | `claims`, `claims-autosubscribe`, `claim-rowfree-rung0` | contention across two real clients, a peer observing a claim, and a claim on a key Ablo never synced |
| Audit ordering and recoverability | `audit-outbox`, `evidence-watermark` | atomic enqueue, ordered sealing, replay, and the state a write reasoned from |
| Safe recovery after restart | `crash-recovery`, `log-store-checkpoint` | Postgres dies mid-session and the stack recovers consistently; a materialised checkpoint reads identically to the log |
| Bounded retained WAL | `postgres-replication-slot-pressure`, `postgres-replication-slot-loss` | a stalled consumer walks the slot to invalidation, and the gap is re-snapshotted rather than skipped |
| Receipt semantics preserved | `confirmed`, `post-commit-receipt`, `endpoint-events-settlement` | `wait: 'confirmed'` resolves on the authoritative echo, with a peer attached and with the peer offline |
| Settlement into the customer's database | `connect-direct`, `connect-drizzle-events`, `connect-kysely-events`, `connect-prisma-events`, `postgres-replication-live` | the write lands in their Postgres and returns through logical replication |

## Tested against somebody else's battery

The strongest tests in the suite are the ones Ablo did not invent, because a suite that writes
its own scenarios grades its own homework.

**Hermitage.** The isolation-anomaly battery is ported directly:

| Port | Claim |
| --- | --- |
| H-P4 lost update | concurrent stale writers cannot silently overwrite |
| H-OTV observed transaction vanishes | a moved row becomes an explicit conflict |
| H-G2 write skew | overlapping reads must serialise disjoint writes |
| fail-first control | bypassing the stale guard **reproduces** the lost-update anomaly |

That last row is the one to notice. A control that deliberately disables the guard and confirms
the anomaly returns is what separates a passing test from a test that would pass with the
feature deleted.

**Differential and property oracles.** Two independent read paths must agree, and a generated
history must land in the same place as a reference model:

| Suite | Oracle |
| --- | --- |
| `fold-oracle.property` | fold-from-log is equivalent to the reference model and to the physical table, under generated random histories, with a monotonic watermark |
| `query-conformance` | the SQL compiler and the log fold agree on results, on generated where/orderBy/limit queries, and on which error they raise |

Agreement between two paths written by different code is evidence in a way that a hand-written
expectation is not.

## The coordination laws

The semantics an agent actually depends on are stated as numbered laws and executed as two
scenarios, agents alone and humans mixed with agents:

| Law | Executed as |
| --- | --- |
| One winner, and the losers settle | N agents contend, one wins, the rest receive `AbloClaimedError` and stop, with no retry storm |
| FIFO fairness | queued claims are granted in enqueue order, each on a fresh row |
| A stale read is refused | an unclaimed write on a stale snapshot is rejected with `AbloStaleContextError` |
| A human is never blocked | a human overwrites an agent-held row under `humansOverwrite`, and the agent is the one rejected |
| That default is universal | the same behaviour holds on a model that declares no conflict axis at all |

This is what makes the ontology argument concrete rather than rhetorical: the disposition
declared beside the fields ([ontology](domains/ontology-and-schema.md)) is the disposition the
suite observes.

## Against the anomalies the literature names

The four concurrency anomalies formalised for multi-agent LLM systems
([research](research/shared-state-concurrency.md#a-machine-checked-anomaly-hierarchy)), scored
honestly:

| Anomaly | Covered | By |
| --- | --- | --- |
| stale-generation | yes | `coordination-laws` law 4, `evidence-watermark`, Hermitage H-P4 |
| causal-cascade | partly | `evidence-watermark` records the basis a write reasoned from, but nothing tests propagation through dependent work |
| tool-effect reordering | partly | `commit-dispatcher` proves commit-ordered dispatch; external effects are not modelled |
| phantom-tool | no | nothing tests a tool appearing or disappearing across an operation |

`evidence-watermark` deserves its own line because it answers a question most systems cannot.
A claimed write records `read_at_sync_id` exactly, while another client advances the
organisation past that point. A deliberately stale write is rejected and **leaves no delta**, so
the denial itself is the audit record. An unclaimed write records null, because no basis was
asserted and none is fabricated.

## What has nothing behind it

Stated plainly, because the rest of this file is only worth reading if this section is honest.

| Claim | Status |
| --- | --- |
| Cross-organisation settlement, signed receipts, disclosure | no journey. The protocol direction in [00-the-vision.md](00-the-vision.md) is unexecuted |
| Physical-world state, freshness, irreversible commands | no journey. Modelled only ([physical-world-state](domains/physical-world-state.md)) |
| Slow-subscriber policy and client apply cost | no journey. The invariant "no unmeasured handoff to an overloaded client" is asserted, not tested |
| A stale owner attempting to continue after failover | partial. Slot loss and slot pressure are covered; a fenced-out processor continuing is not |
| Capacity at the scale the vision implies | measured by benchmark, not by journey ([04-the-evidence.md](04-the-evidence.md)) |

## How a journey earns its name

A journey is named for the claim it proves, not the subsystem it touches, and each one pins an
incident or a law rather than a feature. `confirmed` is a claim about receipts. `org-isolation`
is a claim about tenancy. `fold-oracle.property` is a claim that two read paths cannot disagree.

The suite exists because one day of walking real journeys found roughly ten production bugs that
750 passing unit tests could not see, every one of them an interaction between layers. Unit
tests pin a layer. Journeys pin the contract.

## See it yourself

The engine is private, but the battery is not. Take the
[Hermitage](https://github.com/ept/hermitage) tests against whatever coordination your own stack
uses today, then add the control: disable the guard and confirm the anomaly comes back. If it
does not come back, the test was never proving the guard.
