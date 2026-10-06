---
tags:
  - sema
  - cambios
---

# Registro de cambios por archivo

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.16.0

Qué archivos cambiaron en las últimas versiones (para revisión y blame).

## v1.16.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | Cookie `SameSite=Strict` en login/logout |
| `include/core/Version.hpp` | `1.16.0` |
| `firmware_manifest.json`, `CHANGELOG.md`, `docs/*` | versión y docs |

## v1.15.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/Ads1115Sensor.hpp` + `.cpp` | nuevo driver ADS1115 |
| `src/core/sensors/SensorFactory.cpp` | registro `"ADS1115"` |
| `platformio.ini` | librería ADS1X15 |

## v1.14.0

| Archivo | Cambio |
|---------|--------|
| `include/core/GpioManager.hpp` + `.cpp` | soporte MCP23017 (`expander_addr`) |
| `include/core/ConfigManager.hpp` + `.cpp` | campo `expander_addr` en `GpioSpec` |
| `platformio.ini` | librería MCP23017 |

## v1.13.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/As3935Sensor.hpp` + `.cpp` | nuevo driver AS3935 |
| `src/core/sensors/SensorFactory.cpp` | registro `"AS3935"` |
| `platformio.ini` | librería SparkFun AS3935 |

## v1.12.0

| Archivo | Cambio |
|---------|--------|
| `include/core/web/HttpServer.hpp` + `.cpp` | expiración de sesión (1 h) |

## v1.11.0

| Archivo | Cambio |
|---------|--------|
| `include/core/publishers/MqttPublisher.hpp` + `.cpp` | `configure(..., user, pass)` |
| `include/core/ConfigManager.hpp` + `.cpp` | `mqtt_user` / `mqtt_pass` |

## v1.10.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/Sgp30Sensor.hpp` + `.cpp` | nuevo driver SGP30 |
| `platformio.ini` | librería SGP30 |

## v1.9.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/SensorManager.hpp` | `clear()` |
| `src/core/SemaCore.cpp` + `.hpp` | `applySensors()` (hot reload de sensores) |

## v1.8.0

| Archivo | Cambio |
|---------|--------|
| `include/core/publishers/HttpPublisher.hpp` / `MqttPublisher.hpp` | setters de config |
| `src/core/SemaCore.cpp` + `.hpp` | `applyPublishers()` + publishers como miembros |

## v1.7.0

| Archivo | Cambio |
|---------|--------|
| `include/core/GpioManager.hpp` + `.cpp` | nuevo (GPIO standalone) |
| `include/core/ConfigManager.hpp` + `.cpp` | `gpio[]` |
| `src/core/web/HttpServer.*` | `/api/v1/gpio` GET/POST |

## v1.6.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | rate limiting en login |
