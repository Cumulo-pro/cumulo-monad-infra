# Monad Testnet - Node Monitoring Dashboard

> Part of Cumulo's Monad testnet full node infrastructure documentation.  
> Covers the complete monitoring stack: data collection, push pipeline, and public dashboard.

---

## Overview

This document describes the monitoring setup implemented for `full_Cumulo-1`, Cumulo's Monad testnet full node. The goal was to build a lightweight, public-facing dashboard that shows real-time node health without exposing sensitive data or relying on complex infrastructure.

### Design principles

- **No external monitoring services** - self-hosted, no Grafana Cloud, no Datadog
- **Pull-friendly source** - uses `monad-status`, the official Monad node status script
- **Minimal footprint** - a bash script, a PHP receiver, and a PHP dashboard page
- **Public** - no authentication required to view the dashboard
- **Resilient** - systemd timer ensures continuous push even after reboots

---

## Architecture

```
[Monad Node]
  monad-status (every 30s via systemd timer)
       ↓ bash parser → JSON (~400 bytes)
       ↓ curl POST (HTTPS + shared secret header)
[Hosting — PHP receiver]
  v1/metrics/index.php
       ↓ validates token → writes metrics_cache.json
[Dashboard]
  metrics.php
       ↓ reads metrics_cache.json every 30s (JS fetch)
       → renders live dashboard
```

---

## Prerequisites

### On the node

- Monad full node running (`monad-bft`, `monad-execution`, `monad-rpc`)
- `monad-status` installed (official script from Monad Foundation)
- `curl` and `python3` available (standard on Ubuntu 24.04)

### On the hosting

- PHP-enabled shared hosting or VPS
- HTTPS endpoint (Let's Encrypt or equivalent)
- Write permissions for PHP in the target directory

---

## Step 1 - Install `monad-status`

`monad-status` is the official Monad Foundation script that outputs a structured summary of node health in a single call.

```bash
sudo curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh \
  -o /usr/local/bin/monad-status
sudo chmod +x /usr/local/bin/monad-status
```

Verify it works:

```bash
monad-status
```

Expected output includes consensus status, block height, epoch, round, services, TrieDB usage, statesync progress, and node config.

---

## Step 2 - Generate a shared secret token

The push script and PHP receiver share a secret token for authentication. Generate a strong random token:

```bash
openssl rand -base64 32
```

Save this value - you will need it in both the push script (Step 3) and the PHP receiver (Step 4).

> ⚠️ Never commit this token to a public repository. Store it in a secrets manager or environment variable.

---

## Step 3 - Push script

Create the push script on the node at `/usr/local/bin/monad-push-metrics.sh`:

```bash
#!/bin/bash
# /usr/local/bin/monad-push-metrics.sh
# Collects metrics from monad-status and pushes to hosting receiver

ENDPOINT="https://your-domain.com/services/monad_testnet/v1/metrics/"
TOKEN="your-secret-token-here"

RAW=$(monad-status 2>/dev/null)
if [ -z "$RAW" ]; then
    echo "ERROR: monad-status returned no data" >&2
    exit 1
fi

JSON=$(echo "$RAW" | python3 -c "
import sys, json, re

raw = sys.stdin.read()

def val(key, text):
    m = re.search(rf'(?m)^\s*{key}:\s*(.+)', text)
    return m.group(1).strip() if m else None

def svc(name, text):
    m = re.search(rf'{name}:\s*(\w+)', text)
    return m.group(1) if m else 'unknown'

data = {
    'updated_at': val('date', raw),
    'hostname':   val('hostname', raw),
    'uptime':     val('uptime', raw),
    'version':    val('packageVersion', raw),
    'network':    val('network', raw),
    'chain_id':   val('chainId', raw),
    'secp_pubkey': val('secpPublicKey', raw),
    'consensus': {
        'status':     val('status', raw),
        'mode':       val('mode', raw),
        'peers':      val('peersNumber', raw),
        'epoch':      val('epoch', raw),
        'round':      val('round', raw),
        'block':      val('blockNumber', raw),
        'block_diff': val('blockDifference', raw),
    },
    'statesync': {
        'percentage': val('percentage', raw),
        'progress':   val('progress', raw),
        'target':     val('target', raw),
    },
    'rpc_status': (lambda m: m.group(1) if m else None)(re.search(r'rpc:\s*\n\s+status:\s*(\w+)', raw)),
    'triedb': {
        'model':    val('model', raw),
        'capacity': val('capacity', raw),
        'used':     val('used', raw),
    },
    'services': {
        'monad-bft':       svc('monad-bft', raw),
        'monad-execution': svc('monad-execution', raw),
        'monad-rpc':       svc('monad-rpc', raw),
        'otelcol':         svc('otelcol', raw),
    },
}
print(json.dumps(data))
")

if [ -z "$JSON" ]; then
    echo "ERROR: failed to parse monad-status output" >&2
    exit 1
fi

HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$ENDPOINT" \
    -H "Content-Type: application/json" \
    -H "X-Monad-Token: $TOKEN" \
    -d "$JSON" \
    --max-time 10)

if [ "$HTTP_CODE" != "200" ]; then
    echo "ERROR: hosting returned HTTP $HTTP_CODE" >&2
    exit 1
fi

echo "OK: metrics pushed (HTTP $HTTP_CODE)"
```

Make it executable and test it:

```bash
sudo chmod +x /usr/local/bin/monad-push-metrics.sh
sudo /usr/local/bin/monad-push-metrics.sh
# Expected: OK: metrics pushed (HTTP 200)
```

---

## Step 4 - systemd service and timer

Create the service unit at `/etc/systemd/system/monad-push-metrics.service`:

```ini
[Unit]
Description=Push Monad node metrics to hosting
After=network.target monad-bft.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monad-push-metrics.sh
User=root
StandardOutput=journal
StandardError=journal
```

Create the timer unit at `/etc/systemd/system/monad-push-metrics.timer`:

```ini
[Unit]
Description=Run monad-push-metrics every 30 seconds
Requires=monad-push-metrics.service

[Timer]
OnBootSec=30s
OnUnitActiveSec=30s
AccuracySec=5s

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable monad-push-metrics.timer
sudo systemctl start monad-push-metrics.timer
sudo systemctl status monad-push-metrics.timer
```

Verify data is arriving:

```bash
# Wait 35 seconds after first start
curl -s https://your-domain.com/services/monad_testnet/metrics_cache.json | grep updated_at
```

---

## Step 5 - PHP receiver

Create `v1/metrics/index.php` on the hosting:

```php
<?php
// v1/metrics/index.php — Receiver for monad-status push

define('SECRET_TOKEN', 'your-secret-token-here');
define('CACHE_FILE',   __DIR__ . '/../../metrics_cache.json');

header('Content-Type: application/json');

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['error' => 'Method not allowed']);
    exit;
}

$token = $_SERVER['HTTP_X_MONAD_TOKEN'] ?? '';
if (!hash_equals(SECRET_TOKEN, $token)) {
    http_response_code(403);
    echo json_encode(['error' => 'Forbidden']);
    exit;
}

$raw = file_get_contents('php://input');
if (empty($raw)) {
    http_response_code(400);
    echo json_encode(['error' => 'Empty body']);
    exit;
}

$data = json_decode($raw, true);
if ($data === null) {
    http_response_code(400);
    echo json_encode(['error' => 'Invalid JSON']);
    exit;
}

$written = file_put_contents(CACHE_FILE, json_encode($data, JSON_PRETTY_PRINT), LOCK_EX);
if ($written === false) {
    http_response_code(500);
    echo json_encode(['error' => 'Cannot write cache']);
    exit;
}

http_response_code(200);
echo json_encode(['status' => 'ok']);
```

### File structure on hosting

```
monad_testnet/
├── components/
│   └── get_monad_metrics.php
├── v1/
│   ├── .htaccess
│   └── metrics/
│       └── index.php        ← receiver
├── layout.php
├── menu.php
├── metrics.php              ← dashboard
└── metrics_cache.json       ← written by receiver, read by dashboard
```

---

## Step 6 - Dashboard

`metrics.php` is a PHP page that loads `metrics_cache.json` via JavaScript fetch every 30 seconds and renders the data. It integrates with the existing site layout via `ob_start()` / `include('layout.php')`.

### Data displayed

| Section | Fields |
|---|---|
| Header | Node name, SECP public key |
| Status row | Node status, block height, block diff, epoch, round, mode |
| TrieDB | Disk used %, capacity, model |
| Services | monad-bft, monad-execution, monad-rpc, otelcol |
| Node info | Version, network, chain ID, uptime, RPC status |
| Statesync | Percentage, progress block, target block |
| Quick links | Monad Testnet, Docs, Explorer (with current block), Cumulo.pro |

### Staleness warning

If `metrics_cache.json` has not been updated in more than 2 minutes, a warning banner is shown automatically. This covers cases where the node or the push script is down.

### Auto-refresh

Both the systemd timer (node side) and the JS `setInterval` (dashboard side) run every **30 seconds**. The "Updated Xs ago" indicator in the header shows exact data age.

---

## Troubleshooting

### Push script returns HTTP 406

The hosting WAF is blocking the request. Common causes:

- **User-Agent** — some WAFs block non-browser User-Agents. Add `-H "User-Agent: Mozilla/5.0 (compatible; MonitorBot/1.0)"` to the curl call.
- **Content-Encoding: gzip** — if using OTEL as the push mechanism, set `compression: none` in the exporter config.
- **Payload size** — test with a minimal JSON body to isolate whether the WAF triggers on size or content type.

### Push script returns HTTP 403

Token mismatch. Verify that `SECRET_TOKEN` in `index.php` and `TOKEN` in the push script are identical, including any `=` padding characters.

### Push script returns HTTP 500

PHP error in the receiver. Add a minimal test `index.php` that just returns `{"status":"ok"}` to confirm PHP executes correctly, then restore the full receiver.

### metrics_cache.json not updating

Check the timer is active:

```bash
sudo systemctl status monad-push-metrics.timer
sudo journalctl -u monad-push-metrics --since "5 min ago"
```

### Dashboard shows stale data

The staleness banner appears after 120 seconds without an update. Check the timer on the node and verify the receiver file is writable by PHP.

---

## Notes and lessons learned

- **OTEL push vs pull**: The original approach was to use the OTEL Collector's `otlphttp` exporter to push metrics directly to the hosting. This was abandoned after extended debugging - shared hosting WAFs consistently blocked OTLP payloads (242KB, `Go-http-client` User-Agent) with HTTP 406 regardless of configuration. The `monad-status` approach is simpler, produces a ~400 byte payload, and avoids all WAF issues.

- **monad-status is the right tool**: The official `monad-status` script aggregates all operationally relevant data in a single call — consensus status, block height, epoch, round, services, TrieDB disk usage, statesync, and node config. There is no need to parse raw OTEL metrics for a status dashboard.

- **systemd timer vs cron**: A systemd timer was chosen over cron for consistency with the existing Monad service management pattern and better logging via `journalctl`.

- **Payload size matters on shared hosting**: Even with `compression: none`, the OTEL JSON payload was ~242KB per push. The `monad-status` JSON is ~400 bytes - three orders of magnitude smaller.

- **receiver path**: The PHP receiver must be at `v1/metrics/index.php` (as a directory index), not `v1/metrics.php`. Apache on shared hosting typically does not serve `.php` files at paths without the extension unless `mod_rewrite` is configured.

---

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [General Operations — monad-status](https://docs.monad.xyz/node-ops/general-operations#node-status-with-monad-status)
- [Full Node Installation](https://docs.monad.xyz/node-ops/full-node-installation)
- [Monad Node Announcements (Telegram)](https://t.me/MonadNodeAnnouncements)
- [Monad Testnet Explorer — MonadVision](https://testnet.monadvision.com)
- [Monad Testnet Explorer — MonadScan](https://testnet.monadscan.com)
