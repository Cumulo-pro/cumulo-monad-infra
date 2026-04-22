# Monad Testnet - Metrics Reference

> Reference documentation for all metrics exposed by the Monad node.  
> Metrics are collected via the OTEL Collector and exposed at `http://0.0.0.0:8889/metrics`.  
> All metrics are scoped to `job="monad_full_Cumulo-1"` in this node's configuration.

---

## Metric Sources

| Subsystem | Services |
|---|---|
| Consensus & block production | `monad-bft` |
| Execution & ledger | `monad-execution` |
| RPC | `monad-rpc` |
| Collection pipeline | `otelcol` → Prometheus scrape at `:8889/metrics` |

---

## ⛓ Consensus & Block Production

---

### `Node Status`

**Prometheus metric:** `monad_statesync_syncing`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | boolean (0 / 1) |

Whether the node is currently live or catching up with the chain. Grafana displays this inverted (`1 - monad_statesync_syncing`) so that `1 = LIVE` and `0 = SYNCING`. If the node falls behind, the value flips to `1` automatically until it recovers.

---

### `Block Height`

**Prometheus metric:** `monad_execution_ledger_block_num`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | block number (integer) |

The latest block number processed by the execution layer. This value should increase continuously — a flat line means the node has stalled.

---

### `Block Commits & TX Commits (rate)`

**Prometheus metrics:**
- `monad_state_consensus_events_commit_block` → Blocks/s
- `monad_execution_ledger_num_tx_commits` → TX commits/s

| Field | Value |
|---|---|
| Type | Counter (rate) |
| Unit | blocks/s and transactions/s |

The rate at which the node is processing blocks and transactions. Under normal testnet conditions, expect approximately **1 block/s**. The TX rate reflects the actual volume of transactions finalized by the node.

---

### `Vote Delay (p50 / p90 / p99)`

**Prometheus metrics:**
- `monad_state_vote_delay_ready_after_timer_start_p50_ms`
- `monad_state_vote_delay_ready_after_timer_start_p90_ms`
- `monad_state_vote_delay_ready_after_timer_start_p99_ms`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | milliseconds |

How long it takes the node to be ready to vote after the round timer starts, shown at three percentiles (median, 90th, and worst case). On a full node this reflects observation latency, not actual voting — the log message `not voting on proposal, is not coherent` is completely expected.

---

## ✅ Consensus Events

---

### `QC / TC / NEC Created (rate)`

**Prometheus metrics:**
- `monad_state_consensus_events_created_qc` → QC/s
- `monad_state_consensus_events_created_tc` → TC/s
- `monad_state_consensus_events_created_nec` → NEC/s

| Field | Value |
|---|---|
| Type | Counter (rate) |
| Unit | certificates/s |

Three types of consensus certificates:

- **QC (Quorum Certificate):** Formed when enough validators agree on a block proposal. Its rate should closely track the block commit rate — this is the happy path of consensus.
- **TC (Timeout Certificate):** Produced when a round expires without reaching a QC. Occasional TCs are normal; a sustained high rate indicates consensus instability or network issues.
- **NEC (No-Extension Certificate):** Signals the round did not extend the chain. Should be rare under healthy conditions.

---

### `Validation Errors (rate)`

**Prometheus metrics:**
- `monad_state_validation_errors_invalid_signature`
- `monad_state_validation_errors_invalid_epoch`
- `monad_state_validation_errors_insufficient_stake`
- `monad_state_validation_errors_invalid_author`

| Field | Value |
|---|---|
| Type | Counter (5m rate) |
| Unit | errors/s |

Messages rejected by the node during consensus validation:

- **invalid_signature:** Cryptographic signature is incorrect.
- **invalid_epoch:** The message belongs to a different epoch — can occur briefly during epoch transitions.
- **insufficient_stake:** The sender does not have enough stake to participate.
- **invalid_author:** The declared author is not a recognized validator.

Isolated spikes are normal. A sustained non-zero rate may indicate the node is peered with a stale or misconfigured peer.

---

## 📦 TxPool (Mempool)

---

### `TxPool - Tracked TXs & Addresses`

**Prometheus metrics:**
- `monad_bft_txpool_pool_tracked_txs`
- `monad_bft_txpool_pool_tracked_addresses`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | transactions / addresses |

The current state of the mempool: how many transactions are queued and from how many unique sender addresses. If the TX count keeps growing without stabilizing, the node is not processing blocks fast enough to keep up with incoming transaction volume.

---

### `TxPool - Drop Reasons (rate)`

**Prometheus metrics:**
- `monad_bft_txpool_pool_drop_fee_too_low`
- `monad_bft_txpool_pool_drop_pool_full`
- `monad_bft_txpool_pool_drop_nonce_too_low`
- `monad_bft_txpool_pool_drop_insufficient_balance`
- `monad_bft_txpool_pool_drop_invalid_signature`

| Field | Value |
|---|---|
| Type | Counter (5m rate) |
| Unit | drops/s |

Transactions rejected before entering the mempool, broken down by reason:

- **fee_too_low:** Gas fee is below the pool's minimum threshold.
- **pool_full:** Mempool is at capacity — the incoming transaction is discarded to make room.
- **nonce_too_low:** The nonce has already been used (stale or duplicate transaction).
- **insufficient_balance:** The sender's balance cannot cover value + gas.
- **invalid_signature:** Invalid ECDSA signature — likely a malformed or tampered transaction.

`fee_too_low` and `nonce_too_low` are the most common under normal testnet activity.

---

## 🌐 Peers & Network

---

### `Peers`

**Prometheus metric:** `monad_peer_disc_num_peers`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | peers |

Total number of peers known to the peer discovery layer. A healthy full node should see between **100 and 300+ peers** on testnet. A drop to 0 indicates a network or peer discovery failure.

---

### `Upstream Validators`

**Prometheus metric:** `monad_peer_disc_num_upstream_validators`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | validators |

Number of active upstream validators the node is connected to for receiving blocks. At least **1** is required for the node to receive block proposals. Typical values are between 2 and 5.

---

### `Peer Discovery - Ping/Pong`

**Prometheus metrics:**
- `monad_peer_disc_send_ping` → Pings sent/s
- `monad_peer_disc_recv_pong` → Pongs received/s
- `monad_peer_disc_ping_timeout` → Timeouts/s

| Field | Value |
|---|---|
| Type | Counter (rate) |
| Unit | messages/s |

Health of peer communication. The ping sent and pong received rates should be close to each other — a large gap indicates peers that are not responding. The timeout rate should stay low; a sustained high value suggests connectivity problems.

---

## 📡 RaptorCast & WireAuth

---

### `RaptorCast - UDP Broadcast Latency p99`

**Prometheus metric:** `monad_bft_raptorcast_udp_secondary_broadcast_latency_p99_ms`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | milliseconds |

The p99 latency of RaptorCast's UDP broadcast path. RaptorCast is Monad's erasure-coding-based block propagation protocol. This value reflects the worst-case UDP broadcast performance. Values under **300 ms** are normal on testnet.

---

### `UDP Bytes - Authenticated vs Non-Authenticated`

**Prometheus metrics:**
- `monad_raptorcast_auth_authenticated_udp_bytes_read` → authenticated bytes read/s
- `monad_raptorcast_auth_authenticated_udp_bytes_written` → authenticated bytes written/s
- `monad_raptorcast_auth_non_authenticated_udp_bytes_read` → non-authenticated bytes read/s

| Field | Value |
|---|---|
| Type | Counter (rate) |
| Unit | bytes/s |

UDP traffic volume split by authentication status. Authenticated channels carry block payloads from known validators. Non-authenticated traffic covers peer discovery and general broadcast.

---

### `WireAuth Sessions`

**Prometheus metrics:**
- `monad_wireauth_udp_state_total_sessions`
- `monad_wireauth_udp_state_transport_sessions`
- `monad_wireauth_udp_state_initiating_sessions`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | sessions |

State of the node's WireAuth UDP sessions (the authenticated transport layer):

- **total_sessions:** All known sessions regardless of state.
- **transport_sessions:** Sessions that completed the handshake and are actively transferring data — this is the meaningful value.
- **initiating_sessions:** Sessions currently in the handshake phase. A persistently high value relative to `transport_sessions` may indicate handshake failures with peers.

---

## 🌐 RPC

---

### `RPC - Active Requests`

**Prometheus metric:** `monad_rpc_active_requests`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | requests |

Number of RPC requests currently being processed. Spikes are expected under heavy query load; a sustained high value may indicate slow upstream responses.

---

### `RPC - Request Duration (avg)`

**Prometheus metrics:**
- `monad_rpc_request_duration_seconds_sum` / `_count` → avg total request duration (s)
- `monad_rpc_execution_duration_seconds_sum` / `_count` → avg execution layer duration (s)

| Field | Value |
|---|---|
| Type | Counter (rate ratio) |
| Unit | seconds |

Average latency of RPC requests, shown as two values:
- **Request duration:** Total time from when the request arrives to when the response is sent.
- **Execution duration:** The portion of that time spent in the execution layer. The difference between the two represents networking and serialization overhead in the RPC server.

---

## 🔄 Statesync & BlockSync

---

### `Statesync Progress`

**Prometheus metrics:**
- `monad_statesync_progress_estimate` → current sync block
- `monad_statesync_last_target` → target sync block

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | block number → displayed as percentage in Grafana |

Progress of the initial sync. Grafana computes `(progress / target) * 100` to show the percentage completed. Only relevant while the node is in `SYNCING` mode.

---

### `BlockSync - Payload Requests`

**Prometheus metrics:**
- `monad_state_blocksync_events_payload_response_successful` → OK/s
- `monad_state_blocksync_events_payload_response_failed` → failures/s
- `monad_state_blocksync_events_request_timeout` → timeouts/s

| Field | Value |
|---|---|
| Type | Counter (rate) |
| Unit | responses/s |

Health of the block download process during blocksync. Most responses should be successful. A sustained rate of failures or timeouts during normal operation (outside of sync) may indicate peer connectivity issues or an overloaded upstream.

---

## ⏱ Node Lifecycle

---

### `Uptime`

**Prometheus metric:** `monad_total_uptime_us`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | microseconds → seconds in Grafana (÷ 1,000,000) |

Total time the node process has been running since its last start. Resets to 0 on every service restart, making it easy to detect unexpected restarts — a sudden drop in this value means the node restarted.

---

## PromQL Quick Reference

```promql
# Node status (LIVE=1 / SYNCING=0)
1 - monad_statesync_syncing{job="monad_full_Cumulo-1"}

# Block height
monad_execution_ledger_block_num{job="monad_full_Cumulo-1"}

# Block throughput
rate(monad_state_consensus_events_commit_block{job="monad_full_Cumulo-1"}[1m])

# TX throughput
rate(monad_execution_ledger_num_tx_commits{job="monad_full_Cumulo-1"}[1m])

# Vote delay p99
monad_state_vote_delay_ready_after_timer_start_p99_ms{job="monad_full_Cumulo-1"}

# Peers connected
monad_peer_disc_num_peers{job="monad_full_Cumulo-1"}

# Mempool size
monad_bft_txpool_pool_tracked_txs{job="monad_full_Cumulo-1"}

# TX drop rate (all reasons)
rate(monad_bft_txpool_pool_drop_fee_too_low{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_pool_full{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_nonce_too_low{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_insufficient_balance{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_invalid_signature{job="monad_full_Cumulo-1"}[5m])

# RPC avg latency
rate(monad_rpc_request_duration_seconds_sum{job="monad_full_Cumulo-1"}[5m])
/ rate(monad_rpc_request_duration_seconds_count{job="monad_full_Cumulo-1"}[5m])

# Sync progress (%)
(monad_statesync_progress_estimate{job="monad_full_Cumulo-1"}
/ monad_statesync_last_target{job="monad_full_Cumulo-1"}) * 100

# Uptime (seconds)
monad_total_uptime_us{job="monad_full_Cumulo-1"} / 1000000
```

---

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [OTEL Collector](https://opentelemetry.io/docs/collector/)
- [Grafana Dashboard — Monad Node Monitoring](../grafana/monad-grafana-dashboard-multinode.json)
