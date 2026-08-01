# 04. What has actually been measured

This is the single home for measured numbers. Other files link here rather than repeating them.
Every figure comes from the dated benchmark handoffs in the repository
([10-repo-map.md](10-repo-map.md)); when they disagree with this file, they win. How to read
any of it is in [07-measuring-it.md](07-measuring-it.md).

## The gate

The strict acceptance target on real Aurora:

- at least **100,000 committed deltas per second**
- zero errors
- an exact final cursor
- no duplicate or reordered replay
- final observer drain no greater than **150 ms**

The topology behind those numbers: 48 clients and workers, 7 server processes, 500 operations
per commit, a 20-second measured interval, a segmented log, 2 commit lanes, a 100,000-delta/sec
offered load, sharded writers, a delivery thread, an isolated observer, a 4096 MiB heap, and
partial indexes on the task relation where retained.

This is a defined gate under one topology and one contract. It is not a claim that every
workload shape has cleared it.

## The most recent retained result

Runs 135 and 136 compared the expanded publication object passed between worker and host
against a compact object that preserves the exact external wire representation.

| Run | Worker-host representation | Throughput | Final drain | Ack p95 | Result |
| --- | --- | ---: | ---: | ---: | --- |
| 135 | expanded | 98,394/sec | 267 ms | 356 ms | control |
| 136 | compact | 99,986/sec | 141 ms | 343 ms | retained |

Higher throughput by 1.62 percent, final drain lower by 47.2 percent, lower ack p95, zero
errors, exact final cursor. It missed 100,000/sec by 14 deltas per second, so it was retained
for its mechanism rather than reported as a passed threshold. That distinction is the house
standard and it matters: the run cleared the drain gate and missed the writer gate by 0.014
percent, and saying so is cheaper than defending a rounded number later.

The mechanism is visible in profiling, which is why it was believed:

| Measure | Expanded | Compact |
| --- | ---: | ---: |
| representative 500-UPDATE publication object | 302,873 bytes | 137,313 bytes |
| local structured-clone time | 738 ms | 303 ms |
| profiled cloned bytes per delta | 571.7 | 315.4 |
| worker `postMessage` time | 3,260 ms | 1,842 ms |
| host arrival processing | 10,820 ms | 6,011 ms |
| credit handling | 1,987 ms | 1,319 ms |

The external wire representation was deliberately left unchanged, which isolates an internal
transport cost from any change in client semantics. The reasoning behind representation work is
in [domains/representation-and-memory.md](domains/representation-and-memory.md).

## Other shapes

From an earlier four-shape campaign, not all under the final unified topology and drain
instrumentation:

| Shape | Throughput |
| --- | ---: |
| DELETE | approximately 101,641/sec |
| CREATE | approximately 100,861/sec |
| MIXED | approximately 95,859/sec |
| UPDATE | low to mid 90,000s before the later improvements |

These are directional. Combining them into an all-shapes pass would be false. MIXED remains the
clearest shape below the gate.

Aurora's logical-WAL cache measured an effectively 100 percent hit rate with 1 GiB configured,
so the current limit is not explained by cache misses on that path.

## Where UPDATE cost actually goes

The strongest database-side profile is run 118:

- application SQL latency approximately 338 ms p50, 416 ms p95
- the physical task update alone approximately 321 ms p50, 400 ms p95
- dominant wait class `LWLock:BufferContent`

Run 121 removed every secondary index and reached approximately 100,384/sec. That is causal
proof, not a solution: it destroys the read contract. The evidence points at physical write
amplification on indexed mutable columns rather than a generic need for a larger database. Why
that happens at the storage layer is in
[domains/postgres-under-hot-updates.md](domains/postgres-under-hot-updates.md).

The leading production-safe proposal is a vertical split:

```text
task_identity_and_routing    stable identity, ownership, tenant and routing keys, indexed
task_mutable_state           frequently changing fields, minimal or no secondary indexing
task_query_projection        explicitly measured query dimensions, freshness policy stated
```

The hypothesis is that this keeps the required reads while narrowing the hot update relation,
reducing secondary-index maintenance, buffer contention, tuple churn, WAL volume and vacuum
pressure. It is untested.

## The rejected experiment that taught the most

Runs 124 to 128 tested checkpoint overlap, smaller checkpoints, and independently progressing
publication groups.

| Run | Throughput | Final drain | Ack p95 |
| --- | ---: | ---: | ---: |
| control | 99,851/sec | 417 ms | 299 ms |
| candidate | 98,435/sec | 1,800 ms | 334 ms |

Slower by 1.4 percent with drain 4.3 times worse. Both arms had zero errors and exact cursors,
which is exactly why average throughput alone would have hidden the regression. The
implementation was removed.

This is the single most useful negative result in the history. It rules out the idea that
independently progressing publication groups or more frequent checkpointing automatically
remove the serial cost. They can add coordination and tail-drain overhead while leaving the
limiting stage untouched. The queueing reason is in
[domains/backpressure-and-overload.md](domains/backpressure-and-overload.md).

Two related closed branches: run 123 (checkpoint every 16 deltas) reached 100,515/sec with
1,946 ms drain and was rejected on drain. Runs 129 to 132 replaced generic WAL hydration with
direct typed hydration for a 25.1 percent drain reduction at neutral throughput, and it was
retained.

## Rejected and exhausted directions

Repeating any of these without a new mechanism is not evidence-generating:

- adding more writers or database connections
- ordinary multi-server replication of one hot plane
- hash-partitioning the task table without solving access and ordering cost
- fused self-join or CTE UPDATE forms
- transaction pipelining that weakens or obscures the settlement boundary
- native array update encoding in the tested path
- wider rows
- `wal_buffers` tuning as the primary answer
- fill factor and HOT-only tuning as a complete answer
- more client-envelope parameter sweeps
- a larger Aurora class while database CPU is not saturated
- compact positional external JSON, which trades maintainability and compatibility for a small
  transport win

Negative results belong in the system model. They constrain the hypothesis space, which is
worth more than any single passing run.

## Can Ablo serve customers today

Yes, for bounded and closely operated workloads. The benchmark is not a production-readiness
claim, and the honest split is this:

| Dimension | Present strength | Remaining work before a broad guarantee |
| --- | --- | --- |
| Mediated write correctness | one commit chokepoint, idempotency, claims, fencing, typed conflicts, receipts, WAL confirmation | keep collapsing duplicate contracts; exercise every retry and failure boundary continuously |
| Customer data ownership | customer Postgres stays authoritative, with separated least-privilege roles | provider-specific onboarding, privilege drift detection, uniform operational support |
| Ordered real-time observation | strong implementation, exact-cursor evidence | sustained outage curves, slow-subscriber policy, resnapshot cost, provider failover |
| Single-plane capacity | near 100,000 UPDATE deltas/sec under the documented high-batch workload | one unified pass across all shapes; small transactions, skew, read load, sustained maintenance |
| Multi-plane scale | a natural architectural partition boundary | automated ownership, isolation, rebalancing, quotas, control-plane failure behaviour |
| Headless agent and service use | the transaction and capability concepts exist | make the headless surface the default product contract, not an adjacent path |
| Cross-organisation coordination | a clear protocol direction | signed evidence, trust establishment, revocation, dispute handling, private proof |
| Physical-world control | a coherent model can be defined | actuation and observation semantics, edge operation, safety boundaries, device identity |
| Cost efficiency | disposable benchmark infrastructure and teardown discipline | translate capacity into cost per useful delivered transition on production-shaped instances |

For most early customers, 100,000 deltas/sec is not the requirement. Correct integration,
reliable recovery, predictable p99 freshness and real operational support matter more. The
performance work still matters strategically, because it establishes headroom and exposes
serial assumptions before a customer workload makes them expensive to change.

## Questions this raises

- Run 136 missed the writer gate by 14 deltas/sec. What is the run-to-run variance of that
  metric, and how many repetitions would it take to call the gate passed honestly?
- The topology uses 500 operations per commit. What does the same code do at 1 and at 10
  operations per commit, and which stage becomes dominant there?
- `LWLock:BufferContent` dominates. Which pages, and is the contention on the heap, the index,
  or both?
- The vertical split is a hypothesis. What is the smallest experiment that could falsify it,
  and what read regression would make it not worth doing?
- Drain is measured at the end of a 20-second interval. What does drain look like at 10 minutes
  and at 6 hours, with autovacuum and checkpoints in play?
- Which of the rejected directions were rejected on this topology only, and could return under
  a different unit of work?
