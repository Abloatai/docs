# Subscription-aware fan-out and incremental views

Full-stream fan-out grows as event rate times subscriber count, and becomes impossible well
before world scale.

| At 1,000,000 deltas/sec | Bytes/sec | Bits/sec |
| --- | ---: | ---: |
| Internal cloned data, at the measured 315.4 B/delta | 315.4 MB | 2.52 Gbit |
| One full-stream observer, illustrative 270 B payload | 270 MB | 2.16 Gbit |
| Ten full-stream observers | 2.7 GB | 21.6 Gbit |
| One hundred full-stream observers | 27 GB | 216 Gbit |

Before TLS, WebSocket, TCP and retransmission. The 270-byte payload is illustrative, not a
measured wire average. The conclusion is not a larger network interface, it is that publication
must be subscription-aware and hierarchical.

The commercial version is sharper: the market currently prices fan-out at zero, so every byte
sent to a subscriber that did not need it is margin rather than revenue.

| Electric Cloud, published | Price |
| --- | --- |
| Writes | $1 per million, chunked at 10 KB |
| Retention | $0.10 per GB-month |
| Postgres Sync, writes emitted to the shape log | $2 per million |
| Reads, fan-out, concurrent users, connections | free ([pricing](https://electric.ax/blog/2026/04/02/electric-cloud-pricing)) |

## Two stream products, not one

| | Event-complete | State-convergent |
| --- | --- | --- |
| Consumer | processors, auditors | observers materialising current state |
| Guarantee | every committed transition | latest value per key |
| May coalesce superseded values | no | yes |
| Retention | full history | compaction window |

Kafka ships exactly this split from one log: a consumer at the head sees every message, while a
consumer reading a compacted log from the start sees final state per key plus delete markers in
the retention window ([design](https://kafka.apache.org/documentation/#design)). An auditor
served from a coalescing stream is being quietly lied to.

## What the neighbours do

| System | Approach | Consequence |
| --- | --- | --- |
| Materialize | maintains SQL views over CDC, pushes diffs, carries progress with the data ([SUBSCRIBE](https://materialize.com/docs/sql/subscribe/)) | recomputation avoided; the view definition becomes the subscription |
| Differential Dataflow | changing collections as inputs to an incrementally maintained computation ([paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/01/differentialdataflow.pdf)) | separates data change, logical time, maintained result and progress |
| ElectricSQL | shapes as plain HTTP with offsets ([HTTP API](https://electric.ax/docs/sync/api/http)) | ordinary CDN infrastructure absorbs the fan-out, which is why reads can be free |

A stateful socket per observer is a choice to pay a cost a log-plus-offset design hands to a
cache.

## The mechanisms that scale

Decode once per source rather than once per server. Route only to subscribers whose materialised
shape may change. Publish compact field masks where semantics permit. Retain the audit stream
separately from state-materialisation feeds. Push regional fan-out toward the edge. Serve new or
badly lagged consumers a snapshot plus a recent delta tail.

Subscription matching accepts false positives to avoid false negatives, and the index over
aliases, joins, ranges and large membership sets is what makes that affordable.

## In the code

What a subscriber receives is [wire/feedEvent.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/wire/feedEvent.ts), and
the log row behind it is [log/syncDeltaRow.ts](https://github.com/Abloatai/ablo/blob/main/packages/transaction/src/log/syncDeltaRow.ts).

## See it yourself

An Electric shape is a log you can curl, which makes the whole read-path model concrete in two
commands:

```sh
curl -i "http://localhost:3000/v1/shape?table=items&offset=-1"
```

The response headers carry `electric-offset`; repeat with that offset to get the tail, and watch
for the `up-to-date` and `must-refetch` control messages. That is a cacheable sync protocol with
no socket in it.

## Go deeper

| Read | For |
| --- | --- |
| [Kafka log compaction](https://kafka.apache.org/documentation/#compaction) | the event-complete against state-convergent split, already specified |
| [Differential Dataflow](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/01/differentialdataflow.pdf) | the vocabulary for incremental maintenance |
| [Electric Cloud pricing](https://electric.ax/blog/2026/04/02/electric-cloud-pricing) | what the market says this layer is worth, in dollars |

## Still open

- Whether one decoding pass can safely feed all publication work.
- How a subscriber dependency index stays correct while the schema changes underneath it.
- The cost of an audit stream that must retain every transition while the state stream coalesces.
- What a broad subscription from an agent that barely reads should cost, and who pays for it.
