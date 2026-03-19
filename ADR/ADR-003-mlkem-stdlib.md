# ADR-003 — ML-KEM via Go Standard Library (crypto/mlkem)

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault requires post-quantum key encapsulation for two purposes:
1. Encrypting user data keys (the symmetric AES-256-GCM key is wrapped via ML-KEM before storage/transmission)
2. Establishing shared challenge seeds for Proof of Retrievability (Phase 3)

NIST finalized ML-KEM (Module-Lattice-Based Key-Encapsulation Mechanism) as FIPS 203 in August 2024. ML-KEM is the standardized version of CRYSTALS-Kyber. Any implementation must conform to FIPS 203.

The initial design referenced `liboqs` (Open Quantum Safe library) via its Go binding `liboqs-go`. liboqs is the reference C implementation maintained by the Open Quantum Safe project and is widely used in research and early production deployments. However, `liboqs-go` is a CGO binding, violating ADR-001.

Go 1.24 (released February 2025) introduced `crypto/mlkem` as a standard library package — a pure Go, constant-time implementation of ML-KEM-768 and ML-KEM-1024 authored by the Go cryptography team and reviewed as part of the standard library acceptance process.

---

## Decision

**Use `crypto/mlkem` from the Go standard library (Go ≥ 1.24) for all ML-KEM key encapsulation operations.**

Specifically, use ML-KEM-768 (the NIST-recommended security level, equivalent to ~192-bit classical security).

This requires Go 1.24 as the minimum Go version — this minimum is already established in the project for this reason (bumped from 1.21 in SRD revision 1.1).

---

## Consequences

### Positive
- Pure Go — zero CGO (satisfies ADR-001)
- Standard library inclusion means: reviewed by Go security team, covered by Go's security policy and CVE process, guaranteed to be updated with Go releases
- Constant-time implementation prevents timing side-channel attacks (satisfies NF-17 intent for crypto operations)
- FIPS 203 compliant by design — the standard library implementation tracks the final NIST standard
- No additional dependency to vendor or pin — reduces supply chain risk
- Available on all Go 1.24+ targets including Android/Termux

### Negative / Trade-offs
- Requires Go 1.24 minimum — older Go versions cannot build the project (this is an explicit, documented constraint)
- Only ML-KEM-768 and ML-KEM-1024 are available — ML-KEM-512 (lower security, faster) is not in the stdlib as of 1.24
- The stdlib package is newer than liboqs and has had less external scrutiny, though the Go team's internal review process is rigorous

### Neutral
- ML-KEM-768 is the NIST-recommended security level and appropriate for our use case
- The key sizes (public key: 1184 bytes, ciphertext: 1088 bytes) are larger than classical ECDH keys but acceptable for our gossip and shard placement message sizes

---

## Alternatives Considered

### Alternative A: liboqs / liboqs-go
Rejected. CGO dependency violates ADR-001.

### Alternative B: cloudflare/circl (ML-KEM implementation)
Considered. Cloudflare's `circl` library includes a pure-Go ML-KEM implementation. Rejected in favour of the stdlib because: (a) stdlib has stronger governance and security patch guarantees, (b) avoiding an external dependency is preferable when stdlib covers the need, (c) the stdlib implementation was specifically designed for Go's constant-time guarantees.

### Alternative C: Custom ML-KEM implementation
Rejected immediately. NF-10 explicitly prohibits custom cryptographic implementations: "All cryptographic primitives must use audited libraries; no custom crypto implementations."

### Alternative D: Classic ECDH (X25519) only, defer PQ migration
Rejected. The threat model (THREAT_MODEL.md, assumption 2) explicitly requires post-quantum key encapsulation. The hybrid mode (ADR-006) allows classical ECDH alongside ML-KEM during transition, but ML-KEM must be present from day one.

---

## Hybrid Mode Note

CR-05 (SRD) requires a hybrid mode running classic ECDH alongside ML-KEM for nodes that have not yet upgraded. This is addressed in ADR-006. The hybrid mode does not change this decision — ML-KEM via `crypto/mlkem` is the primary KEM; ECDH via `crypto/elliptic`/`golang.org/x/crypto/curve25519` runs alongside it during the transition period.

---

## Related ADRs
- ADR-001 (Zero CGO Constraint) — primary driver
- ADR-004 (ML-DSA via trailofbits/ml-dsa) — the signing counterpart to this decision
- ADR-006 (Hybrid Classical/PQ Mode) — the transition strategy
