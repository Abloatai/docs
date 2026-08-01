# Bounded backpressure and overload recovery

Average throughput can look healthy while queues fill and drain explodes. Runs 124 to 128 are
the local proof.

| | Control | Candidate | Change |
| --- | ---: | ---: | ---: |
| Throughput | 99,851/sec | 98,435/sec | 1.4% slower |
| Final drain | 417 ms | 1,800 ms | **4.3x worse** |
| Ack p95 | 299 ms | 334 ms | 11.7% worse |
| Errors | 0 | 0 | tie |
| Final cursor | exact | exact | tie |

Throughput alone would have called that a tie. Detail in
[04-the-evidence.md](../04-the-evidence.md#the-rejected-experiment-that-taught-the-most).

## Little's law explains the whole failure mode

```text
L = λW        average work in system = arrival rate × residence time
```

Two systems sustain identical throughput while one accumulates work. Constant λ with rising L
means rising W, and W is drain. The health signal is bounded backlog and predictable recovery,
never a steady-state rate ([Little, 1961](https://pubsonline.informs.org/doi/10.1287/opre.9.3.383)).

## What to capture per stage

| Metric | Why |
| --- | --- |
| Arrival rate | the λ in the law |
| Service-time distribution | the mean hides the tail that fills the queue |
| Queue depth, items **and bytes** | items is the wrong unit when payload width varies |
| Residence time | the W, and the thing a customer feels |
| Utilisation | distance to the knee |
| Drop, disconnect or spill policy | what happens at the bound, stated rather than emergent |
| Recovery rate after overload | a separate measurement from steady state, rarely instrumented |

## Backpressure crosses a boundary in both directions

Downstream, a slow subscriber cannot be allowed to become another customer's latency. Upstream,
the customer's own database can be the slow component while Ablo's replication slot holds WAL on
their disk ([slot retention](logical-decoding-and-cdc.md#what-the-database-gives-you)). An
unbounded queue in Ablo becomes a disk-space incident in a database Ablo does not own.

Overload policy is a product decision, not a tuning parameter. In a multi-tenant system the
question is always which customer absorbs the delay, and answering it implicitly means answering
it badly.

## See it yourself

Little's law is falsifiable in one afternoon. Hold offered load constant, measure queue depth
and residence time every second, and plot L against λW. When they diverge, the system is
accumulating work that throughput is not reporting.

The same shape appears in Postgres: hold a transaction open, watch `safe_wal_size` fall while
commit throughput stays flat. Nothing looks wrong until the slot is invalidated.

## Go deeper

| Read | For |
| --- | --- |
| [Little's law](https://pubsonline.informs.org/doi/10.1287/opre.9.3.383) | the two-page proof behind every queueing intuition |
| [How Not to Measure Latency](https://qconsf.com/sf2012/dl/qcon-sanfran-2012/slides/GilTene_HowNotToMeasureLatency.pdf) | coordinated omission, and why a stalled system reports excellent percentiles |
| [Flink checkpointing](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/fault-tolerance/checkpointing/) | how a streaming system bounds recovery work rather than queue depth |

## Still open

- Where the offered-load knee is under open-loop arrivals, relative to the 100,000/sec gate.
- What happens at the bound for a slow subscriber: disconnect, spill, or coalesce, and whether
  the answer differs for an event-complete and a state-convergent consumer.
- How long recovery takes after a 10x burst ends, which is a number almost nobody instruments.
- Which resource a noisy neighbour contends for first: connections, CPU, memory or socket
  bandwidth.
