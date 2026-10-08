---
tags:
  - sema
  - configuracion
---

# Referencia de configuración (JSON completa)

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.75.0

Referencia **exhaustiva** de cada clave del JSON de configuración de SEMA
(`Config`, `include/core/ConfigManager.hpp`). Editable por `PUT /api/v1/config`,
por `POST /api/v1/backup` o por la web. Esquema actual: **`schema_version: 1`**.

!!! info "Cómo leer esta página"
    - **Clave**: nombre exacto en el JSON.
    - **Tipo**: `bool`, `int`, `float`, `str`, `enum`, `array`.
    - **Default**: valor de fábrica si no se especifica.
    - Todos los campos son opcionales: si faltan, se usa el default.
    - La config es **transaccional**: se valida antes de aplicar; si falla, se hace rollback.

---

## Ejemplo completo

```json
{
  "schema_version": 1,
  "station": { "id": "SEMA-001", "name": "Estación Norte" },
  "network": { "mode": "STA", "ssid": "MiRed", "password": "clave", "hostname": "sema-001", "mdns": true },
  "system": { "timezone": "America/Argentina/Buenos_Aires", "log_level": "INFO" },
  "storage": { "backend": "littlefs", "retention_days": 30 },
  "security": { "api_key": "clave-web", "server_key": "clave-central" },
  "energy": { "rain_pin": 4 },
  "publishers": { "webhook_url": "https://example.com/hook", "mqtt_host": "broker.local", "mqtt_port": 1883, "mqtt_topic": "sema/measurement", "mqtt_user": "sema", "mqtt_pass": "clave" },
  "sensors": [
    { "id": "EXT", "model": "BME280", "sda": 21, "scl": 22 },
    { "id": "SOIL", "model": "DS18B20", "pin": 4 },
    { "id": "BATT", "model": "ADC", "pin": 34, "channel": "voltage", "unit": "V", "scale": 0.00887, "offset": 0.0 }
  ],
  "rules": [ { "name": "high_temp", "sensor_id": "EXT", "channel_id": "temperature", "op": "gt", "value": 40.0 } ],
  "calibrations": [ { "sensor_id": "EXT", "channel_id": "temperature", "gain": 1.0, "offset": -0.5, "has_range": true, "min": -40.0, "max": 85.0 } ],
  "gpio": [ { "id": "relay", "pin": 26, "mode": "output", "initial": 0, "expander_addr": 0 } ]
}
```

---

## Raíz

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `schema_version` | int | `1` | Versión del esquema de configuración |
| `station` | objeto | — | Identidad de la estación |
| `network` | objeto | — | WiFi / mDNS |
| `system` | objeto | — | Zona horaria y log |
| `storage` | objeto | — | Persistencia local |
| `security` | objeto | — | Claves de acceso |
| `energy` | objeto | — | Wake por lluvia |
| `publishers` | objeto | — | Salida HTTP/MQTT |
| `sensors` | array | `[]` | Catálogo de sensores (vacío = catálogo por defecto) |
| `rules` | array | `[]` | Reglas de alarma (vacío = regla por defecto) |
| `calibrations` | array | `[]` | Calibración por canal |
| `gpio` | array | `[]` | GPIO standalone (entradas/salidas) |
| `shift_register` | objeto | — | Shift register (74HC595/74HC165) |

## `station`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `id` | str | `"SEMA-001"` | Identificador lógico de la estación |
| `name` | str | `"Estación Norte"` | Nombre visible |

## `network`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `mode` | enum | `"STA"` | `STA` (cliente) · `AP` (punto de acceso) |
| `ssid` | str | `""` | SSID de la red WiFi |
| `password` | str | `""` | Clave WiFi |
| `hostname` | str | `"sema-001"` | Nombre de host (mDNS `<hostname>.local`) |
| `mdns` | bool | `true` | Habilita mDNS |

## `system`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `timezone` | str | `"America/Argentina/Buenos_Aires"` | Zona horaria IANA |
| `log_level` | enum | `"INFO"` | `DEBUG` · `INFO` · `WARNING` · `ERROR` |

## `storage`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `backend` | enum | `"littlefs"` | `littlefs` · `flash` · `sd` |
| `retention_days` | int | `30` | Días de retención (referencia) |

## `security`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `api_key` | str | `""` | Clave de la web/API local (vacío = sin auth) |
| `server_key` | str | `""` | Clave del Servidor Central → SEMA |

## `energy`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `rain_pin` | int | `0` | GPIO del pluviómetro para wake por lluvia (`0` = deshabilitado) |

## `publishers`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `webhook_url` | str | `""` | URL del webhook HTTP (vacío = deshabilitado) |
| `mqtt_host` | str | `""` | Host del broker MQTT (vacío = deshabilitado) |
| `mqtt_port` | int | `1883` | Puerto del broker MQTT |
| `mqtt_topic` | str | `"sema/measurement"` | Topic de publicación |
| `mqtt_user` | str | `""` | Usuario MQTT (vacío = sin auth) |
| `mqtt_pass` | str | `""` | Contraseña MQTT |

## `sensors[]`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `id` | str | — | Id lógico del sensor (p. ej. `"EXT"`) |
| `model` | str | — | Modelo (ver [Sensores](Sensores.md)) |
| `sda` | int | `21` | Pin SDA (I²C) |
| `scl` | int | `22` | Pin SCL (I²C) |
| `pin` | int | `0` | Pin del sensor (1-Wire, ADC, PCNT) o canal (ADS1115) |
| `rx` | int | `0` | Pin RX (UART, p. ej. PMS5003) |
| `tx` | int | `0` | Pin TX (UART) |
| `channel` | str | `""` | Magnitud analógica (p. ej. `"voltage"`) |
| `unit` | str | `""` | Unidad (p. ej. `"V"`) |
| `scale` | float | `1.0` | Escala del valor |
| `offset` | float | `0.0` | Offset del valor |

## `rules[]`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `name` | str | — | Nombre de la regla (para correlación) |
| `sensor_id` | str | `""` | Sensor objetivo (`""` = cualquiera) |
| `channel_id` | str | — | Canal a vigilar (`"temperature"`, …) |
| `op` | enum | `"gt"` | `gt` · `lt` · `ge` · `le` |
| `value` | float | `0.0` | Umbral |

## `calibrations[]`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `sensor_id` | str | — | Sensor |
| `channel_id` | str | — | Canal (`"sensorId:channelId"`) |
| `gain` | float | `1.0` | Ganancia |
| `offset` | float | `0.0` | Offset |
| `has_range` | bool | `false` | Si define rango válido |
| `min` | float | `0.0` | Mínimo del rango |
| `max` | float | `0.0` | Máximo del rango |

## `gpio[]`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `id` | str | — | Id lógico |
| `pin` | int | `0` | Pin GPIO nativo, o pin del MCP23017 (0–15) |
| `mode` | enum | `"input"` | `output` · `input` · `input_pullup` · `input_pulldown` |
| `initial` | int | `0` | Estado inicial de las salidas (`0`/`1`) |
| `expander_addr` | int | `0` | `0` = pin nativo; `!= 0` = MCP23017 en esa dirección I²C (p. ej. `0x20`) |

## `shift_register`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `type` | enum | `"74HC595"` | `74HC595` (salida) · `74HC165` (entrada) |
| `data_pin` | int | `0` | SER (595) / QH (165) |
| `clock_pin` | int | `0` | SRCLK (595) / CLK (165) |
| `latch_pin` | int | `0` | RCLK (595) / SH-LD (165) |

---

## Validación y rollback

- `ConfigManager::apply()` valida el esquema (`schema_version == 1`), que
  `station.id` no esté vacío y que `network.mode` y `storage.backend` sean válidos.
- Si la validación falla, la config previa se mantiene (rollback).
- `PUT /config` y `POST /backup` re-aplican la config **sin reinicio** (sensores,
  reglas, calibración, GPIO y publicadores).
