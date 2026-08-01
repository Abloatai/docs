# Postgres under a high update rate

## The problem

Ablo's dominant measured cost is the physical UPDATE on an indexed, frequently mutated row.
Run 118 put application SQL at 338 ms p50 with the task update alone at 321 ms p50, dominated by
`LWLock:BufferContent`. Dropping every secondary index took the system to 100,384/sec and broke
the read contract, which is causal proof and not a fix. Numbers in
[04-the-evidence.md](../04-the-evidence.md#where-update-cost-actually-goes).

## What the field knows

The mechanism is documented, and it is specific.

**Heap-only tuples are the fast path, and indexes disqualify you from it.** PostgreSQL can avoid
writing new index entries for an updated row when two conditions hold: the update does not
modify any column referenced by any of the table's indexes, and the page holding the old row has
enough free space for the new version. When that happens, indexes keep pointing at the original
item identifier, which becomes a redirect to the newer version, and obsolete intermediate
versions can be reclaimed during ordinary reads rather than waiting for vacuum.

Miss either condition and every index on the table takes a write for every update, dead tuples
accumulate, and the reclamation moves to vacuum. PostgreSQL's own introduction to indexes states
the tradeoff plainly: indexes improve reads, add overhead to data manipulation, and can prevent
heap-only tuples.

One nuance worth knowing: summarising access methods such as BRIN are flagged `amsummarizing`
and do not disqualify HOT for updates to columns they cover, provided those columns are not used
in index predicates. Index choice, not just index count, changes the write path.

**Vacuum is the other half of the bill.** Dead tuples are not physically removed when a row is
updated or deleted; they persist until vacuumed. A workload that misses HOT therefore pays
twice, once in index maintenance at write time and once in vacuum debt afterwards, and the
second payment arrives later, under different load, which is why short benchmark intervals
flatter it.

## The proposal on the table

A vertical split that keeps stable indexed identity separate from hot mutable state, with query
projections maintained explicitly and their freshness stated. The shape is in
[04-the-evidence.md](../04-the-evidence.md#where-update-cost-actually-goes). It trades write
amplification in one wide indexed row for joins, projection maintenance and an explicit
freshness policy. It is untested.

## What to measure here

Throughput is the least informative metric in this domain. The ones that explain it: SQL
latency by statement, `BufferContent` wait time, WAL bytes per delta, heap and index bytes
written, buffer writes, dead tuple count and vacuum debt over time, projection freshness, and
the read latency of reconstructing one logical entity from multiple physical relations.

## Questions this raises

- What fraction of current updates are HOT-eligible, and what is the exact index that
  disqualifies the rest?
- Is `BufferContent` contention on heap pages, index pages, or both? Does the answer change
  with the number of writer shards?
- Would a BRIN or covering-index arrangement restore HOT eligibility for part of the workload
  without losing the read paths that justify the current indexes?
- After the vertical split, how many physical relations does a common read touch, and what does
  that do to p99 read latency?
- What does sustained autovacuum activity do to the throughput number over an hour, rather than
  over a 20-second interval?
- Fill factor and HOT tuning were already rejected as a complete answer. Were they rejected as
  a partial one, and is there a combination with the vertical split that was never tried?
- Ablo never runs DDL against a customer database. How would a vertical split be offered to a
  customer whose schema Ablo does not own?
