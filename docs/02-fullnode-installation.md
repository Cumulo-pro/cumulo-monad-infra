# Monad Testnet — Full Node Installation

> Continuation of [`monad-validator-setup.md`](./monad-validator-setup.md).  
> Covers full node installation, configuration, snapshot import, and first sync.  
> Part of Cumulo's validator candidacy process.

---

## Node Identity

| Field | Value |
|---|---|
| Node name | `full_Cumulo-1` |
| Public IP | `192.155.100.132` |
| P2P port | `8000` |
| Auth port | `8001` |
| SECP public key | `02ab335bea8131ddd8de77b7bea483313d1278b7c533f4ff48e5f8393370c924b9` |
| BLS public key | `a8b23eeb8fbd077b82c3161e4f34d681ad214f483d352ba938211ba96a62fd49f521575f5cfd6d40dae8558e8d685c31` |
| Monad version | `v0.14.1` |
| Network | `monad_testnet` |

> ⚠️ Private keystores (`id-secp`, `id-bls`) and the keystore password backup are stored in an external secrets vault. These files define the node's identity — losing them requires re-registering with a new identity.

---

## Installation Steps

### 10. Install dependencies

```bash
apt install -y curl nvme-cli aria2 jq screen iptables-persistent
```

`aria2` is required by the official snapshot restore script. `screen` is recommended for long-running processes. `iptables-persistent` persists firewall rules across reboots.

---

### 11. Configure firewall (UFW + iptables)

```bash
ufw allow ssh
ufw allow 8000
ufw allow 8001
ufw enable
ufw status
```

Additional iptables rule to drop short UDP packets — improves robustness against spam traffic on the P2P port:

```bash
iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP
netfilter-persistent save
```

> This rule persists across reboots via `iptables-persistent`. Without it, the rule resets on reboot.

---

### 12. Install OTEL Collector

The `monad` package integrates with OpenTelemetry for metrics collection. Metrics are exposed at `http://0.0.0.0:8889/metrics`.

```bash
OTEL_VERSION="0.139.0"
curl -fsSL "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${OTEL_VERSION}/otelcol_${OTEL_VERSION}_linux_amd64.deb" \
  -o /tmp/otelcol_linux_amd64.deb

dpkg -i /tmp/otelcol_linux_amd64.deb
cp /opt/monad/scripts/otel-config.yaml /etc/otelcol/config.yaml
systemctl restart otelcol
systemctl status otelcol
```

✅ `otelcol.service` active (running).

---

### 13. Download testnet configuration files

Configuration files for **full nodes** on testnet:

```bash
MF_BUCKET=https://bucket.monadinfra.com

curl -o /home/monad/.env \
  $MF_BUCKET/config/testnet/latest/.env.example

curl -o /home/monad/monad-bft/config/node.toml \
  $MF_BUCKET/config/testnet/latest/full-node-node.toml
```

The downloaded `.env` includes remote URLs for automatic `forkpoint.toml` and `validators.toml` fetching on startup (feature available since v0.12.1):

```env
CHAIN=monad_testnet
KEYSTORE_PASSWORD=
REMOTE_VALIDATORS_URL='https://bucket.monadinfra.com/validators/testnet/validators.toml'
REMOTE_FORKPOINT_URL='https://bucket.monadinfra.com/forkpoint/testnet/forkpoint.toml'
RETENTION_LEDGER=600
RETENTION_WAL=300
RETENTION_FORKPOINT=300
RETENTION_VALIDATORS=43200
```

---

### 14. Generate keystore password

```bash
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env

mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup
```

> ⚠️ Store `/opt/monad/backup/keystore-password-backup` in an external password manager or secrets vault immediately. This password is required to decrypt the keystores.

---

### 15. Generate BLS and SECP keystores

```bash
bash <<'EOF'
set -e
source /home/monad/.env

monad-keystore create \
  --key-type secp \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup

monad-keystore create \
  --key-type bls \
  --keystore-path /home/monad/monad-bft/config/id-bls \
  --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup

grep "public key" /opt/monad/backup/secp-backup /opt/monad/backup/bls-backup \
  | tee /home/monad/pubkey-secp-bls

echo "Success: New keystores generated"
EOF
```

Public keys written to `/home/monad/pubkey-secp-bls` for reference.

> ⚠️ Back up `/opt/monad/backup/secp-backup` and `/opt/monad/backup/bls-backup` externally. These files encode the node's network identity.

---

### 16. Sign name record

The name record is a cryptographically signed payload used for peer discovery and network topology management.

```bash
source /home/monad/.env

monad-sign-name-record \
  --address $(curl -s4 ifconfig.me):8000 \
  --authenticated-udp-port 8001 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num 1
```

Output:

```
self_address = "192.155.100.132:8000"
self_record_seq_num = 1
self_auth_port = 8001
self_name_record_sig = "0358853fef23a0f78a9477485da5bc7e090292341ae9adf433c479c5af4660a5334a81cd0a1c8b43b5a19f8c77e638ed51e2f9fc75f705921ff834c3e09641a701"
```

---

### 17. Configure `node.toml`

Edit `/home/monad/monad-bft/config/node.toml` and update the following fields:

```toml
node_name = "full_Cumulo-1"

[peer_discovery]
self_address = "192.155.100.132:8000"
self_record_seq_num = 1
self_name_record_sig = "0358853fef23a0f78a9477485da5bc7e090292341ae9adf433c479c5af4660a5334a81cd0a1c8b43b5a19f8c77e638ed51e2f9fc75f705921ff834c3e09641a701"
```

The `beneficiary` field is set to the burn address (correct for full nodes — only validators need a reward address):

```toml
beneficiary = "0x0000000000000000000000000000000000000000"
```

The following flags were confirmed active in the downloaded config:

```toml
[fullnode_raptorcast]
enable_client = true

[statesync]
expand_to_group = true

[blocksync_override]
peers = []   # empty — public full node, permissionless
```

---

### 18. Set filesystem permissions

```bash
chown -R monad:monad /home/monad/
```

---

### 19. Import testnet snapshot

> ⚠️ **Critical:** Starting the node without a snapshot causes it to sync from block 0, which would take days. Always import a snapshot first.

Run the snapshot restore inside `screen` to prevent interruption from SSH disconnection:

```bash
screen -S monad-snapshot
```

Inside the screen session:

```bash
# Reset workspace (wipes any existing runtime state)
bash /opt/monad/scripts/reset-workspace.sh

# Download and import snapshot from Monad Foundation
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash
```

Expected final output:

```
[2026-04-19 14:57:11] Snapshot imported at block ID: 26462562
[2026-04-19 14:57:11] You can now start the node
```

✅ Snapshot imported at block `26,462,562`.

> **Lesson learned:** If the snapshot import is interrupted (e.g. Ctrl+C), the TrieDB is left in an invalid state. `reset-workspace.sh` must be run again before retrying — attempting to re-import without resetting will produce:  
> `Assertion 'database must be hard reset when loading snapshot' failed`

---

### 20. Enable and start services

```bash
systemctl enable monad-bft monad-execution monad-rpc
systemctl start  monad-bft monad-execution monad-rpc
```

On startup, `monad-bft` automatically fetches `forkpoint.toml` and `validators.toml` from the remote URLs configured in `.env` — no manual download needed.

---

## Verification

### Service status

```bash
systemctl status monad-bft monad-execution monad-rpc
```

All three services showed `active (running)` within seconds of starting.

### Live log monitoring

```bash
# Filter out DEBUG noise
journalctl -u monad-bft -f --output cat | grep -v DEBUG
```

Confirmed peer connections and block commits within ~2 minutes of startup:

```json
{"level":"INFO","fields":{"message":"committed block","num_tx":2,"block_num":26474589}}
{"level":"INFO","fields":{"message":"committed block","num_tx":4,"block_num":26474590}}
{"level":"INFO","fields":{"message":"committed block","num_tx":4,"block_num":26474591}}
```

✅ Node committing live blocks from the testnet.

> **Note:** The WARN message `not voting on proposal, is not coherent` is expected and normal for a full node — it receives validator proposals but does not vote.

---

## Key Files Reference

| Path | Purpose |
|---|---|
| `/home/monad/.env` | Environment variables — chain, password, remote config URLs, retention |
| `/home/monad/monad-bft/config/node.toml` | Node configuration — name, peer discovery, network params |
| `/home/monad/monad-bft/config/id-secp` | SECP keystore (node network identity) |
| `/home/monad/monad-bft/config/id-bls` | BLS keystore (consensus identity) |
| `/home/monad/pubkey-secp-bls` | Public keys — quick reference |
| `/opt/monad/backup/` | Keystore backups and password — **store externally** |
| `/dev/triedb` | Symlink → `nvme1n1p1` (TrieDB raw block device) |
| `/etc/udev/rules.d/99-triedb.rules` | Persistent udev rule for `/dev/triedb` symlink |
| `/etc/otelcol/config.yaml` | OTEL Collector configuration |

---

## Useful Commands

```bash
# Real-time logs (no debug noise)
journalctl -u monad-bft -f --output cat | grep -v DEBUG

# Check committed blocks
journalctl -u monad-bft --output cat | grep "committed block" | tail -20

# Check peer connections
journalctl -u monad-bft --output cat | grep "keepalive" | tail -5

# OTEL metrics endpoint
curl http://0.0.0.0:8889/metrics

# Hard reset (if node needs to be re-synced from snapshot)
systemctl stop monad-bft monad-execution monad-rpc
screen -S monad-snapshot
bash /opt/monad/scripts/reset-workspace.sh
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash
systemctl start monad-bft monad-execution monad-rpc
```

---

## Final Status

| Step | Status |
|---|---|
| Hardware provisioned | ✅ |
| LBA 512b verified | ✅ |
| TrieDB partitioned (`nvme1n1p1`) | ✅ |
| `/dev/triedb` symlink active | ✅ |
| `monad-testnet` package installed | ✅ |
| `monad` user and directories created | ✅ |
| SMT disabled (BIOS — Supermicro IPMI) | ✅ |
| Kernel ≥ 6.8.0-60 (running 6.8.0-110) | ✅ |
| TrieDB initialized (`monad-mpt`) | ✅ |
| Firewall configured (UFW + iptables) | ✅ |
| OTEL Collector running | ✅ |
| Testnet config files downloaded | ✅ |
| Keystores generated (SECP + BLS) | ✅ |
| `node.toml` configured | ✅ |
| Snapshot imported (block 26,462,562) | ✅ |
| Services enabled + started | ✅ |
| Node synced — committing live blocks | ✅ |

---

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [Full Node Installation](https://docs.monad.xyz/node-ops/full-node-installation)
- [Hard Reset Instructions](https://docs.monad.xyz/node-ops/node-recovery/hard-reset)
- [Full Node Block Delivery](https://docs.monad.xyz/node-ops/full-node-block-delivery)
- [Monad Node Announcements (Telegram)](https://t.me/MonadNodeAnnouncements)
