# Monad Testnet — Metrics Reference

> Reference documentation for all Prometheus/OTEL metrics exposed by the Monad node.  
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

### `monad_statesync_syncing`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | boolean (0 / 1) |
| Description | Whether the node is currently in statesync mode. `1` = syncing, `0` = live. |

Used inverted (`1 - monad_statesync_syncing`) to display node status as `LIVE` / `SYNCING`.

> ℹ️ A full node in normal operation shows `0` (not syncing). The value flips to `1` if the node falls behind and re-enters statesync.

---

### `monad_execution_ledger_block_num`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | block number (integer) |
| Description | Latest block number committed by the execution layer. |

Tracks the current chain head from the execution layer's perspective. Should increase continuously — a flat line indicates the node has stalled.

---

### `monad_state_consensus_events_commit_block`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | blocks (cumulative) |
| Description | Total number of blocks committed by the BFT consensus layer. |

Used as a rate: `rate(monad_state_consensus_events_commit_block[1m])` → **Blocks/s**.  
Expected steady value around 1 block/s under normal testnet conditions.

---

### `monad_execution_ledger_num_tx_commits`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | transactions (cumulative) |
| Description | Total number of transactions committed to the execution ledger. |

Used as a rate: `rate(monad_execution_ledger_num_tx_commits[1m])` → **TX commits/s**.  
Reflects actual throughput processed and finalized by the node.

---

### `monad_state_vote_delay_ready_after_timer_start_p50_ms` / `_p90_ms` / `_p99_ms`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | milliseconds |
| Description | Latency distribution between the round timer start and when the node is ready to vote. |

| Metric | Percentile | Meaning |
|---|---|---|
| `_p50_ms` | 50th | Median vote readiness delay |
| `_p90_ms` | 90th | Tail delay for 90% of rounds |
| `_p99_ms` | 99th | Worst-case delay for 99% of rounds |

> ℹ️ For a full node, this metric reflects observation latency, not actual voting. The WARN log `not voting on proposal, is not coherent` is expected — full nodes do not vote.

---

## ✅ Consensus Events Detail

### `monad_state_consensus_events_created_qc`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | events (cumulative) |
| Description | Total Quorum Certificates (QC) created. A QC is formed when a quorum of validators agrees on a block proposal. |

Used as a rate: `rate(...[1m])` → **QC/s**. Should roughly track block commit rate under normal operation.

---

### `monad_state_consensus_events_created_tc`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | events (cumulative) |
| Description | Total Timeout Certificates (TC) created. A TC is produced when a round times out without a QC. |

Used as a rate: `rate(...[1m])` → **TC/s**.  
Occasional TCs are normal; a sustained high rate indicates consensus instability or network issues.

---

### `monad_state_consensus_events_created_nec`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | events (cumulative) |
| Description | Total No-Extension Certificates (NEC) created. Signals that the round did not extend the chain. |

Used as a rate: `rate(...[1m])` → **NEC/s**. Rare under healthy network conditions.

---

### `monad_state_validation_errors_invalid_signature`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | errors (cumulative) |
| Description | Messages rejected due to invalid cryptographic signature. |

---

### `monad_state_validation_errors_invalid_epoch`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | errors (cumulative) |
| Description | Messages rejected because they belong to a different epoch. Can occur briefly during epoch transitions. |

---

### `monad_state_validation_errors_insufficient_stake`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | errors (cumulative) |
| Description | Messages rejected because the sender does not have sufficient stake to participate. |

---

### `monad_state_validation_errors_invalid_author`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | errors (cumulative) |
| Description | Messages rejected because the declared author is not recognized as a valid validator. |

> ℹ️ All validation error metrics are used as 5m rates. Isolated spikes are normal; sustained non-zero rates may indicate peering with a stale or misconfigured node.

---

## 📦 TxPool

### `monad_bft_txpool_pool_tracked_txs`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | transactions |
| Description | Number of transactions currently tracked in the mempool. |

Reflects pending transaction backlog. A continuously growing value may indicate block throughput is lower than incoming TX rate.

---

### `monad_bft_txpool_pool_tracked_addresses`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | addresses |
| Description | Number of unique sender addresses with at least one pending transaction in the pool. |

---

### `monad_bft_txpool_pool_drop_fee_too_low`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | drops (cumulative) |
| Description | Transactions dropped because their gas fee was below the pool's minimum threshold. |

---

### `monad_bft_txpool_pool_drop_pool_full`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | drops (cumulative) |
| Description | Transactions dropped because the mempool reached its maximum capacity. |

---

### `monad_bft_txpool_pool_drop_nonce_too_low`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | drops (cumulative) |
| Description | Transactions dropped because the nonce was already used (stale transaction). |

---

### `monad_bft_txpool_pool_drop_insufficient_balance`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | drops (cumulative) |
| Description | Transactions dropped because the sender's on-chain balance cannot cover value + gas. |

---

### `monad_bft_txpool_pool_drop_invalid_signature`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | drops (cumulative) |
| Description | Transactions dropped due to invalid ECDSA signature — likely malformed or tampered transactions. |

> ℹ️ All drop metrics are used as 5m rates. `fee_too_low` and `nonce_too_low` are the most common under normal network activity.

---

## 🌐 Peers & Network

### `monad_peer_disc_num_peers`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | peers |
| Description | Total number of peers currently known to the peer discovery layer. |

Healthy full nodes typically see 100–300+ peers on testnet. A drop to 0 indicates a network or peer discovery failure.

---

### `monad_peer_disc_num_upstream_validators`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | validators |
| Description | Number of active upstream validators the node is connected to for block delivery. |

> ℹ️ Full nodes receive blocks from upstream validators. A value ≥ 1 is required for the node to receive proposals. Values of 2–5 are typical.

---

### `monad_peer_disc_send_ping`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | pings (cumulative) |
| Description | Ping messages sent to peers for liveness checking. |

Used as a rate: `rate(...[1m])` → **Ping sent/s**.

---

### `monad_peer_disc_recv_pong`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | pongs (cumulative) |
| Description | Pong replies received from peers. |

Used as a rate: `rate(...[1m])` → **Pong recv/s**. `send_ping` and `recv_pong` rates should be close; a large gap indicates unresponsive peers.

---

### `monad_peer_disc_ping_timeout`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | timeouts (cumulative) |
| Description | Ping attempts that did not receive a pong reply within the timeout window. |

Used as a rate: `rate(...[1m])` → **Timeouts/s**. Occasional timeouts are expected; a sustained high rate suggests connectivity problems.

---

## 📡 RaptorCast & WireAuth

### `monad_bft_raptorcast_udp_secondary_broadcast_latency_p99_ms`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | milliseconds |
| Description | p99 latency for UDP broadcast via the RaptorCast secondary broadcast path. |

RaptorCast is Monad's erasure-coding-based block propagation protocol. The p99 latency reflects worst-case UDP broadcast performance. Values under 300 ms are normal on testnet.

---

### `monad_raptorcast_auth_authenticated_udp_bytes_read` / `_written`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | bytes (cumulative) |
| Description | Bytes read/written on authenticated UDP channels (WireAuth sessions). |

Used as a rate: `rate(...[1m])` → **bytes/s**. Authenticated channels carry block payloads from known validators.

---

### `monad_raptorcast_auth_non_authenticated_udp_bytes_read`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | bytes (cumulative) |
| Description | Bytes read on non-authenticated UDP channels. Covers peer discovery and unauthenticated broadcast traffic. |

---

### `monad_wireauth_udp_state_total_sessions`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | sessions |
| Description | Total WireAuth UDP sessions tracked, including established, initiating, and transport-layer sessions. |

---

### `monad_wireauth_udp_state_transport_sessions`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | sessions |
| Description | WireAuth sessions that have completed the handshake and are in active transport state. |

---

### `monad_wireauth_udp_state_initiating_sessions`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | sessions |
| Description | WireAuth sessions currently in the handshake initiation phase. |

A persistently high value relative to `transport_sessions` may indicate handshake failures with peers.

---

## 🌐 RPC

### `monad_rpc_active_requests`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | requests |
| Description | Number of RPC requests currently being processed. |

Reflects instantaneous RPC load. Spikes are expected under heavy query load; sustained high values may indicate slow upstream responses.

---

### `monad_rpc_request_duration_seconds_sum` / `_count`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | seconds / requests (cumulative) |
| Description | Total time and request count for RPC requests. Used together to compute average request duration. |

**PromQL:** `rate(..._sum[5m]) / rate(..._count[5m])` → **Avg request duration (s)**.

---

### `monad_rpc_execution_duration_seconds_sum` / `_count`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | seconds / requests (cumulative) |
| Description | Time spent in the execution layer per RPC request. Subset of total request duration. |

**PromQL:** `rate(..._sum[5m]) / rate(..._count[5m])` → **Avg execution duration (s)**.

> ℹ️ The difference between `request_duration` and `execution_duration` represents networking and serialization overhead in the RPC layer.

---

## 🔄 Statesync & BlockSync

### `monad_statesync_progress_estimate`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | block number |
| Description | Estimated current block number reached during statesync. |

---

### `monad_statesync_last_target`

| Field | Value |
|---|---|
| Type | Gauge |
| Unit | block number |
| Description | Target block number for the current statesync run. |

Used together to compute sync progress percentage:  
`(monad_statesync_progress_estimate / monad_statesync_last_target) * 100` → **Sync %**.

---

### `monad_state_blocksync_events_payload_response_successful`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | responses (cumulative) |
| Description | Successful block payload responses received during blocksync. |

Used as a rate: `rate(...[1m])` → **Payload OK/s**.

---

### `monad_state_blocksync_events_payload_response_failed`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | responses (cumulative) |
| Description | Failed block payload responses during blocksync (peer returned an error). |

Used as a rate: `rate(...[1m])` → **Payload fail/s**.

---

### `monad_state_blocksync_events_request_timeout`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | timeouts (cumulative) |
| Description | Block payload requests that timed out before receiving a response. |

Used as a rate: `rate(...[1m])` → **Timeouts/s**. Sustained timeouts during live operation may indicate peer connectivity issues or an overloaded upstream.

---

## ⏱ Node Lifecycle

### `monad_total_uptime_us`

| Field | Value |
|---|---|
| Type | Counter |
| Unit | microseconds (cumulative) |
| Description | Total time the node process has been running since last start. |

Converted to seconds in Grafana: `monad_total_uptime_us / 1000000`. Resets to 0 on service restart, making it useful for detecting unexpected restarts.

---

## PromQL Quick Reference

```promql
# Node live/syncing
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

# TxPool size
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

# Sync progress
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
