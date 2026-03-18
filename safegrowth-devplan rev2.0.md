# SafeGrowth Vault — Development Plan

**Version**: 1.0  
**Target Team**: 3 developers (Glenn + 2 remote collaborators)  
**Development Environment**: Lenovo Flex (primary), OptiPlex 990, Dell Latitude, home-built PC, Android phones

---

## Overview

Development is organized into three milestones aligned directly with the three deployment phases:

- **Milestone 1 — Local Testnet** (Weeks 1–4): working personal backup across devices on a LAN
- **Milestone 2 — Trusted Group Network** (Weeks 5–10): expand to remote friends, NAT traversal, invite system
- **Milestone 3 — Open Participation** (Weeks 11–20): Sybil resistance, PoR, reputation, full trustless operation

Each milestone produces a working, usable system — not just a prototype. Complexity is added incrementally; nothing is built that isn't immediately testable.

---

## Repository Structure

```
safegrowth/
├── node/                    # Node daemon (Go)
│   ├── cmd/main.go          # Entry point
│   ├── identity/            # Keypair generation, persistence
│   ├── storage/             # Shard storage engine (RocksDB/sled)
│   ├── crypto/              # Encryption, signing, key exchange
│   ├── sharding/            # Reed-Solomon erasure coding
│   ├── gossip/              # Epidemic gossip protocol
│   ├── consensus/           # PBFT-lite quorum voting
│   ├── por/                 # Proof of Retrievability engine
│   ├── reputation/          # Reputation tracking and scoring
│   ├── credits/             # Credit ledger
│   ├── replication/         # Shard placement and replication manager
│   ├── api/                 # REST/gRPC API for client communication
│   └── config/              # Configuration management
├── cli/                     # Command-line client (Go)
│   └── cmd/
├── app/                     # Flutter client (iOS/Android/Desktop)
│   └── lib/
├── crypto/                  # Shared crypto utilities
├── proto/                   # Protobuf definitions
├── scripts/
│   ├── backup-linux.sh
│   ├── backup-windows.ps1
│   └── install/
├── docs/
│   ├── whitepaper.md
│   ├── requirements.md
│   └── devplan.md
└── .github/
    └── workflows/
        └── build.yml        # CI: build Linux + Windows binaries on merge
```

---

## Milestone 1: Local Testnet

**Duration**: Weeks 1–4  
**Goal**: Personal backup working across OptiPlex, Dell Latitude, Lenovo Flex, and Android phones on the home LAN. No internet, no trust model needed yet — all devices are personally owned.

**Requirements covered**: ID-01 through ID-05, ST-01 through ST-07, CR-01, CR-04, CR-07, SH-01, SH-02, SH-09, PD-01, PD-05, CLI-01, CLI-02 (subset), CLI-05, BI-01, BI-02, NF-14, NF-15

---

### Week 1: Identity and Storage Foundation

**Tasks:**

- [ ] Initialize Git repository; set up Go module (`go mod init github.com/you/safegrowth`)
- [ ] Implement `identity` package
  - Generate ML-DSA + ML-KEM keypairs using liboqs Go bindings on first run
  - Persist to `~/.safegrowth/identity` encrypted with AES-256-GCM + Argon2id passphrase
  - Load and unlock on startup
  - Expose `NodeID()` → stable string derived from public key
- [ ] Implement `storage` package
  - Wrap RocksDB (gorocksdb) with: `PutShard(id, data)`, `GetShard(id)`, `DeleteShard(id)`, `HasShard(id)`
  - Maintain shard manifest in a separate RocksDB column family
  - Enforce configurable storage quota
- [ ] Implement `api` package — minimal REST server on localhost:8080
  - `GET /status` → node ID, uptime, storage used/available
  - `POST /shards` → store a shard
  - `GET /shards/{id}` → retrieve a shard
- [ ] Write unit tests for identity (keypair round-trip) and storage (write/read/delete)

**Deliverable**: `safegrowth-node` binary that starts, generates identity, stores shards via REST API.

**Test**:
```bash
curl -X POST http://localhost:8080/shards -d @testfile.bin
curl http://localhost:8080/shards/{returned-id} -o retrieved.bin
diff testfile.bin retrieved.bin  # must pass
```

---

### Week 2: Encryption and Sharding

**Tasks:**

- [ ] Implement `crypto` package
  - `EncryptFile(data []byte, recipientPubKey) → ciphertext` using AES-256-GCM with ML-KEM key encapsulation
  - `DecryptFile(ciphertext []byte, privKey) → data`
  - `SignMessage(msg, privKey) → signature`
  - `VerifySignature(msg, sig, pubKey) → bool`
  - Hash utilities: `SHA3256(data)`, `BLAKE3(data)`
- [ ] Implement `sharding` package using klauspost/reedsolomon
  - `SplitFile(data, dataShards, parityShards) → []Shard`
  - `ReconstructFile(shards []Shard, dataShards, parityShards) → data`
  - Default: 2-of-3 (1 data, 2 parity — any 2 reconstruct)
  - Each shard carries: shard index, total shards, file ID (hash of original), shard hash
- [ ] Integrate encryption into storage: `PutShard` encrypts before writing; `GetShard` decrypts after reading
- [ ] Write unit tests: encrypt→shard→reconstruct→decrypt round-trip on test files of various sizes (1KB, 1MB, 100MB)

**Deliverable**: Files can be encrypted, split into shards, stored, retrieved, and reconstructed correctly.

---

### Week 3: mDNS Discovery and Peer Replication

**Tasks:**

- [ ] Add go-libp2p dependency; implement `gossip/mdns.go`
  - Announce node ID + IP:port via mDNS every 30 seconds
  - Discover peer announcements; add to peer table
  - Persist peer table to `~/.safegrowth/peers.json` on update
  - Reload peer table on startup
- [ ] Implement basic replication in `replication` package
  - On shard store: push shard to all known peers (Phase 1 = all LAN peers)
  - `GET /peers` API endpoint returning current peer table
  - `POST /shards/receive` endpoint for accepting inbound shards from peers
- [ ] Wire up: upload file → encrypt → shard → store locally + push to peers
- [ ] Test on real LAN: run daemon on OptiPlex and Lenovo Flex simultaneously; verify peer discovery and replication

**Deliverable**: Store a file on the Flex; verify shards appear on the OptiPlex automatically.

**Test**:
```bash
# On Lenovo Flex
safegrowth-cli backup ~/Documents/test.pdf

# SSH to OptiPlex
curl http://localhost:8080/status  # should show shards received from Flex
```

---

### Week 4: CLI, Linux Backup Integration, Cross-Compile

**Tasks:**

- [ ] Implement full CLI using cobra
  - `safegrowth-cli node start` / `stop` / `status`
  - `safegrowth-cli network status` → table of all discovered peers with status
  - `safegrowth-cli backup <path>` → one-shot backup
  - `safegrowth-cli restore <vault-path> --to <local-path>`
- [ ] Linux backup integration
  - inotify directory watcher: auto-backup on file change
  - systemd service unit file: `safegrowth-node.service`
  - `scripts/install-linux.sh`: installs service, sets up config
- [ ] Cross-compile for Windows and Android (Termux)
  ```bash
  GOOS=windows GOARCH=amd64 go build -o dist/safegrowth-node.exe ./node/cmd
  GOOS=linux GOARCH=arm64 go build -o dist/safegrowth-node-arm64 ./node/cmd
  ```
- [ ] Install on Android via Termux; verify mDNS discovery works when phone is on same WiFi
- [ ] Set up GitHub Actions CI: build all targets on every push to main

**Deliverable**: Milestone 1 complete. Personal backup working on all your devices. Linux auto-backup on file change. Windows and Android daemons joinable via CLI.

---

## Milestone 2: Trusted Group Network

**Duration**: Weeks 5–10  
**Goal**: Two remote friends can join the network. NAT traversal works. All connections are authenticated. Credits begin accruing.

**New participants**: Friend 1 (Linux), Friend 2 (Windows)

**Requirements covered**: ID-08, CR-05, CR-08, SH-04, SH-08, GP-01 through GP-06, BC-01 through BC-05, PD-02, PD-03, NAT-01 through NAT-05, IN-01 through IN-05, CL-01 through CL-06, CLI-04, BI-03, BI-04, BI-05, BI-06, NF-08

---

### Week 5: Mutual TLS and Signed Gossip

**Tasks:**

- [ ] Upgrade all peer connections to mutual TLS
  - Derive TLS certificate from node ML-DSA keypair (self-signed, identity-linked)
  - Both sides present certificate on connection; reject connections without valid cert
- [ ] Add message signing to gossip layer
  - All gossip messages serialized as protobuf, signed with sender's ML-DSA key
  - Receiver verifies signature before processing; log and drop invalid messages
- [ ] Define protobuf schemas for all gossip message types: `PeerAnnouncement`, `ShardPlacement`, `CreditUpdate`, `PeerList`
- [ ] Unit test: tampered gossip message is rejected; valid message is processed

---

### Week 6: Invite System

**Tasks:**

- [ ] Implement invite token generation and validation
  - Token = base58(nodeID + expiry_timestamp + random_nonce + signature)
  - Single-use: token marked consumed after first successful use
  - Default expiry: 24 hours
- [ ] Implement join flow
  - Joining node generates keypair, signs join request with invite token
  - Issuing node verifies, adds to trusted peer list, gossips new peer to network
  - Joining node receives full trusted peer list from issuing node
- [ ] CLI commands:
  ```bash
  safegrowth-cli invite generate --expires 24h
  # Output: sgv-invite-a3f9c2b1d4e7...

  safegrowth-cli join \
    --invite sgv-invite-a3f9c2b1d4e7 \
    --bootstrap yourname.duckdns.org:4001
  ```
- [ ] Test: generate invite on OptiPlex, join from a fresh VM, verify peer appears in network

---

### Week 7: NAT Traversal and DDNS

**Tasks:**

- [ ] Set up DuckDNS for OptiPlex
  - Register `yourname.duckdns.org`
  - Cron job on OptiPlex updates IP every 5 minutes:
    ```bash
    */5 * * * * curl -s "https://www.duckdns.org/update?domains=yourname&token=TOKEN&ip=" > /dev/null
    ```
  - Open port 4001 UDP on home router → OptiPlex LAN IP (this is the only router config needed)
- [ ] Implement UDP hole punching via libp2p's built-in hole punching support
  - Use OptiPlex as rendezvous point for initial coordination
  - Fall back to relay if hole punching fails within 10 seconds
- [ ] Implement relay node functionality in daemon
  - Relay forwards encrypted QUIC packets between peers
  - Relay has no access to packet content (E2EE preserved)
  - Configurable: any anchor node can act as relay
- [ ] Test: Friend 1 joins from a different home network; verify direct P2P connection established (check relay is not needed for most cases)

---

### Week 8: Bootstrap Peer Discovery and Phase 2 Replication

**Tasks:**

- [ ] Implement bootstrap peer config in `~/.safegrowth/config.json`
  ```json
  {
    "bootstrap_peers": ["yourname.duckdns.org:4001"],
    "erasure_scheme": {"data": 3, "parity": 2}
  }
  ```
- [ ] On startup: connect to bootstrap peers, request peer list, attempt connections to all trusted peers
- [ ] Upgrade erasure coding to 3-of-5 for internet-connected nodes
- [ ] Upgrade replication manager to use reputation-aware placement (Phase 2: basic uptime score only)
  - Prefer nodes that have been online for at least 1 hour
  - Track per-peer uptime in peer table
- [ ] Test full flow: backup file on Flex → shards distributed across Flex, OptiPlex, Latitude, Friend 1, Friend 2

---

### Week 9: Credit System

**Tasks:**

- [ ] Implement credit ledger
  - Append-only log stored in RocksDB
  - Entries: `{timestamp, nodeID, type, amount, signature}`
  - Types: `uptime_reward`, `storage_reward`, `retrieval_spend`
- [ ] Credit accrual daemon: runs hourly, calculates earned credits since last run, submits to ledger
- [ ] Implement lightweight consensus for credit validation (3-node quorum, simple majority for Phase 2)
- [ ] CLI: `safegrowth-cli credits balance`, `safegrowth-cli credits history`
- [ ] Test: run network for 24 hours; verify credits accrue on nodes with uptime; verify no credits on offline nodes

---

### Week 10: Windows Backup, Android Production, Integration Testing

**Tasks:**

- [ ] Windows backup integration
  - PowerShell FileSystemWatcher script for directory monitoring
  - NSSM service setup script for auto-start
  - `scripts/install-windows.ps1`
- [ ] Android background service (Flutter)
  - Begin Flutter app scaffold
  - Background service that keeps daemon running on WiFi + charging
  - termux-wake-lock integration for Termux path
- [ ] End-to-end integration test: all 7 nodes online (your 4 + 2 friends + phone)
  - Store 1GB test file from Flex
  - Verify shards distributed across all nodes
  - Take OptiPlex offline; verify retrieval still works (3-of-5 intact)
  - Bring OptiPlex back; verify re-replication restores full redundancy

**Deliverable**: Milestone 2 complete. Three-person distributed storage network across real home internet connections. All traffic encrypted and authenticated. Basic credits accruing.

---

## Milestone 3: Open Participation

**Duration**: Weeks 11–20  
**Goal**: Unknown participants can join. The network defends itself against malicious nodes cryptographically. Full PoR, reputation, and Sybil resistance operational.

**Requirements covered**: All remaining P3 requirements

---

### Weeks 11–12: Sybil Resistance

**Tasks:**

- [ ] Implement resource proof challenge on join
  - Joining node must store a random 256MB challenge dataset and prove it within 60 seconds
  - Challenge is generated by the inviting node; response verified before invite is finalized
- [ ] Implement reputation probation period (72 hours for new nodes)
  - No credit accrual during probation
  - Not eligible for primary shard placement during probation
  - Probation status gossiped to all peers
- [ ] Add computational puzzle to join flow (one-time moderate PoW to make mass node creation expensive)
- [ ] Test: attempt to flood network with 20 fake nodes from a single machine; verify resource challenges block it

---

### Weeks 13–14: Proof of Retrievability

**Tasks:**

- [ ] Implement PoR challenge generation
  - On shard placement, pre-compute 100 random challenges: `{start, end, nonce, expected_hash}`
  - Store challenge set encrypted alongside shard metadata
- [ ] Implement PoR challenge issuance
  - Daemon periodically selects random shards it's responsible for verifying
  - Sends challenge to the node holding the shard; expects response within 5 seconds
- [ ] Implement PoR challenge response
  - Daemon responds to incoming challenges instantly if shard is present
  - Failed response: log, decrement reputation score, gossip failure
- [ ] Reputation impact: 3 consecutive failures → trigger shard migration
- [ ] Test: manually delete a shard from one node without notifying the network; verify PoR challenges detect it and trigger re-replication

---

### Weeks 15–16: ZK-PoR and Full Byzantine Consensus

**Tasks:**

- [ ] Integrate arkworks-rs or bellman for zk-SNARK operations (via CGO or WASM bridge)
- [ ] Implement ZK-PoR: node proves shard possession via zk-SNARK without transmitting shard
  - Circuit: prove knowledge of preimage to committed challenge hash
  - Proof size target: under 2KB per challenge response
- [ ] Upgrade consensus to full PBFT-lite with 5-7 node quorums
  - Leader election per decision round
  - 3-phase commit: pre-prepare, prepare, commit
  - View change on leader failure
- [ ] Require ZK-PoR proof for all credit claims in Phase 3
- [ ] Performance test: measure proof generation time on OptiPlex 990 (target: under 2 seconds per proof)

---

### Weeks 17–18: Full Reputation System and Adaptive Replication

**Tasks:**

- [ ] Implement composite reputation score
  - Weighted sum: uptime_ratio × 0.4 + por_pass_rate × 0.4 + latency_score × 0.2
  - Score range: 0.0 (fully untrusted) to 1.0 (fully trusted)
- [ ] Gossip-propagated signed reputation records
  - Each node publishes its observed reputation for peers it has interacted with
  - Aggregate reputation = weighted average of all peer observations
- [ ] Quarantine mechanism: nodes below 0.6 score receive no new shard placements; existing shards migrate off
- [ ] Adaptive replication manager
  - Select placement nodes by: reputation ≥ 0.85, tier (prefer anchor), geo-diversity (3+ regions), ASN diversity (3+ ASNs)
  - Re-evaluate placement continuously; migrate shards proactively if holding node's reputation trends downward
- [ ] Contribution accounting: bytes-in vs bytes-out fairness; throttle persistent net takers

---

### Weeks 19–20: Flutter App, DHT, and Phase 3 Testing

**Tasks:**

- [ ] Complete Flutter app
  - QR onboarding flow
  - File manager (upload, download, share with expiry)
  - Node dashboard with reputation scores and credit balance
  - Geo-diversity map visualization
  - Push notifications for node offline / backup complete
  - Gamified onboarding badges
- [ ] Implement DHT-based peer discovery (libp2p Kademlia DHT)
  - Replace bootstrap-only discovery with fully distributed peer lookup
  - Bootstrap nodes still serve as DHT entry points but are no longer required for ongoing operation
- [ ] Upgrade erasure coding to 5-of-9 for Phase 3 nodes
- [ ] Full Phase 3 integration test
  - Introduce a "malicious" test node that fails PoR challenges intentionally
  - Verify reputation drops, quarantine triggers, shards migrate off
  - Introduce a Sybil test (10 nodes from same machine); verify resource proofs block them
  - Simulate 4 simultaneous node failures; verify data recoverable from remaining 5-of-9

**Deliverable**: Milestone 3 complete. Fully trustless open participation network operational.

---

## Development Workflow

### Branching Strategy
```
main          ← stable, always builds, always passes tests
├── m1/       ← Milestone 1 integration branch
├── m2/       ← Milestone 2 integration branch
├── m3/       ← Milestone 3 integration branch
└── feature/  ← individual feature branches (merged via PR)
```

### Pull Request Process
- All features developed on `feature/` branches
- PRs require review from at least 1 other developer before merge
- CI must pass (build + unit tests) before merge
- No direct commits to `main`

### CI Pipeline (GitHub Actions)
```yaml
# .github/workflows/build.yml
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - go test ./...
      - GOOS=linux GOARCH=amd64 go build ./node/cmd
      - GOOS=windows GOARCH=amd64 go build ./node/cmd
      - GOOS=linux GOARCH=arm64 go build ./node/cmd
      - Upload artifacts (Linux, Windows, ARM64 binaries)
```

### Testing Standards
- Unit tests required for all packages before PR merge
- Integration tests run manually at end of each week against real hardware
- Each milestone ends with a documented end-to-end test run on all physical devices

### Communication (3-Person Team)
- Weekly async sync via shared dev log (markdown file in repo)
- Issues tracked on GitHub Issues
- Breaking changes documented in `CHANGELOG.md` with migration notes

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| liboqs Go bindings unstable or incompatible | Medium | High | Evaluate early in Week 2; fallback to age encryption for M1 |
| ZK-SNARK proof generation too slow on OptiPlex | Medium | Medium | Benchmark in Week 15; consider proof batching or lighter proof system (e.g., Bulletproofs) |
| QUIC hole punching fails for one friend's NAT | Low | Medium | Relay fallback is already designed; test early in Week 7 |
| Android background service killed by battery optimization | High | Medium | Document workaround; use termux-wake-lock; Flutter background service has known workarounds |
| Reed-Solomon reconstruction fails with corrupted shards | Low | High | Test extensively in Week 2; include checksum validation before reconstruction |
| Gossip storm on large network | Low | Medium | Version vectors and bandwidth caps prevent this; test with simulated 50-node network in M3 |

---

## Milestone Summary

| Milestone | Weeks | Deliverable | Trust Model |
|---|---|---|---|
| 1 — Local Testnet | 1–4 | Personal backup on LAN, all platforms | Fully trusted (personal devices) |
| 2 — Trusted Group | 5–10 | Remote friends, NAT traversal, credits | Web of trust via invite chain |
| 3 — Open Network | 11–20 | Any hardware, PoR, reputation, Sybil resistance | Zero trust; math-enforced |
