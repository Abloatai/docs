# How these systems fail, and how much of the gain is real

Read this before designing any experiment that claims a coordination layer helped. Two of the
results here are direct threats to claims Ablo would naturally want to make, which is exactly
why they belong in the pack.

## How much coordination gain is real

[How Much Coordination Gain Is Real? A Paired Noise-Floor Protocol for Multi-Agent LLM
Benchmarks](https://arxiv.org/abs/2606.20695), Kaliyev and Maryanskyy, June 2026. Preprint.

The authors ask a prior question: how much paired disagreement do two coordination protocols
produce on the same model and benchmark when their inputs are configuration-equivalent, verified
by code inspection and a byte-level audit. On Claude Haiku 4.5 against tau-squared-bench retail,
two protocols that are inert at trial zero still produce signed paired gaps of +10 and 0
percentage points across two seeds, pooling to +5 points with a Wilson interval of -2 to +12,
not significant. The largest single-seed contrast, +18 points, failed to reproduce at the second
seed, coming back at -3. The observed envelope spans -3 to +18 points.

The conclusion is the uncomfortable part. Seven of ten recent multi-agent coordination
architectures report headline effects below that local noise floor, and one more sits inside the
envelope. Whether any of them survives a same-model paired replication is untested in their
original settings.

This is one model on one benchmark, and the floor elsewhere may differ. It still sets a
standard. Any Ablo coordination result that reports a margin in single-digit percentage points,
without paired replication across seeds, is reporting noise until proven otherwise. The house
rule against re-rolling a benchmark for a lucky pass
([07-measuring-it.md](../07-measuring-it.md#mechanism-backed-evidence)) is the same discipline
applied to throughput, and it should transfer to coordination benchmarks without argument.

## Protocol structure dominates model capability

[DPBench: Structural Determinants of Multi-Agent LLM Coordination Under Simultaneous Resource
Contention](https://arxiv.org/abs/2602.13255), Hasan and BusiReddyGari, February 2026, revised
June 2026. Preprint.

DPBench adapts the dining philosophers problem so that agents contend for the same resources
simultaneously, then varies communication protocol, network structure and group size
independently. Across six models including GPT-5.2, Claude Opus 4.5 and Gemini 2.5 Flash, the
finding is that protocol design dominates model capability: three rounds of pre-commitment
communication produced a 0.0 percent deadlock rate where a single round produced 86.7 percent,
and minimal prompts approached total failure, while concurrency primitives drove deadlock to
near zero.

If that result holds, it is the strongest available argument for the existence of a coordination
layer as a product. It says the failure mode is structural rather than cognitive, so a better
model does not fix it and a protocol does. It is also a warning about attribution: a benchmark
that changes the protocol and the model at once cannot tell you which one moved the number.

## Why multi-agent systems fail

[Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/abs/2503.13657) introduces the MAST
taxonomy from more than 1,600 annotated traces across seven frameworks, sorting 14 failure modes
into system design, inter-agent misalignment, and verification.

It is the necessary counterweight to every architecture paper, including this one. Reliable
transport does not compensate for ambiguous roles, bad delegation, missing verification, or
agents that receive relevant information and fail to use it. A perfect coordination substrate
can still host a system that fails on every task.

[SILO-BENCH](https://arxiv.org/abs/2603.01045) (Zhang et al., 2026, accepted at ACL 2026) makes
the same point quantitatively with thirty algorithmic tasks over distributed information,
reporting a communication and reasoning gap: agents often exchange enough information and still
fail to integrate it, and coordination overhead eventually erases the parallel gain.

## Evaluate the layers separately

Which produces the practical requirement. An evaluation of an Ablo-backed system has to measure
these independently, or it will credit the substrate for a model's reasoning, or blame a correct
substrate for an agent's failure to use what it was given.

1. **Transport and settlement**: was every authorised transition committed, ordered, published,
   acknowledged and recoverable?
2. **View coherence**: did each actor receive or retrieve the state and invalidations it needed
   before acting?
3. **Coordination behaviour**: did actors assign work, honour dependencies and respond to
   conflicts correctly?
4. **Task outcome**: was the final world state correct under domain invariants?
5. **Efficiency**: what wall time, tokens, bytes, retries, compensations, conflicts and human
   interventions were consumed?

[Coordination as an Architectural Layer](https://arxiv.org/abs/2605.03310) (Nechepurenko and
Shuvalov, 2026, preprint) argues the same separation from the other direction: hold model,
tools, prompts and information access fixed, and vary only the coordination configuration.
That is the experimental design the layer-separation above implies.

The [Always-On Agents](https://arxiv.org/abs/2606.30306) survey proposes a pilot evaluation
contract, AOEP-v0, that scores state mutation and recovery obligations rather than answer
quality. Layers 1 and 2 above are exactly what it tries to make scoreable, and it is worth
reading before building an evaluation harness from scratch.

## Questions this raises

- What is the noise floor of Ablo's own coordination benchmark, measured with paired
  configuration-equivalent runs on one model? Until that number exists, no coordination result
  from it can be interpreted.
- Which of the 14 MAST failure modes can Ablo's substrate prevent, and which are the
  application's problem? Being honest about the second list is what makes the first list
  credible.
- DPBench suggests structure beats model capability. Does that hold on a task where the shared
  state is a real database rather than a synthetic resource?
- Layer 1 (transport and settlement) is the only layer Ablo fully controls. Is it measured
  separately today, or only through end-to-end task success?
- What is the coordination overhead per agent, in tokens and in wall time, and at what agent
  count does it erase the parallel gain?
- If a customer's agents fail on a task, what does Ablo let them see that tells them which layer
  failed? That diagnostic may be worth more commercially than the coordination itself.
