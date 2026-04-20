# Monad Testnet — Metrics Reference

> Documentación de referencia de todas las métricas expuestas por el nodo Monad.  
> Las métricas se recogen vía OTEL Collector y se exponen en `http://0.0.0.0:8889/metrics`.  
> Todas las métricas usan el scope `job="monad_full_Cumulo-1"` en la configuración de este nodo.

---

## Fuentes de métricas

| Subsistema | Servicios |
|---|---|
| Consenso y producción de bloques | `monad-bft` |
| Ejecución y ledger | `monad-execution` |
| RPC | `monad-rpc` |
| Pipeline de recolección | `otelcol` → Prometheus scrape en `:8889/metrics` |

---

## ⛓ Consenso y Producción de Bloques

---

### `Node Status`

**Métrica Prometheus:** `monad_statesync_syncing`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | booleano (0 / 1) |

El nodo tiene dos estados posibles: **LIVE** (operando con normalidad, valor `0`) o **SYNCING** (resincronizando con la cadena, valor `1`). En Grafana se muestra invertido (`1 - monad_statesync_syncing`) para que `1 = LIVE`. Si el nodo se queda atrás respecto a la cadena, el valor vuelve a `1` automáticamente hasta que se recupere.

---

### `Block Height`

**Métrica Prometheus:** `monad_execution_ledger_block_num`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | número de bloque (entero) |

Número del último bloque procesado por la capa de ejecución. Debe subir de forma continua — si la línea se queda plana, el nodo ha dejado de avanzar.

---

### `Block Commits & TX Commits (rate)`

**Métricas Prometheus:**
- `monad_state_consensus_events_commit_block` → Bloques/s
- `monad_execution_ledger_num_tx_commits` → TX commits/s

| Campo | Valor |
|---|---|
| Tipo | Counter (tasa) |
| Unidad | bloques/s y transacciones/s |

Muestra el ritmo al que el nodo está procesando bloques y transacciones. En condiciones normales de testnet se espera aproximadamente **1 bloque/s**. La tasa de TX refleja el volumen real de transacciones finalizadas por el nodo.

---

### `Vote Delay (p50 / p90 / p99)`

**Métricas Prometheus:**
- `monad_state_vote_delay_ready_after_timer_start_p50_ms`
- `monad_state_vote_delay_ready_after_timer_start_p90_ms`
- `monad_state_vote_delay_ready_after_timer_start_p99_ms`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | milisegundos |

Tiempo que tarda el nodo en estar listo para votar desde que arranca el temporizador de ronda, medido en percentiles (mediana, 90% y peor caso). En un nodo completo (*full node*) este valor refleja la latencia de observación, no el voto real — el log `not voting on proposal, is not coherent` es completamente normal.

---

## ✅ Eventos de Consenso

---

### `QC / TC / NEC Created (rate)`

**Métricas Prometheus:**
- `monad_state_consensus_events_created_qc` → QC/s
- `monad_state_consensus_events_created_tc` → TC/s
- `monad_state_consensus_events_created_nec` → NEC/s

| Campo | Valor |
|---|---|
| Tipo | Counter (tasa) |
| Unidad | certificados/s |

Tres tipos de certificados de consenso:

- **QC (Quorum Certificate):** Se forma cuando suficientes validadores coinciden en una propuesta de bloque. Su tasa debería seguir de cerca la tasa de bloques — es el camino feliz del consenso.
- **TC (Timeout Certificate):** Se produce cuando una ronda expira sin alcanzar QC. Alguno ocasional es normal; una tasa sostenida indica inestabilidad en el consenso o problemas de red.
- **NEC (No-Extension Certificate):** Indica que la ronda no extendió la cadena. Debe ser raro en condiciones saludables.

---

### `Validation Errors (rate)`

**Métricas Prometheus:**
- `monad_state_validation_errors_invalid_signature`
- `monad_state_validation_errors_invalid_epoch`
- `monad_state_validation_errors_insufficient_stake`
- `monad_state_validation_errors_invalid_author`

| Campo | Valor |
|---|---|
| Tipo | Counter (tasa a 5m) |
| Unidad | errores/s |

Mensajes rechazados por el nodo durante la validación de consenso:

- **invalid_signature:** Firma criptográfica incorrecta.
- **invalid_epoch:** El mensaje pertenece a una época distinta — puede ocurrir brevemente durante transiciones de época.
- **insufficient_stake:** El remitente no tiene stake suficiente para participar.
- **invalid_author:** El autor declarado no es un validador reconocido.

Picos aislados son normales. Una tasa no nula sostenida puede indicar que el nodo está conectado a un peer obsoleto o mal configurado.

---

## 📦 TxPool (Mempool)

---

### `TxPool — Tracked TXs & Addresses`

**Métricas Prometheus:**
- `monad_bft_txpool_pool_tracked_txs`
- `monad_bft_txpool_pool_tracked_addresses`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | transacciones / direcciones |

Muestra el estado actual del mempool: cuántas transacciones están en cola y desde cuántas direcciones únicas provienen. Si el número de TXs crece sin parar, el nodo no está procesando bloques a la misma velocidad que llegan las transacciones.

---

### `TxPool — Drop Reasons (rate)`

**Métricas Prometheus:**
- `monad_bft_txpool_pool_drop_fee_too_low`
- `monad_bft_txpool_pool_drop_pool_full`
- `monad_bft_txpool_pool_drop_nonce_too_low`
- `monad_bft_txpool_pool_drop_insufficient_balance`
- `monad_bft_txpool_pool_drop_invalid_signature`

| Campo | Valor |
|---|---|
| Tipo | Counter (tasa a 5m) |
| Unidad | drops/s |

Transacciones rechazadas antes de entrar al mempool, clasificadas por motivo:

- **fee_too_low:** La comisión de gas es inferior al mínimo aceptado por el pool.
- **pool_full:** El mempool está lleno — la TX más nueva se descarta para dejar sitio.
- **nonce_too_low:** El nonce ya fue usado (transacción obsoleta o repetida).
- **insufficient_balance:** El saldo del remitente no cubre el valor + gas de la transacción.
- **invalid_signature:** La firma ECDSA es inválida — transacción malformada o manipulada.

`fee_too_low` y `nonce_too_low` son los más habituales en actividad normal de testnet.

---

## 🌐 Peers y Red

---

### `Peers`

**Métrica Prometheus:** `monad_peer_disc_num_peers`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | peers |

Número total de peers conocidos por la capa de descubrimiento. Un nodo completo saludable debería ver entre **100 y 300+ peers** en testnet. Una caída a 0 indica un fallo de red o de descubrimiento de peers.

---

### `Upstream Validators`

**Métrica Prometheus:** `monad_peer_disc_num_upstream_validators`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | validadores |

Número de validadores activos a los que el nodo está conectado para recibir bloques. Se necesita **al menos 1** para que el nodo reciba propuestas de bloque. Los valores habituales oscilan entre 2 y 5.

---

### `Peer Discovery — Ping/Pong`

**Métricas Prometheus:**
- `monad_peer_disc_send_ping` → Pings enviados/s
- `monad_peer_disc_recv_pong` → Pongs recibidos/s
- `monad_peer_disc_ping_timeout` → Timeouts/s

| Campo | Valor |
|---|---|
| Tipo | Counter (tasa) |
| Unidad | mensajes/s |

Muestra la salud de la comunicación con los peers. Las tasas de ping enviado y pong recibido deben ser similares — una diferencia grande indica peers que no responden. La tasa de timeouts debe ser baja; un valor sostenido alto sugiere problemas de conectividad.

---

## 📡 RaptorCast y WireAuth

---

### `RaptorCast — UDP Broadcast Latency p99`

**Métrica Prometheus:** `monad_bft_raptorcast_udp_secondary_broadcast_latency_p99_ms`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | milisegundos |

Latencia en el percentil 99 del broadcast UDP de RaptorCast, el protocolo de propagación de bloques de Monad basado en erasure coding. Refleja el peor caso de rendimiento en la difusión de bloques por UDP. Valores por debajo de **300 ms** son normales en testnet.

---

### `UDP Bytes — Authenticated vs Non-Authenticated`

**Métricas Prometheus:**
- `monad_raptorcast_auth_authenticated_udp_bytes_read` → bytes autenticados leídos/s
- `monad_raptorcast_auth_authenticated_udp_bytes_written` → bytes autenticados escritos/s
- `monad_raptorcast_auth_non_authenticated_udp_bytes_read` → bytes no autenticados leídos/s

| Campo | Valor |
|---|---|
| Tipo | Counter (tasa) |
| Unidad | bytes/s |

Volumen de tráfico UDP desglosado por autenticación. Los canales autenticados transportan los payloads de bloques desde validadores conocidos. Los no autenticados corresponden al descubrimiento de peers y broadcast general.

---

### `WireAuth Sessions`

**Métricas Prometheus:**
- `monad_wireauth_udp_state_total_sessions`
- `monad_wireauth_udp_state_transport_sessions`
- `monad_wireauth_udp_state_initiating_sessions`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | sesiones |

Estado de las sesiones WireAuth (protocolo de autenticación UDP del nodo):

- **total_sessions:** Todas las sesiones conocidas, en cualquier estado.
- **transport_sessions:** Sesiones que completaron el handshake y están activas — este es el valor útil.
- **initiating_sessions:** Sesiones en proceso de handshake. Un valor persistentemente alto respecto a `transport_sessions` puede indicar fallos de handshake con los peers.

---

## 🌐 RPC

---

### `RPC — Active Requests`

**Métrica Prometheus:** `monad_rpc_active_requests`

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | peticiones |

Número de peticiones RPC procesándose en este momento. Los picos son normales bajo alta carga; un valor alto sostenido puede indicar respuestas lentas del upstream.

---

### `RPC — Request Duration (avg)`

**Métricas Prometheus:**
- `monad_rpc_request_duration_seconds_sum` / `_count` → Duración media total de la petición (s)
- `monad_rpc_execution_duration_seconds_sum` / `_count` → Duración media en la capa de ejecución (s)

| Campo | Valor |
|---|---|
| Tipo | Counter (ratio de tasas) |
| Unidad | segundos |

Latencia media de las peticiones RPC. Se muestran dos valores:
- **Request duration:** Tiempo total desde que llega la petición hasta que se envía la respuesta.
- **Execution duration:** Porción de ese tiempo gastada en la capa de ejecución. La diferencia entre ambos representa el overhead de red y serialización del servidor RPC.

---

## 🔄 Statesync y BlockSync

---

### `Statesync Progress`

**Métricas Prometheus:**
- `monad_statesync_progress_estimate` → bloque actual de sync
- `monad_statesync_last_target` → bloque objetivo de sync

| Campo | Valor |
|---|---|
| Tipo | Gauge |
| Unidad | número de bloque → porcentaje en Grafana |

Progreso de la sincronización inicial. Grafana calcula `(progreso / objetivo) * 100` para mostrar el porcentaje completado. Solo es relevante cuando el nodo está en modo `SYNCING`.

---

### `BlockSync — Payload Requests`

**Métricas Prometheus:**
- `monad_state_blocksync_events_payload_response_successful` → OK/s
- `monad_state_blocksync_events_payload_response_failed` → fallos/s
- `monad_state_blocksync_events_request_timeout` → timeouts/s

| Campo | Valor |
|---|---|
| Tipo | Counter (tasa) |
| Unidad | respuestas/s |

Salud del proceso de descarga de bloques durante el blocksync. La mayor parte de respuestas deben ser exitosas. Una tasa sostenida de fallos o timeouts durante la operación normal (no durante sync) puede indicar peers con problemas de conectividad o sobrecarga en el upstream.

---

## ⏱ Ciclo de Vida del Nodo

---

### `Uptime`

**Métrica Prometheus:** `monad_total_uptime_us`

| Campo | Valor |
|---|---|
| Tipo | Counter |
| Unidad | microsegundos → segundos en Grafana (÷ 1 000 000) |

Tiempo total que lleva en ejecución el proceso del nodo desde el último arranque. Se resetea a 0 en cada reinicio del servicio, lo que permite detectar reinicios inesperados — si el valor baja de golpe, el nodo se reinició.

---

## PromQL — Referencia rápida

```promql
# Estado del nodo (LIVE=1 / SYNCING=0)
1 - monad_statesync_syncing{job="monad_full_Cumulo-1"}

# Altura de bloque
monad_execution_ledger_block_num{job="monad_full_Cumulo-1"}

# Bloques por segundo
rate(monad_state_consensus_events_commit_block{job="monad_full_Cumulo-1"}[1m])

# Transacciones por segundo
rate(monad_execution_ledger_num_tx_commits{job="monad_full_Cumulo-1"}[1m])

# Vote delay p99
monad_state_vote_delay_ready_after_timer_start_p99_ms{job="monad_full_Cumulo-1"}

# Peers conectados
monad_peer_disc_num_peers{job="monad_full_Cumulo-1"}

# Tamaño del mempool
monad_bft_txpool_pool_tracked_txs{job="monad_full_Cumulo-1"}

# Tasa de drops del mempool (todos los motivos)
rate(monad_bft_txpool_pool_drop_fee_too_low{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_pool_full{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_nonce_too_low{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_insufficient_balance{job="monad_full_Cumulo-1"}[5m])
+ rate(monad_bft_txpool_pool_drop_invalid_signature{job="monad_full_Cumulo-1"}[5m])

# Latencia media RPC
rate(monad_rpc_request_duration_seconds_sum{job="monad_full_Cumulo-1"}[5m])
/ rate(monad_rpc_request_duration_seconds_count{job="monad_full_Cumulo-1"}[5m])

# Progreso de sync (%)
(monad_statesync_progress_estimate{job="monad_full_Cumulo-1"}
/ monad_statesync_last_target{job="monad_full_Cumulo-1"}) * 100

# Uptime (segundos)
monad_total_uptime_us{job="monad_full_Cumulo-1"} / 1000000
```

---

## Referencias

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [OTEL Collector](https://opentelemetry.io/docs/collector/)
- [Grafana Dashboard — Monad Node Monitoring](../grafana/monad-grafana-dashboard-multinode.json)
