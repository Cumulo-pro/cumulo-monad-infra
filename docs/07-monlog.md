# Monad Testnet — Monlog

> Monlog es una herramienta de diagnóstico para nodos Monad que extrae y resume los logs de journald de los últimos 60 segundos.  
> No tiene daemon, no corre en background y no abre puertos — se ejecuta puntualmente y termina.  
> Binario descargado en `/home/monad/monlog`.

---

## Prerrequisitos

El usuario `monad` necesita acceso al grupo `systemd-journal` para poder leer los logs del sistema.

Ejecutar como root o con sudo:

```bash
usermod -a -G systemd-journal monad
```

> ℹ️ El cambio de grupo requiere reconectar la sesión para que tome efecto. Si se usa `su - monad` directamente después del `usermod`, el grupo no estará activo todavía.

---

## Instalación

Cambiar al usuario monad:

```bash
sudo su - monad
```

Descargar el binario:

```bash
cd /home/monad
curl -sSL https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev/scripts/monlog -O
chmod u+x ./monlog
```

Verificar el checksum:

```bash
sha256sum ./monlog
```

Checksum documentado en la [documentación oficial de Monad](https://docs.monad.xyz/node-ops/general-operations):

```
f8a1066d8c093bbdb5f722f5b3d89904c17a76fa01fa1dd62e950f445e88327f
```

> ⚠️ El binario puede ser actualizado por Monad sin que la documentación oficial se actualice simultáneamente. Si el checksum no coincide, verificar en el canal `#build_anything` de Discord antes de ejecutar. El checksum real del binario actualmente descargado es:
> ```
> 9ea3c58df6d390e7ba73832c330a077d91f49102274b3b95ecbcbc297573bc22
> ```
> Confirmado como válido por el equipo de Monad (actualización reciente del binario).

---

## Uso

### Ejecución básica

```bash
./monlog
```

Analiza los últimos 60 segundos de logs de journald y muestra un resumen estructurado por secciones.

### Modo watch (monitorización continua)

```bash
watch -d "./monlog"
```

Ejecuta el binario cada 2 segundos y resalta los cambios en pantalla. El impacto en el sistema es negligible.

### Flag `-n` (mapeo de validadores)

```bash
./monlog -n
```

Muestra nombres de validadores mapeados a sus SECP keys. Útil para identificar quién propone cada bloque. Prometheus/OTEL solo expone la key cruda — este flag añade contexto legible.

---

## Interpretación del output

Ejemplo de output en nodo sincronizado y operativo:

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

### Referencia de secciones

| Sección | Estado esperado en nodo sano |
|---|---|
| **VERSION INFORMATION** | Versión APT y RPC coinciden |
| **STARTUP STATUS** | `No startup/forkpoint messages` — el nodo lleva tiempo corriendo |
| **STATE SYNC STATUS** | `No StateSync messages` — nodo ya sincronizado |
| **BLOCK SYNC STATUS** | `ZERO BlockSync overrides` — operación normal en modo live |
| **RAPTORCAST STATUS** | `✅ Confirming group join!` — el nodo está recibiendo bloques |
| **CONSENSUS STATUS** | `✅ Blocks processing and being committed` — bloques avanzando |

> ℹ️ Las secciones StateSync y BlockSync vacías son normales una vez el nodo está sincronizado. Solo muestran actividad durante el proceso inicial de sync o en caso de re-sync.

### Señales de alerta

| Output | Significado |
|---|---|
| `RaptorCast: No group join` | El nodo no está recibiendo bloques — revisar conectividad |
| `Blocks NOT being committed` | Consenso parado — revisar servicios y peers |
| Mensajes activos en StateSync | El nodo está re-sincronizando |
| Timeouts o dropping proposals | Posibles problemas de red o de peers |

---

## Relación con Prometheus/OTEL

Monlog extrae información de los logs que no siempre está disponible como métrica en Prometheus:

| Dato | Prometheus/OTEL | Monlog |
|---|---|---|
| Block height, epoch, round | ✅ Métrica numérica | ✅ Visible en output |
| Estado de sync | ✅ `monad_statesync_syncing` | ✅ Sección StateSync |
| RaptorCast group join | ❌ No expuesto | ✅ Visible en output |
| Nombres de validadores | ❌ Solo key cruda | ✅ Con flag `-n` |
| Timeouts y dropping proposals | Parcialmente en validation errors | ✅ Detalle en logs |
| Diagnósticos cualitativos | ❌ No aplica | ✅ Sugerencias de estado |

Monlog es complementario al stack Prometheus/Grafana — útil para diagnóstico puntual cuando algo va mal, no como fuente de métricas continuas.

---

## Referencias

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops/general-operations)
- [05 — Metrics Reference](05-monad-metrics-reference.md)
