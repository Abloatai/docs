# Logical decoding and change data capture

## The problem

The WAL echo is what promotes an Ablo receipt from accepted to confirmed
([02-the-contract.md](../02-the-contract.md#the-settlement-rule)). That makes the decoder a
correctness component, not a plumbing detail. It also makes it an availability dependency on
infrastructure Ablo does not own: the customer's database, its disk, and its failover behaviour.

## What the field knows

**A replication slot is a promise the customer's disk has to keep.** A slot holds WAL until its
consumer acknowledges. PostgreSQL bounds the damage with `max_slot_wal_keep_size`, which caps
what slots may retain so the disk cannot be exhausted, and `wal_keep_size`, which sets a
retention floor. Cross the cap and the required segments are removed, which terminates the
connection and invalidates the slot.

The observability for this is already in the database and is worth wiring into any product that
depends on it. `pg_replication_slots` exposes `safe_wal_size`, the bytes that can still be
written before the slot is at risk, and `invalidation_reason`, which names the cause: required
WAL removed, required rows removed, `wal_level` insufficient for logical decoding, or an idle
timeout. A product that reports "replication is behind" without those two columns is guessing.

**Failover is newer and more conditional than it looks.** PostgreSQL 17 added failover slots:
`pg_create_logical_replication_slot(..., failover => true)` or `ALTER_REPLICATION_SLOT ...
FAILOVER`, combined with `synchronized_standby_slots` on the primary, synchronises slots to
standbys so a consumer can resume after promotion. Readiness has to be verified rather than
assumed, with the documented check being that a slot is `synced`, not `temporary`, and has a
null `invalidation_reason`.

Debezium's connector makes the version boundary explicit: its `slot.failover` property defaults
to false and is ignored entirely when the primary runs PostgreSQL 16 or earlier. Ablo supports
customers on Aurora, RDS, Neon, Supabase and self-managed Postgres, so the honest position is
that failover behaviour is per-provider and per-version, not a single guarantee.

**The read path has a working commercial shape next door.** ElectricSQL syncs shapes out of
Postgres over plain HTTP: a client requests a shape with an `offset`, uses `-1` for an initial
sync, and follows an append-only shape log with control messages including `up-to-date`,
`snapshot-end`, and `must-refetch` when the server can no longer serve continuity from the
client's position. `must-refetch` is the same admission Ablo's bounded-resnapshot path makes,
made visible in the protocol rather than hidden in an error.

## The architectural choice ahead

Repeating decode, projection and object construction in each server process multiplies CPU and
memory bandwidth, and independent consumers duplicate subscription lookup. The alternative is
one fenced decoder per source feeding a replayable internal segment stream, so each WAL
transition is decoded once and publication workers route compact records independently.

That introduces a second acknowledgement and recovery boundary, and one invariant governs it:
the decoder must never acknowledge source WAL past data that cannot be reconstructed after the
relevant failure. The safe acknowledgement point is the highest contiguous LSN across all
downstream partitions, which is not the same as the highest LSN any of them has reached.

## Questions this raises

- Are `safe_wal_size` and `invalidation_reason` surfaced to operators and to the customer today?
  At what threshold does anyone get told?
- Which supported providers actually expose PostgreSQL 17 failover slots, and what happens on
  the ones that do not?
- What is the measured crossover between replaying from a retained slot and taking a fresh
  snapshot, as a function of lag and table size?
- What does an observer see during a resnapshot? Is there an equivalent of `must-refetch` in the
  Ablo protocol, and is it a first-class message or an error?
- If decode moved to one fenced decoder per source, what is the new recovery story, and how is
  the highest contiguous safe LSN computed cheaply?
- A write that bypasses Ablo still shows up in the WAL. How is it attributed, and what does a
  customer see when it appears in their audit trail with no capability behind it?
- How long can a customer's database be unreachable before Ablo's position becomes
  unrecoverable, and is that number written down anywhere a customer can act on?
