# 07. How to read a performance claim, including ours

The standard by which the numbers in [04-the-evidence.md](04-the-evidence.md) should be read.
It applies to Ablo's own runs first.

## Mechanism-backed evidence

A strong experiment declares, before it runs:

1. the observed bottleneck
2. the proposed causal mechanism
3. the metric expected to move
4. the metric that must not regress
5. the correctness risks
6. the control and candidate configurations
7. the number of repetitions and the decision rule

Running a candidate repeatedly until random variation produces a passing row is selection bias
with extra steps. The house rule is that a candidate is never re-rolled for a lucky pass, and
the reason run 136 is described as retained rather than passed is exactly this: the mechanism
was visible in profiling, the drain gate cleared, the writer gate did not, and both facts get
reported.

## Open loop and closed loop

Closed-loop clients slow down when the system slows, which makes an overloaded system look
stable. Open-loop scheduling keeps arrivals independent of completions and exposes the
saturation knee by showing offered load against achieved load.

Latency measurement has to avoid coordinated omission, where the load generator stops issuing
requests during a stall and therefore never records the requests that would have waited longest.
A system that stalls for two seconds and reports a 5 ms p99 is reporting the requests it managed
to send, not the experience of the requests it did not.

End-to-end timing needs stage timestamps: enqueue, admission, database commit, WAL observation,
publication, socket delivery, client application, final cursor.

## Workload dimensions

A complete claim names its position on each axis. The current 500-operation profile is one
point on this grid and cannot stand alone as a capacity claim.

| Dimension | Representative values |
| --- | --- |
| Mutation shape | CREATE, UPDATE, DELETE, realistic MIXED |
| Transaction size | 1, 10, 100, 500 operations |
| Conflict rate | zero, low, medium, high, single hot key |
| Key distribution | uniform, Zipfian, moving hotspot |
| Payload | narrow, representative, wide with TOAST |
| Planes | one hot plane, many cold planes, mixed tenancy |
| Subscribers | 1, 10, 100 or more, varied selectivity |
| Consumer behaviour | healthy, slow, disconnected, reconnecting |
| Duration | short diagnostic, sustained through checkpoint and vacuum, soak |
| Failure | server crash, decoder crash, database failover, slot loss, network partition |

## Resource currencies

Throughput alone underdetermines whether a change is good:

CPU time per committed delta by stage, allocations and garbage collection, bytes copied between
worker and host, internal and external serialised bytes, database rows and indexes touched, WAL
bytes and retained WAL lag, buffer and lock waits, queue depth and residence time, network bytes
and socket backpressure, client apply and persistence time, and cost per million settled and
delivered deltas.

## Correctness evidence

Performance evidence is worth more when it arrives with:

- property-based histories for idempotency and ordering
- model-based tests of claims, leases and fencing
- crash injection at every persistence and acknowledgement boundary
- replay equivalence between snapshot-plus-tail and uninterrupted execution
- duplicate, delay, loss and reordering injection on internal channels
- failover tests where a stale owner attempts to continue
- long-running bounded-memory assertions
- invariants checked from independent observer state rather than server counters

For the subset of operations that claims linearizability or serializability, a history checker
is appropriate. For operations whose contract is causal or frontier-based, a global
linearizability check is the wrong instrument and will produce confident nonsense.

## When a change is retained

Evidence is strongest when the intended mechanism is visible in profiling, a predeclared primary
metric moves beyond noise, every correctness gate passes, no unbounded queue appears elsewhere,
the public contract is preserved or explicitly revised, the result survives a sustained run and
a relevant failure test, and the added complexity is justified by the gain.

## Questions this raises

- Is the current benchmark open-loop or closed-loop, and where is that decided in the driver?
- Which stage timestamps exist today, and which would have to be added to attribute drain?
- What is the measured run-to-run variance of throughput and of final drain on this topology?
- Has any result been produced under injected failure rather than a clean run?
- Which correctness properties are checked from observer state rather than server counters?
- What is the cost, in dollars, of one full benchmark campaign, and does that shape which
  experiments get run?
