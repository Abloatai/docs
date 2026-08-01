# 10. Where this lives in code

The contracts, the client, the coordination primitives and the CLI are public in
[Abloatai/ablo](https://github.com/Abloatai/ablo). Every link below points at real source. The
server that implements the write path is not public, so this map names the contract each concept
is defined by rather than the engine that enforces it.

## The packages

| Package | What it is |
| --- | --- |
| [transaction](https://github.com/Abloatai/ablo/tree/main/packages/transaction) | the canonical contracts: reads, commits, settlement, claims, durable observation. Everything below lives here unless noted |
| [ablo](https://github.com/Abloatai/ablo/tree/main/packages/ablo) | the published SDK, for coordinated reads, commits, claims, observation and reactive applications |
| [humans](https://github.com/Abloatai/ablo/tree/main/packages/humans) | the optional human-facing local-state package: presence, live queries, React bindings |
| [cli](https://github.com/Abloatai/ablo/tree/main/packages/cli) | the `ablo` command line: set up, connect a database, push a schema |
| [agent](https://github.com/Abloatai/ablo/tree/main/packages/agent) | AI SDK composition and coordination adapters |

## The concepts, in source

| Concept | File |
| --- | --- |
| Commit request shape | [wire/commit.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/commit.ts) |
| Delta | [wire/delta.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/delta.ts) |
| Cursor | [wire/feedCursor.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/feedCursor.ts) |
| Feed events | [wire/feedEvent.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/feedEvent.ts) |
| Claims, on the wire | [wire/claims.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/claims.ts), [wire/claimEvent.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/claimEvent.ts) |
| Error envelope and code registry | [wire/errorEnvelope.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/errorEnvelope.ts), [errorCodes.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/errorCodes.ts) |
| Protocol version | [wire/protocolVersion.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/protocolVersion.ts) |

## Schema and ontology

The declaration everything else derives from ([ontology-and-schema](domains/ontology-and-schema.md)).

| Concept | File |
| --- | --- |
| A model: fields plus options, row type inferred | [schema/model.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/model.ts) |
| Fields and their metadata | [schema/field.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/field.ts) |
| Relations between things | [schema/relation.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/relation.ts) |
| What happens when two actors write | [schema/coordination.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/coordination.ts) |
| Who can see it | [schema/tenancy.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/tenancy.ts) |
| Where it may live | [schema/residency.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/residency.ts) |
| Documentation, generated from the schema | [schema/openapi.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/openapi.ts) |
| Drift against the real database | [schema/diff.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/diff.ts) |
| Table provisioning | [schema/ddl.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/ddl.ts) |

`coordination.ts` is the surprising one: conflict disposition is authored beside the fields, so
`coordination.humansOverwrite().agentsReject()` is part of the model rather than a policy file.

## Settlement

| Concept | File |
| --- | --- |
| Commit envelope | [settlement/commitEnvelope.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/transactions/settlement/commitEnvelope.ts) |
| Idempotency key | [settlement/idempotencyKey.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/transactions/settlement/idempotencyKey.ts) |
| Accepted but unconfirmed work | [settlement/pendingWrite.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/transactions/settlement/pendingWrite.ts) |
| Durable writes | [durableWrites.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/durableWrites.ts) |
| The delta row in the log | [log/syncDeltaRow.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/log/syncDeltaRow.ts), [syncLog/contract.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/syncLog/contract.ts) |

`pendingWrite.ts` is the file to read if only one: it is `queued` against `confirmed` expressed
as a type ([the settlement rule](02-the-contract.md#the-settlement-rule)).

## Coordination

| Concept | File |
| --- | --- |
| Waiting for a claim | [coordination/awaitClaimGrant.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/coordination/awaitClaimGrant.ts) |
| Keeping one alive | [coordination/claimHeartbeatLoop.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/coordination/claimHeartbeatLoop.ts) |
| Deciding two writes collide | [coordination/targetConflict.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/coordination/targetConflict.ts) |
| What a claim covers | [footprint.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/footprint.ts), [coordination/locator.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/coordination/locator.ts) |

## Credentials

| Concept | File |
| --- | --- |
| Key kinds | [auth/credentialKind.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/credentialKind.ts), [auth/apiKey.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/apiKey.ts) |
| Minting a scoped session | [auth/sessionMint.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/sessionMint.ts) |
| Scope and its lifecycle | [auth/capability.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/capability.ts), [auth/capabilityLifecycle.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/capabilityLifecycle.ts) |
| Who is acting | [auth/identity.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/identity.ts) |
| What a browser may hold | [auth/browserCredentialSafety.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/auth/browserCredentialSafety.ts) |

## The client side

| Concept | File |
| --- | --- |
| Applying deltas | [local/SyncClient.ts](https://github.com/Abloatai/ablo/blob/main/packages/humans/src/local/SyncClient.ts) |
| The durable position | [local/logPosition.ts](https://github.com/Abloatai/ablo/blob/main/packages/humans/src/local/logPosition.ts) |
| Local persistence | [local/persistence.ts](https://github.com/Abloatai/ablo/blob/main/packages/humans/src/local/persistence.ts), [local/mutationPersistence.ts](https://github.com/Abloatai/ablo/blob/main/packages/humans/src/local/mutationPersistence.ts) |
| The local graph | [local/Database.ts](https://github.com/Abloatai/ablo/blob/main/packages/humans/src/local/Database.ts) |

## The convention that shapes all of it

[packages/transaction/CONVENTIONS.md](https://github.com/Abloatai/ablo/blob/main/packages/transaction/CONVENTIONS.md)
states the rule the codebase is organised around: one definition site per shape, everything else
derived. Schemas define, types are inferred, documentation is generated, and a hand-written
second copy of a contract is treated as a defect because nothing fails when the copies drift.

This pack follows the same rule for prose, which is why each fact appears in one file and
everything else links to it.

## Not public

The engine is a separate, private repository: the commit chokepoint, the logical-replication
reader, publication and fan-out, and the benchmark topology behind
[04-the-evidence.md](04-the-evidence.md). Nothing in this pack names its internals, because the
useful thing to read is the contract rather than one implementation of it.

That split is deliberate rather than a redaction. Every guarantee in
[02-the-contract.md](02-the-contract.md) is expressed as a type above, and a type is checkable:
you can read what `pendingWrite` distinguishes without trusting a description of the server that
produces it.
