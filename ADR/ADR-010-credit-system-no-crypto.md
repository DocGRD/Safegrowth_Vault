# ADR-010 — Credit System Without Cryptocurrency

**Status**: Accepted  
**Date**: 2026-03  
**Authors**: SafeGrowth Team  
**Deciders**: Glenn + remote collaborators  

---

## Context

SafeGrowth Vault needs a contribution-accounting mechanism that rewards nodes for storing data and providing uptime, and that throttles nodes that consume without contributing. Without such a mechanism, the network is vulnerable to free-riding: nodes that consume storage and bandwidth without contributing anything in return.

The obvious existing models are cryptocurrency-based: Filecoin uses FIL tokens, Sia uses Siacoin, Storj uses STORJ tokens. These systems have been in production for years and have proven that crypto-economic incentives can drive decentralized storage participation.

However, the SRD constraint is explicit: "No cryptocurrency, blockchain, or token mechanisms" (Section 8, Constraints). The reasons for this constraint are:
1. Token volatility makes the system's economic parameters unpredictable — a node's credit balance becomes worthless if the token crashes
2. Cryptocurrency requirements add technical and legal complexity for ordinary users
3. Cryptocurrency exchanges create regulatory exposure
4. Existing token-based systems have struggled with free-rider problems and market manipulation
5. The system is intended for personal and small-group use first — cryptocurrency overkill for this use case

---

## Decision

**Implement a non-transferable, non-tradable internal credit system inspired by BitTorrent's tit-for-tat mechanism.**

Core design:
- Credits are earned for contribution: storage (+1 credit/GB/month), uptime (+0.5/hr anchor, +0.25/hr mid-tier)
- Credits are spent for consumption: priority bandwidth during retrieval, faster re-replication
- Credits exist only within the SafeGrowth network — no external exchange mechanism
- Credits are non-transferable between nodes
- Credits are capped per epoch via consensus to prevent runaway accumulation
- Contribution accounting tracks bytes-stored-for-others vs bytes-stored-by-others; persistent net takers are throttled (not blocked)
- Phase 3: all credit claims require a valid ZK-PoR proof of actual storage

The credit ledger is an append-only log stored in Pebble, validated by PBFT-lite consensus (ADR-008). No blockchain, no global ledger, no token.

---

## Consequences

### Positive
- No token volatility — credits are internal accounting units, not financial instruments
- No cryptocurrency wallet required — zero barrier for non-technical users
- No regulatory exposure from token issuance or exchange
- Tit-for-tat model is proven in BitTorrent at massive scale
- Credits as internal units can be recalibrated (accrual rates, spending costs) by network-wide governance without affecting external markets
- Non-transferability eliminates credit markets, speculation, and griefing via credit accumulation and dumping

### Negative / Trade-offs
- No external economic incentive for strangers to join and contribute hardware — unlike FIL/STORJ, you cannot "earn money" by running a node
- Free-rider deterrence is softer than cryptocurrency: a persistent free-rider is throttled, not financially penalized
- The credit system provides relative priority rather than hard guarantees — a node with many credits gets faster retrieval, but a node with no credits still gets retrieval (eventually)
- Without transferability, there is no way to compensate a node operator for their electricity and hardware costs in monetary terms — participation must be intrinsically motivated (data sovereignty, supporting friends) rather than financially motivated

### Neutral
- The non-financial nature of credits simplifies legal analysis (credits are not securities, not money, not property in most jurisdictions — consult LEGAL_NOTES.md when that document is created)
- Phase 1 and Phase 2 do not require credits at all — the system is useful without them; credits add fairness incentives for Phase 3 open participation

---

## Anti-Gaming Mechanisms

The following mechanisms prevent credit gaming (see also THREAT_MODEL.md T-06):

1. **ZK-PoR requirement (Phase 3)**: credit claims require proof of actual shard possession — you cannot claim credits for shards you don't hold
2. **Probation period**: new nodes earn zero credits for 72 hours — prevents rapid Sybil farming
3. **Epoch caps**: credits per node are capped per epoch via consensus — prevents runaway accumulation
4. **Reputation dependency**: nodes below reputation 0.6 earn zero credits — poor behavior cuts off earning
5. **Consensus validation**: all credit claims are validated by a BFT quorum — a single node cannot self-issue credits

---

## Alternatives Considered

### Alternative A: Filecoin / FIL integration
Rejected. Token volatility, cryptocurrency wallet requirement, and explicit SRD constraint against cryptocurrency mechanisms.

### Alternative B: Storj-style micropayments
Rejected. Same reasons as Filecoin. Also, Storj's satellite coordination model contradicts ADR-009.

### Alternative C: No contribution accounting (pure altruism model)
Considered for Phase 1/2. Acceptable for the personal and trusted-group phases where all participants have social relationships. Rejected for Phase 3: open participation without contribution accounting creates free-rider dynamics that degrade the network for legitimate users.

### Alternative D: Reputation-only (no credits) — throttle low-reputation nodes without a credit balance
Considered as a simplification. The reputation system (Section 3.8, SRD) already throttles bad actors. Credits add a positive incentive layer on top of the negative reputation layer — both are needed for a healthy contribution economy. Credits also provide a smoother UX signal ("your credit balance is low — store more data for others") than the binary reputation threshold.

---

## Future Considerations

If the network grows to a point where external economic incentives are needed to attract hardware, the credit system can be extended (not replaced) with:
- Credit-for-service exchanges within a closed group of known participants
- Integration with existing P2P credit systems (mutual credit, LETS)

These would be implemented as network-level extensions and would not require a protocol change to the core credit ledger format. They are explicitly not planned for v1.0 and would require a new ADR before implementation.

---

## Related ADRs
- ADR-008 (PBFT-lite Consensus) — credit ledger validation mechanism
- ADR-009 (No Central Servers) — credit ledger must not require a central server
