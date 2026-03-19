# ADR-004 — ML-DSA via trailofbits/ml-dsa

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault requires post-quantum digital signatures for:
1. Node identity (NodeID is derived from ML-DSA public key hash)
2. TLS certificate derivation (self-signed, identity-bound)
3. All gossip message signing (GP-03)
4. Shard Merkle hash signing (SH-08)
5. Invite token signing (IN-01)
6. Credit ledger entry signing (CL-01)
7. MessageEnvelope signing (Section 4, PROTOCOL_SPEC)

NIST finalized ML-DSA (Module-Lattice-Based Digital Signature Algorithm) as FIPS 204 in August 2024. ML-DSA is the standardized version of CRYSTALS-Dilithium.

Unlike ML-KEM (which was added to the Go standard library in Go 1.24), ML-DSA has not yet been added to the Go standard library as of Go 1.24. An external library is therefore required.

The initial design referenced `liboqs` for ML-DSA as well, which is CGO and violates ADR-001.

---

## Decision

**Use `github.com/trailofbits/ml-dsa/mldsa65` for all ML-DSA digital signature operations.**

Specifically, use the ML-DSA-65 parameter set (NIST security level 3, equivalent to ~128-bit classical security — appropriate for our signature use case).

Trail of Bits is a well-regarded security auditing and engineering firm. Their ml-dsa library is:
- Pure Go — zero CGO (satisfies ADR-001)
- NIST FIPS 204 compliant
- Authored by security professionals whose primary business is finding flaws in exactly this class of code
- Pinned to a specific tagged version in go.mod (risk register: low likelihood of API-breaking changes)

**Canonical encoding**: ML-DSA signatures in the MessageEnvelope are encoded as raw bytes (not DER). This is Open Question #1 from the THREAT_MODEL.md — resolution assigned to Glenn, Week 1. Once resolved, this ADR must be updated with the encoding decision and rationale.

---

## Consequences

### Positive
- Pure Go — satisfies ADR-001
- FIPS 204 compliant
- Authored and maintained by a dedicated security team — higher confidence than a community implementation
- ML-DSA-65 key sizes are manageable: public key 1952 bytes, signature 3309 bytes — acceptable for gossip message overhead
- Constant-time implementation (critical for a signing key that represents node identity)

### Negative / Trade-offs
- External dependency (not stdlib) — must be pinned and monitored for security updates
- Trail of Bits ml-dsa is younger than liboqs's Dilithium implementation in terms of production deployment history
- Signature size (3309 bytes for ML-DSA-65) is significantly larger than Ed25519 signatures (64 bytes) — this increases gossip message sizes and must be accounted for in bandwidth budgeting (GP-06)
- When Go standard library eventually adds ML-DSA (anticipated in a future Go version), this library should be replaced with the stdlib version for governance consistency — migration will be required

### Neutral
- All gossip messages are already protobuf-framed with a 1MB size cap (GP-09) — the signature overhead is acceptable within this budget
- The NodeID derivation (SHA3-256 of ML-DSA public key) is stable regardless of which library generates the keypair, ensuring future library migration does not change node identity

---

## Alternatives Considered

### Alternative A: liboqs / liboqs-go
Rejected. CGO dependency violates ADR-001.

### Alternative B: cloudflare/circl (Dilithium/ML-DSA implementation)
Considered. Cloudflare's circl library includes Dilithium3 (the pre-standardization name for ML-DSA-65). Rejected because: (a) cloudflare/circl implements draft Dilithium, not final FIPS 204 ML-DSA — the two have subtle differences; (b) trailofbits/ml-dsa explicitly targets FIPS 204; (c) Trail of Bits' security pedigree is better suited to this use case than a general cryptography library.

### Alternative C: Wait for Go stdlib ML-DSA
Rejected for now. The timeline for stdlib inclusion is uncertain. The project cannot block Week 1 identity implementation on a library that may not arrive until Go 1.26 or later. The trailofbits library is the right bridge. When stdlib ML-DSA is available, migration should be done — create a tracking issue when Go announces ML-DSA stdlib inclusion.

### Alternative D: Ed25519 (classical signatures only)
Rejected. The threat model (THREAT_MODEL.md, assumption 2) requires quantum-resistant signatures. Ed25519 is vulnerable to quantum adversaries with Shor's algorithm. The hybrid mode (ADR-006) allows Ed25519 alongside ML-DSA during transition, but ML-DSA must be primary.

---

## Migration Path

When the Go standard library adds `crypto/mldsa` (or equivalent):
1. Verify FIPS 204 compliance of the stdlib implementation
2. Verify key format compatibility — the raw public key bytes should be identical between trailofbits and stdlib implementations for the same key material
3. Replace `github.com/trailofbits/ml-dsa/mldsa65` import with stdlib equivalent
4. Run full test suite — keypair round-trip, signature verification across implementations
5. Update this ADR status to "Superseded" with reference to the migration PR

---

## Open Questions
- **Encoding**: Raw bytes vs DER for MessageEnvelope signatures? Assigned to Glenn, Week 1. Update this ADR when resolved.

---

## Related ADRs
- ADR-001 (Zero CGO Constraint) — primary driver
- ADR-003 (ML-KEM via crypto/mlkem) — the encapsulation counterpart
- ADR-006 (Hybrid Classical/PQ Mode) — transition strategy
