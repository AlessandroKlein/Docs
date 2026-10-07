---
tags:
  - sema
  - identidad
---

# Identidad y estados

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.44.0

Identidad de la estación y estados/máquinas de estado del firmware.

## Identidad

| Campo | Fuente | Descripción |
|-------|--------|-------------|
| `station.id` | config | Id lógico (default `"SEMA-001"`) |
| `station.name` | config | Nombre visible (default `"Estación Norte"`) |
| `firmware` | `SEMA_FW_VERSION` | `1.44.0` |
| `hw` | `SEMA_HW_VERSION` | `rev0` |
| `config_schema` | `SEMA_CONFIG_SCHEMA_VERSION` | `1` |
| `protocol` | `SEMA_PROTOCOL_VERSION` | `1` |
| `network.hostname` | config | host mDNS `<hostname>.local` |

La identidad viaja en las mediciones (`station_id`), en MQTT/webhook y en
`GET /api/v1/status` / `/api/v1/system`.

## Estados de salud (`HealthMonitor`)

`HEALTHY` → `DEGRADED` → `ERROR`, según heartbeat (30 s) y sensores online.

## Estados de energía (`EnergyProfile`)

`Performance` · `Normal` · `LowPower` · `UltraLowPower`.

## Wake reasons (deep sleep)

`wake_reason` (de `esp_sleep_get_wakeup_cause()`): timer RTC o GPIO de lluvia (`ext0`).

## Calidad de medición (`Quality`)

`VALID` · `INVALID` · `STALE` · `TIMEOUT` · `OUT_OF_RANGE` ·
`CALIBRATION_ERROR` · `COMMUNICATION_ERROR` · `SENSOR_DISCONNECTED`.

## Estado de red

| Campo | Valores |
|-------|---------|
| `network.mode` | `STA` · `AP` |
| `connected` | `true` · `false` |
| `ip` | dirección IP |
| `rssi` | dBm |

## Estado de sensores

Cada sensor del catálogo reporta `healthy` (true/false) en `GET /api/v1/sensors`.

## Estados del dashboard (sesión)

- **Sin sesión**: se muestra el login.
- **Con sesión válida**: se muestra el dashboard (estado, histórico, gráfico, config).
- **Sesión expirada** (1 h sin actividad): vuelve al login.
