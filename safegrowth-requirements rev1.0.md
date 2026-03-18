# SafeGrowth Vault — Software Requirements Document (SRD)

**Version**: 1.0  
**Status**: Draft  
**Scope**: Node Daemon, CLI Client, Flutter App, Networking Layer, Crypto Layer, Credit System

---

## 1. Overview

This document defines the functional and non-functional requirements for the SafeGrowth Vault software stack. Requirements are organized by component and tagged with priority: **[P1]** = must-have for Phase 1, **[P2]** = required for Phase 2, **[P3]** = required for Phase 3.

---

## 2. System Components

```
┌─────────────────────────────────────────────────────┐
│                  Flutter App / CLI                   │  ← User interface
└───────────────────────┬─────────────────────────────┘
                        │ gRPC / REST (mutual TLS)
┌───────────────────────▼─────────────────────────────┐
│                   Node Daemon                        │  ← Core service
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │ Identity │ │ Storage  │ │ Gossip   │ │ PoR    │ │
│  │ Manager  │ │ Engine   │ │ Protocol │ │ Engine │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │ Crypto   │ │ Shard    │ │ Credit   │ │ Reputa-│ │
│  │ Layer    │ │ Manager  │ │ Ledger   │ │ tion   │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
└───────────────────────┬─────────────────────────────┘
                        │ libp2p (QUIC + TCP)
┌───────────────────────▼─────────────────────────────┐
│              P2P Networking Layer                    │  ← Transport
│   mDNS Discovery | QUIC Transport | Relay | DHT     │
└─────────────────────────────────────────────────────┘
```

---

## 3. Node Daemon Requirements

### 3.1 Identity Management

| ID | Requirement | Priority |
|---|---|---|
| ID-01 | On first run, generate an ML-DSA keypair and persist it to `~/.safegrowth/identity` encrypted with a user passphrase | P1 |
| ID-02 | Generate a separate ML-KEM keypair for key encapsulation; persist alongside identity key | P1 |
| ID-03 | Node identity must be stable across IP address changes, reboots, and network migrations | P1 |
| ID-04 | Identity file must be encrypted at rest using AES-256-GCM with a key derived from user passphrase via Argon2id | P1 |
| ID-05 | Node must expose its public key and current IP:port via mDNS announcements | P1 |
| ID-06 | Support keypair export and import for hardware migration | P2 |
| ID-07 | Support keypair revocation broadcast via gossip (marks keypair as invalid network-wide) | P3 |

### 3.2 Storage Engine

| ID | Requirement | Priority |
|---|---|---|
| ST-01 | Store shards as opaque binary blobs keyed by shard ID (SHA3-256 hash of content) in RocksDB or sled | P1 |
| ST-02 | Support shard write, read, delete, and existence-check operations | P1 |
| ST-03 | Enforce configurable maximum storage quota; reject shard writes when quota is exceeded | P1 |
| ST-04 | All shards must be stored encrypted; plaintext must never be written to disk | P1 |
| ST-05 | Maintain a shard manifest: shard ID, size, owner public key, placement timestamp, last verified timestamp | P1 |
| ST-06 | Support atomic shard commit (write-verify-confirm) to prevent partial writes | P2 |
| ST-07 | Expose storage metrics: total capacity, used space, shard count, per-owner breakdown | P1 |

### 3.3 Encryption Layer

| ID | Requirement | Priority |
|---|---|---|
| CR-01 | Encrypt all user data with AES-256-GCM before sharding; key derived from user's ML-KEM keypair | P1 |
| CR-02 | Use ML-KEM (CRYSTALS-Kyber, NIST FIPS 203) for all key encapsulation operations | P1 |
| CR-03 | Use ML-DSA (CRYSTALS-Dilithium, NIST FIPS 204) for all digital signatures | P1 |
| CR-04 | Use SHA3-256 or BLAKE3 for all hashing operations | P1 |
| CR-05 | Implement hybrid mode: run classic ECDH alongside ML-KEM for nodes that have not yet upgraded | P2 |
| CR-06 | Implement forward-secrecy ratchet for group shared keys (Signal-style double ratchet or equivalent) | P2 |
| CR-07 | Integrate liboqs or equivalent for post-quantum primitives; no custom cryptographic implementations | P1 |
| CR-08 | All inter-node connections must use mutual TLS; TLS certificates derived from node keypairs | P2 |

### 3.4 Sharding and Replication

| ID | Requirement | Priority |
|---|---|---|
| SH-01 | Split files using Reed-Solomon erasure coding; scheme configurable per phase (2-of-3, 3-of-5, 5-of-9) | P1 |
| SH-02 | Default scheme: 2-of-3 for Phase 1, 3-of-5 for Phase 2, 5-of-9 for Phase 3 | P1 |
| SH-03 | Replication manager must enforce exact target replica count; trigger re-replication when count drops below threshold | P1 |
| SH-04 | Shard placement must prefer anchor-tier nodes; fall back to mid-tier only when anchors are insufficient | P2 |
| SH-05 | Enforce geographic diversity: no more than 2 replicas in the same geographic region (MaxMind GeoLite2, bundled) | P2 |
| SH-06 | Enforce ASN diversity in Phase 3: replicas must span at least 3 distinct autonomous systems | P3 |
| SH-07 | Replication manager must migrate shards off nodes whose reputation falls below a configurable threshold | P3 |
| SH-08 | Each shard must carry a Merkle hash signed by the originating node; verify signature on retrieval | P2 |
| SH-09 | Reconstruction must succeed with any qualifying subset of shards (e.g., any 2 of 3) | P1 |

### 3.5 Gossip Protocol

| ID | Requirement | Priority |
|---|---|---|
| GP-01 | Implement epidemic-style gossip: each node maintains a partial peer list and fans out updates to random peers | P1 |
| GP-02 | Use version vectors to deduplicate messages; discard messages with version already seen | P1 |
| GP-03 | All gossip messages must be signed by the originating node's ML-DSA key; unsigned messages discarded | P2 |
| GP-04 | Support gossip message types: peer announcement, shard placement, credit update, reputation update, invite validation, node revocation | P2 |
| GP-05 | Implement anti-entropy: nodes periodically sync full state with a random peer to repair divergence | P2 |
| GP-06 | Smart filtering: gossip bandwidth must not exceed a configurable cap (default 10% of available upload) | P2 |
| GP-07 | Regional grouping: prefer gossiping with geographically close peers first to reduce latency | P3 |

### 3.6 Byzantine Consensus

| ID | Requirement | Priority |
|---|---|---|
| BC-01 | Implement lightweight PBFT-inspired consensus for quorums of 5–7 nodes | P2 |
| BC-02 | Use consensus for: credit ledger updates, shard placement decisions, reputation changes, invite validation | P2 |
| BC-03 | Tolerate up to f = floor((n-1)/3) Byzantine participants in a quorum of n nodes | P2 |
| BC-04 | Consensus must not require full network participation; quorums are formed from available trusted peers | P2 |
| BC-05 | Consensus decisions must be signed by all participating nodes and gossiped as a committed record | P2 |

### 3.7 Proof of Retrievability

| ID | Requirement | Priority |
|---|---|---|
| PR-01 | On shard placement, pre-compute a set of at least 100 random PoR challenges against the shard content | P3 |
| PR-02 | Challenges must be of the form: hash(bytes[start:end] XOR nonce) where start, end, and nonce are random | P3 |
| PR-03 | Node daemon must respond to incoming PoR challenges within 5 seconds for shards under 1GB | P3 |
| PR-04 | Failed PoR challenge: immediately reduce node reputation score by configurable penalty | P3 |
| PR-05 | Three consecutive failed challenges from the same node: trigger shard migration off that node | P3 |
| PR-06 | PoR challenges must be issued by random peers, not a central authority | P3 |
| PR-07 | Implement ZK-PoR: node proves shard possession via zk-SNARK without transmitting shard content | P3 |
| PR-08 | Challenge frequency: at least 1 challenge per shard per 6-hour window per replica | P3 |

### 3.8 Reputation System

| ID | Requirement | Priority |
|---|---|---|
| RP-01 | Each node maintains a signed reputation record for all known peers | P3 |
| RP-02 | Reputation score is computed from: uptime ratio, PoR pass rate, retrieval latency, peer reports | P3 |
| RP-03 | Reputation records are gossip-propagated and signed; unsigned or unverifiable records discarded | P3 |
| RP-04 | New nodes enter a 72-hour probation period: no credit accrual, not eligible for primary shard placement | P3 |
| RP-05 | Nodes with reputation below configurable threshold (default 0.6 / 1.0) are quarantined: no new shard placements | P3 |
| RP-06 | Quarantined nodes trigger automatic shard migration to healthy nodes | P3 |

### 3.9 Credit Ledger

| ID | Requirement | Priority |
|---|---|---|
| CL-01 | Maintain a local append-only credit log; entries are signed by the issuing node | P2 |
| CL-02 | Credit claims must be validated by consensus before acceptance | P2 |
| CL-03 | Credit accrual rates: +1/GB/month stored, +0.5/hr uptime (anchor), +0.25/hr uptime (mid) | P2 |
| CL-04 | In Phase 3: credit claims must include a valid ZK-PoR proof | P3 |
| CL-05 | Credits are non-transferable and non-tradable; no external exchange mechanism | P2 |
| CL-06 | New nodes in probation period accrue zero credits | P3 |
| CL-07 | Implement contribution accounting: track bytes-stored-for-others vs. bytes-stored-by-others | P3 |
| CL-08 | Nodes with persistent negative contribution balance (net takers) receive throttled retrieval priority | P3 |

---

## 4. Networking Layer Requirements

### 4.1 Transport

| ID | Requirement | Priority |
|---|---|---|
| NT-01 | Primary transport: QUIC over UDP (go-quic or equivalent) | P1 |
| NT-02 | Fallback transport: TCP for networks that block UDP | P1 |
| NT-03 | All connections use mutual TLS; certificate derived from node ML-DSA keypair | P2 |
| NT-04 | Connection establishment must complete within 10 seconds on typical home internet | P2 |

### 4.2 Peer Discovery

| ID | Requirement | Priority |
|---|---|---|
| PD-01 | Implement mDNS peer discovery for LAN; nodes announce keypair + current IP:port every 30 seconds | P1 |
| PD-02 | Support bootstrap peer list in config: list of DDNS hostnames or IP:port for initial internet connectivity | P2 |
| PD-03 | On startup, connect to all bootstrap peers and request their full trusted peer list | P2 |
| PD-04 | Implement DHT-based global peer discovery for Phase 3 | P3 |
| PD-05 | Peer table must survive restarts; persist to disk and reload on startup | P1 |

### 4.3 NAT Traversal

| ID | Requirement | Priority |
|---|---|---|
| NAT-01 | Implement UDP hole punching via a rendezvous mechanism (can use a bootstrap peer as rendezvous) | P2 |
| NAT-02 | Implement relay fallback for symmetric NAT; relay node forwards encrypted packets between peers | P2 |
| NAT-03 | Relay nodes must not be able to decrypt relayed traffic (E2EE preserved end-to-end) | P2 |
| NAT-04 | Node daemon must detect its NAT type on startup and log it | P2 |
| NAT-05 | No static IP, DHCP reservation, or router port forwarding required for non-relay nodes | P2 |

### 4.4 Invite System

| ID | Requirement | Priority |
|---|---|---|
| IN-01 | Generate single-use, time-limited invite tokens signed by the issuing node's keypair | P2 |
| IN-02 | Default invite expiry: 24 hours; configurable | P2 |
| IN-03 | Joining node signs its keypair with the invite token; issuing node verifies before adding to peer list | P2 |
| IN-04 | After successful join, invalidate token and gossip the new trusted peer to all network members | P2 |
| IN-05 | Invite tokens must be safe to transmit over insecure channels (Signal, email); no secrets in the token itself | P2 |

---

## 5. Client Application Requirements

### 5.1 CLI

| ID | Requirement | Priority |
|---|---|---|
| CLI-01 | Cross-compile to Linux (amd64, arm64), Windows (amd64), and Android (arm64 via Termux) | P1 |
| CLI-02 | Commands: `node start`, `node stop`, `node status`, `invite generate`, `join`, `backup`, `restore`, `network status`, `credits balance` | P1 |
| CLI-03 | `network status` must display: all known nodes, online/offline status, latency, shard health summary | P1 |
| CLI-04 | `backup` must support directory watching mode (continuous sync on file change) | P2 |
| CLI-05 | `backup` must support one-shot mode (single backup run) | P1 |
| CLI-06 | All CLI output must support `--json` flag for machine-readable output | P2 |

### 5.2 Flutter App

| ID | Requirement | Priority |
|---|---|---|
| APP-01 | Support iOS, Android, macOS, Windows, Linux from single codebase | P2 |
| APP-02 | QR code onboarding flow: scan bootstrap QR → auto-configure → join network | P2 |
| APP-03 | File manager: browse vault, drag-and-drop upload, download, delete, share with expiry controls | P2 |
| APP-04 | Node dashboard: per-node status, uptime, storage, credits, reputation score | P2 |
| APP-05 | Geo-diversity visualization: map or diagram showing shard distribution across nodes | P2 |
| APP-06 | Push or local notifications for: node offline, low reputation warning, new peer joined, backup complete | P2 |
| APP-07 | Gamified onboarding: step-by-step setup with badges and progress indicators | P2 |
| APP-08 | No required accounts, no cloud sign-in, no telemetry without explicit opt-in | P1 |

### 5.3 Platform Backup Integration

| ID | Requirement | Priority |
|---|---|---|
| BI-01 | Linux: inotify-based directory watcher that triggers incremental backup on file change | P1 |
| BI-02 | Linux: systemd service unit file for daemon auto-start on boot | P1 |
| BI-03 | Windows: PowerShell FileSystemWatcher integration for directory monitoring | P2 |
| BI-04 | Windows: NSSM service configuration for daemon auto-start | P2 |
| BI-05 | Android: background service (Termux for dev, Flutter background service for production) | P2 |
| BI-06 | Android: wake lock management — maintain uptime while on WiFi and charging | P2 |
| BI-07 | All platforms: configurable exclude patterns (e.g., `*.tmp`, `.git/`) | P2 |

---

## 6. Non-Functional Requirements

### 6.1 Performance

| ID | Requirement |
|---|---|
| NF-01 | Node daemon idle CPU usage must not exceed 2% on anchor-tier hardware (OptiPlex 990 class) |
| NF-02 | Node daemon idle RAM usage must not exceed 150MB |
| NF-03 | Gossip protocol must not consume more than 10% of available upload bandwidth by default |
| NF-04 | Shard retrieval of a 1GB file must complete in under 60 seconds on 50Mbps home internet |
| NF-05 | Node startup to peer discovery must complete in under 30 seconds on LAN; under 60 seconds over internet |

### 6.2 Reliability

| ID | Requirement |
|---|---|
| NF-06 | System must tolerate up to floor((n-1)/3) simultaneous node failures without data loss at 5-of-9 coding |
| NF-07 | Re-replication must begin within 10 minutes of detecting a node has gone offline |
| NF-08 | No single node — including bootstrap or relay nodes — must be a single point of failure |
| NF-09 | Daemon must recover cleanly from crash without manual intervention; use WAL for storage operations |

### 6.3 Security

| ID | Requirement |
|---|---|
| NF-10 | All cryptographic primitives must use audited libraries; no custom crypto implementations |
| NF-11 | No plaintext user data must ever be written to disk or transmitted over the network |
| NF-12 | Node daemon must have no content inspection capability — no MIME detection, no file scanning |
| NF-13 | Security-relevant configuration must not be changeable at runtime without passphrase re-authentication |

### 6.4 Portability

| ID | Requirement |
|---|---|
| NF-14 | Node daemon must run on Linux (Ubuntu 20.04+, Debian 11+, Raspberry Pi OS), Windows 10+, Android 10+ (Termux) |
| NF-15 | Node daemon must run on hardware as constrained as Raspberry Pi 3B (1GB RAM, 4-core ARM Cortex-A53) |
| NF-16 | No dependency on systemd, Docker, or any container runtime for core operation |

---

## 7. Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Node daemon language | Go 1.22+ | Cross-compilation, low resource use, strong concurrency primitives |
| Networking | libp2p (go-libp2p) | Battle-tested; QUIC, mDNS, DHT, hole punching built-in |
| Post-quantum crypto | liboqs (Go bindings) | NIST-validated ML-KEM / ML-DSA implementations |
| Symmetric crypto | AES-256-GCM (Go stdlib) | Standard library, hardware-accelerated |
| Erasure coding | klauspost/reedsolomon | High performance, well-maintained Go library |
| ZK proofs | arkworks-rs or bellman | zk-SNARK generation and verification |
| Local storage | RocksDB (gorocksdb) or sled | Embedded K/V, WAL, compaction |
| Serialization | protobuf (google.golang.org/protobuf) | Compact, versioned, cross-language |
| Client app | Flutter 3.x | Single codebase for all platforms |
| CLI framework | cobra (Go) | Standard Go CLI library |
| DDNS | DuckDNS (free tier) | Zero cost, simple API, reliable |
| Geo data | MaxMind GeoLite2 (bundled) | No external API calls at runtime |

---

## 8. Constraints

- No central servers in the critical path for any core operation (storage, retrieval, consensus)
- No cryptocurrency, blockchain, or token mechanisms
- No user accounts or identity tied to anything other than keypairs
- No telemetry, analytics, or usage reporting without explicit opt-in
- All dependencies must be open source with compatible licenses (MIT, Apache 2.0, BSD)
- The system must remain functional when the original developers are unavailable — no proprietary backend
