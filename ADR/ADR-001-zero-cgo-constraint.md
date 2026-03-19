# ADR-001 — Zero CGO Constraint on Node Daemon

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault must cross-compile and run on Linux (amd64 and arm64), Windows (amd64), and Android arm64 via Termux — from a single `go build` invocation with no C toolchain installed on the build machine or any deployment target. The initial dependency selections for several components (RocksDB via gorocksdb, liboqs for post-quantum crypto, bellman/arkworks-rs for ZK proofs) required CGO — a Go mechanism that calls into C libraries. CGO compilation requires a C toolchain (gcc/clang) to be present on both the build machine and, in some configurations, the target platform.

The team operates across a Lenovo Flex (primary), OptiPlex 990, Dell Latitude, a home-built PC, and Android phones. Installing and maintaining C toolchains across all of these platforms — and on CI — adds non-trivial operational complexity and creates brittle builds that fail in non-obvious ways when the C toolchain version or configuration differs between environments.

Early in development, the CI pipeline failed because the Linux arm64 cross-compilation target could not find the aarch64 GCC cross-compiler. This was the triggering incident for formalizing the zero-CGO constraint.

---

## Decision

**All node daemon dependencies must compile with `CGO_ENABLED=0`. No C toolchain is required on any build or deployment platform. This is a hard constraint, not a preference.**

CI enforces this by:
1. Not installing any C toolchain on the CI runner (intentional)
2. Building all targets with `CGO_ENABLED=0` as an explicit env var
3. Any PR that introduces a CGO dependency will fail CI at the build step — this is the intended gate

---

## Consequences

### Positive
- Single `go build` command produces binaries for all four targets
- No C toolchain installation required on any developer machine, CI runner, or deployment device
- No cross-compilation toolchain complexity (no need for `aarch64-linux-gnu-gcc`, `x86_64-w64-mingw32-gcc`, etc.)
- Dependency tree is fully auditable in Go — no opaque C code in the critical path
- Android/Termux deployment is straightforward: copy binary and run
- Eliminates entire class of "works on my machine" build failures

### Negative / Trade-offs
- Restricts library choices: the most mature or highest-performance implementations of some primitives have CGO wrappers (e.g., RocksDB, liboqs)
- Pure-Go alternatives must be found for every dependency (see ADR-002, ADR-003, ADR-004, ADR-005)
- Some pure-Go libraries are less battle-tested than their C counterparts at the time of writing

### Neutral
- Go 1.24+ is required anyway (for `crypto/mlkem`), and the pure-Go ecosystem for all required primitives is mature at this Go version
- The gnark ZK library (pure Go) is actually more ergonomic for Go developers than the Rust-based bellman/arkworks alternatives

---

## Alternatives Considered

### Alternative A: Allow CGO for specific packages, manage cross-compilation toolchains
Rejected. The operational complexity of maintaining cross-compilation toolchains across all target platforms was deemed higher than the cost of finding pure-Go alternatives. CGO also makes static binaries harder to produce, which matters for the Android/Termux deployment target.

### Alternative B: Use Docker for cross-compilation
Rejected. The requirements document explicitly states: "No dependency on systemd, Docker, or any container runtime for core operation" (NF-16). Using Docker for builds would create a Docker dependency on every developer machine and CI runner.

### Alternative C: Compile only for native platform and distribute per-platform
Rejected. A core design goal is that any team member can build the full release from any machine. Requiring platform-specific build machines breaks this.

---

## Verification

```bash
# This must pass on every PR. No C toolchain is installed on CI.
CGO_ENABLED=0 GOOS=linux   GOARCH=amd64 go build ./node/cmd
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build ./node/cmd
CGO_ENABLED=0 GOOS=linux   GOARCH=arm64 go build ./node/cmd
CGO_ENABLED=0 GOOS=android GOARCH=arm64 go build ./node/cmd
```

---

## Related ADRs
- ADR-002 (Pebble over RocksDB) — direct consequence of this constraint
- ADR-003 (ML-KEM via crypto/mlkem) — direct consequence of this constraint
- ADR-004 (ML-DSA via trailofbits/ml-dsa) — direct consequence of this constraint
- ADR-005 (gnark over bellman/arkworks) — direct consequence of this constraint
