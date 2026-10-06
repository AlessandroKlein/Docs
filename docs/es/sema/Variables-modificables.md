---
tags:
  - sema
  - configuracion
---

# Variables modificables (configuración JSON)

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.31.0

Resumen de las variables que se pueden modificar en caliente (`PUT /api/v1/config`)
y su efecto. Todas viven en el JSON de configuración; ver la referencia completa en
[Referencia de configuración](Referencia-configuracion.md).

## Aplicación en caliente (sin reinicio)

| Área | Claves | Se re-aplica al instante |
|------|--------|--------------------------|
| Reglas de alarma | `rules[]` | ✅ |
| Calibración | `calibrations[]` | ✅ |
| GPIO standalone | `gpio[]` | ✅ |
| Publicadores | `publishers.*` | ✅ (webhook + MQTT) |
| Sensores | `sensors[]` | ✅ (re-crea el catálogo) |
| Identidad | `station.id`, `station.name` | ✅ (visible en `/api/v1/status`) |

## Requieren reinicio

| Área | Claves | Motivo |
|------|--------|--------|
| Red WiFi | `network.mode`, `ssid`, `password`, `hostname` | `WiFiManager::begin()` corre solo en `setup()` |
| Zona horaria / log | `system.*` | se aplican en el arranque |
| Retención | `storage.retention_days` | referencia de configuración |
| Wake por lluvia | `energy.rain_pin` | configura el `ext0` en el arranque |

## Variables críticas para la identidad

| Variable | Efecto |
|----------|--------|
| `station.id` | `station_id` en mediciones, MQTT y webhook |
| `station.name` | nombre visible en el dashboard y `/api/v1/status` |
| `network.hostname` | host mDNS (`<hostname>.local`) |

## Variables de seguridad

| Variable | Efecto |
|----------|--------|
| `security.api_key` | clave del panel web/API local; también es la contraseña del login |
| `security.server_key` | clave que usa el Servidor Central para operar SEMA |

> Las claves se muestran enmascaradas por seguridad en la API (solo se devuelven a
> peticiones autenticadas).

## Semántica de `sensors[]`

- Si `sensors[]` está **vacío** (y `SEMA_FIXED_HARDWARE == 0`), se usa el catálogo
  por defecto (BME280, SHT40, DS18B20, BH1750, AHT20, batería ADC).
- Si `SEMA_FIXED_HARDWARE == 1` (PCB propio), `sensors[]` se **ignora** y se usan
  los pines fijos de `BoardProfile.hpp`.
- Cada sensor es una entrada del array; el `id` debe ser único.

## Semántica de `gpio[]` y actuadores

- `mode: "output"` + `initial` define un **actuador** (relé, led, válvula).
- Escribirlo por `POST /api/v1/gpio` con `{ "pin": 26, "value": 1 }`.
- Con `expander_addr` distinto de `0`, el pin se enruta a un MCP23017 por I²C.
