# ADR-008 — PBFT-lite Quorum Consensus (No Blockchain)

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault requires distributed agreement for several operations that must be consistent across nodes:
- Credit ledger updates (preventing double-spending and fraudulent claims)
- Shard placement decisions (ensuring all nodes agree on where shards live)
- Reputation score changes (preventing a single node from unilaterally damaging another's reputation)
- Invite token validation (ensuring a token is consumed exactly once)
- Credit epoch opening (establishing a globally unique sequence number — CL-09)

These operations require Byzantine fault tolerance — the ability to reach correct agreement even when some participants are actively adversarial (not just faulty or offline).

The obvious reference implementations are blockchain-based (Bitcoin PoW, Ethereum PoS, Tendermint BFT). SafeGrowth Vault explicitly prohibits cryptocurrency mechanisms: "No cryptocurrency, blockchain, or token mechanisms" (SRD Section 8, Constraints).

---

## Decision

**Implement a lightweight PBFT-inspired (Practical Byzantine Fault Tolerance) consensus protocol running over small quorums of 5–7 nodes, without a full blockchain.**

Key design parameters:
- Quorum size: 5–7 nodes (not full network) — decisions do not require network-wide participation
- Fault tolerance: f = floor((n-1)/3) Byzantine nodes — for n=7, f=2 (2 malicious nodes cannot corrupt a 7-node quorum)
- Three-phase commit: pre-prepare, prepare, commit (Castro & Liskov 1999)
- Leader election per decision round; view change on leader failure
- All consensus decisions are signed by participating nodes and gossiped as committed records
- Implementation: custom, in the `consensus/` package — no external BFT library dependency

---

## Consequences

### Positive
- No cryptocurrency, no token, no blockchain — consistent with constraints
- Small quorums (5–7 nodes) keep overhead low — most consensus rounds complete in < 1 second on a LAN, < 3 seconds over internet
- Byzantine tolerance at f = floor((n-1)/3) is the theoretical maximum for BFT — no additional margin is possible without more nodes
- Quorums are formed from available trusted peers — full network participation is never required
- Committed records are gossiped — any node can verify a past decision without re-running consensus

### Negative / Trade-offs
- Phase 2 has only ~7 nodes total — the entire trusted group may be the quorum, meaning there is no separation between "network" and "quorum"
- PBFT has O(n²) message complexity within the quorum — fine for n=7, becomes expensive at n=20+. Phase 3 must cap quorum size at 7 and select quorum members carefully
- View change (leader failover) adds latency when the leader node is offline — up to 3 seconds per view change
- Custom implementation carries implementation risk — unit tests must cover all Byzantine scenarios before Phase 2 launch
- PBFT requires a stable trusted membership list — dynamic membership changes during consensus rounds are complex and must be handled carefully

### Neutral
- The credit ledger uses an append-only log model that maps naturally to a replicated state machine — the standard BFT application pattern
- Phase 2's "simple majority for credit validation" (dev plan Week 9) is a simplified version of this — full PBFT is introduced in Phase 3 (Weeks 15-16)

---

## Alternatives Considered

### Alternative A: Blockchain (Bitcoin PoW, Ethereum PoS, Tendermint)
Rejected. Explicitly prohibited by SRD constraints: "No cryptocurrency, blockchain, or token mechanisms." Beyond the policy constraint, blockchain adds token volatility risk, external infrastructure dependency, and significant complexity for a use case that only needs small-quorum agreement.

### Alternative B: Raft consensus
Considered. Raft is simpler to implement correctly than PBFT and is widely used in distributed databases (etcd, CockroachDB). Rejected because: Raft is crash-fault-tolerant, not Byzantine-fault-tolerant. A malicious node can violate Raft's assumptions and corrupt the log. Our threat model (Profile B — malicious node operator) requires Byzantine tolerance.

### Alternative C: HotStuff BFT
Considered. HotStuff (used in DiemBFT/LibraBFT) has O(n) message complexity vs PBFT's O(n²), making it more scalable. Rejected for Phase 2/3 because: (a) HotStuff is more complex to implement correctly; (b) at quorum sizes of 5–7, O(n²) vs O(n) is not meaningful; (c) our team has more reference material for PBFT. HotStuff is noted as the preferred upgrade path if Phase 3 quorum sizes need to grow beyond 10.

### Alternative D: No consensus — last-write-wins
Rejected immediately. Without consensus, the credit ledger can be corrupted by any node that can issue gossip messages. The entire economic model of the network depends on tamper-resistant credit accounting.

---

## Phased Implementation

| Phase | Consensus Approach | Justification |
|---|---|---|
| Phase 1 | None needed | All devices trusted, single user |
| Phase 2 (Week 9) | Simple majority (3-node quorum) | Enough for trusted group; full PBFT overkill |
| Phase 3 (Weeks 15-16) | Full PBFT-lite, 5–7 node quorums | Required for open participation with malicious nodes |

---

## Related ADRs
- ADR-007 (Erasure Coding) — Byzantine fault tolerance analysis informs quorum sizing
- ADR-009 (No Central Servers) — consensus must work without a central coordinator
