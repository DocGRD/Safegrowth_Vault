# ADR-009 — No Central Servers in the Critical Path

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault is a decentralized storage network. A common failure mode in systems that claim decentralization is the introduction of "soft" centralization — components that are technically optional but practically required for the system to function. Examples from comparable systems:
- Storj's satellite coordination nodes: technically replaceable, but in practice the Storj network requires Storj Inc.'s satellites to function
- IPFS pinning services: IPFS itself is decentralized, but persistence requires centralized pinning services (Pinata, Infura)
- Many "decentralized" systems that use a central discovery server, a central relay, or a central billing system

The SRD constraint is explicit: "No central servers in the critical path for any core operation (storage, retrieval, consensus)" (SRD Section 8, Constraints). And: "The system must remain functional when the original developers are unavailable — no proprietary backend" (SRD Section 8).

This decision documents what "no central servers" means concretely, which components could accidentally become central, and how the architecture prevents it.

---

## Decision

**No single node, server, or service operated by the SafeGrowth development team or any third party is required for the core operations of storage, retrieval, peer discovery (Phase 3), or consensus.**

The following components serve bootstrap/convenience roles but are explicitly designed to be replaceable and non-critical for ongoing operation:

| Component | Role | Is It Critical? | Failure Mode |
|---|---|---|---|
| Bootstrap peers (DDNS) | Initial internet connectivity | No — once connected, DHT takes over (Phase 3) | New nodes cannot join; existing nodes continue |
| DuckDNS | DDNS for bootstrap hostname | No — any DDNS or static IP works | Bootstrap hostname unresolvable; see OPERATIONS_RUNBOOK Section 3 |
| Relay nodes | NAT traversal fallback | No — ~70-80% of connections use hole punching | ~5-10% of nodes fall back to direct connection via relay; relay failure degrades but does not break |
| GitHub (source code) | Source distribution | No — the compiled binary is self-contained | Cannot distribute new versions; existing deployments unaffected |

---

## Consequences

### Positive
- The network survives the original developers becoming unavailable (meets the SRD "proprietary backend" constraint)
- No vendor lock-in to any cloud provider, DNS service, or hosting company
- Bootstrap peer failure is graceful: Phase 3 DHT means ongoing peer discovery does not require bootstrap nodes at all
- Resilient against targeted attacks on bootstrap infrastructure — taking down one or two DDNS entries does not take down the network
- Users own their data completely: no account, no subscription, no central key store

### Negative / Trade-offs
- Phase 2 has a soft centralization risk: with only 2 bootstrap nodes, losing both simultaneously would prevent new nodes from joining (but would not affect existing connected nodes)
- Phase 2 mitigation: PD-06 requires 2 bootstrap peers from distinct physical locations; the risk register notes this and the DHT (Phase 3) eliminates it
- Self-hosted relay infrastructure means the team must maintain at least one relay-capable anchor node — but any anchor node can serve as relay, so no single node is critical
- Without central telemetry, monitoring the health of the overall network requires querying individual nodes — no central dashboard exists

### Neutral
- DuckDNS is a free third-party service. If DuckDNS disappears, any other DDNS service (No-IP, afraid.org, Cloudflare DDNS) or a static IP works as a drop-in replacement. The protocol uses a hostname string — the specific service providing it is irrelevant.
- GitHub Actions CI is a convenience, not a runtime dependency. The build can be reproduced locally with `go build` on any developer machine.

---

## What Would Violate This Decision

The following would violate this ADR and must not be introduced:
- Any API call to a server operated by SafeGrowth developers required for storage, retrieval, or consensus
- Any licensing or telemetry check-in to a remote server
- Any credit or reputation operation that requires a specific node operated by the team
- Any hardcoded bootstrap address that resolves to a SafeGrowth-operated server (bootstrap addresses must be user-configurable and replaceable)
- Any cryptographic operation that requires a remote service (e.g., remote key signing, remote certificate authority)

---

## Alternatives Considered

### Alternative A: Central coordinator for shard placement and credit validation
Rejected. This is the Storj satellite model. It creates a single point of failure and a liability concentration point. It also contradicts the SRD constraint directly.

### Alternative B: Use IPFS infrastructure for peer discovery
Considered. IPFS DHT infrastructure is widely deployed and could serve as a bootstrap layer. Rejected because: (a) it introduces a dependency on the IPFS ecosystem's health; (b) SafeGrowth's Phase 3 DHT (libp2p Kademlia) is architecturally equivalent and does not require external infrastructure; (c) mixing SafeGrowth nodes into the IPFS DHT creates routing confusion and potential privacy exposure.

### Alternative C: Use a well-known hardcoded bootstrap node list (like Bitcoin's DNS seeds)
Rejected. A hardcoded list controlled by the development team would be soft centralization. The current design (user-configurable bootstrap peers in config.json) puts control with the node operator, not the developers.

---

## Related ADRs
- ADR-008 (PBFT-lite Consensus) — consensus must not require a central coordinator
- ADR-010 (Credit System Without Cryptocurrency) — credits must not require a central ledger server
