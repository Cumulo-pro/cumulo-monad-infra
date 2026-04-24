# cumulo-monad-infra

Infrastructure documentation for Cumulo's Monad testnet validator node.

## Node Identity

| Field | Value |
|---|---|
| Node name | `full_Cumulo-1` |
| Network | `monad_testnet` |
| Monad version | `v0.14.1` |
| SECP public key | `02ab335bea8131ddd8de77b7bea483313d1278b7c533f4ff48e5f8393370c924b9` |

## Live Dashboard

Real-time node metrics available at:  
**[cumulo.pro/services/monad_testnet/metrics.php](https://cumulo.pro/services/monad_testnet/metrics.php)**

Updates every 30 seconds via a systemd timer push from the node.

Grafana dashboard (Prometheus-based, 19 panels):  
**[grafana/monad-grafana-dashboard-multinode.json](grafana/monad-grafana-dashboard-multinode.json)**

## Documentation

| Doc | Description |
|---|---|
| [01 — Validator setup](docs/01-validator-setup.md) | Hardware provisioning, disk layout, OS config, TrieDB init |
| [02 — Full node installation](docs/02-fullnode-installation.md) | Dependencies, firewall, keystore generation, snapshot import, sync |
| [03 — Node monitoring](docs/03-monad-node-monitoring.md) | Monitoring stack: monad-status, push pipeline, PHP receiver, public dashboard |
| [04 — Grafana & Prometheus](docs/04-grafana-prometheus.md) | Prometheus scrape setup, firewall config, Grafana dashboard import |
| [05 — Metrics reference](docs/05-monad-metrics-reference.md) | All Prometheus metrics exposed by the node, mapped to Grafana panels |
| [06 — CLI reference](docs/06-cli-reference.md) | CLI commands for day-to-day node management: services, logs, RPC, keys, recovery, upgrades |
| [07 — Monlog](docs/07-monlog.md) | BFT log analyser: installation, usage, output reference and relation to Prometheus |

## Repository Structure

```
cumulo-monad-infra/
├── docs/
│   ├── 01-validator-setup.md
│   ├── 02-fullnode-installation.md
│   ├── 03-monad-node-monitoring.md
│   ├── 04-grafana-prometheus.md
│   ├── 05-monad-metrics-reference.md
│   ├── 06-cli-reference.md
│   └── 07-monlog.md
├── grafana/
│   └── monad-grafana-dashboard-multinode.json
└── README.md
```

## Status

Node committing live blocks on Monad testnet since April 2026.

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
