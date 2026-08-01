# Agents concurrently mutating shared state

The closest research to Ablo, and the fastest-moving part of the field. Almost everything below
appeared between February and July 2026, and almost all of it is preprint. Status is given per
item.

The through-line: once several autonomous actors mutate the same external state, they are doing
concurrency control, whether or not anyone designed it. The 2026 work is the field noticing
this and reaching for database vocabulary, which is the same vocabulary
[02-the-contract.md](../02-the-contract.md) is written in.

## Atomix: progress-aware transactions for tool use

[arXiv 2602.14849](https://arxiv.org/abs/2602.14849), Mohammadi, Potamitis, Klein, Arora and
Bindschaedler, February 2026, revised May 2026. Preprint.

Atomix opens with the observation that common orchestrators treat the tool's return as the
settlement trigger, so faults, speculation and concurrent agents leave partial effects,
losing-branch residue, stale writes or irreversible sends. It argues that correct settlement
needs two facts that retries, checkpoint replay, locks and compensation each conflate: which
effects must settle together, and when earlier conflicting work is exhausted. Its transactions
commit only when per-resource frontiers confirm no earlier conflicting work is pending, suppress
uncommitted effects, compensate reversible externalised effects, and prevent effects classified
as irreversible from leaking.

This is the closest independent statement of three things Ablo already asserts: that a tool
returning is not settlement ([the receipt model](../02-the-contract.md#the-settlement-rule)),
that progress has to be expressed per resource rather than globally
([frontiers](../domains/ordering-and-frontiers.md)), and that irreversible effects need a
different path from reversible ones
([physical-world-state](../domains/physical-world-state.md)). Independent arrival at the same
distinctions is evidence the distinctions are real.

## CoAgent: a pre-decided serialisation order

[arXiv 2606.15376](https://arxiv.org/abs/2606.15376), Lyu et al., June 2026. Preprint,
submitted to USENIX ATC 2026.

CoAgent states the problem precisely: agent transactions span minutes of inference, read sets
are broad and opaque, and writes take effect immediately, so classical mechanisms fit poorly.
MTPO fixes a serialisation order at launch, permits speculative in-place tool effects, notifies
agents when another action may invalidate their premises, and uses registered saga-style
inverses to undo and reorder misplaced writes. It guarantees serializability at quiescence. Its
distinctive move is using the LLM itself to judge whether a conflicting write invalidates a
plan, and to repair the operations that depended on it.

The pre-decided order is what makes a serializability result possible despite speculative
external effects. The assumptions are equally important: the guarantee holds at quiescence,
reads are filtered through the selected order, tools must declare footprints and register
inverses, and notified agents must judge affected premises correctly.

Those assumptions fit bounded agent jobs better than an open-ended human and agent workspace.
In a continuous session there may be no moment when every actor commits and the system
quiesces. A human's edit cannot be treated as a mechanically reversible agent action, and a
human cannot be required to consume a notification before the next keystroke. This does not
refute MTPO. It identifies different target semantics: batch-like agent transactions with a
serial result, against an always-live surface with per-action authority and explicit conflict
handling.

## S-Bus: read sets without agent cooperation

[arXiv 2605.17076](https://arxiv.org/abs/2605.17076), Sajjad Khan, May 2026. Preprint.

S-Bus addresses concurrency control for agents sharing mutable state over HTTP where the agents
cannot be modified to declare their read sets. A server-side DeliveryLog reconstructs each
agent's read set at commit time from observed HTTP GET traffic. The property it provides is
Observable-Read Isolation, described as a partial causal consistency over the HTTP-observable
read projection.

This attacks the assumption CoAgent needs and Ablo would also need: that footprints can be
declared. Inferring the read set from traffic the middleware already sees is a genuinely
different answer, and it is available to any system that sits on the request path, which Ablo
does. The property is deliberately partial, and the honesty about that partiality is what makes
it useful.

It also bears directly on a gap in premise validation. Validating only the row being written is
insufficient for cross-object invariants. If an actor reads `geo.image` and then creates
`geo-canary` from that value, the relevant premise is attached to `geo`, even though the write
target is `geo-canary`. Comparing the current version of the write target catches nothing. A
serious protocol has to model read dependencies or claims over premises. Whether the system then
rejects, retries, invalidates or asks a human is a policy choice built on top of that dependency
information, and it is a separate decision from acquiring it.

## A machine-checked anomaly hierarchy

[arXiv 2606.17182](https://arxiv.org/abs/2606.17182), Sajjad Khan, June 2026. Preprint.

This models agent state sharing as long-running read-generate-write operations under
deterministic-generation semantics, the regime durable-execution engines enforce through
deterministic replay, and formalises four concurrency anomalies in TLA+, each with a TLC
counter-example:

- **stale-generation**: a generation produced from state that has since changed
- **phantom-tool**: a tool appearing or disappearing across the operation
- **causal-cascade**: an invalidation propagating through dependent work
- **tool-effect reordering**: externalised effects landing out of order

These are structural analogues of the classical isolation anomalies. The paper builds a verified
hierarchy L0 to L4 with Verus obligations, realises the lower levels in deployed Rust runtimes,
and reproduces concrete failures in real systems: a silent lost update in ByteDance's deer-flow,
formalised as an L0 to L1 refinement, and tool-effect reordering in LangGraph's ToolNode on
unmodified output, removed by a commit-order sequencer.

The practical value for Ablo is a test battery rather than a design. Those four anomalies are
answerable questions about the current implementation, and reproduced failures in widely used
frameworks make them concrete rather than theoretical.

## STORM: mediated writes against branch and merge

[arXiv 2605.20563](https://arxiv.org/abs/2605.20563), Liu, Chen, Xu, Jiang and Dong, May 2026.
Preprint.

STORM mediates agent interactions with a shared workspace so each agent operates on a consistent
view and conflicting edits are detected at write time, rather than deferred to a post-hoc merge.
Its baseline is the current default in agent tooling: give each agent a git worktree and merge
afterwards. It reports gains over that baseline on Commit0-Lite and PaperBench, at comparable
or better cost.

This is the most direct empirical evidence available for Ablo's core architectural bet. Write
time mediation against isolate and merge is exactly the choice between a commit chokepoint and a
branch, and one paper reporting a gain is not settled science. Read it alongside the noise-floor
result in [evaluation-and-failure.md](evaluation-and-failure.md#how-much-coordination-gain-is-real)
before treating the margin as large.

## ATM: admission before the write

[arXiv 2607.00041](https://arxiv.org/abs/2607.00041), Eagl Huang, June 2026. Preprint.

ATM isolates a narrower problem: before any governed shared mutation is applied, something must
decide which concurrently formed write intents may proceed in parallel, which require
deterministic composition or serialisation, and which take a fail-closed path. It maps write
intents to semantic units, brokering admission by content identifier. The authors are explicit
that the evaluation is single-domain and claims no broad comparative superiority.

The framing is worth more than the results here. Admission control before the write is the same
position Ablo's commit chokepoint occupies, and the three-way split of parallel, serialise, or
fail closed is a cleaner statement of the decision than "resolve conflicts" ever is. See
[backpressure-and-overload](../domains/backpressure-and-overload.md) for the capacity side of
the same boundary.

## Convergence, coherence and branches

Three earlier directions, all still relevant as alternatives rather than complements:

- [CodeCRDT](https://arxiv.org/abs/2510.18893) (Pugachev, 2025, preprint) has agents observe a
  shared CRDT-backed coding state and coordinate through convergent updates rather than messages
  or locks. Across 600 trials, performance ranges from speedup to slowdown while all runs
  converge without merge failure. Convergence without merge failure is not the same as domain
  validity, authorisation, or serial equivalence, which is the standing limit on any CRDT answer
  ([physical-world-state](../domains/physical-world-state.md#the-relationship-to-convergence)).
- [Token Coherence](https://arxiv.org/abs/2603.15183) (Parakhin, 2026, preprint) maps MESI-style
  cache invalidation onto agent artifacts with lazy invalidation, monotonic versions and
  single-writer rules, model-checked in TLA+, to cut repeated full-state prompt
  synchronisation. It optimises the cost of keeping many private agent contexts coherent, which
  is a real cost Ablo also pays, though Ablo must additionally settle mutations and serve actors
  with no token context at all.
- [Effective Strategies for Asynchronous Software Engineering Agents](https://arxiv.org/abs/2603.21489)
  (Geng and Neubig, 2026, preprint) proposes CAID: centralised dependency-aware delegation,
  isolated workspaces, asynchronous execution, branch-and-merge integration, executable tests.
  It is the strongest form of the alternative Ablo is arguing against, and it is a good answer
  whenever state can be forked and merged later. Ablo matters where effects are live,
  continuously visible, cross-domain, or physically consequential, and cannot sit on a private
  branch.

## Structure, memory and governance

- [Provable Coordination for LLM Agents via Message Sequence Charts](https://arxiv.org/abs/2604.17612)
  (Bollig, Függer and Nowak, 2026, accepted at ISoLA 2026) projects a global message-sequence
  specification into deadlock-free local programs while leaving LLM calls, tools and human
  control points nondeterministic. It formalises safe message structure and control flow, which
  complements rather than replaces data consistency and settlement.
- [Multi-Agent Memory from a Computer Architecture Perspective](https://arxiv.org/abs/2603.10062)
  (Yu et al., 2026, position preprint) frames shared and distributed agent memory as an
  architecture problem and names cross-layer consistency as the most pressing gap.
- [Governed Shared Memory for Multi-Agent LLM Systems](https://arxiv.org/abs/2606.24535) (June
  2026, preprint) treats shared memory as a distributed-systems governance problem: scoped
  access, temporal correctness, provenance, synchronisation and policy-controlled propagation.
- [Always-On Agents](https://arxiv.org/abs/2606.30306) (Ding, Nannapaneni, Liu and Zhang, June
  2026, survey preprint) covers 435 works on persistent memory, state and governance, reading
  each state item along authority, scope, mutability, provenance, recoverability and
  actionability. Its headline finding is the useful one: the literature concentrates far more on
  accumulating and retrieving state than on governing, recovering or relinquishing it, and the
  agenda it draws connects to databases, distributed systems, formal methods and capability
  security. That is a description of the space Ablo is claiming, written by people counting
  papers rather than selling a product.
- [Isolation as a First-Class Principle for LLM-Agent System Safety](https://arxiv.org/abs/2607.12406)
  (Jing et al., July 2026, survey preprint) organises agent safety failures around five
  boundaries: user-agent, agent-tool, agent-execution, agent-agent, and system-environment. It
  argues that prompt injection, tool misuse and memory poisoning share a structural cause, which
  is the loss of isolation at one of those boundaries. Worth reading against
  [capabilities-and-evidence](../domains/capabilities-and-evidence.md), because Ablo's tenancy
  and capability model is an isolation claim at two of those five boundaries.

## Questions this raises

- Which of the four verified anomalies (stale-generation, phantom-tool, causal-cascade,
  tool-effect reordering) can be reproduced against Ablo today? That is a concrete afternoon's
  work and the result is worth knowing either way.
- Ablo sits on the request path, so S-Bus's argument applies directly: could read sets be
  reconstructed from traffic Ablo already sees, rather than requiring declaration?
- Where does Ablo's premise model attach a dependency: to the write target, or to the object
  that was read? If it is the target, cross-object premises are unguarded.
- CoAgent's guarantee holds at quiescence. Does an always-live workspace ever quiesce, and if
  not, what is the strongest guarantee available without it?
- Atomix separates "which effects settle together" from "when earlier conflicting work is
  exhausted". Are those two facts separable in Ablo's current commit path, or conflated?
- STORM's baseline is git worktrees. What is Ablo's honest baseline, and has anyone run it?
- If a customer's agents can genuinely work on private branches and merge later, is Ablo the
  wrong tool for them? Naming that customer clearly is worth more than winning the argument.
