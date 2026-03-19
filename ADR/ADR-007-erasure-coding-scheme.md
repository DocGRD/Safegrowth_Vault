# ADR-007 — Erasure Coding Scheme Progression (2-of-3 → 3-of-5 → 5-of-9)

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault distributes user data as erasure-coded shards across the network. The erasure coding scheme determines how many shards are needed to reconstruct a file (k) out of how many total shards exist (n). A higher n-k difference means more node failures can be tolerated before data is lost.

The choice of scheme involves a three-way trade-off between:
1. **Storage overhead**: a k-of-n scheme uses n/k times the original file size in total network storage
2. **Fault tolerance**: the system can lose (n-k) nodes simultaneously without data loss
3. **Minimum network size**: you need at least n distinct nodes to place shards on

The network grows across three phases — from 4 personal devices on a LAN (Phase 1) to a trusted group of ~7 nodes (Phase 2) to open participation with 10+ nodes (Phase 3). A single fixed scheme cannot serve all phases efficiently.

---

## Decision

**Use a three-phase erasure coding scheme that scales with the network:**

| Phase | Scheme | Storage Overhead | Fault Tolerance | Min Nodes Required |
|---|---|---|---|---|
| Phase 1 (LAN, personal devices) | 2-of-3 | 1.5× | 1 node loss | 3 |
| Phase 2 (trusted group, internet) | 3-of-5 | 1.67× | 2 node losses | 5 |
| Phase 3 (open participation) | 5-of-9 | 1.8× | 4 node losses | 9 |

The scheme is configurable per node via `config.json` (`erasure_scheme: {"data": k, "parity": n-k}`) and is applied at backup time. Existing shards are not retroactively re-encoded when the scheme changes — re-encoding would require re-downloading and re-uploading all data, which is impractical. New backups after a scheme upgrade use the new scheme.

Implementation: `github.com/klauspost/reedsolomon` — pure Go, SIMD-accelerated, zero CGO.

---

## Consequences

### Positive
- Phase 1 scheme (2-of-3) works with as few as 3 nodes — matches the initial 4-device personal setup
- Phase 3 scheme (5-of-9) tolerates 4 simultaneous node failures — appropriate for an open network with unknown node reliability
- Storage overhead grows gracefully: 1.5× → 1.67× → 1.8× — never more than 2× the original data size
- Phase 3 (5-of-9) satisfies NF-06: "tolerate up to floor((n-1)/3) simultaneous node failures without data loss at 5-of-9 coding" — floor((9-1)/3) = 2 Byzantine nodes, but 4 simple failures
- klauspost/reedsolomon is pure Go with SIMD acceleration — satisfies ADR-001 and performs well on all target hardware

### Negative / Trade-offs
- Files backed up under Phase 1 scheme (2-of-3) are less redundant than Phase 2/3 files — users should be informed and re-backup critical data after upgrading scheme
- Managing multiple schemes simultaneously in the shard manifest adds complexity to the replication manager
- 5-of-9 requires 9 distinct nodes — the network must reach this size before Phase 3 can be fully activated for new backups

### Neutral
- Reed-Solomon reconstruction succeeds with any qualifying subset (any k of n) — partial node availability is handled gracefully
- Shard sizes are approximately equal (original file size / k, plus parity overhead) regardless of scheme — storage per node is predictable

---

## Fault Tolerance Analysis

For the most security-sensitive case (Phase 3, 5-of-9):
- Random node failures: can lose any 4 of 9 nodes — probability of data loss with independent 10% per-node failure rate = C(9,5) × 0.1^5 × 0.9^4 ≈ 0.0001% — effectively zero
- Targeted attack: an attacker must simultaneously corrupt shards on at least 5 of 9 nodes — combined with geographic and ASN diversity enforcement (SH-05, SH-06), this requires a highly coordinated multi-location attack
- Byzantine nodes: the Byzantine fault tolerance model (BC-03) tolerates f = floor((n-1)/3) = 2 Byzantine nodes in a 9-node quorum — consistent with the 5-of-9 scheme's tolerance

---

## Alternatives Considered

### Alternative A: Fixed 5-of-9 from Phase 1
Rejected. Phase 1 only has 4 nodes — a 5-of-9 scheme requires 9 nodes and cannot be used. Starting with a fixed high-n scheme would block Phase 1 entirely.

### Alternative B: Fixed 3-of-5 from Phase 1
Rejected. Phase 1 has 4 nodes; a 3-of-5 scheme requires 5. Would still block Phase 1.

### Alternative C: 4-of-7 for Phase 3
Considered. Slightly lower storage overhead (1.75×) than 5-of-9 (1.8×). Rejected because: the 5-of-9 naming is conventional and widely understood in distributed storage literature, and the NF-06 requirement specifically references 5-of-9. The difference in overhead (0.05×) is negligible.

### Alternative D: RAID-style mirroring (3× replication) instead of erasure coding
Considered for Phase 1 simplicity. Rejected because: (a) mirroring uses 3× storage vs 1.5× for 2-of-3 erasure coding with equivalent fault tolerance; (b) the klauspost/reedsolomon library is simple enough that starting with proper erasure coding from day one avoids a migration later.

---

## Related ADRs
- ADR-001 (Zero CGO Constraint) — klauspost/reedsolomon must be CGO-free (it is)
- ADR-008 (PBFT-lite Consensus) — quorum sizing is informed by the erasure coding scheme's Byzantine fault tolerance analysis
