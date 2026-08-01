# 06. One million per second is three different targets

## First, define the unit

| Ablo's benchmark unit | Value |
| --- | --- |
| Operations per commit | 500 |
| So 1,000,000 deltas/sec is | about 2,000 commits/sec |
| At 1 operation per commit, the same delta count is | 1,000,000 commits/sec |

TigerBeetle is the clearest illustration in the field because it says so out loud: its rates are
per transfer, transfers batch up to a server-configured 8,189 events per request to amortise
consensus and I/O, inserting a million one at a time yields a fraction of the potential rate,
and batches shrink under light load to favour latency
([requests](https://docs.tigerbeetle.com/coding/requests)). The batch is not an implementation
detail, it is why the number exists.

Every result should therefore state: deltas/sec, transactions/sec, rows/sec, bytes/sec at each
handoff, operations per transaction, mutation-shape mix, payload and row width, index set, plane
count, subscriber count and selectivity, durability and receipt boundary, final drain and
steady-state lag, and failure and recovery behaviour.

## The three targets

| Target | What it is | Hardest part |
| --- | --- | --- |
| **Aggregate** | 1M/sec over 100 planes at 10k each | partitioning and operations |
| **One hot plane** | 1M/sec with one order and one cursor | the serial fraction |
| **Delivered** | 1M/sec reaching each subscriber | bytes, and who needs to hear it |

### Aggregate

Reachable by horizontal scaling, if plane assignment is stable and rebalancing is fenced,
metadata lookup does not become the new global bottleneck, per-plane accounting and fairness
exist, control-plane failure does not interrupt existing data-plane owners, no shared audit or
billing table serialises all planes, and sources and subscribers can reconnect to the current
owner safely.

### One hot plane

Needs either a sequencer sustaining the whole rate, or a change to the ordering model so
independent transitions progress without one global serialisation point. Candidate models and
costs: [ordering and frontiers](domains/ordering-and-frontiers.md).

The partitioning function has to follow the conflict graph rather than hashing bytes evenly. Two
updates to unrelated lamps need no mutual order. A transfer between two accounts does. A robot
reserving a shared corridor may conflict with several entities at once.

### Delivered

| At 1,000,000 deltas/sec | Bytes/sec | Bits/sec |
| --- | ---: | ---: |
| Internal clone, at the measured 315.4 B/delta | 315.4 MB | 2.52 Gbit |
| One full-stream observer, illustrative 270 B | 270 MB | 2.16 Gbit |
| Ten observers | 2.7 GB | 21.6 Gbit |
| One hundred observers | 27 GB | 216 Gbit |

Before TLS, WebSocket, TCP and retransmission. The answer is subscription-aware hierarchical
publication, not a bigger network card
([fan-out](domains/fanout-and-incremental-views.md)).

## Regime A: aggregate, 100k to 1M

```text
                 control plane
              mapping + lease epochs
                      |
       +--------------+--------------+
       |              |              |
    owner A         owner B        owner C
    planes 1..n     planes n..m    planes m..k
       |              |              |
   DB/WAL/log     DB/WAL/log      DB/WAL/log
       |              |              |
   regional fan-out and subscriber-specific routing
```

Concerns: monotonic ownership epochs, zero-overlap or explicitly fenced handoff, continuation
during control-plane outages, per-plane admission control and cost accounting, automated
hot-plane isolation, bounded rebalancing that does not replay a whole source, and multi-tenant
fairness across CPU, database, memory and network.

## Regime B: one hot plane at 1M

```text
commit T
  read/write set or declared coordination keys
                  |
                  v
        deterministic lane assignment
          /          |          \
       lane 0      lane 1      lane 2
       epoch/e0    epoch/e0    epoch/e0
       offset/a    offset/b    offset/c
          \          |          /
                  frontier
          {lane0:a, lane1:b, lane2:c}
```

Single-lane transactions execute and publish independently. Cross-lane transactions take a
deterministic order, barrier or reservation. The consumer's position becomes a frontier rather
than an integer.

The unresolved problems, in rough order of difficulty: declaring or inferring coordination keys
without expensive runtime read-set tracking, whether capabilities and claims can double as
partition hints, cross-lane deadlock, frontier compactness as lanes split and merge, snapshot
acquisition at a frontier, moving subscriptions between fan-out workers without gaps, serving
scalar-cursor clients from a deterministic merge, recovery when one lane lags, and which
operations need a global barrier.

## Regime C: geographically distributed

Latency makes one synchronous leader wrong for most physical and interactive actions. The shape
is hierarchical: local safety and control loops, regional coordination, global policy and audit,
explicit ownership transfer between regions, operation under disconnection, and reconciliation
afterwards with authority preserved.

A warehouse robot takes a fenced lease over a zone from a regional authority, moves at local
latency inside it while emitting observations, and a global service audits and revokes future
authority without approving each motor command. The lease epoch is what stops a disconnected
controller from believing it still owns the zone.

## Go deeper

| Read | For |
| --- | --- |
| [TigerBeetle requests](https://docs.tigerbeetle.com/coding/requests) | how a headline number is constructed, stated honestly |
| [Kafka design](https://kafka.apache.org/documentation/#design) | partition-local order as the shipped compromise |
| [Spanner](https://research.google.com/archive/spanner-osdi2012.pdf) | the machinery global consistency actually requires |
| [FoundationDB architecture](https://apple.github.io/foundationdb/architecture.html) | roles scaling independently under one transaction contract |

## Still open

- Which target the business needs first, since the answer changes what gets built next.
- The largest single plane a real customer would produce, and the rate it implies.
- The cost per million delivered deltas at the measured topology.
- Whether any customer workload genuinely requires a scalar cursor, and what breaks for them
  under regime B.
