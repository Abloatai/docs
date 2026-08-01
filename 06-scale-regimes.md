# 06. One million per second is three different targets

## First, define the unit

No throughput claim means anything without the unit, workload, ordering domain, fan-out,
payload, durability point and observation point. The current Ablo benchmark uses 500 operations
per commit, so 1,000,000 deltas/sec is about 2,000 commits/sec. A workload with one operation
per transaction has radically different commit, WAL, acknowledgement and network costs at the
same delta count.

TigerBeetle is the clearest illustration in the field, because it is explicit about it. Its
headline rates are per transfer, and transfers are batched up to a server-configured maximum of
8,189 events per request, specifically to amortise the cost of consensus and I/O across the
batch. Its own documentation says that inserting one million transfers one at a time yields a
fraction of the potential rate, and that batch sizes shrink automatically under light load to
favour latency. The batch is not an implementation detail, it is the reason the number exists.

Ablo's 500 operations per commit is the same trick. Stating it plainly is the difference between
an honest benchmark and a misleading one.

Every result should therefore state: deltas/sec, transactions/sec, rows/sec, bytes/sec at each
handoff, operations per transaction, mutation-shape distribution, payload and row width, index
set, number of planes, number and selectivity of subscribers, durability and receipt boundary,
final drain and steady-state lag, and failure and recovery behaviour.

## Target 1: aggregate throughput across independent planes

One million deltas/sec spread over 100 independent planes at 10,000/sec each is primarily a
partitioning and operations problem. Each plane can have its own owner, database, WAL source,
cursor, queue and failure domain.

This is commercially important and reachable by horizontal scaling, provided that:

- plane assignment is stable and rebalancing is fenced
- metadata lookup does not become the new global bottleneck
- per-plane resource accounting and fairness exist
- control-plane failure does not interrupt existing data-plane owners
- no shared audit, log or billing table serialises all planes
- sources and subscribers can reconnect to the current owner safely

It is not the same problem as one hot shared world, and conflating the two is the most common
way to overstate capacity.

## Target 2: one hot plane at one million per second

Maintaining one total order, one scalar cursor and low drain at that rate requires either a
sequencer and data path that sustains the whole rate, or a change to the ordering model so
independent transitions progress without a single global serialisation point.

The second is more promising and it changes client-visible semantics. Candidate models and
their costs are in [domains/ordering-and-frontiers.md](domains/ordering-and-frontiers.md).

The partitioning function has to follow the conflict graph rather than hashing bytes evenly.
Two updates to unrelated lamps need no mutual order. A transfer between two accounts does. A
robot reserving a shared corridor may conflict with several entities at once.

## Target 3: delivered and applied throughput

One million committed deltas/sec does not imply one million can be delivered to each client.
Fan-out dominates almost immediately.

Using the measured compact worker-host representation of
[315.4 bytes per delta](04-the-evidence.md#the-most-recent-retained-result):

```text
1,000,000 deltas/sec * 315.4 bytes = 315.4 MB/sec of internal cloned data
                                   = approximately 2.52 Gbit/sec before framing and runtime overhead
```

For an illustrative narrow application payload of 270 bytes per delivered delta:

```text
270 MB/sec = approximately 2.16 Gbit/sec per full-stream observer
```

Ten full-stream observers would need roughly 21.6 Gbit/sec before TLS, WebSocket, TCP and
retransmission overhead. One hundred would need roughly 27 GB/sec of payload. The conclusion is
not to buy a larger network interface. It is that publication at this scale must be
subscription-aware and hierarchical, which is
[domains/fanout-and-incremental-views.md](domains/fanout-and-incremental-views.md).

The 270-byte figure is illustrative, not a measured wire average. Real records are often larger.

## Regime A: 100,000 to 1,000,000 aggregate

Keep a simple scalar order inside each plane and scale by assigning planes to independently
fenced owners.

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

The engineering concerns are consistent assignment with monotonic ownership epochs, zero-overlap
or explicitly fenced handoff, continuation during control-plane outages, per-plane admission
control and cost accounting, automated hot-plane isolation, bounded rebalancing that does not
replay an entire source, and multi-tenant fairness across CPU, database, memory and network.

This is the shortest credible route to an aggregate million per second.

## Regime B: one hot plane at one million

This requires partitioning the plane's dependency graph.

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

Transactions touching one lane execute and publish independently. Transactions spanning lanes
take a deterministic order, barrier or reservation across them. A consumer's applied position
becomes a frontier rather than an integer.

Open problems, in rough order of difficulty:

- how coordination keys get declared or inferred without expensive dynamic read-set tracking
- whether capabilities and claims can double as partition hints
- how cross-lane deadlock is avoided
- how compact the frontier stays as lanes split and merge
- how a new client gets a consistent snapshot corresponding to a frontier
- how subscriptions move between fan-out workers without gaps
- whether scalar-cursor clients can be served by a deterministic merge view
- what recovery looks like when one lane is behind and others are current
- which operations require a global barrier, and how expensive those may become

This is the most fertile systems problem in the near-term architecture.

## Regime C: geographically distributed world state

At global scale, latency makes one synchronous leader wrong for most physical and interactive
actions. The shape is hierarchical: local or device-level safety and control loops, site or
regional coordination, global policy and audit, explicit ownership transfer between regions,
operation under temporary disconnection, and later reconciliation with authority and evidence
preserved.

Strong global serialisation gets reserved for transitions that genuinely need it. A warehouse
robot takes a fenced lease over a physical zone from a regional authority, moves at local
latency inside that lease while emitting observations, and a global service audits and can
revoke future authority without approving each motor command. The lease epoch is what stops a
disconnected stale controller from believing it still owns the zone.

## Questions this raises

- Which target does the business actually need first, and what does the answer change about
  what gets built next?
- What is the largest single plane any real customer would produce, and what rate does it imply?
- What is the cost per million delivered deltas today, at the measured topology?
- If the unit changed from 500 operations per commit to 10, what would the current
  architecture's ceiling be?
- Is there a customer workload where a frontier is unacceptable and a scalar cursor is a hard
  requirement? What breaks for them under regime B?
