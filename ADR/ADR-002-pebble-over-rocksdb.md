# ADR-002 — Pebble over RocksDB for Local Storage

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

The node daemon requires a local embedded key-value store for three distinct purposes:
1. Shard storage (opaque binary blobs keyed by SHA3-256 hash)
2. Shard manifest (metadata: shard ID, size, owner pubkey, placement timestamp, last verified timestamp)
3. Credit ledger (append-only log of signed credit entries)

The initial design specified RocksDB via the `gorocksdb` Go binding. RocksDB is the industry standard for embedded key-value storage in high-performance distributed systems — it is battle-tested, well-documented, and used in production by CockroachDB, TiKV, and many others.

However, `gorocksdb` is a CGO binding. It requires librocksdb to be compiled and linked, which violates the zero-CGO constraint established in ADR-001. Additionally, RocksDB's memory footprint on constrained hardware (Raspberry Pi 3B: 1GB RAM, NF-15) is a documented concern — RocksDB's block cache and write buffer management can consume hundreds of MB even at minimal configuration.

---

## Decision

**Use `github.com/cockroachdb/pebble` as the embedded key-value store.**

Pebble is CockroachDB's purpose-built replacement for RocksDB. It is:
- Pure Go — zero CGO, compiles with `CGO_ENABLED=0` on all targets
- API-compatible with RocksDB at the conceptual level (similar iterator/batch/compaction model)
- WAL-backed for crash safety (satisfies NF-09: daemon must recover cleanly from crash)
- Lower memory footprint than RocksDB at equivalent workloads
- Actively maintained by a production engineering team (CockroachDB uses it in production at scale)

The three use cases (shard storage, manifest, credit ledger) are separated by Pebble key prefixes: `shards/`, `manifest/`, and `credits/` — allowing independent iteration and backup of each namespace without separate database instances.

---

## Consequences

### Positive
- Satisfies zero-CGO constraint (ADR-001) — no C toolchain required
- WAL guarantees crash safety without additional work (satisfies NF-09, ST-06)
- Lower memory footprint suits Raspberry Pi 3B target (NF-15)
- Single Pebble instance handles all three data namespaces, reducing file handle usage
- RocksDB-compatible mental model means team documentation and debugging intuition transfers

### Negative / Trade-offs
- Pebble is less widely deployed outside of CockroachDB than RocksDB
- Some RocksDB-specific tuning guides don't apply directly to Pebble
- Pebble's compaction behaviour at very large shard stores (>1TB) is less documented than RocksDB's

### Neutral
- Both RocksDB and Pebble use LSM-tree architecture — performance characteristics are similar for the read/write patterns of shard storage (mostly write-once, read-occasionally)
- The key-prefix namespace separation is a Pebble convention; it would have been needed with RocksDB as well

---

## Alternatives Considered

### Alternative A: RocksDB via gorocksdb
Rejected. CGO dependency violates ADR-001.

### Alternative B: bbolt (BoltDB)
Considered. bbolt is pure Go and widely used. Rejected because bbolt uses a B-tree (not LSM), which performs poorly for write-heavy workloads at the shard sizes we expect (individual shards can be hundreds of MB). bbolt also holds the entire database file mmap'd, which is problematic on memory-constrained devices.

### Alternative C: SQLite via modernc/sqlite (pure Go CGO-free port)
Considered. modernc/sqlite compiles without CGO via transpilation. Rejected because SQL semantics add unnecessary complexity for a key-value workload, and the transpiled C-to-Go code is less auditable than a native Go implementation.

### Alternative D: Custom flat-file storage
Rejected. Implementing crash-safe atomic writes, compaction, and range iteration correctly is a significant engineering investment. Using Pebble gives all of these for free.

---

## Related ADRs
- ADR-001 (Zero CGO Constraint) — the primary driver of this decision
