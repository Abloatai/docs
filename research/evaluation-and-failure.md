# How these systems fail, and how much of the gain is real

Read this before designing any experiment that claims a coordination layer helped. Two of these
results cut against claims Ablo would naturally want to make, which is why they are here.

## How much coordination gain is real

[Paired Noise-Floor Protocol](https://arxiv.org/abs/2606.20695), Kaliyev and Maryanskyy, June
2026, preprint. It asks a prior question: how much do two protocols disagree on the same model
and benchmark when their inputs are configuration-equivalent, verified by code inspection and a
byte audit.

| On Claude Haiku 4.5, tau-squared-bench retail | Result |
| --- | --- |
| Two protocols, both inert at trial 0, two seeds | +10pp and 0pp |
| Pooled | +5pp, Wilson CI [-2, +12], not significant |
| Largest single-seed contrast | +18pp (p 0.012), reproduced at seed 2 as **-3pp** |
| Observed envelope across seeds | -3pp to +18pp |
| Recent architectures reporting effects **below** this floor | **7 of 10**, with one more inside the envelope |

One model, one benchmark, and the floor elsewhere may differ. It still sets a standard: a
coordination result with a single-digit margin and no paired replication across seeds is
reporting noise until proven otherwise. Same discipline as the rule against re-rolling a
throughput benchmark ([07-measuring-it.md](../07-measuring-it.md#mechanism-backed-evidence)).

## Protocol structure dominates model capability

[DPBench](https://arxiv.org/abs/2602.13255), Hasan and BusiReddyGari, February 2026, revised
June, preprint. Dining philosophers adapted so agents contend for the same resources
simultaneously, varying communication protocol, network structure and group size independently
across six models including GPT-5.2, Claude Opus 4.5 and Gemini 2.5 Flash.

| Condition | Deadlock rate |
| --- | ---: |
| Single round of pre-commitment communication | 86.7% |
| Three rounds of pre-commitment | 0.0% |
| Minimal prompts | approaching 100% |
| With concurrency primitives | near zero |

If it holds, this is the strongest available argument that a coordination layer is a product:
the failure is structural, so a better model does not fix it and a protocol does. It is equally
a warning about attribution, since a benchmark that changes protocol and model together cannot
say which moved the number.

## Why they fail

| Source | Evidence |
| --- | --- |
| [MAST](https://arxiv.org/abs/2503.13657) | 14 failure modes from 1,600+ annotated traces across seven frameworks, in three groups: system design, inter-agent misalignment, verification |
| [SILO-BENCH](https://arxiv.org/abs/2603.01045) | thirty algorithmic tasks over distributed information. Agents often exchange enough information and still fail to integrate it, and coordination overhead eventually erases the parallel gain |

The counterweight to every architecture paper, including this one: reliable transport does not
compensate for ambiguous roles, bad delegation, missing verification, or agents that receive
what they need and fail to use it.

## Evaluate the layers separately

| Layer | Question | Who owns it |
| --- | --- | --- |
| 1. Transport and settlement | was every authorised transition committed, ordered, published, acknowledged, recoverable? | the substrate |
| 2. View coherence | did each actor get the state and invalidations it needed before acting? | the substrate |
| 3. Coordination behaviour | did actors assign work, honour dependencies, respond to conflicts? | shared |
| 4. Task outcome | was the final world state correct under domain invariants? | the application |
| 5. Efficiency | wall time, tokens, bytes, retries, compensations, conflicts, human interventions | shared |

Without the separation, an experiment credits the substrate for a model's reasoning, or blames a
correct substrate for an agent's failure to use what it was given.
[Coordination as an Architectural Layer](https://arxiv.org/abs/2605.03310) argues the same from
the other direction: hold model, tools, prompts and information access fixed, vary only the
coordination configuration. The [Always-On Agents](https://arxiv.org/abs/2606.30306) survey
proposes AOEP-v0, a pilot contract scoring state mutation and recovery obligations rather than
answer quality, which is layers 1 and 2 made scoreable.

## Go deeper

| Read | For |
| --- | --- |
| [Paired noise floor](https://arxiv.org/abs/2606.20695) | the standard any coordination benchmark now has to clear |
| [DPBench](https://arxiv.org/abs/2602.13255) | structure against capability, measured |
| [MAST](https://arxiv.org/abs/2503.13657) | the failure taxonomy, from real traces |
| [Always-On Agents](https://arxiv.org/abs/2606.30306) | an evaluation contract for state, not answers |

## Still open

- The noise floor of any Ablo coordination benchmark, measured with paired
  configuration-equivalent runs on one model. Without it, no result from that benchmark can be
  interpreted.
- Which MAST failure modes a substrate can prevent, against those that stay the application's.
  Being honest about the second list is what makes the first credible.
- Whether DPBench's result holds when the shared state is a real database rather than a
  synthetic resource.
- Coordination overhead per agent, in tokens and wall time, and the agent count at which it
  erases the parallel gain.
