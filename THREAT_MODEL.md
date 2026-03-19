# SafeGrowth Vault — Threat Model

**Version**: 1.0  
**Status**: Draft  
**Date**: March 2026  
**Authors**: SafeGrowth Team  
**Scope**: Node Daemon, Networking Layer, Crypto Layer, Credit System, Client Applications

---

## Revision History

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0 | 2026-03 | SafeGrowth Team | Initial threat model — covers all three deployment phases |

---

## 1. Purpose and Scope

This document defines the formal threat model for SafeGrowth Vault. It establishes trust boundaries, attacker profiles, data flow trust zones, and the cryptographic and architectural controls that mitigate each identified threat.

This document is the authoritative reference for security design decisions across all milestones. When a new feature is designed, the implementing developer must reference this document to verify the feature does not introduce unmitigated threats or violate a stated trust boundary.

**In scope:**
- All node-to-node communication (gossip, shard placement, PoR, credit claims)
- Identity management and key storage
- Shard encryption, storage, and retrieval
- The invite and join system
- The credit and reputation system
- Client-to-node communication (CLI and Flutter app)
- NAT traversal and relay infrastructure

**Out of scope:**
- Physical hardware security of node operator premises
- Operating system or kernel-level vulnerabilities on node hardware
- Social engineering attacks against node operators outside the protocol
- Supply chain attacks against Go toolchain or upstream libraries
- Legal and regulatory compliance (see LEGAL_NOTES.md — TBD)

---

## 2. System Overview and Trust Zones

The system is divided into four trust zones. Every data flow crosses at least one zone boundary, and every boundary has defined authentication and authorization controls.

```
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 1: USER DEVICE (Fully Trusted)                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Flutter App / CLI  ←→  Node Daemon (localhost only)     │  │
│  │  Keypair (encrypted at rest)  |  Plaintext data in RAM   │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │ mTLS (ML-DSA cert), encrypted shards only
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 2: TRUSTED PEER NETWORK (Phase 2 — Known Participants)   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Invited nodes only  |  Invite chain verified            │  │
│  │  Gossip signed by ML-DSA  |  mTLS on all connections     │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │ Same transport; additional Sybil/PoR controls
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 3: OPEN PARTICIPATION NETWORK (Phase 3 — Unknown Nodes)  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Any hardware  |  Zero-trust enforcement                 │  │
│  │  PoR challenges  |  Reputation scoring  |  Sybil proofs  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │ Relay path (E2EE preserved)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 4: RELAY / BOOTSTRAP INFRASTRUCTURE (Untrusted Transit)  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Relay nodes see only ciphertext  |  No content access   │  │
│  │  Bootstrap nodes share peer lists only                   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Zone Boundary Controls

| Boundary | Control |
|---|---|
| Zone 1 → Zone 2/3 | mTLS with ML-DSA-derived certificate; all shard data AES-256-GCM encrypted before leaving Zone 1 |
| Zone 2 → Zone 3 | Invite chain validated; resource proof challenge on join |
| Any zone → Zone 4 | E2EE preserved; relay node has no decryption capability by architecture |
| Zone 1 internal | REST/gRPC on localhost only; no external exposure of plaintext API |

---

## 3. Attacker Profiles

### Profile A — Passive Network Observer
**Capability**: Can intercept all network traffic between nodes. Cannot break AES-256-GCM or ML-KEM with current hardware.  
**Goal**: Identify what data is being stored, by whom, and the network topology.  
**Phase active**: All phases.

### Profile B — Malicious Node Operator (Insider Threat)
**Capability**: Operates one or more legitimate nodes in the network. Has full access to their own node's software, storage, and network traffic.  
**Goal**: Read stored data, claim false credits, destabilize the network, drop shards without detection.  
**Phase active**: Phase 2 onwards (requires joining the network).

### Profile C — Sybil Attacker
**Capability**: Can spin up large numbers of fake nodes from a single machine or small cluster.  
**Goal**: Gain disproportionate influence over shard placement, credit accrual, consensus decisions, or reputation scoring. Eclipse a legitimate node.  
**Phase active**: Phase 3 (open participation).

### Profile D — Byzantine / Colluding Nodes
**Capability**: Controls up to f = floor((n-1)/3) nodes within a consensus quorum.  
**Goal**: Corrupt credit ledger entries, falsify reputation scores, approve fraudulent shard placements, or prevent consensus from completing.  
**Phase active**: Phase 2 onwards.

### Profile E — Relay / Bootstrap Node Operator (Privileged Position)
**Capability**: Operates a node that other nodes depend on for connectivity or peer discovery.  
**Goal**: Perform traffic analysis, selectively drop relay traffic, feed false peer lists to joining nodes (eclipse attack).  
**Phase active**: Phase 2 onwards.

### Profile F — Compromised Developer / Supply Chain
**Capability**: Injects malicious code into a dependency or the build pipeline.  
**Goal**: Introduce a backdoor that exfiltrates keys or plaintext.  
**Phase active**: All phases.  
**Note**: This is explicitly out of active scope for v1.0 controls — documented here for completeness and future audit planning.

---

## 4. Data Classification

| Data Type | Sensitivity | Where It Lives | Notes |
|---|---|---|---|
| User plaintext file data | **Critical** | RAM only (Zone 1) | Never written to disk unencrypted; never transmitted |
| User encryption keys (ML-KEM private) | **Critical** | Encrypted at rest (Zone 1 disk) | AES-256-GCM + Argon2id passphrase |
| Node signing keys (ML-DSA private) | **Critical** | Encrypted at rest (Zone 1 disk) | Same keystore as ML-KEM key |
| Encrypted shard data | **High** | Pebble store on node disk (any zone) | Ciphertext only; nodes cannot decrypt |
| Shard manifest (shard IDs, placement metadata) | **Medium** | Pebble store; gossiped across network | Reveals file structure but not content |
| Peer table (NodeIDs, IP:ports, pubkeys) | **Medium** | Disk + gossiped | Reveals network topology |
| Credit ledger entries | **Medium** | Pebble store; consensus-validated | Reveals contribution patterns |
| Reputation scores | **Low** | Gossiped | Public by design |
| Node public keys / NodeIDs | **Low** | Gossiped; in mDNS announcements | Public by design |

---

## 5. Threat Register

Each threat is identified with a unique ID, mapped to the attacker profile(s) that execute it, and linked to the control(s) that mitigate it.

---

### 5.1 Data Confidentiality Threats

#### T-01 — Passive Traffic Analysis (Content Inference)
**Profile**: A  
**Description**: A passive observer captures encrypted shard traffic and attempts to infer file content or identity from packet sizes, timing, or frequency patterns.  
**Impact**: User data privacy compromise; network topology exposure.  
**Phase**: All.

**Controls**:
- All shards padded to nearest power-of-two boundary before encryption (NF-17 / Section 9.4 of PROTOCOL_SPEC)
- Retrieval responses include 0–500ms random jitter delay before transmission (NF-17)
- Shard IDs are SHA3-256 hashes — reveal nothing about content
- mTLS encrypts all transport metadata

**Residual Risk**: Low. Padding and jitter substantially reduce size and timing correlation. A sophisticated long-term observer with many vantage points could still perform probabilistic traffic analysis, but this is beyond the threat model for v1.0.

---

#### T-02 — Node Reads Stored Shard Content
**Profile**: B  
**Description**: A malicious node operator attempts to read the content of shards stored on their node.  
**Impact**: User data exposure.  
**Phase**: All.

**Controls**:
- Nodes store only AES-256-GCM encrypted ciphertext; encryption keys never leave Zone 1
- ML-KEM key exchange ensures only the data owner can derive the symmetric key
- Node daemon has no MIME detection, content scanning, or plaintext access hooks (NF-12)
- ZK-PoR (Phase 3) proves shard possession without exposing content even during verification

**Residual Risk**: Negligible. This is an architectural guarantee, not a policy control.

---

#### T-03 — Key Exfiltration from Node Keystore
**Profile**: B, F  
**Description**: An attacker gains access to the node's filesystem and attempts to extract private keys.  
**Impact**: Identity theft; ability to impersonate node; decryption of historical data if forward secrecy is not in use.  
**Phase**: All.

**Controls**:
- Private keys stored encrypted with AES-256-GCM, key derived from user passphrase via Argon2id (ID-04)
- Passphrase never stored; required on daemon startup
- Forward secrecy ratchet (CR-06, Phase 2) limits exposure of historical data even if current key is compromised
- Security-relevant config requires passphrase re-authentication to change (NF-13)

**Residual Risk**: Medium. A keylogger or memory dump on a running node could expose the passphrase or derived key. Mitigated by OS-level security (out of scope for this model) and key rotation procedure (see OPERATIONS_RUNBOOK.md).

---

### 5.2 Integrity Threats

#### T-04 — Corrupt Shard Injection
**Profile**: B, D  
**Description**: A malicious node substitutes a corrupt or tampered shard in response to a retrieval request.  
**Impact**: File reconstruction failure; silent data corruption.  
**Phase**: All.

**Controls**:
- Every shard carries a SHA3-256 Merkle hash signed by the originating node (SH-08)
- Receiver verifies signature before accepting shard; corrupt or unsigned shards are rejected (Section 9.3 of PROTOCOL_SPEC)
- Reed-Solomon reconstruction can recover from up to (n-k) corrupt or missing shards
- PoR challenges continuously verify shard integrity on holder nodes (Phase 3)

**Residual Risk**: Low. Reed-Solomon + signed Merkle hashes provide strong integrity guarantees. A colluding majority of shard holders could corrupt an un-recoverable file, mitigated by geographic and ASN diversity placement.

---

#### T-05 — Gossip Message Forgery
**Profile**: B, D  
**Description**: A node injects forged gossip messages — fake peer announcements, fraudulent credit updates, false reputation scores — into the network.  
**Impact**: Network state corruption; false credit accrual; reputation manipulation.  
**Phase**: Phase 2 onwards.

**Controls**:
- All gossip messages signed by originating node's ML-DSA key (GP-03)
- Unsigned or invalid-signature messages are dropped without processing (Section 8, PROTOCOL_SPEC)
- Version vectors deduplicate messages; replayed old messages are discarded (GP-02)
- Timestamp validation in MessageEnvelope: messages older than 30 seconds rejected (Section 4.3, PROTOCOL_SPEC)
- msg_id deduplication with 60-second sliding window prevents replay attacks (Section 4.3)

**Residual Risk**: Low. A node can only forge messages under its own identity; it cannot forge messages from other nodes without their private key.

---

#### T-06 — Credit Ledger Manipulation
**Profile**: B, D  
**Description**: A node submits fraudulent credit claims or attempts to double-spend credits.  
**Impact**: Unfair resource allocation; economic imbalance in the network.  
**Phase**: Phase 2 onwards.

**Controls**:
- Credit claims require consensus validation before acceptance (CL-02)
- Consensus tolerates up to f = floor((n-1)/3) Byzantine nodes (BC-03)
- Each epoch has a globally unique sequence number established by consensus before opening (CL-09)
- Conflicting claims for the same epoch resolved in favour of higher-reputation quorum (CL-09)
- Phase 3: credit claims require a valid ZK-PoR proof of actual storage (CL-04)
- 72-hour probation period prevents new nodes from accruing credits before reputation is established (CL-06)

**Residual Risk**: Medium (Phase 2), Low (Phase 3). Phase 2 consensus without ZK-PoR is vulnerable to colluding nodes within a quorum. Phase 3 ZK-PoR requirement closes this gap.

---

### 5.3 Availability Threats

#### T-07 — Shard Dropping (Silent Data Loss)
**Profile**: B  
**Description**: A node accepts shard placement, acknowledges storage, but quietly deletes or never stores the shard. It continues to respond to metadata queries as if holding the shard.  
**Impact**: Reduced redundancy; potential data loss if enough holders drop shards simultaneously.  
**Phase**: All.

**Controls**:
- PoR challenges (Phase 3) detect absence within a 6-hour challenge window (PR-08)
- Three consecutive PoR failures trigger shard migration to a healthy node (PR-05)
- Reputation score penalized on each failed challenge (PR-04)
- Replication manager continuously monitors replica count and re-replicates when below threshold (SH-03, NF-07)

**Residual Risk**: Medium (Phase 1/2 — no PoR), Low (Phase 3). In Phase 1 and 2, silent shard dropping is only detected when retrieval is attempted. Mitigated by erasure coding redundancy (2-of-3, 3-of-5).

---

#### T-08 — Gossip Storm / Bandwidth Exhaustion
**Profile**: B, C  
**Description**: A malicious or misconfigured node floods the gossip layer with high-volume messages, consuming peer bandwidth.  
**Impact**: Network degradation; node resource exhaustion.  
**Phase**: Phase 2 onwards.

**Controls**:
- Per-source gossip rate limiting: 100 messages/second per peer; excess messages dropped (GP-08, Section 8.5 of PROTOCOL_SPEC)
- Temporary 5-minute ban after 3 rate-limit violations within 60 seconds (Section 8.5)
- Maximum gossip message size: 1MB; oversized messages dropped and ban counter incremented (GP-09)
- Gossip bandwidth cap: default 10% of available upload (GP-06)
- Version vectors prevent redundant re-propagation of already-seen messages (GP-02)

**Residual Risk**: Low. Rate limiting and size caps bound worst-case bandwidth consumption per peer.

---

#### T-09 — Bootstrap / Relay Node Denial of Service
**Profile**: A, B, E  
**Description**: The bootstrap or relay node becomes unavailable (attack or failure), preventing new nodes from joining or relayed nodes from communicating.  
**Impact**: Network partitioning for nodes behind symmetric NAT; inability to bootstrap new participants.  
**Phase**: Phase 2 onwards.

**Controls**:
- Phase 2 requires minimum 2 bootstrap peers from distinct physical locations (PD-06, NF-08)
- Daemon logs startup warning if fewer than 2 bootstrap peers are reachable (PD-06)
- DHT (Phase 3) removes bootstrap dependency for ongoing operation (PD-04)
- No single node is a single point of failure by design (NF-08)
- Relay is fallback only; direct P2P via hole punching is preferred path for ~70-80% of connections

**Residual Risk**: Low (Phase 3). Medium (Phase 2) if only one bootstrap is configured — the requirement for two distinct bootstrap peers must be enforced operationally.

---

### 5.4 Identity and Authentication Threats

#### T-10 — Sybil Attack (Mass Fake Node Creation)
**Profile**: C  
**Description**: An attacker creates many fake node identities from a single machine to gain disproportionate influence over shard placement, reputation, or consensus.  
**Impact**: Attacker controls placement of shards; can coordinate to drop data; can dominate consensus quorums.  
**Phase**: Phase 3 (open participation).

**Controls**:
- Resource proof challenge on join: joining node must store 256MB of random challenge data and prove it within 60 seconds (Weeks 11-12)
- Computational puzzle on join makes mass node creation expensive (Weeks 11-12)
- 72-hour probation: new nodes cannot influence consensus or primary shard placement (RP-04)
- Reputation takes time to build; new nodes cannot immediately achieve quorum influence
- Invite chain in Phase 2 limits Sybil risk to trusted participants

**Residual Risk**: Medium. Resource proofs and PoW raise the cost of Sybil attacks but a well-resourced attacker with diverse hardware could still attempt it. Social vouching and invite chains (Phase 2) provide stronger guarantees for the trusted group phase.

---

#### T-11 — Eclipse Attack
**Profile**: C, E  
**Description**: An attacker gains control of all peer connections to a target node, feeding it a false view of the network — routing all of its gossip, peer discovery, and shard requests through attacker-controlled nodes.  
**Impact**: Target node receives false data; its shards are misdirected; it may lose contact with legitimate peers.  
**Phase**: Phase 2 onwards.

**Controls**:
- Peer list diversity enforcement: replicas must span at least 3 distinct ASNs and geographic regions (SH-05, SH-06)
- Bootstrap peers from at least 2 distinct physical locations (PD-06)
- Peer table persists to disk; daemon cannot be fully eclipsed by a single connection sequence
- DHT (Phase 3) distributes peer discovery; no single node controls the routing table
- mDNS discovery (Phase 1) is broadcast-based and not spoofable on a LAN without ARP poisoning

**Residual Risk**: Low (Phase 3 with DHT). Medium (Phase 2 without DHT, if bootstrap diversity requirement is not operationally enforced).

---

#### T-12 — Man-in-the-Middle on Shard Transfer
**Profile**: A, E  
**Description**: An attacker intercepts shard transfer between nodes, attempting to read or modify the shard data in transit.  
**Impact**: Data confidentiality or integrity breach during transfer.  
**Phase**: All.

**Controls**:
- All inter-node connections use mutual TLS 1.3 (NT-03, CR-08)
- TLS certificate is derived from the node's ML-DSA keypair; impersonation requires key compromise
- Certificate verification checks that presented cert's public key matches the expected NodeID — not standard X.509 chain (Section 5.2 of PROTOCOL_SPEC)
- ML-KEM key exchange provides quantum-resistant confidentiality
- Shard Merkle hash verified after receipt; tampered shards are rejected before storage

**Residual Risk**: Negligible. mTLS with identity-bound certificates and quantum-resistant key exchange provides strong protection.

---

#### T-13 — Invite Token Interception and Replay
**Profile**: A, B  
**Description**: An attacker intercepts an invite token transmitted over an insecure channel (email, Signal) and attempts to use it before the legitimate recipient.  
**Impact**: Attacker joins the network as a trusted peer using a stolen token.  
**Phase**: Phase 2 onwards.

**Controls**:
- Tokens are single-use: consumed on first valid JOIN_REQUEST; consumed state gossiped to all peers (IN-01, Section 7.1 of PROTOCOL_SPEC)
- Default expiry: 24 hours (IN-02)
- No secrets are embedded in the token itself — intercepting the token does not expose keys (IN-05)
- If a race condition occurs (both attacker and legitimate node attempt to use simultaneously), first valid use wins; issuing node should be notified out-of-band if unexpected join occurs

**Residual Risk**: Low. Single-use tokens with short expiry minimize the attack window. Users should be advised to share tokens via authenticated channels when possible.

---

### 5.5 Protocol-Level Threats

#### T-14 — Timestamp / Replay Attack on Message Envelope
**Profile**: B  
**Description**: An attacker records a legitimate signed gossip or protocol message and replays it later to cause duplicate processing (double credit claim, duplicate shard placement, etc.).  
**Impact**: State corruption; duplicate credit claims.  
**Phase**: Phase 2 onwards.

**Controls**:
- MessageEnvelope timestamp validation: messages older than 30 seconds rejected (E-005)
- msg_id deduplication: 60-second sliding window of seen message IDs; duplicates silently dropped (Section 4.3 of PROTOCOL_SPEC)
- Credit epoch sequence number prevents claims referencing closed epochs (CL-09)

**Residual Risk**: Low. 30-second replay window is industry standard for this class of protocol.

---

#### T-15 — PoR Pre-computation (Fake Storage)
**Profile**: B  
**Description**: A node attempts to pass PoR challenges without actually holding the shard, by pre-computing challenge responses in advance.  
**Impact**: Node earns credits and maintains reputation for shards it is not actually storing; data availability is degraded.  
**Phase**: Phase 3.

**Controls**:
- Challenge parameters {start, end, nonce} are derived fresh by the verifier from shared seed + a new random salt per challenge (PR-02, Section 10.4 of PROTOCOL_SPEC)
- Salt is unknown to the holder until the challenge is issued; pre-computation is architecturally prevented
- Shared challenge seed established via ML-KEM key exchange between holder and at least 3 verifying peers (PR-01)
- Holder stores only the seed, not pre-computed answers (PR-01)

**Residual Risk**: Low. The fresh-salt architecture is specifically designed to prevent pre-computation. Correctness of this design is a critical review item for the Week 13-14 implementation.

---

#### T-16 — Version Downgrade Attack
**Profile**: A, B  
**Description**: An attacker forces protocol version negotiation to an older, less secure version of the protocol that lacks mTLS, signed gossip, or PoR.  
**Impact**: Loss of security properties available in higher protocol versions.  
**Phase**: All.

**Controls**:
- Version negotiation occurs during authenticated handshake; both nodes must present valid ML-DSA certificates
- Negotiated version = min(peer_max, local_max); neither side can force a version the other does not support
- Nodes should be configured with a minimum acceptable version; operator can set `min_version` to reject downgrades (Section 2.1 of PROTOCOL_SPEC)
- Unknown fields in messages trigger E-010 log entries; unknown message types are rejected with E-002

**Residual Risk**: Low. Version negotiation is authenticated and bounded; forced downgrade requires compromise of a peer's identity key.

---

## 6. Threat Summary Matrix

| ID | Threat | Profile | Phase | Impact | Controls | Residual Risk |
|---|---|---|---|---|---|---|
| T-01 | Passive traffic analysis | A | All | Medium | Padding, jitter, mTLS | Low |
| T-02 | Node reads shard content | B | All | Critical | E2EE, content-blind arch | Negligible |
| T-03 | Key exfiltration | B, F | All | Critical | Encrypted keystore, Argon2id, forward secrecy | Medium |
| T-04 | Corrupt shard injection | B, D | All | High | Signed Merkle hash, RS coding | Low |
| T-05 | Gossip message forgery | B, D | P2+ | High | ML-DSA signatures, replay protection | Low |
| T-06 | Credit ledger manipulation | B, D | P2+ | Medium | BFT consensus, ZK-PoR (P3) | Medium (P2), Low (P3) |
| T-07 | Silent shard dropping | B | All | High | PoR challenges (P3), replication monitoring | Medium (P1/2), Low (P3) |
| T-08 | Gossip storm | B, C | P2+ | Medium | Rate limiting, size caps, BW cap | Low |
| T-09 | Bootstrap/relay DoS | A, B, E | P2+ | High | 2+ bootstrap peers, DHT (P3) | Medium (P2), Low (P3) |
| T-10 | Sybil attack | C | P3 | High | Resource proof, PoW, probation | Medium |
| T-11 | Eclipse attack | C, E | P2+ | High | ASN/geo diversity, DHT (P3) | Medium (P2), Low (P3) |
| T-12 | MITM on shard transfer | A, E | All | High | mTLS, ML-KEM, Merkle verify | Negligible |
| T-13 | Invite token replay | A, B | P2+ | Medium | Single-use, 24h expiry | Low |
| T-14 | Message replay | B | P2+ | Medium | Timestamp window, msg_id dedup | Low |
| T-15 | PoR pre-computation | B | P3 | High | Fresh-salt challenge arch | Low |
| T-16 | Version downgrade | A, B | All | Medium | Authenticated negotiation, min_version | Low |

---

## 7. Controls Not Yet Implemented (Gap Register)

The following controls are designed but not yet implemented. Each gap has an elevated residual risk until the corresponding milestone delivers the control.

| Gap | Threat(s) Affected | Milestone | Risk Until Resolved |
|---|---|---|---|
| PoR challenge engine | T-07, T-15 | M3 (Weeks 13-14) | Medium — shard dropping undetected in P1/P2 |
| ZK-PoR proof requirement for credit claims | T-06 | M3 (Weeks 15-16) | Medium — credit fraud possible in P2 |
| Resource proof and PoW on join | T-10 | M3 (Weeks 11-12) | High — Sybil attacks uninhibited in P2 (mitigated by invite chain) |
| DHT peer discovery | T-09, T-11 | M3 (Weeks 19-20) | Medium — bootstrap dependency in P2 |
| Forward secrecy ratchet (CR-06) | T-03 | M2 (Week 5-6) | Medium — key compromise exposes full history in P1 |
| ASN diversity enforcement | T-11 | M3 (SH-06) | Low — geo diversity (M2) partially mitigates |

---

## 8. Security Assumptions

The following assumptions are explicitly made by this threat model. If any assumption is violated, the associated threats must be re-evaluated.

1. **Go standard library cryptographic implementations are correct.** `crypto/mlkem`, `crypto/aes`, `crypto/cipher`, `crypto/sha3`, `crypto/hkdf` are assumed to be correctly implemented and free of exploitable bugs.

2. **Trail of Bits `ml-dsa` implementation is correct.** The `github.com/trailofbits/ml-dsa` library is authored by a reputable security firm and is assumed to implement NIST FIPS 204 correctly.

3. **NIST FIPS 203/204 post-quantum primitives are not broken.** ML-KEM and ML-DSA are assumed to be secure against both classical and quantum adversaries for the duration of this system's operational life.

4. **AES-256-GCM provides 128-bit quantum-resistant symmetric security.** Grover's algorithm halves symmetric key security; AES-256 provides ~128 bits of post-quantum security, which is considered sufficient.

5. **Argon2id with appropriate parameters resists offline dictionary attacks.** The passphrase KDF is assumed to make brute-force attacks on the keystore infeasible on commodity hardware.

6. **The operating system process isolation prevents cross-process memory reads.** The daemon's in-memory plaintext data (decrypted shards during reconstruction) is assumed to be inaccessible to other processes without root/admin access.

7. **The node operator's physical hardware is not under adversary control.** Physical access attacks (cold boot, DMA, hardware implants) are out of scope.

---

## 9. Open Security Questions

The following questions require resolution before the relevant milestone. They represent design decisions with security implications that have not yet been finalized.

| # | Question | Impact | Target | Owner |
|---|---|---|---|---|
| 1 | What is the canonical encoding for ML-DSA signatures in MessageEnvelope — raw bytes or DER? Encoding ambiguity could enable signature bypass on heterogeneous implementations. | T-05, T-12 | Week 1 | Glenn |
| 2 | Should HANDSHAKE_HELLO include a PoW field in Phase 3 for Sybil resistance at the connection layer, or is join-layer resource proof sufficient? | T-10 | Week 10 | TBD |
| 3 | What is the Argon2id parameterization (memory, iterations, parallelism) for the keystore KDF? Parameters must be validated against the weakest target hardware (Raspberry Pi 3B) to ensure both security and usability. | T-03 | Week 1 | Glenn |
| 4 | How are challenge seeds rotated if a verifier peer leaves the network permanently? Stale seeds could allow a holder to retain challenges whose verifier is no longer reachable. | T-15 | Week 13 | TBD |
| 5 | Should gossip reputation updates be aggregated before propagation to limit a single observer's ability to skew aggregate scores? | T-05, T-06 | Week 17 | TBD |

---

## 10. Relationship to Other Documents

| Document | Relationship |
|---|---|
| `PROTOCOL_SPEC.docx` | Implements controls for T-05, T-12, T-13, T-14, T-15, T-16. This threat model is the security basis for protocol design decisions therein. |
| `safegrowth-requirements.md` | Security requirements (NF-10 through NF-17, CR-01 through CR-08) derive from threats identified here. |
| `safegrowth-devplan.md` | Milestone schedule determines when gap-register controls are delivered and residual risks are resolved. |
| `OPERATIONS_RUNBOOK.md` (TBD) | Operational controls for T-03 (key rotation), T-09 (bootstrap failover), T-07 (manual re-replication) |
| `BENCHMARKS.md` (TBD) | ZK-PoR performance (T-15 control) must meet the ≤2s target on OptiPlex 990 documented there |
| `LEGAL_NOTES.md` (TBD) | Content-blind architecture (T-02 control) is the basis for the legal conduit argument |
