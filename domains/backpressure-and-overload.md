# Bounded backpressure and overload recovery

## The problem

Average throughput can look healthy while queues fill and final drain explodes. Runs 124 to 128
are the local proof: the candidate held 98,435/sec, within 1.4 percent of the control, with zero
errors and an exact cursor, while final drain went 4.3 times worse. Throughput alone would have
called that a tie. Detail in
[04-the-evidence.md](../04-the-evidence.md#the-rejected-experiment-that-taught-the-most).

## What the field knows

**Little's law explains the whole failure mode.** For a stable system, average in-system work
equals arrival rate multiplied by average residence time:

```text
L = λW
```

Two systems can sustain identical throughput while one accumulates work and the other does not.
Constant λ with rising L means rising W, which is drain. It is why the right health signal is
bounded backlog and predictable recovery rather than a steady-state rate, and why a benchmark
that reports only throughput cannot distinguish a healthy system from one that is quietly
falling behind.

Model each stage as a queueing network and capture arrival rate, service-time distribution,
queue depth in items and in bytes, residence time, utilisation, the drop or disconnect or spill
policy, and the recovery rate after the overload ends.

**Overload policy is a product decision, not a tuning parameter.** The mechanisms are standard:
per-stage credits, explicit service-level budgets, durable fallback, and admission control early
enough that work is refused before it is half-done. What varies is who the policy protects. In a
multi-tenant system the question is always which customer absorbs the delay, and answering it
implicitly means answering it badly.

## The Ablo-specific edge

Backpressure here crosses an administrative boundary in both directions. Downstream, a slow
subscriber cannot be allowed to become another customer's latency. Upstream, the customer's own
database is an independent system that can be the slow component, and Ablo's replication slot is
holding WAL on their disk while it happens
([logical-decoding-and-cdc.md](logical-decoding-and-cdc.md)). An unbounded queue in Ablo becomes
a disk-space incident in a database Ablo does not own.

## Questions this raises

- What is the offered-load knee, measured with open-loop arrivals? Where is it relative to the
  100,000/sec gate?
- What is the queue depth in bytes, not items, at each stage? Items are the wrong unit when
  payload width varies.
- What happens at the bound for a slow subscriber: disconnect, spill, or coalesce? Is the policy
  the same for an event-complete consumer and a state-convergent one?
- Is there admission control at the commit boundary today, and what does a rejected commit look
  like to an agent that will immediately retry?
- How long does recovery take after a 10x burst ends? Recovery rate is a separate measurement
  from steady-state rate and is rarely instrumented.
- Which resource does a noisy neighbour actually contend for first: database connections, CPU,
  memory, or socket bandwidth?
- If the customer's database is the slow component, what does Ablo do, and at what point does it
  tell someone?
