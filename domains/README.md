# The eight domains

Ablo sits at the intersection of fields normally sold as separate products or taught as
separate subjects. Each file below owns one of them: what the problem is, what Ablo does today,
what the surrounding field already knows, and what stays open.

The useful approach is not to find one established category that already contains Ablo. It is to
understand what each neighbouring category guarantees, then decide which responsibilities Ablo
keeps and which it delegates.

| Domain | The question it owns |
| --- | --- |
| [postgres-under-hot-updates.md](postgres-under-hot-updates.md) | why an indexed UPDATE costs what it costs |
| [logical-decoding-and-cdc.md](logical-decoding-and-cdc.md) | how a committed change becomes an observable event, and what breaks |
| [ordering-and-frontiers.md](ordering-and-frontiers.md) | which order is semantically necessary, and what a cursor is |
| [representation-and-memory.md](representation-and-memory.md) | what the data is shaped like between stages |
| [fanout-and-incremental-views.md](fanout-and-incremental-views.md) | who needs to hear about a change |
| [backpressure-and-overload.md](backpressure-and-overload.md) | what happens when a consumer cannot keep up |
| [capabilities-and-evidence.md](capabilities-and-evidence.md) | who was allowed, and who can later prove it |
| [physical-world-state.md](physical-world-state.md) | how a commitment relates to a physical fact |

## Responsibility map

Each layer states exactly what it knows. The system is strongest when `sent`, `committed`,
`published`, `applied`, `acknowledged` and `observed` are never treated as synonyms.

| Layer | Proves | Does not prove |
| --- | --- | --- |
| Customer Postgres | atomic row changes, constraints, isolation, durable WAL | actor delegation, external side-effect completion, subscriber freshness |
| Ablo write authority | capability, scope, claims, fencing, idempotency, mediated commit result | physical truth, or delivery to every observer |
| WAL and CDC path | that database changes committed | business intent, permission before a bypass write, client application |
| Publication log | ordered replay and recovery position | that a slow consumer has applied the transition |
| Client materialisation | locally useful current state and reactive behaviour | global settlement, unless tied to a confirmed cursor |
| Device or edge gateway | local protocol, command delivery, observation, safety interlocks | global organisational authority, unless explicitly connected to it |
| Audit and evidence layer | attribution, reconstruction, potentially signed confirmation | confidentiality, unless disclosure is deliberately minimised |
