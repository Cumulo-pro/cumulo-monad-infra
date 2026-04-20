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

## Documentation

| Doc | Description |
|---|---|
| [01 — Validator setup](docs/01-validator-setup.md) | Hardware provisioning, disk layout, OS config, TrieDB init |
| [02 — Full node installation](docs/02-fullnode-installation.md) | Dependencies, firewall, keystore generation, snapshot import, sync |
| [03 — Node monitoring](docs/03-monad-node-monitoring.md) | Monitoring stack: monad-status, push pipeline, PHP receiver, public dashboard |

## Status

Node committing live blocks on Monad testnet since April 2026.

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
