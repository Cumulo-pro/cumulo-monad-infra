# Monad Testnet — Grafana & Prometheus Monitoring Setup

> Documentation of the monitoring stack integration for the Monad full node.  
> The node already exposes all metrics via the OTEL Collector. This document covers
> connecting an existing Prometheus + Grafana server to that endpoint and importing
> the Cumulo dashboard.

---

## Architecture

The monitoring pipeline relies on the OTEL Collector already running on the node,
which exports all Monad metrics in Prometheus format at port `8889`:

```
monad-bft / monad-execution / monad-rpc
       ↓ OTLP
OTEL Collector  (running on the node)
       ↓ Prometheus exporter → :8889/metrics
Prometheus      (external server — 54.39.128.229:9099)
       ↓ scrape every 15s
Grafana         (same external server)
```

No additional exporters or agents are required on the node.

---

## Prerequisites

| Component | Location | Notes |
|---|---|---|
| OTEL Collector | Node (`192.155.100.132`) | Already running, exposes `:8889/metrics` |
| Prometheus | External server (`54.39.128.229`) | Listening on port `9099` |
| Grafana | External server (`54.39.128.229`) | Version 12.3.2 |

---

## Step 1 — Open port 8889 on the node

The OTEL metrics endpoint must be reachable only from the Prometheus server.
`ufw` was not pre-installed — install it first, then add the rule:

```bash
# On the Monad node
sudo apt update && sudo apt install -y ufw

sudo ufw allow from 54.39.128.229 to any port 8889
sudo ufw enable
sudo ufw status | grep 8889
```

Expected output:

```
8889                       ALLOW       54.39.128.229
```

Verify the metrics endpoint is reachable from the node itself:

```bash
curl -s http://localhost:8889/metrics | head -5
```

A response starting with `# HELP` and `# TYPE` lines confirms the OTEL Collector
is exporting correctly.

---

## Step 2 — Add the Monad job to Prometheus

On the external Prometheus server, edit `/etc/prometheus/prometheus.yml` and add
the following job at the end of `scrape_configs`:

```yaml
- job_name: monad_full_Cumulo-1
  static_configs:
  - targets: ['192.155.100.132:8889']
```

Reload Prometheus without restarting:

```bash
sudo systemctl reload prometheus
```

If `reload` is not supported:

```bash
curl -X POST http://localhost:9099/-/reload
```

---

## Step 3 — Verify the target is UP

Prometheus on this server listens on port `9099`. Check the scrape status:

```bash
curl -s "http://localhost:9099/api/v1/targets?state=active" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for t in data['data']['activeTargets']:
    if 'monad' in t['scrapePool']:
        print(t['scrapePool'], '->', t['health'], t.get('lastError',''))
"
```

Expected output:

```
monad_full_Cumulo-1 -> up
```

---

## Step 4 — Import the Grafana dashboard

The Cumulo dashboard JSON is available in this repository:

**[`grafana/monad-grafana-dashboard-multinode.json`](../grafana/monad-grafana-dashboard-multinode.json)**

Import it from the command line on the Grafana server:

```bash
curl -s -X POST http://admin:YOUR_PASSWORD@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d "{\"dashboard\": $(cat grafana/monad-grafana-dashboard-multinode.json), \"overwrite\": true, \"folderId\": 0}"
```

Alternatively, import via the Grafana UI: **Dashboards → Import → Upload JSON file**.

> The dashboard datasource variable is pre-configured as `prometheus`. If your
> datasource has a different name, update it in **Dashboard settings → Variables**
> after importing.

---

## Dashboard Overview

**Title:** Monad Node Monitoring  
**Refresh:** 30 seconds  
**Default time range:** Last 3 hours  
**Node selector:** dropdown variable populated from `label_values(monad_execution_ledger_block_num, job)` — supports multiple nodes in the same Grafana instance.

### Panels

| Panel | Type | Description |
|---|---|---|
| Node Status | Stat | LIVE / SYNCING state (`1 - monad_statesync_syncing`) |
| Block Height | Stat | Latest block committed by the execution layer |
| Peers | Stat | Total peers known to peer discovery |
| Upstream Validators | Stat | Active upstream validators connected |
| Uptime | Stat | Node process uptime in seconds |
| Block Commits & TX Commits (rate) | Time series | Blocks/s and transactions/s |
| Vote Delay (p50 / p90 / p99) | Time series | Round vote readiness latency in ms |
| TxPool — Tracked TXs & Addresses | Time series | Mempool size and unique sender addresses |
| TxPool — Drop Reasons (rate) | Time series | TX drops by reason (fee, nonce, balance, full, signature) |
| RaptorCast — UDP Broadcast Latency p99 | Time series | Worst-case block propagation latency in ms |
| UDP Bytes — Authenticated vs Non-Authenticated | Time series | RaptorCast traffic volume by auth status |
| Peer Discovery — Ping/Pong | Time series | Ping sent / pong received / timeouts rate |
| WireAuth Sessions | Time series | Total, transport, and initiating UDP sessions |
| RPC — Active Requests | Time series | Instantaneous RPC load |
| RPC — Request Duration (avg) | Time series | Avg total and execution duration per request |
| Statesync Progress | Gauge | Sync completion percentage (only relevant during SYNCING) |
| BlockSync — Payload Requests | Time series | Successful, failed, and timed-out block payload responses |
| QC / TC / NEC Created (rate) | Time series | Consensus certificate rates |
| Validation Errors (rate) | Time series | Consensus message rejection reasons |

For a full description of each metric and its Prometheus source, see
[`05-monad-metrics-reference.md`](05-monad-metrics-reference.md).

---

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [OTEL Collector](https://opentelemetry.io/docs/collector/)
- [Grafana Dashboard JSON](../grafana/monad-grafana-dashboard-multinode.json)
- [Metrics Reference](05-monad-metrics-reference.md)
