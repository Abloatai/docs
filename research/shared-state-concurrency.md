# Agents concurrently mutating shared state

The closest research to Ablo, and the fastest-moving part of the field. Almost all of it is
2026 and almost all of it is preprint.

The through-line: once several autonomous actors mutate the same external state they are doing
concurrency control, designed or not. The 2026 work is the field noticing that and reaching for
database vocabulary, which is the vocabulary
[02-the-contract.md](../02-the-contract.md) is already written in.

| Work | Date | Status | Mechanism | For Ablo |
| --- | --- | --- | --- | --- |
| [Atomix](https://arxiv.org/abs/2602.14849) | Feb 2026, rev May | preprint | progress-aware transactions; commit when per-resource frontiers show no earlier conflicting work | names the same settlement mistake Ablo's receipt model avoids |
| [CoAgent](https://arxiv.org/abs/2606.15376) | Jun 2026 | preprint, submitted USENIX ATC | MTPO fixes a serialisation order at launch; LLM judges invalidation; saga inverses repair | closest direct comparison, and the strongest theorem |
| [S-Bus](https://arxiv.org/abs/2605.17076) | May 2026 | preprint | reconstructs read sets at commit from observed HTTP GETs; Observable-Read Isolation | read sets without asking agents to declare anything |
| [Verified anomalies](https://arxiv.org/abs/2606.17182) | Jun 2026 | preprint | four anomalies in TLA+, verified L0-L4 hierarchy in Verus | a test battery Ablo can be held to |
| [STORM](https://arxiv.org/abs/2605.20563) | May 2026 | preprint | mediates the shared workspace, detects conflicts at write time | evidence for mediated writes over branch-and-merge |
| [ATM](https://arxiv.org/abs/2607.00041) | Jun 2026 | preprint | pre-write admission: proceed in parallel, serialise, or fail closed | the commit chokepoint stated as an admission problem |
| [CodeCRDT](https://arxiv.org/abs/2510.18893) | 2025 | preprint | agents observe shared CRDT state, coordinate by convergence | convergence without authority, and its limits |
| [Token Coherence](https://arxiv.org/abs/2603.15183) | 2026 | preprint | MESI-style invalidation over agent artifacts, TLA+ checked | the cost of keeping many private contexts coherent |
| [CAID](https://arxiv.org/abs/2603.21489) | 2026 | preprint | dependency-aware delegation, isolated workspaces, branch and merge | the strongest form of the alternative |
| [ZipperGen](https://arxiv.org/abs/2604.17612) | 2026 | ISoLA 2026 | global message-sequence spec projected into deadlock-free local programs | safe control flow, orthogonal to data consistency |
| [Governed shared memory](https://arxiv.org/abs/2606.24535) | Jun 2026 | preprint | scoped access, provenance, policy-controlled propagation | memory as a governance problem |
| [Multi-agent memory](https://arxiv.org/abs/2603.10062) | 2026 | position | three-layer hierarchy; cross-layer consistency as the open gap | vocabulary, not a system |
| [Always-On Agents](https://arxiv.org/abs/2606.30306) | Jun 2026 | survey | 435 works coded on authority, scope, mutability, provenance, recoverability, actionability | the field accumulates state far more than it governs it |
| [Isolation as a principle](https://arxiv.org/abs/2607.12406) | Jul 2026 | survey | five boundaries: user-agent, agent-tool, agent-execution, agent-agent, system-environment | where isolation loss becomes a safety failure |

## Atomix: progress-aware transactions for tool use

Opens by observing that common orchestrators treat the tool's **return** as the settlement
trigger, so faults, speculation and concurrent agents leave partial effects, losing-branch
residue, stale writes or irreversible sends. It argues correct settlement needs two facts that
retries, checkpoint replay, locks and compensation each conflate: which effects settle together,
and when earlier conflicting work is exhausted.

Three independent arrivals at Ablo's own distinctions: a tool returning is not settlement
([receipts](../02-the-contract.md#the-settlement-rule)), progress is per resource
([frontiers](../domains/ordering-and-frontiers.md)), and irreversible effects need their own path
([physical state](../domains/physical-world-state.md)).

## CoAgent: a pre-decided serialisation order

States the problem precisely: agent transactions span minutes of inference, read sets are broad
and opaque, writes take effect immediately. MTPO fixes the serialisation order at launch,
permits speculative in-place effects, notifies agents when another action may invalidate their
premises, and repairs through registered inverses. Serializability holds **at quiescence**.

The assumptions are the interesting part: reads are filtered through the chosen order, tools
declare footprints and register inverses, and notified agents judge affected premises correctly.
Those fit bounded agent jobs. A continuous human and agent workspace may never quiesce, a
human's edit is not a mechanically reversible agent action, and a person cannot be required to
consume a notification before the next keystroke. Different target semantics, not a refutation.

## S-Bus: read sets without agent cooperation

A server-side DeliveryLog reconstructs each agent's read set at commit time from HTTP GET
traffic the middleware already sees, giving Observable-Read Isolation, a partial causal
consistency over the observable read projection. It attacks the assumption CoAgent needs and
Ablo would also need, and it is available to anything sitting on the request path, which Ablo is.

It also names a real gap in premise validation. Validating only the row being written misses
cross-object invariants: read `geo.image`, create `geo-canary` from that value, and the premise
lives on `geo` while the write target is `geo-canary`. Comparing the target's version catches
nothing. Acquiring the dependency and choosing the policy (reject, retry, invalidate, ask a
human) are two separate decisions.

## A machine-checked anomaly hierarchy

| Anomaly | What it is |
| --- | --- |
| stale-generation | a generation produced from state that has since changed |
| phantom-tool | a tool appearing or disappearing across the operation |
| causal-cascade | an invalidation propagating through dependent work |
| tool-effect reordering | externalised effects landing out of order |

Structural analogues of the classical isolation anomalies, formalised in TLA+ with TLC
counter-examples, then a verified L0 to L4 hierarchy in Verus realised in three Rust runtimes.
It reproduces a silent lost update in ByteDance's deer-flow and tool-effect reordering in
LangGraph's ToolNode on unmodified output. Concrete rather than theoretical, and answerable
against any implementation.

## STORM: mediated writes against branch and merge

Mediates agent interaction with a shared workspace so each agent sees a consistent view and
conflicts surface at write time, against a baseline of one git worktree per agent merged
afterwards. Reports +18.7 on Commit0-Lite and +1.4 on PaperBench at comparable or better cost.

Write-time mediation against isolate-and-merge is exactly the commit chokepoint against the
branch. One paper is not settled science, and the margins should be read next to the
[noise floor](evaluation-and-failure.md#how-much-coordination-gain-is-real).

## See it yourself

Stale-generation takes two terminals and no framework. Read a row, wait thirty seconds standing
in for inference, then write a value derived from what you read. During the wait, change the same
row from the other terminal.

The unguarded version lands a write computed from state that no longer exists, and nothing
reports an error. The guarded version refuses it:

```ts
await using claim = await ablo.orders.claim({ id });

const priced = await pricingAgent(claim.data);   // thirty seconds of inference

await ablo.orders.update({ id, data: priced, claim, wait: 'confirmed' });
// rejected if the premise moved while the agent was thinking
```

Then extend it: have the second terminal write a *different* row that the first one read from.
That is the cross-object premise case, and most systems, including systems that pass the first
test, fail this one.

## Go deeper

| Read in this order | Why |
| --- | --- |
| [Atomix](https://arxiv.org/abs/2602.14849) | the settlement boundary, stated by someone who arrived at it independently |
| [CoAgent](https://arxiv.org/abs/2606.15376) | the concurrency problem and the strongest current theorem |
| [Verified anomalies](https://arxiv.org/abs/2606.17182) | the four failure shapes, with counter-examples |
| [S-Bus](https://arxiv.org/abs/2605.17076) | read sets acquired rather than declared |
| [STORM](https://arxiv.org/abs/2605.20563) and [CAID](https://arxiv.org/abs/2603.21489) | the two sides of mediate against fork |
| [Always-On Agents](https://arxiv.org/abs/2606.30306) | the map of the whole area, with the gap named |

## Still open

- Whether an always-live workspace ever quiesces, and what the strongest available guarantee is
  when it does not.
- Whether premises attach to the write target or to the object that was read. If the target,
  cross-object premises are unguarded.
- Whether "which effects settle together" and "when earlier conflicting work is exhausted" are
  separable in a commit path, or conflated by default.
- Where the boundary sits between customers whose effects are live and customers whose work can
  sit on a branch. Naming that customer is worth more than winning the argument.
