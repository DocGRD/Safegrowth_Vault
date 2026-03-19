# SafeGrowth Vault — Operations Runbook

**Version**: 1.0  
**Status**: Draft  
**Date**: March 2026  
**Authors**: SafeGrowth Team  
**Scope**: Node Daemon Operations, Recovery Procedures, Maintenance Tasks

> **Living Document**: This runbook must be updated whenever a new operational procedure is established or an existing one changes. All three team members must review changes before merge. Procedures marked **[M2]** or **[M3]** are not yet active but are documented here in advance of the milestone that introduces the relevant feature.

---

## Revision History

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0 | 2026-03 | SafeGrowth Team | Initial runbook — covers Milestone 1 procedures; M2/M3 stubs included |

---

## Table of Contents

1. [Node Bootstrap and Startup Sequence](#1-node-bootstrap-and-startup-sequence)
2. [Health Checks and Status Verification](#2-health-checks-and-status-verification)
3. [DDNS Update Failure — Detection and Recovery](#3-ddns-update-failure--detection-and-recovery)
4. [Shard Re-replication — Manual Trigger and Verification](#4-shard-re-replication--manual-trigger-and-verification)
5. [Credit Ledger — Verification and Recovery](#5-credit-ledger--verification-and-recovery)
6. [Key Rotation Procedure](#6-key-rotation-procedure)
7. [Node Offboarding and Peer Table Cleanup](#7-node-offboarding-and-peer-table-cleanup)
8. [Backup Verification and Test Restore](#8-backup-verification-and-test-restore)
9. [Adding a New Node (Invite Flow)](#9-adding-a-new-node-invite-flow)
10. [Emergency Procedures](#10-emergency-procedures)
11. [Scheduled Maintenance Tasks](#11-scheduled-maintenance-tasks)
12. [Log Reference](#12-log-reference)
13. [Node Inventory](#13-node-inventory)

---

## 1. Node Bootstrap and Startup Sequence

### 1.1 Prerequisites Checklist

Before starting a node for the first time, verify:

- [ ] Go 1.24+ installed: `go version` — must show `go1.24` or higher
- [ ] Identity directory exists or will be created: `~/.safegrowth/`
- [ ] Storage quota configured in `~/.safegrowth/config.json`
- [ ] Passphrase decided and stored securely (password manager, not a text file)
- [ ] **[M2]** Bootstrap peers configured: minimum 2 entries in `bootstrap_peers`
- [ ] **[M2]** Port 4001 UDP open on router (anchor nodes only)
- [ ] **[M2]** DDNS hostname registered and cron job active (anchor nodes only)

### 1.2 First-Run Identity Generation

On first run the daemon generates both keypairs and encrypts them to disk:

```bash
# Start the daemon — it will prompt for a new passphrase on first run
safegrowth-node start

# Expected output on first run:
# [INFO] No identity found at ~/.safegrowth/identity — generating new keypair
# [INFO] Generated ML-DSA-65 keypair
# [INFO] Generated ML-KEM-768 keypair
# [INFO] Enter passphrase to encrypt identity (min 12 characters):
# [INFO] Confirm passphrase:
# [INFO] Identity encrypted and saved to ~/.safegrowth/identity
# [INFO] NodeID: <64-char hex string>
# [INFO] Daemon listening on :8080 (REST) and :4001 (P2P)
```

**Record the NodeID immediately.** Copy it to your node inventory (Section 13). It is the stable identity of this node for its entire lifetime.

### 1.3 Normal Startup Sequence

```bash
# Manual start
safegrowth-node start

# Startup health check criteria (all must pass within 60 seconds):
safegrowth-cli node status
# Expected:
# Status:        ONLINE
# NodeID:        <hex>
# Uptime:        0h 0m Xs
# Storage Used:  X.X GB / Y GB (Z%)
# Peers (LAN):   N   ← Phase 1: should be > 0 within 30s on LAN
# Peers (WAN):   N   ← Phase 2+: should be > 0 within 60s
# Bootstrap:     OK  ← Phase 2+: at least 2 reachable
```

**Startup fails if**: passphrase is wrong, identity file is corrupt, storage path is inaccessible, or port 4001 is already in use.

### 1.4 Startup via systemd (Linux)

```bash
# Enable and start
sudo systemctl enable safegrowth-node
sudo systemctl start safegrowth-node

# Check status
sudo systemctl status safegrowth-node

# View logs (last 100 lines)
journalctl -u safegrowth-node -n 100 --no-pager

# Follow logs live
journalctl -u safegrowth-node -f
```

**Note on passphrase with systemd**: The systemd service prompts for passphrase via a `StandardInput=tty` directive. On headless servers, use `systemd-ask-password` integration or store the passphrase in a systemd credential (see `systemd-creds` — do not store passphrase in the unit file).

### 1.5 Startup via NSSM (Windows) **[M2]**

```powershell
# Check service status
nssm status SafeGrowthNode

# Start service
nssm start SafeGrowthNode

# View logs
Get-Content "C:\ProgramData\SafeGrowth\logs\node.log" -Tail 100

# Follow logs
Get-Content "C:\ProgramData\SafeGrowth\logs\node.log" -Wait -Tail 20
```

### 1.6 Startup on Android (Termux)

```bash
# Start the daemon
safegrowth-node start &

# Check status
safegrowth-cli node status

# Keep alive on charging
termux-wake-lock
```

---

## 2. Health Checks and Status Verification

### 2.1 Quick Health Check (30-second assessment)

Run this sequence whenever you want a fast system health snapshot:

```bash
# 1. Node daemon status
safegrowth-cli node status

# 2. Peer connectivity
safegrowth-cli network status

# 3. Storage health
curl -s http://localhost:8080/status | python3 -m json.tool

# 4. [M2] Credit balance
safegrowth-cli credits balance
```

### 2.2 Interpreting Node Status Output

| Field | Healthy Value | Action if Unhealthy |
|---|---|---|
| `Status` | `ONLINE` | See Section 10.1 — Daemon Not Starting |
| `Peers (LAN)` | ≥ 1 (if other devices on LAN) | Check mDNS; verify devices on same subnet |
| `Peers (WAN)` **[M2]** | ≥ 2 | Check DDNS; check bootstrap config; see Section 3 |
| `Bootstrap` **[M2]** | `OK` | At least 2 of configured bootstrap peers reachable |
| `Storage Used` | < 90% of quota | Increase quota or reduce stored shards |
| `Shard Health` | `HEALTHY` or `DEGRADED` | `DEGRADED` = some replicas below target; see Section 4 |
| `Reputation` **[M3]** | ≥ 0.85 | Below 0.6 = quarantine risk; investigate PoR failures |

### 2.3 Full Network Status

```bash
safegrowth-cli network status
# Output columns:
# NODE_ID (first 12 chars)  STATUS    LATENCY   SHARDS_HELD  UPTIME   REPUTATION
# a3f9c2b1d4e7...           ONLINE    12ms      847          99.2%    0.97
# b82f1c3a9d21...           ONLINE    34ms      612          87.1%    0.91
# c94d2e7b1f33...           OFFLINE   -         -            -        0.74
```

**Action for OFFLINE nodes**: if offline > 10 minutes, re-replication should trigger automatically. Verify with Section 4.

### 2.4 REST API Direct Health Check

```bash
# Full status JSON
curl -s http://localhost:8080/status

# Specific shard check
curl -s http://localhost:8080/shards/{shard-id}

# Peer table
curl -s http://localhost:8080/peers
```

---

## 3. DDNS Update Failure — Detection and Recovery

**Applicable nodes**: Anchor nodes (OptiPlex, Friend 1 anchor) — any node running a DDNS cron job.  
**Phase**: M2+

### 3.1 What DDNS Failure Looks Like

Symptoms of a failed DDNS update:
- Remote nodes report inability to connect to bootstrap peers
- `safegrowth-cli network status` shows WAN peers dropping off
- New nodes cannot complete join (`--bootstrap yourname.duckdns.org:4001` times out)
- The DDNS hostname resolves to an old IP address

### 3.2 Verify DDNS Status

```bash
# Check what IP your DDNS hostname resolves to
dig yourname.duckdns.org +short
# or
nslookup yourname.duckdns.org 8.8.8.8

# Check your current public IP
curl -s https://api.ipify.org

# If the two IPs differ — the DDNS record is stale
```

### 3.3 Manual DDNS Update

```bash
# DuckDNS manual update (replace TOKEN and DOMAIN)
curl -s "https://www.duckdns.org/update?domains=DOMAIN&token=TOKEN&ip="

# Expected response: "OK"
# If response is "KO" — check that token is correct in your config
```

### 3.4 Verify Cron Job is Running

```bash
# List crontab
crontab -l

# Expected entry (runs every 5 minutes):
# */5 * * * * curl -s "https://www.duckdns.org/update?domains=DOMAIN&token=TOKEN&ip=" >> /var/log/duckdns.log 2>&1

# Check last update log
tail -20 /var/log/duckdns.log
# Each line should show "OK"

# If cron is not running:
sudo systemctl status cron    # Debian/Ubuntu
sudo systemctl status crond   # RHEL/Fedora
```

### 3.5 Restore Cron Job

```bash
# Add DDNS cron job (replace TOKEN and DOMAIN)
(crontab -l 2>/dev/null; echo "*/5 * * * * curl -s \"https://www.duckdns.org/update?domains=DOMAIN&token=TOKEN&ip=\" >> /var/log/duckdns.log 2>&1") | crontab -

# Verify
crontab -l
```

### 3.6 Router Port Forward Verification

If DDNS resolves correctly but remote connections still fail:

```bash
# Test from a remote machine (not your LAN)
nc -zuv yourname.duckdns.org 4001
# Expected: "Connection to yourname.duckdns.org 4001 port [udp] succeeded"

# If this fails, the port forward is not working
# Router config required: UDP 4001 → <LAN IP of anchor node>
# Find your anchor's LAN IP:
ip route get 1.1.1.1 | awk '{print $7; exit}'
```

---

## 4. Shard Re-replication — Manual Trigger and Verification

### 4.1 When Re-replication Is Needed

Re-replication should trigger automatically (within 10 minutes per NF-07) when:
- A node goes offline
- A node's reputation drops below the quarantine threshold **[M3]**
- PoR challenges detect shard absence **[M3]**

Manual trigger is needed when:
- Automatic re-replication has not started after 15 minutes
- You are proactively retiring a node (see Section 7)
- Storage quota on a node is exceeded and shards must be redistributed
- Integration testing requires verifying replication logic

### 4.2 Check Current Replication Status

```bash
# Overall shard health summary
safegrowth-cli network status

# Detailed shard placement for all files in your vault
safegrowth-cli backup status --verbose
# Output shows: FILE_ID  SHARDS  REPLICAS  HEALTH
# a3f9...  5/5     3/3       HEALTHY
# b72c...  5/5     2/3       DEGRADED  ← needs re-replication

# Check a specific file's shard distribution
safegrowth-cli backup status --file <file-id> --verbose
```

### 4.3 Manual Re-replication Trigger

```bash
# Trigger re-replication for all degraded files
safegrowth-cli replication repair

# Trigger for a specific file
safegrowth-cli replication repair --file <file-id>

# Trigger for all shards held by a specific node (use before offboarding)
safegrowth-cli replication repair --evacuate-node <node-id>

# Watch re-replication progress (poll every 10s)
watch -n 10 "safegrowth-cli backup status --verbose | grep DEGRADED"
```

### 4.4 Verify Re-replication Completed

```bash
# All files should show HEALTHY when done
safegrowth-cli backup status --verbose | grep -v HEALTHY
# Empty output = all files healthy

# Confirm target replica count on a specific file
safegrowth-cli backup status --file <file-id>
# Expected:
# File:     <file-id>
# Shards:   5/5 intact
# Replicas: 3/3 (target: 3)  ← should match your erasure scheme target
# Health:   HEALTHY
```

### 4.5 Re-replication Is Not Starting

If `replication repair` returns immediately with no activity:

1. Verify there are enough online peers to meet the target replica count: `safegrowth-cli network status` — count ONLINE nodes
2. Phase 1 (2-of-3): need at least 3 online nodes. Phase 2 (3-of-5): need at least 5.
3. If not enough peers, bring additional nodes online before re-replication can complete
4. Check daemon logs for replication errors: `journalctl -u safegrowth-node -n 200 | grep -i replic`

---

## 5. Credit Ledger — Verification and Recovery

**Phase**: M2+

### 5.1 Check Credit Balance and History

```bash
# Current balance
safegrowth-cli credits balance
# Output:
# Node:     <your-node-id>
# Balance:  142.50 credits
# Earned:   187.25 (lifetime)
# Spent:    44.75 (lifetime)

# Full transaction history
safegrowth-cli credits history

# History for last 24 hours
safegrowth-cli credits history --since 24h

# Export to JSON for analysis
safegrowth-cli credits history --json > credits_export.json
```

### 5.2 Verify Ledger Integrity

```bash
# Validate the local credit ledger (checks signatures on all entries)
safegrowth-cli credits verify

# Expected output:
# Verifying 247 ledger entries...
# All entries valid. No anomalies detected.

# If anomalies are found:
# [WARN] Entry #183: signature invalid — node abc123... (possible tampering or clock skew)
```

### 5.3 Credit Ledger Corruption — Recovery

If `credits verify` reports signature errors on local entries:

```bash
# Step 1: Export current ledger for backup
safegrowth-cli credits history --json > credits_backup_$(date +%Y%m%d).json

# Step 2: Identify the first corrupt entry
safegrowth-cli credits verify --verbose 2>&1 | grep -i invalid | head -5

# Step 3: Request ledger sync from a trusted peer
# The ledger is consensus-validated — a peer with a clean copy can resync you
safegrowth-cli credits sync --from-peer <trusted-node-id>

# Step 4: Verify again
safegrowth-cli credits verify

# Step 5: If sync does not resolve it, truncate to last known good entry
# WARNING: this will lose credit history after the truncation point
# Only do this after consulting all team members
safegrowth-cli credits truncate --to-entry <last-valid-entry-id> --confirm
```

### 5.4 Credits Not Accruing

Expected accrual: anchor nodes earn +0.5 credits/hour of uptime. If balance is not increasing after 2+ hours online:

1. Verify node has been online continuously: `safegrowth-cli node status` → check uptime
2. **[M2]** Verify reputation is ≥ 0.6: `safegrowth-cli network status` → check your own reputation score
3. **[M3]** Verify you are past the 72-hour probation period (new nodes accrue zero credits during probation)
4. **[M3]** Verify ZK-PoR proofs are generating successfully: `journalctl -u safegrowth-node | grep -i "por\|proof"`
5. Check credit epoch is open: `journalctl -u safegrowth-node | grep -i "epoch"`

---

## 6. Key Rotation Procedure

> **Warning**: Key rotation is a significant operation. It changes your node's identity on the network. Follow this procedure exactly and ensure all team members are notified before rotating any anchor node's keys.

### 6.1 When to Rotate Keys

Rotate keys when:
- You suspect private key compromise (laptop stolen, unauthorized disk access)
- A team member who had physical access to a node leaves the project
- Routine annual rotation (recommended)

Do **not** rotate keys as a debugging step for unrelated issues.

### 6.2 Pre-Rotation Checklist

- [ ] Notify all other node operators via out-of-band channel (Signal, email)
- [ ] Verify at least 2 other nodes are online and healthy (your shards will need to survive your temporary absence)
- [ ] Export current identity for backup: `cp -r ~/.safegrowth/identity ~/.safegrowth/identity.bak.$(date +%Y%m%d)`
- [ ] Document your current NodeID: `safegrowth-cli node status | grep NodeID`
- [ ] **[M2]** Ensure shards held by your node are replicated to at least 2 other nodes before proceeding

### 6.3 Key Rotation Steps

```bash
# Step 1: Stop the daemon gracefully
sudo systemctl stop safegrowth-node
# Wait for clean shutdown (up to 30 seconds)
# Verify: sudo systemctl status safegrowth-node → "inactive (dead)"

# Step 2: Back up the existing identity
cp -r ~/.safegrowth/identity ~/.safegrowth/identity.bak.$(date +%Y%m%d_%H%M%S)

# Step 3: Remove the existing identity to force regeneration
rm -rf ~/.safegrowth/identity

# Step 4: Start the daemon — it will generate a new keypair
safegrowth-node start
# You will be prompted for a new passphrase
# Record the new NodeID immediately

# Step 5: [M2] Rejoin the network using an invite from another node
# Your new keypair is unknown to the network — you must rejoin as a new node
safegrowth-cli join \
  --invite <invite-token-from-team-member> \
  --bootstrap yourname.duckdns.org:4001

# Step 6: Verify new identity is propagated
safegrowth-cli network status
# Your new NodeID should appear in the peer table on other nodes within 60 seconds

# Step 7: Monitor re-replication
# Shards previously held by your old NodeID will be re-replicated
# Watch progress:
watch -n 10 "safegrowth-cli backup status --verbose | grep DEGRADED"
```

### 6.4 Post-Rotation Verification

```bash
# Verify new NodeID is active
safegrowth-cli node status | grep NodeID

# Verify old NodeID is no longer in peer tables
safegrowth-cli network status | grep <old-node-id>
# Should return no results

# Update node inventory (Section 13) with new NodeID
# Notify team members of new NodeID via out-of-band channel
```

---

## 7. Node Offboarding and Peer Table Cleanup

Use this procedure when permanently removing a node from the network (hardware retirement, team member departure).

### 7.1 Pre-Offboarding Checklist

- [ ] Identify all shards held by the departing node: `safegrowth-cli network status --node <node-id>`
- [ ] Verify at least (k) other nodes are online to accept re-replicated shards (k = your erasure scheme's data shard count)
- [ ] Notify all other node operators out-of-band

### 7.2 Evacuation and Shutdown

```bash
# Step 1: Trigger shard evacuation from the departing node
# Run this from any other healthy node
safegrowth-cli replication repair --evacuate-node <departing-node-id>

# Step 2: Monitor until evacuation is complete
watch -n 10 "safegrowth-cli network status --node <departing-node-id>"
# Wait until SHARDS_HELD drops to 0

# Step 3: On the departing node — graceful shutdown
sudo systemctl stop safegrowth-node
sudo systemctl disable safegrowth-node

# Step 4: [M2] Broadcast node revocation to the network
# Run from any still-active node
safegrowth-cli node revoke <departing-node-id>
# This gossips a NODE_REVOCATION message to all peers

# Step 5: Verify revocation propagated
# Wait 60 seconds, then check from another node:
safegrowth-cli network status | grep <departing-node-id>
# Should show REVOKED or not appear at all within 2 minutes
```

### 7.3 Peer Table Cleanup on All Nodes

After revocation propagates, peer tables clean up automatically. If a departed node's entry persists after 10 minutes:

```bash
# Manual peer removal (run on each affected node)
safegrowth-cli peer remove <departing-node-id>

# Verify removal
safegrowth-cli network status | grep <departing-node-id>
# Should return no results
```

---

## 8. Backup Verification and Test Restore

### 8.1 Verify a Backup Completed Successfully

```bash
# List all backed-up files
safegrowth-cli backup list

# Verify integrity of a specific backup (checks shard hashes without restoring)
safegrowth-cli backup verify --file <file-id>
# Expected:
# Verifying: documents/report.pdf
# Shards:    5/5 retrievable
# Hashes:    all match
# Status:    VERIFIED

# Verify all backups
safegrowth-cli backup verify --all
# Summary at end: X files verified, Y warnings, Z errors
```

### 8.2 Test Restore (Routine — Monthly Recommended)

**Always test restore to a temporary location, never overwrite originals.**

```bash
# Restore a single file to a temp directory
safegrowth-cli restore \
  --file <file-id-or-vault-path> \
  --to /tmp/safegrowth-test-restore/

# Verify the restored file
diff /original/path/to/file.pdf /tmp/safegrowth-test-restore/file.pdf
# No output = files are identical

# Clean up test restore
rm -rf /tmp/safegrowth-test-restore/
```

### 8.3 Test Restore with Simulated Node Failure

This is the most important test. Run it at the end of each milestone.

```bash
# Step 1: Note which nodes hold shards for a test file
safegrowth-cli backup status --file <test-file-id> --verbose

# Step 2: Take one shard holder offline (stop its daemon)
# SSH to target node:
sudo systemctl stop safegrowth-node

# Step 3: Restore from the remaining nodes
safegrowth-cli restore \
  --file <test-file-id> \
  --to /tmp/test-restore/

# Step 4: Verify restored file integrity
diff /original/file /tmp/test-restore/file
echo $?  # Must be 0

# Step 5: Bring the stopped node back online
# SSH back and:
sudo systemctl start safegrowth-node

# Step 6: Verify re-replication restores full redundancy
safegrowth-cli backup status --file <test-file-id>
# Expected: HEALTHY with full replica count after a few minutes
```

### 8.4 Restore After Total Node Failure

If your primary machine is lost and you need to restore from scratch:

```bash
# Step 1: Set up safegrowth-node on the new hardware
# Install Go, build from source, or download release binary

# Step 2: [M2] Rejoin the network via invite
safegrowth-cli join \
  --invite <token-from-team-member> \
  --bootstrap peername.duckdns.org:4001

# Step 3: Restore all your files
safegrowth-cli restore --all --to ~/Restored/

# Step 4: Verify restored files
# Compare file sizes, checksums, spot-check content
```

---

## 9. Adding a New Node (Invite Flow)

**Phase**: M2+

### 9.1 On the Inviting Node (Existing Network Member)

```bash
# Generate an invite token (default 24-hour expiry)
safegrowth-cli invite generate --expires 24h

# Output:
# Invite token: sgv-invite-a3f9c2b1d4e7f8b9c2d3e4f5a6b7c8d9
# Expires: 2026-03-19 14:32:00 UTC
# Share this token via a trusted channel (Signal, encrypted email)
# Token is single-use and will expire after 24 hours

# [Optional] Generate with longer expiry for scheduled joins
safegrowth-cli invite generate --expires 72h
```

### 9.2 Share the Token Securely

- Preferred: Signal (encrypted, ephemeral)
- Acceptable: Email with PGP encryption
- Acceptable: In-person (QR code scan via Flutter app)
- **Not recommended**: Unencrypted email, SMS, Slack (token itself contains no secrets, but intercept risk from Section 5.4 T-13 of THREAT_MODEL.md applies)

### 9.3 On the Joining Node (New Participant)

```bash
# Step 1: Install safegrowth-node on the new hardware

# Step 2: Join the network
safegrowth-cli join \
  --invite sgv-invite-a3f9c2b1d4e7f8b9c2d3e4f5a6b7c8d9 \
  --bootstrap peername.duckdns.org:4001

# Expected output:
# [INFO] Generating identity keypair...
# [INFO] New NodeID: <64-char hex>
# [INFO] Connecting to bootstrap peer peername.duckdns.org:4001...
# [INFO] Handshake complete
# [INFO] Sending JOIN_REQUEST...
# [INFO] Join accepted. Received 3 trusted peers.
# [INFO] Connecting to all peers...
# [INFO] Successfully joined SafeGrowth network.
# [INFO] Your node is in 72-hour probation. Credits will begin accruing after: 2026-03-22 14:35 UTC

# Step 3: Verify join
safegrowth-cli node status
safegrowth-cli network status
```

### 9.4 On the Inviting Node — Verify New Peer Appeared

```bash
safegrowth-cli network status | grep <new-node-id-first-12-chars>
# Should show the new node as ONLINE within 60 seconds

# Update node inventory (Section 13) with new participant's NodeID
```

### 9.5 Set Up New Node as Co-Bootstrap (Anchor Nodes Only)

If the new node is an anchor (always-on desktop or server), add it as a co-bootstrap immediately per PD-06 requirement:

```bash
# On the new anchor node: set up DDNS
# 1. Register a hostname at duckdns.org
# 2. Add the cron job (replace TOKEN and DOMAIN):
(crontab -l 2>/dev/null; echo "*/5 * * * * curl -s \"https://www.duckdns.org/update?domains=DOMAIN&token=TOKEN&ip=\" >> /var/log/duckdns.log 2>&1") | crontab -
# 3. Open port 4001 UDP on their router

# On ALL existing nodes — add new bootstrap peer to config:
# Edit ~/.safegrowth/config.json:
# "bootstrap_peers": [
#   "existinganchor.duckdns.org:4001",
#   "newanchor.duckdns.org:4001"    ← ADD THIS
# ]

# Restart daemon on each node to pick up the new config
sudo systemctl restart safegrowth-node
```

---

## 10. Emergency Procedures

### 10.1 Daemon Not Starting

```bash
# Check for error on manual start
safegrowth-node start 2>&1 | head -30

# Common causes and fixes:

# 1. Wrong passphrase
# Error: "failed to decrypt identity: cipher: message authentication failed"
# Fix: verify passphrase. If forgotten, identity is lost — must rotate (Section 6)

# 2. Port already in use
# Error: "listen udp :4001: bind: address already in use"
# Fix:
lsof -i :4001
# Kill the conflicting process or change the port in config.json

# 3. Identity file corrupt
# Error: "failed to parse identity file: unexpected EOF"
# Fix: restore from backup
cp ~/.safegrowth/identity.bak.YYYYMMDD ~/.safegrowth/identity
# If no backup exists, must perform key rotation (Section 6)

# 4. Storage path inaccessible
# Error: "failed to open pebble store at /path/to/storage: permission denied"
# Fix: check path permissions
ls -la ~/.safegrowth/storage/
chmod 700 ~/.safegrowth/storage/
```

### 10.2 Node Isolated — No Peers Reachable

```bash
# Step 1: Verify network connectivity
ping 8.8.8.8
curl -s https://api.ipify.org

# Step 2: [M2] Check bootstrap peer DNS resolution
dig existinganchor.duckdns.org +short
# If this returns the wrong IP → DDNS has failed on the bootstrap node (contact that operator)

# Step 3: [M2] Try connecting to bootstrap manually
nc -zuv existinganchor.duckdns.org 4001
# If connection refused → port forward may be down on bootstrap node (contact that operator)

# Step 4: Check local firewall
sudo iptables -L INPUT -n | grep 4001
# or
sudo ufw status | grep 4001

# Step 5: If on LAN — verify mDNS is not blocked
# mDNS uses multicast to 224.0.0.251:5353
sudo tcpdump -i any -n 'host 224.0.0.251' 2>/dev/null | head -10
# Should see mDNS packets if other nodes are announcing

# Step 6: Force peer rediscovery
safegrowth-cli peer discover --force
```

### 10.3 Data Appears Missing or Unrestorable

```bash
# Step 1: Verify the file was actually backed up
safegrowth-cli backup list | grep <filename>

# Step 2: Check shard availability
safegrowth-cli backup status --file <file-id> --verbose
# Note which shards are MISSING and which nodes hold them

# Step 3: Check if those nodes are online
safegrowth-cli network status
# If holders are offline, wait and retry — shards may still be retrievable from other replicas

# Step 4: Attempt restore explicitly
safegrowth-cli restore --file <file-id> --to /tmp/emergency-restore/

# Step 5: If restore fails with "insufficient shards"
# This means more replicas are lost than the erasure scheme can tolerate
# Phase 1 (2-of-3): need 2 of 3 shards → can tolerate 1 node loss
# Phase 2 (3-of-5): need 3 of 5 shards → can tolerate 2 node losses
# If you have lost more nodes than the scheme tolerates, data is unrecoverable from this backup
# Check if you have local copies elsewhere before declaring data loss
```

### 10.4 Suspected Key Compromise

**Treat this as a high-urgency incident.**

```bash
# Step 1: Immediately notify all other node operators out-of-band
# Use Signal or phone — do not use the potentially compromised machine

# Step 2: On the compromised machine — stop the daemon immediately
sudo systemctl stop safegrowth-node

# Step 3: Disconnect the machine from the network if practical

# Step 4: From a trusted machine — request peer revocation of the compromised node
safegrowth-cli node revoke <compromised-node-id>
# This gossips a NODE_REVOCATION to all peers

# Step 5: On the compromised machine — perform key rotation (Section 6)
# Generate new keypair on a clean machine if possible

# Step 6: Rejoin the network with the new keypair (Section 9)

# Step 7: Document the incident in the team dev log
```

---

## 11. Scheduled Maintenance Tasks

### 11.1 Daily (Automated — Verify Cron is Running)

| Task | Mechanism | Verify |
|---|---|---|
| DDNS update | Cron `*/5 * * * *` | `tail /var/log/duckdns.log` — each line must show `OK` |
| Credit accrual **[M2]** | Daemon internal (hourly) | `safegrowth-cli credits history --since 24h` |
| PoR challenge responses **[M3]** | Daemon internal (continuous) | `journalctl -u safegrowth-node | grep "por passed"` |

### 11.2 Weekly

```bash
# 1. Network status snapshot
safegrowth-cli network status > ~/safegrowth-weekly-$(date +%Y%m%d).txt

# 2. Backup verification
safegrowth-cli backup verify --all

# 3. Storage quota check
safegrowth-cli node status | grep "Storage Used"
# Alert if > 80%

# 4. Prune old logs
journalctl --vacuum-time=30d
```

### 11.3 Monthly

```bash
# 1. Test restore (see Section 8.2 — restore one file to temp location)

# 2. Verify DDNS resolves correctly from outside your LAN
# Ask a team member to run: dig yourname.duckdns.org +short
# and confirm it matches your current public IP

# 3. Check for Go security updates
go version
# Compare against https://go.dev/security/

# 4. Check for dependency updates
cd /path/to/safegrowth && go list -u -m all 2>/dev/null | grep "\["

# 5. Update node inventory (Section 13) if any hardware has changed
```

### 11.4 At Each Milestone Boundary

- Run the full end-to-end integration test documented in the dev plan for that milestone
- Run `safegrowth-cli backup verify --all` on all nodes
- Update this runbook with any new procedures introduced by the milestone
- Review the Gap Register in THREAT_MODEL.md — mark resolved gaps as closed

---

## 12. Log Reference

### 12.1 Log Locations

| Platform | Log Location | How to Access |
|---|---|---|
| Linux (systemd) | journald | `journalctl -u safegrowth-node` |
| Linux (manual) | `~/.safegrowth/logs/node.log` | `tail -f ~/.safegrowth/logs/node.log` |
| Windows (NSSM) **[M2]** | `C:\ProgramData\SafeGrowth\logs\node.log` | `Get-Content ... -Wait -Tail 50` |
| Android (Termux) | `~/.safegrowth/logs/node.log` | `tail -f ~/.safegrowth/logs/node.log` |

### 12.2 Important Log Patterns

```bash
# Filter by severity
journalctl -u safegrowth-node | grep -E "\[(ERROR|WARN)\]"

# Peer events (joins, disconnects)
journalctl -u safegrowth-node | grep -i "peer"

# Shard events (placements, retrievals, failures)
journalctl -u safegrowth-node | grep -i "shard"

# PoR events [M3]
journalctl -u safegrowth-node | grep -i "por\|challenge\|proof"

# Credit events [M2]
journalctl -u safegrowth-node | grep -i "credit\|epoch"

# Security events (signature failures, rate limits)
journalctl -u safegrowth-node | grep -E "E-00[1-6]|signature|rate.limit"
```

### 12.3 Key Log Messages and Meanings

| Log Message | Meaning | Action |
|---|---|---|
| `[INFO] Peer discovered via mDNS: <id>` | New LAN peer found | Normal — verify in `network status` |
| `[INFO] Handshake established: <id>` | Peer connection active | Normal |
| `[WARN] Bootstrap peer unreachable: <hostname>` | DDNS or network issue | See Section 3 |
| `[WARN] Shard replica count below target: <file-id>` | Re-replication needed | See Section 4 |
| `[ERROR] E-006: Invalid signature from <id>` | Possible tampering or misconfigured peer | Investigate; may need peer removal |
| `[ERROR] E-011: Rate limit exceeded from <id>` | Peer sending too many gossip messages | Peer will be temporarily banned automatically |
| `[WARN] Storage quota 90% full` | Disk space low | Increase quota or add storage |
| `[ERROR] Failed to open Pebble store` | Storage corruption or permission error | See Section 10.1 |
| `[INFO] PoR challenge passed: <shard-id>` **[M3]** | Shard verified present | Normal |
| `[WARN] PoR challenge failed: <shard-id>` **[M3]** | Shard may be missing | Investigate immediately; see Section 4 |

---

## 13. Node Inventory

**Keep this section current.** Update whenever hardware changes, NodeIDs rotate, or network topology changes.

### 13.1 Node Registry

| Node Name | Hardware | OS | Role | NodeID (first 16 chars) | DDNS Hostname | LAN IP | Owner |
|---|---|---|---|---|---|---|---|
| opti-anchor | OptiPlex 990 | Ubuntu 22.04 | Anchor / Bootstrap | *(record on first run)* | yourname.duckdns.org | 192.168.1.x | Glenn |
| flex-mid | Lenovo Flex | Windows 11 | Mid-tier | *(record on first run)* | — | 192.168.1.x | Glenn |
| latitude-mid | Dell Latitude | Ubuntu 22.04 | Mid-tier | *(record on first run)* | — | 192.168.1.x | Glenn |
| android-edge | Android Phone | Android 13 | Edge | *(record on first run)* | — | 192.168.1.x | Glenn |
| friend1-anchor | TBD | TBD | Anchor / Bootstrap **[M2]** | *(record on join)* | friend1.duckdns.org | — | Friend 1 |
| friend2-mid | TBD | TBD | Mid-tier **[M2]** | *(record on join)* | — | — | Friend 2 |

### 13.2 Network Configuration

| Parameter | Value |
|---|---|
| Primary bootstrap | `yourname.duckdns.org:4001` |
| Secondary bootstrap **[M2]** | `friend1.duckdns.org:4001` |
| P2P port (UDP) | 4001 |
| REST API port (localhost) | 8080 |
| Phase 1 erasure scheme | 2-of-3 |
| Phase 2 erasure scheme | 3-of-5 |
| Phase 3 erasure scheme | 5-of-9 |
| Storage quota (OptiPlex) | *(set in config.json)* |
| Gossip bandwidth cap | 10% of upload |

### 13.3 Key Contacts (Out-of-Band)

| Name | Role | Signal / Contact |
|---|---|---|
| Glenn | Lead developer, anchor operator | *(stored separately — not in repo)* |
| Friend 1 | Remote collaborator, anchor operator **[M2]** | *(stored separately — not in repo)* |
| Friend 2 | Remote collaborator **[M2]** | *(stored separately — not in repo)* |

> **Security Note**: Do not store contact details, passphrases, or DDNS tokens in this document if it is committed to the repository. Store sensitive values in a password manager or encrypted notes shared out-of-band.
