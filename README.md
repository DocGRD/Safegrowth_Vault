# SafeGrowth Vault

**Trust the Math, Not the Hardware.**

![Status: Pre-Implementation](https://img.shields.io/badge/status-pre--implementation-orange)
![Language: Go 1.24+](https://img.shields.io/badge/language-Go%201.24%2B-blue)
![CGO: Zero](https://img.shields.io/badge/CGO-zero-green)
![License: MIT](https://img.shields.io/badge/license-MIT-green)
![Crypto: Post-Quantum](https://img.shields.io/badge/crypto-post--quantum-purple)
![Tokens: None](https://img.shields.io/badge/tokens-none-brightgreen)

---

## ⚠️ Project Status: Fully Designed. Zero Code Yet.

SafeGrowth Vault is a **complete architecture looking for its engineers.**

This repository contains a full technical specification — whitepaper, protocol spec, 11 architecture decision records, threat model, software requirements document, and 20-week development plan — for a decentralized storage network. What it does not yet contain is working code.

**The single most valuable contribution right now is a domain expert willing to read the architecture and say whether it is sound.** See [How to Help](#how-to-help).

---

## What Is SafeGrowth Vault?

SafeGrowth Vault is an open-source decentralized storage network designed for genuine data sovereignty:

- **Your files are encrypted** with post-quantum cryptography (ML-KEM + ML-DSA, NIST FIPS 203/204) before they leave your device
- **Erasure-coded shards** (Reed-Solomon) are distributed across hardware you trust — your own devices, family NAS units, trusted friends' servers
- **Proof of Retrievability** challenges continuously verify that nodes actually hold what they claim to hold — cryptographic challenges that cannot be faked without possessing the data
- **No node can read your data** — including the one run by this project's creator. Content-blind by construction.
- **No token. No subscription. No central server.** Ever.

Any hardware can participate: a Raspberry Pi 3B, a NAS device (Synology, QNAP, TrueNAS), an old laptop, an Android phone. A single `go build` with `CGO_ENABLED=0` cross-compiles to Linux, Windows, Android, and FreeBSD.

---

## Origin

In August 2025, Glenn Dewar (a non-developer) wanted to move his family's data out of the cloud without sacrificing resilience. The original idea: a NAS at his home and one at each adult child's home, with distributed backup. That question — how do you make distributed family backup *trustless*? — led to a long design session with Grok, drawing inspiration from Hashgraph's Byzantine consensus approach (adapted without patent infringement), and produced a whitepaper.

The project sat until March 2026, when Glenn resumed with Claude Sonnet 4.6 and produced the full documentation suite. What would have taken a solo distributed systems expert 6-12 months of part-time documentation work was compressed into intensive AI-assisted design sessions.

**Current team: Glenn (design, documentation, vision) + Grok + Claude Sonnet 4.6. No human engineers yet.**

---

## Architecture at a Glance

| Layer | Technology | Why |
|-------|-----------|-----|
| Node daemon | Go 1.24+ (`CGO_ENABLED=0`) | Cross-compiles to all 5 targets with no C toolchain |
| Key encapsulation | `crypto/mlkem` (Go stdlib) | NIST FIPS 203; pure Go; zero CGO |
| Digital signatures | `trailofbits/ml-dsa` | NIST FIPS 204; pure Go; Trail of Bits audited |
| Storage | `cockroachdb/pebble` | Pure Go; WAL-backed; ZFS/Btrfs compatible |
| Erasure coding | `klauspost/reedsolomon` | Pure Go; SIMD-accelerated |
| Networking | `go-libp2p` (QUIC, mDNS, Kademlia) | Battle-tested at IPFS scale |
| ZK proofs | `consensys/gnark` | Pure Go; Groth16; production in Linea L2 |
| Consensus | Custom PBFT-lite | No blockchain; tolerates f=⌊(n-1)/3⌋ Byzantine nodes |

### Hardware Tiers (derived from observed behaviour, not self-reported)

| Tier | Example Hardware | Role | Credits/hr |
|------|-----------------|------|-----------|
| **Super Anchor** | NAS (Synology, QNAP, TrueNAS) | Preferred first replica | +0.75 |
| Anchor | Desktop, OptiPlex, always-on | Primary shard holders | +0.50 |
| Mid | Laptops, mini PCs | Backup shard holders | +0.25 |
| Edge | Phones, tablets | Small shards, metadata | +0.10 |
| Transient | Browser sessions | Relay and cache only | 0 |

### Deployment Phases

| Phase | Weeks | Scope | Trust Model |
|-------|-------|-------|-------------|
| 1 — Local Testnet | 1-4 | Personal backup on LAN | Fully trusted (personal devices) |
| 2 — Trusted Group | 5-10 | Remote participants, NAT traversal, NAS support | Web of trust via invite chain |
| 3 — Open Network | 11-20 | Any hardware, full Sybil resistance, ZK-PoR | Zero trust; math-enforced |

---

## Documentation

| Document | Description |
|----------|-------------|
| [`WHITEPAPER.docx`](docs/WHITEPAPER.docx) | Full architecture, security model, and deployment philosophy (v2.2) |
| [`SAFEGROWTH_REQUIREMENTS.docx`](docs/SAFEGROWTH_REQUIREMENTS.docx) | 55+ functional and non-functional requirements (SRD v1.2) |
| [`ADR_CONSOLIDATED.docx`](docs/ADR_CONSOLIDATED.docx) | 11 Architecture Decision Records — every significant design choice |
| [`PROTOCOL_SPEC.docx`](docs/PROTOCOL_SPEC.docx) | Complete wire-level specification: message formats, state machines (v1.1) |
| [`THREAT_MODEL.docx`](docs/THREAT_MODEL.docx) | 30+ attack vectors, mitigations, residual risk register (v1.1) |
| [`DEVPLAN.docx`](docs/DEVPLAN.docx) | 20-week phased development plan with week-by-week task breakdown (v1.3) |
| [`OPERATIONS_RUNBOOK.docx`](docs/OPERATIONS_RUNBOOK.docx) | Node operator procedures for all platforms including NAS hardware (v1.1) |
| [`FOR_DISCUSSION.docx`](docs/FOR_DISCUSSION.docx) | Open architectural questions requiring decisions before implementation |

---

## How to Help

### 🔍 Domain Expert — Architecture Sanity Check *(Most Needed Right Now)*

The most valuable contribution at this stage is an honest technical review by someone who knows distributed systems, cryptography, or Go at depth.

**What I'm asking:** Read the whitepaper and ADRs (~35 pages). Tell me honestly whether the architecture is sound, or where it isn't. No ongoing commitment required or expected.

See [`docs/MKTG-5-Sanity-Check-Request.docx`](docs/MKTG-5-Sanity-Check-Request.docx) for the full request, or contact Glenn directly.

### 🛠️ Go Engineer — First Implementation

The architecture is specified to wire-protocol level. Good starting points:

- **Week 1:** `identity/` package — ML-DSA + ML-KEM keypair generation with Argon2id-encrypted persistence
- **Week 1:** `storage/` package — Pebble wrapper with `PutShard`, `GetShard`, `DeleteShard`, `HasShard`
- **Must resolve first:** [`FOR_DISCUSSION.docx`](docs/FOR_DISCUSSION.docx) item A-01 (ML-DSA canonical encoding — raw bytes vs DER)
- **Read first:** ADR-001 (Zero CGO constraint) explains every dependency choice

```bash
git clone https://github.com/[username]/safegrowth
cd safegrowth
CGO_ENABLED=0 go build -o ./safegrowth-node ./node/cmd
# Week 1 goal: this command builds and generates keypairs on first run
```

### 💛 Everyone Else

- ⭐ Star this repository
- Share with Go engineers, cryptographers, and privacy advocates
- Contact Glenn if you know a foundation that funds open-source infrastructure

---

## Contact

**Glenn Dewar** | grd@dewarfamily.net | 802 370 2071

---

## License

MIT License — see [LICENSE](LICENSE).

---

## Acknowledgements

Architecture developed through AI-assisted design sessions with Grok (August 2025) and Claude Sonnet 4.6 (Anthropic, March 2026). Grounded in peer-reviewed research:

- Castro & Liskov (1999) — Practical Byzantine Fault Tolerance
- Ateniese et al. (2007) and Juels & Kaliski (2007) — Proof of Retrievability
- Demers et al. (1987) — Epidemic algorithms for replicated database maintenance
- NIST FIPS 203/204 (2024) — ML-KEM and ML-DSA post-quantum standards
