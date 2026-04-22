# Monad Testnet - CLI Reference

> Part of Cumulo's Monad testnet full node infrastructure documentation.  
> CLI commands organised by category for the day-to-day management of node `full_Cumulo-1`.

---

## Table of Contents

1. [Node status](#1-node-status)
2. [Service management (systemd)](#2-service-management-systemd)
3. [Logs and diagnostics](#3-logs-and-diagnostics)
4. [Block monitoring and RPC](#4-block-monitoring-and-rpc)
5. [MonadDB / TrieDB](#5-monaddb--triedb)
6. [monlog — BFT log analysis](#6-monlog--bft-log-analysis)
7. [ledger-tail — consensus data](#7-ledger-tail--consensus-data)
8. [OTEL Collector](#8-otel-collector)
9. [Push pipeline monitoring (Cumulo)](#9-push-pipeline-monitoring-cumulo)
10. [Version and CLI help](#10-version-and-cli-help)
11. [Keys — backup and restore](#11-keys--backup-and-restore)
12. [Node recovery](#12-node-recovery)
13. [Package upgrade](#13-package-upgrade)
14. [Firewall (ufw)](#14-firewall-ufw)

---

## 1. Node status

```bash
# Full node status in a single call
monad-status

# Continuous status with watch (refreshes every 5s)
watch -n 5 monad-status
```

---

## 2. Service management (systemd)

### Enable services on boot

```bash
sudo systemctl enable monad-bft monad-execution monad-rpc
```

### Start services

```bash
sudo systemctl start monad-bft monad-execution monad-rpc
```

### Stop services

```bash
sudo systemctl stop monad-bft monad-execution monad-rpc
```

### Restart services

```bash
sudo systemctl restart monad-bft monad-execution monad-rpc
```

### Status of all services (compact view)

```bash
sudo systemctl status monad-bft monad-execution monad-rpc --no-pager -l
```

### Status per individual service

```bash
sudo systemctl status monad-bft
sudo systemctl status monad-execution
sudo systemctl status monad-rpc
```

### Verify services are active (clean list)

```bash
systemctl list-units --type=service monad-bft.service monad-execution.service monad-rpc.service
```

---

## 3. Logs and diagnostics

### Follow logs in real time

```bash
# BFT (consensus)
sudo journalctl -u monad-bft -f --output cat

# BFT without DEBUG messages
sudo journalctl -u monad-bft -f --output cat | grep -v DEBUG

# Execution
sudo journalctl -u monad-execution -f --output cat

# RPC
sudo journalctl -u monad-rpc -f --output cat
```

### Last N lines of log

```bash
# Last 20 lines of execution
sudo journalctl -u monad-execution --output cat | tail -20

# Last 50 lines of BFT
sudo journalctl -u monad-bft --output cat | tail -50
```

### Filter by keyword (blocks, sync)

```bash
# Blocks received, sync, finalized, height
sudo journalctl -u monad-bft --output cat | grep -i "block\|sync\|finalized\|height" | tail -20

# Errors only
sudo journalctl -u monad-bft --output cat | grep -i "error\|panic\|fatal" | tail -20

# Logs since N minutes/hours ago
sudo journalctl -u monad-bft --since "5 min ago"
sudo journalctl -u monad-bft --since "1 hour ago"
```

### Follow all Monad services at once

```bash
sudo journalctl -u monad-bft -u monad-execution -u monad-rpc -f --output cat
```

---

## 4. Block monitoring and RPC

> The RPC service starts listening on port `8080` once statesync is complete.

### Current block height (eth_blockNumber)

```bash
curl http://localhost:8080/ \
  -X POST \
  -H "Content-Type: application/json" \
  --data '{"method":"eth_blockNumber","params":[],"id":1,"jsonrpc":"2.0"}'
```

### RPC client version

```bash
curl http://localhost:8080/ \
  -X POST \
  -H "Content-Type: application/json" \
  --data '{"method":"web3_clientVersion","params":[],"id":1,"jsonrpc":"2.0"}'
```

### Net version (chain ID)

```bash
curl http://localhost:8080/ \
  -X POST \
  -H "Content-Type: application/json" \
  --data '{"method":"net_version","params":[],"id":1,"jsonrpc":"2.0"}'
```

---

## 5. MonadDB / TrieDB

> MonadDB compacts automatically when reaching 80% capacity to preserve optimal SSD performance.

```bash
# Check disk usage and retained block history
monad-mpt --storage /dev/triedb

# Expected output example:
# MPT database on storages:
#   Capacity: 3.49 Tb | Used: 24.30 Gb | 0.68%
#   History: 599928 blocks, earliest 36784905, latest 37384832
```

---

## 6. monlog - BFT log analysis

> Lightweight tool maintained by Category Labs. Scrapes BFT logs from the last 60 seconds.

### Setup (once, as root)

```bash
# Grant the monad user access to systemd journal logs
usermod -a -G systemd-journal monad
```

### Installation (as monad user)

```bash
cd /home/monad
curl -sSL https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev/scripts/monlog -O
chmod u+x ./monlog

# Verify checksum
sha256sum ./monlog
# f8a1066d8c093bbdb5f722f5b3d89904c17a76fa01fa1dd62e950f445e88327f  ./monlog
```

### Usage

```bash
# Basic run
./monlog

# Continuous live updates
watch -d "./monlog"

# Show last 10 lines of grabbed logs
./monlog -r

# Show secpKey → validator name mapping
./monlog -n

# No colour output (useful for pipelines or plain logging)
./monlog --no-color
```

---

## 7. ledger-tail - consensus data

> Parses ledger artifacts and streams consensus events in JSON format.

```bash
# Start the service
systemctl start monad-ledger-tail

# Follow output in real time
journalctl -fu monad-ledger-tail
```

---

## 8. OTEL Collector

```bash
# Status
sudo systemctl status otelcol

# Restart
sudo systemctl restart otelcol

# Follow logs in real time
sudo journalctl -u otelcol -f

# Last 30 lines
sudo journalctl -u otelcol --output cat | tail -30

# Verify metrics endpoint is exposed on port 8889
curl -s http://localhost:8889/metrics | head -10
```

---

## 9. Push pipeline monitoring (Cumulo)

### Metrics push timer

```bash
# Timer status
sudo systemctl status monad-push-metrics.timer

# Push logs (last 5 minutes)
sudo journalctl -u monad-push-metrics --since "5 min ago"

# Trigger a manual push
sudo /usr/local/bin/monad-push-metrics.sh
```

### Verify hosting cache

```bash
# Confirm metrics_cache.json is being updated
curl -s https://cumulo.pro/services/monad_testnet/metrics_cache.json | grep updated_at
```

---

## 10. Version and CLI help

```bash
# Node version
monad-node --version

# Version per binary
monad-rpc --version
monad --version

# CLI argument help
monad-rpc --help
monad --help
monad-bft --help
```

---

## 11. Keys - backup and restore

### Export key backups

```bash
source /home/monad/.env

# Rotate existing backups with a timestamp suffix
[ -f /opt/monad/backup/secp-backup ] && mv /opt/monad/backup/secp-backup \
  "/opt/monad/backup/secp-backup.$(date +%Y%m%d%H%M%S).bak"
[ -f /opt/monad/backup/bls-backup ] && mv /opt/monad/backup/bls-backup \
  "/opt/monad/backup/bls-backup.$(date +%Y%m%d%H%M%S).bak"

# Export SECP key
monad-keystore recover \
  --password "$KEYSTORE_PASSWORD" \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --key-type secp > /opt/monad/backup/secp-backup

# Export BLS key
monad-keystore recover \
  --password "$KEYSTORE_PASSWORD" \
  --keystore-path /home/monad/monad-bft/config/id-bls \
  --key-type bls > /opt/monad/backup/bls-backup
```

> ⚠️ These files contain the node's private keys - they define its identity. Store them in an external location (password manager or secrets vault).

### Import keys from backup

```bash
source /home/monad/.env

# Rotate existing keystores
[ -f /home/monad/monad-bft/config/id-secp ] && mv /home/monad/monad-bft/config/id-secp \
  "/home/monad/monad-bft/config/id-secp.$(date +%Y%m%d%H%M%S).bak"
[ -f /home/monad/monad-bft/config/id-bls ] && mv /home/monad/monad-bft/config/id-bls \
  "/home/monad/monad-bft/config/id-bls.$(date +%Y%m%d%H%M%S).bak"

# Extract IKM secrets from backups
SECP_IKM=$(grep -E "Keystore secret:|Keep your IKM secure:" /opt/monad/backup/secp-backup | awk '{print $NF}')
BLS_IKM=$(grep -E "Keystore secret:|Keep your IKM secure:" /opt/monad/backup/bls-backup | awk '{print $NF}')

# Import SECP key
monad-keystore import \
  --ikm "$SECP_IKM" \
  --password "$KEYSTORE_PASSWORD" \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --key-type secp

# Import BLS key
monad-keystore import \
  --ikm "$BLS_IKM" \
  --password "$KEYSTORE_PASSWORD" \
  --keystore-path /home/monad/monad-bft/config/id-bls \
  --key-type bls
```

---

## 12. Node recovery

### Soft Reset (automated - v0.12.1+)

> Use when the node tip is close to the network tip. `monad-bft` auto-fetches configs on startup.

```bash
# Restart services — monad-bft will auto-download configs on startup
systemctl restart monad-bft monad-execution monad-rpc

# Confirm services are running
systemctl list-units --type=service monad-bft.service monad-execution.service monad-rpc.service

# Follow logs to confirm catch-up
journalctl -u monad-bft -f
```

### Soft Reset (manual - testnet)

```bash
# 1. Stop services
systemctl stop monad-bft monad-execution monad-rpc

# 2. Download latest forkpoint and validators config
MF_BUCKET=https://bucket.monadinfra.com
VALIDATORS_FILE=/home/monad/monad-bft/config/validators/validators.toml

curl -sSL $MF_BUCKET/scripts/testnet/download-forkpoint.sh | bash
curl $MF_BUCKET/validators/testnet/validators.toml -o $VALIDATORS_FILE
chown monad:monad $VALIDATORS_FILE

# 3. Start services
systemctl start monad-bft monad-execution monad-rpc
```

### Hard Reset (testnet)

> Wipes local state and restores from a chain snapshot. Use when the node is far behind.

```bash
# 1. Stop services and wipe the workspace
bash /opt/monad/scripts/reset-workspace.sh

# 2a. Restore snapshot — Monad Foundation provider
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash

# 2b. Restore snapshot — Category Labs provider (alternative)
CL_BUCKET=https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev
curl -sSL $CL_BUCKET/scripts/testnet/restore_from_snapshot.sh | bash

# 3. (Optional if auto-fetch not configured) Download runtime configs
MF_BUCKET=https://bucket.monadinfra.com
VALIDATORS_FILE=/home/monad/monad-bft/config/validators/validators.toml

curl -sSL $MF_BUCKET/scripts/testnet/download-forkpoint.sh | bash
curl $MF_BUCKET/validators/testnet/validators.toml -o $VALIDATORS_FILE
chown monad:monad $VALIDATORS_FILE

# 4. Start services
systemctl start monad-bft monad-execution monad-rpc
```

### Remote config - environment variables (.env)

```bash
# View configured variables
cat /home/monad/.env | grep -E "REMOTE|KEYSTORE"

# Temporarily disable auto-fetch
unset REMOTE_VALIDATORS_URL
unset REMOTE_FORKPOINT_URL
```

---

## 13. Package upgrade

```bash
# Upgrade to a specific version
sudo apt update && sudo apt install --reinstall monad=0.14.1 -y \
  --allow-downgrades \
  --allow-change-held-packages

# Restart and verify
sudo systemctl restart monad-bft monad-execution monad-rpc
sudo systemctl status monad-bft monad-execution monad-rpc --no-pager -l

# Confirm running version
monad-node --version
```

---

## 14. Firewall (ufw)

```bash
# Install ufw if not available
sudo apt update && sudo apt install -y ufw

# Allow OTEL metrics only from the Prometheus server
sudo ufw allow from 54.39.128.229 to any port 8889

# Enable firewall
sudo ufw enable

# Verify the rule
sudo ufw status | grep 8889

# Show all rules
sudo ufw status verbose
```

---

## Quick reference - common operation sequences

| Situation | Key commands |
|---|---|
| Check node status | `monad-status` |
| Normal restart | `sudo systemctl restart monad-bft monad-execution monad-rpc` |
| Check if in sync | `monad-status` → `blockDifference: 0` |
| Diagnose an issue | `sudo journalctl -u monad-bft -f --output cat \| grep -v DEBUG` |
| Node slightly behind | Automated soft reset (restart services) |
| Node far behind | Hard reset with snapshot |
| Upgrade version | `sudo apt install --reinstall monad=X.Y.Z` + restart |
| Back up keys | `monad-keystore recover ...` |

---

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [General Operations](https://docs.monad.xyz/node-ops/general-operations)
- [Upgrade Instructions](https://docs.monad.xyz/node-ops/upgrade-instructions)
- [Recovering a Node](https://docs.monad.xyz/node-ops/node-recovery)
- [Monad Node Announcements (Telegram)](https://t.me/MonadNodeAnnouncements)
- [Public Dashboard](https://cumulo.pro/services/monad_testnet/metrics.php)
