# Ontology: coordinating over declared things

Ablo does not coordinate changes to bytes. It coordinates changes to a **declared thing**: a
document, an order, a button, an aircraft. The thing has a name, fields with types, relations to
other things, and rules about who may change it and what happens when two actors try at once.

An ontology is to a knowledge graph what a schema is to a database: the statement of what kinds
of things exist and what may be true of them. Ablo's schema is that statement, and it is the
reason the rest of the system can say anything precise at all.

## Coordination is impossible without a declaration

Every guarantee in [02-the-contract.md](../02-the-contract.md) bottoms out in something the
schema declares.

| What the system must decide | What it needs declared |
| --- | --- |
| Whether two writes collide | the identity of the thing being written, and which fields |
| Whether this actor may act | the model name and the operations over it |
| What a delta means to a subscriber | field names and types |
| Who must be told | the shapes subscribers hold |
| What happens when a human and an agent disagree | a conflict disposition per model |
| What a person reads in the audit trail | names, not opaque payloads |
| Whether a premise expired | the field the premise was read from |

An untyped blob supports none of these. You cannot detect a conflict on a value whose identity
you cannot name, and you cannot scope authority to a thing that has no shape.

## Every organisation has its own ontology

This is the part that decides whether the approach works at all. There is no universal
vocabulary. A "task" in a hospital is not a task in a factory. An "order" means one thing to a
restaurant and another to a broker. Two departments in the same company will disagree, in good
faith, about what a "customer" is.

Global ontology projects have spent decades on that problem.
[schema.org](https://schema.org/) succeeded by staying shallow and optional.
[OWL and RDF](https://www.w3.org/TR/owl2-overview/) are expressive and are adopted where one
authority can impose a vocabulary. Neither produces agreement between two companies who each
already named their own things.

So Ablo does not ship a world ontology. It takes the vocabulary the organisation has already
declared, usually the models that already exist in their ORM, and adds a **coordination
overlay** on top of it.

| Layer | Who owns it | Varies by organisation |
| --- | --- | --- |
| The things and their fields | the customer | always |
| Identity, ownership, conflict disposition, tenancy, freshness | Ablo | never |

The overlay is what is universal. `Task` is not, and never will be. That split is why the layer
can be sold to a hospital and a factory without either adopting the other's words.

## What a declaration carries

A model is a Zod object plus the options the engine needs. The row type is inferred from the
schema, so there is no second type system to keep in sync
([model.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/model.ts)):

```ts
const tasks = model({
  title: z.string(),
  status: z.enum(['todo', 'doing', 'done']).default('todo'),
  projectId: z.string().optional(),
}, {
  relations: { project: relation.belongsTo('projects', 'projectId') },
  load: 'lazy',
});
```

The interesting part is that behaviour is declared beside the fields, not configured elsewhere.
Conflict disposition is authored in the same DSL
([coordination.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/coordination.ts)):

```ts
conflict: coordination.humansOverwrite().agentsReject()
// → { user: 'overwrite', agent: 'reject' }
```

A human's write wins, an agent's yields. That is an ontological statement about this kind of
thing, and it travels with the model rather than living in a policy file somebody forgets.

| Declared beside the fields | File |
| --- | --- |
| Fields and their types | [field.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/field.ts) |
| Relations between things | [relation.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/relation.ts) |
| What happens when two actors write | [coordination.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/coordination.ts) |
| Who can see it | [tenancy.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/tenancy.ts) |
| Where it may live | [residency.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/residency.ts) |

## Everything else derives

One definition, many surfaces. The API documentation is generated from the schema
([openapi.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/openapi.ts)),
drift against the real database is computed from it
([diff.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/schema/diff.ts)),
credential scopes are typed off the model names so `can: { tasks: ['read', 'update'] }` becomes
the wire allowlist `task.read`, `task.update`, and the client materialises the same shapes
locally. A hand-written second copy of any of that is a defect, for the same reason a second
copy of an ontology is: nothing fails when the copies drift.

## Agents already run on declared schemas

This is the part that makes the argument current rather than academic. Every MCP tool definition
carries a `name`, a `description`, an `inputSchema` in JSON Schema, and optionally an
`outputSchema` that clients validate results against
([spec](https://modelcontextprotocol.io/docs/concepts/tools)). The agent ecosystem already
decided that a model calling a tool needs a declared shape.

The gap is that the **tool call** is typed and the **state underneath it** usually is not. An
agent knows the shape of the arguments it must send, and nothing about what the thing currently
is, who else is changing it, or what happened after the call returned. Declaring the state, not
just the call, is what closes that.

For a model, a declared ontology is also the difference between recall and guessing. It gives
the agent a vocabulary to ground a claim in, and gives a checker something to validate against.

## Who else builds this way

The argument is not novel, and the company that made it first made it at scale.

**Palantir's Ontology** has three primitives: object types for the things you operate on, link
types for how they relate, and action types for the governed operations that change them
([the Ontology system](https://www.palantir.com/docs/foundry/architecture-center/ontology-system)).
Every write goes through an action type, which carries validation, approvals, audit and side
effects. Their agents do not touch tables and do not roam the schema; they act through
Ontology-exposed objects, functions and actions, which means an agent is bounded by the same
permissions as a human operator and is reviewable in the same log.

That is the same sentence Ablo would write. The differences are where the data lives and how the
ontology gets built:

| | Palantir Ontology | Ablo |
| --- | --- | --- |
| Where the data sits | brought into the platform | stays in the customer's Postgres |
| How the ontology is authored | modelled in the platform, usually with deployment engineers | declared in code, derived from the models the customer's ORM already has |
| The governed write | an action type | a commit through one typed API |
| Multi-writer coordination | permissions and validation on the action | claims, fences, and premise staleness across an agent's inference |
| Shape of the product | an enterprise platform | an SDK and an API |

**8090's Software Factory** starts from the same belief one layer up: put human engineers and AI
agents in one system where the work is governed, documented and auditable rather than
free-form, aimed at regulated industries. It raised a
[$135M Series A led by Salesforce Ventures](https://www.businesswire.com/news/home/20260626795833/en/8090-Raises-$135M-Series-A-to-Accelerate-Their-Rollout-of-Software-Factory)
in June 2026, and EY launched
[EY.ai PDLC on it](https://www.ey.com/en_us/newsroom/2026/03/ernst-young-llp-and-8090-launch-ey-ai-pdlc)
in March 2026. Their declared structure governs how software gets built; Ablo's governs how
state gets changed once it is running.

The convergence is the signal worth taking from this: three different companies, from three
directions, concluding that an agent has to act inside a declared structure rather than on raw
data.

## The same mechanism for physical things

A button, a lamp, a vehicle and an aircraft are declared things too, and the robotics standards
say so in their own words. A [W3C WoT Thing Description](https://w3c.github.io/wot-thing-description/)
declares a Thing's properties, actions and events. [VDA 5050](https://github.com/VDA5050/VDA5050)
declares exactly which fields a vehicle reports in its state message.

So the physical direction is not a new mechanism, it is the same one with two additions:
observations carry a validity interval, and some commands cannot be undone. See
[physical-world-state](physical-world-state.md).

## See it yourself

Declare the same field twice with different dispositions and watch the engine treat them
differently:

```ts
const notes = model({ body: z.string() }, {
  conflict: coordination.humansOverwrite().agentsReject(),
});

const ledger = model({ balance: z.number() }, {
  conflict: coordination.agentsReject(),
});
```

Have an agent and a human write each row at the same time. On `notes` the human wins and the
agent is told. On `ledger` nobody silently overwrites. Nothing about the writers changed, only
what the ontology says about the kind of thing being written.

## Go deeper

| Read | For |
| --- | --- |
| [MCP tools](https://modelcontextprotocol.io/docs/concepts/tools) | the agent ecosystem's existing commitment to declared schemas, down to `inputSchema` and `outputSchema` |
| [W3C WoT Thing Description](https://w3c.github.io/wot-thing-description/) | the same idea for a physical object |
| [schema.org](https://schema.org/) | what a shallow, optional, widely adopted vocabulary looks like |
| [OWL 2 overview](https://www.w3.org/TR/owl2-overview/) | what a deep formal ontology costs, and buys |
| [Governed shared memory](https://arxiv.org/abs/2606.24535) | the 2026 argument that agent memory needs scope, provenance and policy, not just storage |

## Still open

- Schema evolution against replay: a delta authored under an older shape has to remain
  interpretable, and the rules for that are not fully written down.
- Cross-organisation vocabulary. Two companies coordinating through Ablo each have their own
  names, and something has to map them without either adopting the other's ontology.
- How much behaviour belongs in the schema. Conflict disposition and tenancy clearly do.
  Freshness and validity intervals probably do. The boundary is not settled.
- Whether an agent should be able to propose a schema change, and what approval that requires.
