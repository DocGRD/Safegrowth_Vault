# ADR-006 — Hybrid Classical / Post-Quantum Cryptography Mode

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault is designed from the ground up with post-quantum cryptography (ML-KEM for key encapsulation, ML-DSA for signatures). However, the network will undergo a deployment period during which not all nodes have upgraded to Go 1.24+ and not all clients support the PQ primitives. This is a standard migration challenge in any cryptographic protocol upgrade.

Additionally, the maturity of post-quantum implementations in production is lower than classical implementations. Running both classical and PQ primitives simultaneously during a transition period provides defense-in-depth: even if a flaw is discovered in the PQ primitive, the classical primitive still provides security against classical adversaries.

NIST guidance on hybrid PQ/classical schemes (from NIST IR 8413, 2022) recommends hybrid approaches during the transition period precisely for this reason.

---

## Decision

**Implement a hybrid mode (CR-05, SRD) that runs classic ECDH (X25519) alongside ML-KEM-768, and Ed25519 alongside ML-DSA-65, during the Phase 2 transition period.**

The hybrid scheme works as follows:
- **Key encapsulation**: derive the symmetric key as `HKDF(ML-KEM-shared-secret || ECDH-shared-secret)` — an attacker must break both to recover the symmetric key
- **Signatures**: nodes sign with both ML-DSA-65 and Ed25519; receivers verify whichever they support, preferring ML-DSA; both signatures are included in the MessageEnvelope during the transition period
- **Protocol version**: hybrid mode is a v1/v2 feature; v3 nodes may require ML-DSA-only if the network has fully migrated

The hybrid mode is activated per-connection based on version negotiation (PROTOCOL_SPEC Section 2): a v1 node connecting to a v2 node uses the hybrid mode. A v2 node connecting to another v2 node can negotiate PQ-only if both support it.

Classical implementations used:
- ECDH: `golang.org/x/crypto/curve25519` (X25519) — pure Go, stdlib-adjacent
- Signatures: `crypto/ed25519` — Go standard library

---

## Consequences

### Positive
- Defense-in-depth: security requires breaking both classical and PQ primitives simultaneously
- Smooth migration path: nodes can upgrade incrementally without breaking connectivity
- Aligned with NIST and IETF recommendations for PQ transition (RFC 9420 hybrid guidance)
- Classical primitives (Ed25519, X25519) are extremely well-audited and battle-tested
- Both classical libraries are pure Go, maintaining ADR-001 compliance

### Negative / Trade-offs
- Increased message sizes during hybrid period: two public keys, two signatures, two ciphertexts per connection
- Increased computational cost: two key operations per handshake instead of one
- Additional code complexity in the crypto layer — two code paths must be maintained until classical mode is fully deprecated
- Hybrid mode must eventually be deprecated — requires a coordinated network-wide migration to PQ-only

### Neutral
- The overhead is bounded: X25519 keys are 32 bytes, Ed25519 signatures are 64 bytes — negligible compared to ML-KEM/ML-DSA sizes
- The hybrid period duration is determined by the network's upgrade rate — no hard deadline is set here, but tracking should begin once Phase 3 is live

---

## Alternatives Considered

### Alternative A: PQ-only from day one, no hybrid mode
Considered. Simpler codebase. Rejected because: (a) PQ implementations are younger and less battle-tested; (b) no backward compatibility with older nodes during deployment; (c) NIST itself recommends hybrid during transition.

### Alternative B: Classical-only now, migrate to PQ later
Rejected. The threat model explicitly requires PQ key encapsulation (THREAT_MODEL.md, security assumption 2). A "harvest now, decrypt later" adversary who stores encrypted traffic today could decrypt it when quantum computers become capable. Starting with PQ (even in hybrid mode) protects data with long secrecy requirements.

### Alternative C: ECDH + ML-KEM but Ed25519-only signatures (no ML-DSA during transition)
Considered as a simplification. Rejected because signatures also need PQ protection — a quantum adversary could forge Ed25519 signatures and impersonate nodes. Both the KEM and the signature scheme must be PQ-protected.

---

## Deprecation Criteria

Hybrid mode can be deprecated (PQ-only mode becomes mandatory) when:
1. All known nodes in the network are running Go 1.24+ with ML-KEM/ML-DSA support
2. No v1 nodes have been seen on the network for at least 30 days
3. The team agrees via explicit vote documented in the dev log

---

## Related ADRs
- ADR-003 (ML-KEM via crypto/mlkem) — the PQ KEM being hybridized
- ADR-004 (ML-DSA via trailofbits/ml-dsa) — the PQ signature being hybridized
