# Monad Testnet — Monlog

> Monlog is a diagnostic tool for Monad nodes that extracts and summarises the last 60 seconds of journald logs.  
> It has no daemon, runs no background process and opens no ports — it executes on demand and exits.  
> Binary installed at `/home/monad/monlog`.

---

## Prerequisites

The `monad` user requires access to the `systemd-journal` group to read system logs.

Run as root or with sudo:

```bash
usermod -a -G systemd-journal monad
```

> ℹ️ The group change requires reconnecting the session to take effect. Switching to `monad` immediately after `usermod` via `su - monad` will fail — use `sudo su - monad` or `sudo -u monad -i` instead.

---

## Installation

Switch to the monad user:

```bash
sudo su - monad
```

Download the binary:

```bash
cd /home/monad
curl -sSL https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev/scripts/monlog -O
chmod u+x ./monlog
```

Verify the checksum:

```bash
sha256sum ./monlog
```

Checksum documented in the [official Monad docs](https://docs.monad.xyz/node-ops/general-operations):

```
f8a1066d8c093bbdb5f722f5b3d89904c17a76fa01fa1dd62e950f445e88327f
```

> ⚠️ The binary may be updated by Monad without the documentation being updated simultaneously. If the checksum does not match, verify in the `#build_anything` Discord channel before running. The checksum of the currently downloaded binary is:
> ```
> 9ea3c58df6d390e7ba73832c330a077d91f49102274b3b95ecbcbc297573bc22
> ```
> Confirmed as valid by the Monad team (recent binary update, docs not yet updated).

---

## Usage

### Basic execution

```bash
./monlog
```

Analyses the last 60 seconds of journald logs and displays a structured summary organised by section.

### Watch mode (continuous monitoring)

```bash
watch -d "./monlog"
```

Runs the binary every 2 seconds and highlights changes on screen. System impact is negligible.

### Flag `-n` (validator name mapping)

```bash
./monlog -n
```

Displays validator names mapped to their SECP keys. Useful for identifying which validator proposed each block. Prometheus/OTEL only exposes the raw key — this flag adds human-readable context.

---

## Output Reference

Example output from a synced and healthy node:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  MONLOG v0.7.3 - Monad Log Analyzer
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Network: testnet

📦 VERSION INFORMATION
APT Package:
  ii monad 0.14.2 amd64 Monad BFT stack (with debug symbols)
RPC Version:
  {"jsonrpc":"2.0","result":"Monad/0.14.2","id":1}

🚀 STARTUP STATUS
No startup/forkpoint messages in the last 60 seconds.

🔄 STATE SYNC STATUS
No StateSync messages.

📦 BLOCK SYNC STATUS
ZERO BlockSync overrides detected.
No BlockSync messages.

📡 RAPTORCAST STATUS
✅ RaptorCast: Confirming group join!
RaptorCast: No ping requests seen in the last 60 seconds...

📊 CONSENSUS STATUS
Most recent round: 27499195
Most recent epoch: 548
Most recent block: 27356898
✅ Blocks processing and being committed
```

### Section reference

| Section | Expected state on healthy node |
|---|---|
| **VERSION INFORMATION** | APT and RPC versions match |
| **STARTUP STATUS** | `No startup/forkpoint messages` — node has been running for a while |
| **STATE SYNC STATUS** | `No StateSync messages` — node already in sync |
| **BLOCK SYNC STATUS** | `ZERO BlockSync overrides` — normal operation in live mode |
| **RAPTORCAST STATUS** | `✅ Confirming group join!` — node is receiving blocks |
| **CONSENSUS STATUS** | `✅ Blocks processing and being committed` — blocks advancing |

> ℹ️ Empty StateSync and BlockSync sections are expected once the node is synced. They only show activity during the initial sync process or in the event of a re-sync.

### Warning signals

| Output | Meaning |
|---|---|
| `RaptorCast: No group join` | Node is not receiving blocks — check connectivity |
| `Blocks NOT being committed` | Consensus stalled — check services and peers |
| Active messages in StateSync | Node is re-syncing |
| Timeouts or dropping proposals | Possible network or peer issues |

---

## Relation to Prometheus/OTEL

Monlog extracts information from logs that is not always available as a Prometheus metric:

| Data | Prometheus/OTEL | Monlog |
|---|---|---|
| Block height, epoch, round | ✅ Numeric metric | ✅ Visible in output |
| Sync status | ✅ `monad_statesync_syncing` | ✅ StateSync section |
| RaptorCast group join | ❌ Not exposed | ✅ Visible in output |
| Validator names | ❌ Raw key only | ✅ With `-n` flag |
| Timeouts and dropping proposals | Partially via validation errors | ✅ Detail in logs |
| Qualitative diagnostics | ❌ Not applicable | ✅ Status suggestions |

Monlog is complementary to the Prometheus/Grafana stack — best used for point-in-time diagnosis when something goes wrong, not as a continuous metrics source.

---

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops/general-operations)
- [05 — Metrics Reference](05-monad-metrics-reference.md)
