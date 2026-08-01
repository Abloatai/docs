# Internal representation and memory movement

At 100,000 deltas per second the shape of the data between stages is architecture. The most
recent retained change altered no algorithm: it changed one object passed between worker and
host.

| Measure | Expanded | Compact | Change |
| --- | ---: | ---: | ---: |
| 500-UPDATE publication object | 302,873 bytes | 137,313 bytes | 54.7% smaller |
| Structured-clone time, local | 738 ms | 303 ms | 59% faster |
| Cloned bytes per delta, profiled | 571.7 | 315.4 | 44.8% fewer |
| Worker `postMessage` | 3,260 ms | 1,842 ms | 43.5% faster |
| Host arrival processing | 10,820 ms | 6,011 ms | 44.4% faster |
| Credit handling | 1,987 ms | 1,319 ms | 33.6% faster |
| **Final drain** | **267 ms** | **141 ms** | **47.2% lower** |

Full runs in [04-the-evidence.md](../04-the-evidence.md#the-most-recent-retained-result). The
external wire format was deliberately unchanged, which isolates internal transport cost from
client semantics.

Structured cloning across a worker boundary is serial and synchronous on the publication path,
so it lands on drain rather than being absorbed by parallelism elsewhere.

## The arithmetic that makes it matter

| At 100,000 deltas/sec | Cost |
| --- | --- |
| 1 µs of CPU per delta | 10% of one core, permanently |
| 300 bytes copied per delta | 30 MB/sec of memory traffic |
| one global lock or lookup per delta | a serialisation point, not an overhead |

## Batching is the other lever

TigerBeetle amortises consensus and I/O across batches of up to 8,189 events per request, and
shrinks the batch under light load to favour latency
([requests](https://docs.tigerbeetle.com/coding/requests)). Its documentation states that
inserting a million transfers one at a time yields a fraction of the potential rate. Batching is
throughput bought with latency. Ablo's 500 operations per commit sits on the same curve
([the unit](../06-scale-regimes.md#first-define-the-unit)).

## Candidate directions

| Direction | Removes |
| --- | --- |
| Transferable `ArrayBuffer` ownership | the structured clone entirely, at the cost of shared access |
| Fixed header plus payload arena | repeated object expansion |
| Shared-memory ring buffer with epoch and credit | the copy and the message boundary |
| Direct Postgres tuple decode into the internal record | one intermediate form |
| Vectored socket writes and batch framing | per-delta syscall overhead |

Zero-copy is local to a boundary. The data is still decoded, transformed, encrypted, framed,
retransmitted and copied in the kernel and on the client, so a representation change is worth
what the end-to-end stage profile says it is worth. Compact positional external JSON was
rejected: a small transport win for a permanent compatibility cost.

## In the code

The external representation that was deliberately held fixed is
[wire/delta.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/delta.ts) and
[wire/frames.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/frames.ts). The client side of the same bytes is
[local/SyncClient.ts](https://github.com/Abloatai/ablo/blob/main/packages/humans/src/local/SyncClient.ts).

## See it yourself

The clone cost is measurable in ten lines of Node, and the result is usually larger than people
expect:

```js
const obj = { rows: Array.from({ length: 500 }, (_, i) => ({ id: i, data: 'x'.repeat(200) })) };
console.time('clone'); for (let i = 0; i < 1000; i++) structuredClone(obj); console.timeEnd('clone');
```

Halve the payload and the time roughly halves, which is the whole mechanism behind the table
above. `node --cpu-prof` on the real path shows where the remaining time sits.

## Go deeper

| Read | For |
| --- | --- |
| [TigerBeetle batching](https://docs.tigerbeetle.com/coding/requests) | how a headline throughput number is actually constructed |
| [Little's law](https://pubsonline.informs.org/doi/10.1287/opre.9.3.383) | why a cheaper stage shows up as lower drain rather than higher throughput |
| [How Not to Measure Latency](https://qconsf.com/sf2012/dl/qcon-sanfran-2012/slides/GilTene_HowNotToMeasureLatency.pdf) | the measurement traps that make representation work look free |

## Still open

- How much of the remaining 315.4 bytes per delta is payload the client needs, against envelope.
- Whether the client apply path pays a symmetric cost, measured on a low-end device rather than
  a developer machine.
- Where a typed binary record crosses a version boundary, since it needs a schema version at
  that seam before it ships rather than after.
- How much of drain is allocation pressure rather than copying.
