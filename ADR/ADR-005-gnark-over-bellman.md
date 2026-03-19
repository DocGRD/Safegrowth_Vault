# ADR-005 — gnark over bellman/arkworks for ZK Proofs

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault Phase 3 requires zero-knowledge proofs for two purposes:
1. **ZK-PoR** (PR-07): a node proves shard possession via zk-SNARK without transmitting shard content — enabling blind verification by any peer
2. **Credit claims** (CL-04): all Phase 3 credit claims must include a valid ZK-PoR proof

The initial design referenced `arkworks-rs` and `bellman` — mature Rust zkSNARK frameworks widely used in the ZK ecosystem (Zcash, Filecoin, and others use bellman). These are high-quality libraries with significant production deployment history.

However:
1. The node daemon is written in Go — using Rust libraries requires either FFI (CGO, violating ADR-001) or rewriting the proof generation in a separate Rust sidecar process (significant architectural complexity and operational overhead)
2. Both bellman and arkworks are CGO when called from Go via their FFI bindings

The Go ZK ecosystem has matured significantly. `github.com/consensys/gnark` is a pure-Go zkSNARK library authored by ConsenSys (now Linea) with Groth16 and PLONK backends.

---

## Decision

**Use `github.com/consensys/gnark` for all zero-knowledge proof operations, with Groth16 as the primary backend.**

PLONK is available as a fallback if Groth16 proving time on the OptiPlex 990 exceeds the 2-second target established in the dev plan (Week 15 benchmark gate).

**This decision is gated on a benchmark gate at the start of Week 15 (before any circuit code is written):**
- Write a minimal test circuit (prove knowledge of a SHA3-256 preimage)
- Measure Groth16 prove time on OptiPlex 990 and Raspberry Pi 3B
- Target: ≤ 2 seconds on OptiPlex 990; ≤ 10 seconds on Pi 3B
- If target is missed: evaluate PLONK backend or proof batching
- If neither meets the target: defer ZK-PoR to post-MVP and document in BENCHMARKS.md

Results must be documented in `docs/BENCHMARKS.md` before circuit implementation begins.

---

## Consequences

### Positive
- Pure Go — satisfies ADR-001
- Both Groth16 and PLONK backends available in a single library
- Single language for the entire daemon codebase — no FFI, no sidecar, no IPC between Go and Rust
- gnark is used in production (Linea L2 rollup uses gnark for proof generation at scale)
- Circuit writing uses Go syntax — accessible to the existing team without learning a new DSL
- Proof size: Groth16 produces ~128-256 byte proofs; PLONK produces ~1-2KB proofs — both within the PR-07 target of under 2KB

### Negative / Trade-offs
- gnark is younger than bellman/arkworks — less community documentation and fewer StackOverflow answers for debugging circuit errors
- gnark's Groth16 proving time may be slower than Rust implementations on equivalent hardware — this is the direct risk the Week 15 benchmark gate addresses
- Groth16 requires a trusted setup per circuit — the ceremony must be conducted once and the toxic waste discarded. This is a one-time operational requirement before Phase 3 launch.
- gnark's API has changed between minor versions — pin to a specific version and review changelog before upgrading (risk register)

### Neutral
- The ZK-PoR circuit design (proving knowledge of shard bytes satisfying a hash challenge) is mathematically equivalent regardless of the underlying library
- gnark supports bn254, bls12-381, and other curves — bn254 is the default and appropriate for our use case

---

## Alternatives Considered

### Alternative A: bellman (Rust, via Go FFI/CGO)
Rejected. CGO dependency violates ADR-001. Additionally, a Rust sidecar architecture would require IPC between Go daemon and Rust prover, adding latency and operational complexity.

### Alternative B: arkworks-rs (Rust, via Go FFI/CGO)
Rejected. Same reasoning as bellman.

### Alternative C: groth16-go (community port)
Rejected. Multiple community Groth16 implementations exist in Go but none have production deployment history comparable to gnark. gnark is the clear community standard for production Go ZK work.

### Alternative D: Bulletproofs (no trusted setup)
Considered as a fallback. Bulletproofs do not require a trusted setup ceremony, which is operationally simpler. However, Bulletproof verification is O(n) in proof size — significantly slower than Groth16's O(1) verification. For continuous PoR challenges at scale, fast verification is important. Bulletproofs remain documented as a fallback if gnark proving time is unacceptable on target hardware.

### Alternative E: Defer ZK-PoR entirely (Phase 3+ or post-MVP)
Acceptable as a fallback if the Week 15 benchmark gate is not met. The system remains secure without ZK-PoR — the non-ZK PoR challenge protocol (challenge/response with hash computation) is sufficient for Phase 3 launch. ZK-PoR adds blind verification as an enhancement, not a base security requirement.

---

## Trusted Setup Requirement

Groth16 requires a per-circuit trusted setup (Powers of Tau ceremony). For Phase 3 launch:
1. Generate circuit-specific parameters using an existing universal Powers of Tau (Hermez/Zcash ceremonies are publicly available and auditable)
2. Conduct a small multi-party computation ceremony among the three team members to add our own randomness contribution
3. Document the ceremony process and outputs in `docs/zk-setup/`
4. The toxic waste (random contributions) must be verifiably deleted by each participant

PLONK (the fallback backend) also requires a universal trusted setup but can reuse existing ceremonies without circuit-specific parameters — operationally simpler if we switch.

---

## Benchmark Gate Record

*(To be filled in at Week 15)*

| Hardware | Backend | Circuit | Prove Time | Verify Time | Decision |
|---|---|---|---|---|---|
| OptiPlex 990 | Groth16 | SHA3-256 preimage | TBD | TBD | TBD |
| Raspberry Pi 3B | Groth16 | SHA3-256 preimage | TBD | TBD | TBD |
| OptiPlex 990 | PLONK | SHA3-256 preimage | TBD | TBD | TBD |

---

## Related ADRs
- ADR-001 (Zero CGO Constraint) — primary driver
