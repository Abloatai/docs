# Internal representation and memory movement

## The problem

At 100,000 deltas per second, the shape of the data between stages is an architectural decision.
The most recent retained change did not alter an algorithm: it changed the object passed between
worker and host, cutting profiled cloned bytes per delta from 571.7 to 315.4, and final drain by
47.2 percent. Full profile in
[04-the-evidence.md](../04-the-evidence.md#the-most-recent-retained-result).

The reason this is worth attention rather than micro-optimisation is that structured cloning
across a worker boundary is a serial, synchronous cost paid on the publication path, so it lands
directly on drain rather than being absorbed by parallelism elsewhere.

## What the field knows

**Batching is the strongest single lever, and its cost is stated in latency.** TigerBeetle
amortises consensus and I/O across batches of up to 8,189 events per request, and shrinks the
batch automatically under light load to favour latency instead. That is the honest form of the
tradeoff: batching is not free throughput, it is throughput bought with latency, tuned by load.
Ablo's 500 operations per commit sits at the same point on the same curve. See
[06-scale-regimes.md](../06-scale-regimes.md#first-define-the-unit).

**Zero-copy is local to a boundary.** Data that avoids one copy is still decoded, transformed,
encrypted, framed, retransmitted, and copied again in the kernel and on the client. A
representation change is worth exactly what the end-to-end stage profile says it is worth, which
is why the 135/136 result was accepted only after profiling showed the copied bytes had actually
moved.

## Candidate directions

- fixed headers plus variable payload arenas
- transferable `ArrayBuffer` ownership instead of structured cloning
- shared-memory ring buffers with epoch and credit protection
- schema-versioned columnar batches
- decoding Postgres tuples directly into the internal record, skipping an intermediate form
- vectored socket writes and batch framing

A typed, versioned binary internal record is the general shape: avoid repeated object expansion
and serialisation while keeping the external protocol stable. Keeping the external wire format
fixed is what made run 136 interpretable, and compact positional external JSON was already
rejected because it trades maintainability and compatibility for a small transport win.

## Questions this raises

- What is the current end-to-end stage profile in CPU time per delta? Which stage would a
  transferable buffer actually remove?
- How much of the remaining 315.4 bytes per delta is payload the client needs, and how much is
  envelope?
- Where does the internal representation cross a version boundary today? A binary record needs a
  schema version at that seam, and needs it before it ships, not after.
- Does the client apply path pay a symmetric cost, and has it been measured on a low-end device
  rather than a developer machine?
- At what offered load should the commit batch shrink automatically, and does anything do that
  today?
- Garbage collection: how much of drain is allocation pressure rather than copying?
