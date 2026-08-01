# The research landscape around agent coordination

"Agent collaboration" names several different research problems. Some work studies how language
models discuss a question. Some studies task decomposition and delegation. A much smaller set
studies what happens when autonomous actors concurrently read and mutate the same external
state. That last category is the one Ablo lives in, and as of mid-2026 it is moving fast.

| File | Covers |
| --- | --- |
| [foundations-and-conversation.md](foundations-and-conversation.md) | classical multi-agent theory, and the LLM wave that treats conversation as the coordination substrate |
| [shared-state-concurrency.md](shared-state-concurrency.md) | the closest work: agents concurrently mutating shared state |
| [evaluation-and-failure.md](evaluation-and-failure.md) | how these systems fail, and how much of the reported gain is real |
| [humans-in-the-loop.md](humans-in-the-loop.md) | what changes when one participant is a person |

## Reading publication status

This area moves faster than peer review. The lists below distinguish established peer-reviewed
foundations, recent conference papers, and arXiv preprints. A preprint can carry an important
idea without independently reviewed evidence behind it, and every reported result is the
authors' result under their workload rather than an established property of the world. Dates
and identifiers are given so a reader can check the current status rather than inherit this
pack's snapshot.

Everything cited here was checked on 2026-08-01.

## The three layers

The literature sorts into three layers, and most confusion in this area comes from comparing
work across them.

| Layer | Concern | Representative work |
| --- | --- | --- |
| Cognitive and organisational coordination | roles, delegation, conversation, debate, shared plans, verification | SharedPlans, STEAM, AutoGen, MetaGPT, ChatDev, MAST |
| Concurrent artifact coordination | stale views, conflicts, branches, convergence, invalidation, undo, message structure | CoAgent, S-Bus, Atomix, STORM, CodeCRDT, Token Coherence, CAID, ZipperGen |
| Durable world-state coordination | authority, transaction boundaries, ordering, replay, settlement, fan-out, device acknowledgement, physical observation | classical databases and distributed systems, and the layer Ablo is composing for heterogeneous actors |

Most visible LLM-agent work treats communication, prompts, private branches or bounded agent
jobs as the coordination boundary. Ablo's intended boundary is the shared state itself,
including enduring human participation and effects that reach organisations, devices and
physical reality.

That statement needs care. This is a selected review. It does not prove that no prior system
combines these properties, and it establishes nothing about novelty or patentability. What it
does show is where the attention currently is, and where it is not.

## The vocabulary mapping

Classical coordination concepts map onto Ablo primitives
([02-the-contract.md](../02-the-contract.md#vocabulary)) closely enough that the mapping is
worth stating, because it is also a checklist of what a coordination layer owes its
participants.

| Coordination concept | Ablo primitive |
| --- | --- |
| joint goal or plan | a durable plan object with explicit participants |
| delegation | a scoped capability plus a claim or fence |
| an actor's belief | a versioned observation with a cursor and provenance |
| notification | an ordered delta or invalidation, with delivery state |
| action | an authorised transaction with dependencies and idempotency identity |
| completion | a commit receipt, and where relevant a device acknowledgement |
| recovery | replay from durable state, not reconstruction from chat history |
| physical truth | commanded, acknowledged and observed kept distinct |

## Suggested reading sequence

For Ablo's immediate design questions, in this order:

1. [Atomix](shared-state-concurrency.md#atomix-progress-aware-transactions-for-tool-use) for the
   settlement boundary, because it names the same mistake Ablo's receipt model exists to avoid
2. [CoAgent](shared-state-concurrency.md#coagent-a-pre-decided-serialisation-order) for the
   concurrency problem and its strongest theorem
3. [the verified anomaly hierarchy](shared-state-concurrency.md#a-machine-checked-anomaly-hierarchy)
   for a test battery Ablo can be held to
4. [S-Bus](shared-state-concurrency.md#s-bus-read-sets-without-agent-cooperation) for read sets
   without agent cooperation
5. [STORM](shared-state-concurrency.md#storm-mediated-writes-against-branch-and-merge) for the
   empirical case for mediated writes over branch-and-merge
6. [the noise-floor protocol](evaluation-and-failure.md#how-much-coordination-gain-is-real) and
   [DPBench](evaluation-and-failure.md#protocol-structure-dominates-model-capability) before
   running any coordination benchmark of your own
7. the classical joint-intention and SharedPlans work for the deeper semantics of commitment
