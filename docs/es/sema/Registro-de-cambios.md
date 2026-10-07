---
tags:
  - sema
  - cambios
---

# Registro de cambios por archivo

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.68.0

Qué archivos cambiaron en las últimas versiones (para revisión y blame).

## v1.33.0

| Archivo | Cambio |
|---------|--------|
| `src/core/storage/HistoryStore.*` | retención por tiempo (`prune`) |
| `src/core/web/HttpServer.cpp` | `/api/v1/history?format=csv` + botón CSV |
| `src/core/SemaCore.cpp` | poda periódica del histórico |

## v1.32.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | gráficos con series, ejes y rangos |
| `docs/ETHERNET-Y-BUILDFLAGS.md` | documentación Ethernet + build_flags |

## v1.31.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | catálogo de tarjetas + añadir/eliminar + gráficos |

## v1.30.0

| Archivo | Cambio |
|---------|--------|
| `data/gridstack-all.min.js` + `gridstack.min.css` | Gridstack servido desde LittleFS |
| `src/core/web/HttpServer.*` | rutas `/gridstack.min.css` y `/gridstack-all.min.js` |
| `platformio.ini` | `board_build.filesystem = littlefs` |

## v1.29.0

| Archivo | Cambio |
|---------|--------|
| `src/core/web/HttpServer.cpp` | dashboard Gridstack + endpoints `/wind/resistors` y `/dashboard/layout` |
| `src/core/derived/DerivedCalculator.cpp` | veleta por tabla de 8 resistencias |
| `ConfigManager` | `system.wind_resistors[]`, `wind_rpull`, `dashboard_layout` |

## v1.28.0

| Archivo | Cambio |
|---------|--------|
| `include/core/derived/DerivedCalculator.hpp` + `.cpp` | magnitudes derivadas + unidades |
| `include/core/ConfigManager.hpp` + `.cpp` | `system.units`/`altitude`/veleta + `sensors[].enabled` |
| `src/core/web/HttpServer.*` | `/api/v1/sensors` (derivadas) + `/api/v1/wind/north` |

## v1.27.0

| Archivo | Cambio |
|---------|--------|
| `firmware_manifest.json` | multi-chip (un binario + SHA por board) |
| `partitions_{4,8,16}mb.csv` | tablas de particiones por tamaño de flash |
| `platformio.ini` | env `esp32-wroom-32u` (16 MB) + particiones por env |
| `HwProfile.hpp` | `SEMA_BOARD_ID` + `SEMA_FLASH_MB` + `BOARD_ESP32_WROOM32U` |

## v1.26.0

| Archivo | Cambio |
|---------|--------|
| `include/hw/HwProfile.hpp` | perfil de hardware (board + features + pines) |
| `platformio.ini` | `build_flags` de features y variantes |
| `SemaCore`/`HttpServer`/managers | compilación condicional `#if SEMA_USE_*` |

## v1.25.0

| Archivo | Cambio |
|---------|--------|
| `src/core/EthernetManager.cpp` | W5500 vía driver ESP-IDF `esp_eth` (integrado a lwIP) |
| `include/core/ConfigManager.hpp` + `.cpp` | campos `irq`/`sck`/`miso`/`mosi` |

## v1.24.0

| Archivo | Cambio |
|---------|--------|
| `include/core/EthernetManager.hpp` + `.cpp` | Ethernet LAN8720A (nativo) / W5500 (SPI) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `ethernet` |
| `platformio.ini` | `[env:base]` + `build_flags` (`BOARD_ESP32_WROOM`/`BOARD_ESP32_S3`) |

## v1.23.0

| Archivo | Cambio |
|---------|--------|
| `include/core/ZigbeeManager.hpp` + `.cpp` | nuevo Zigbee (ZNP por UART) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `zigbee` |
| `src/core/web/HttpServer.*` | `/api/v1/zigbee` GET/POST |

## v1.22.0

| Archivo | Cambio |
|---------|--------|
| `include/core/LoraManager.hpp` + `.cpp` | nuevo LoRa SX1262 (RadioLib) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `lora` |
| `src/core/web/HttpServer.*` | `/api/v1/lora` GET/POST |

## v1.21.0

| Archivo | Cambio |
|---------|--------|
| `include/core/CanManager.hpp` + `.cpp` | nuevo CAN/TWAI |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `can` |
| `src/core/web/HttpServer.*` | `/api/v1/can` GET/POST |

## v1.20.0

| Archivo | Cambio |
|---------|--------|
| `include/core/ModbusManager.hpp` + `.cpp` | nuevo maestro Modbus RTU |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `modbus` |
| `src/core/web/HttpServer.*` | `/api/v1/modbus` |

## v1.19.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/SolarSensor.hpp` + `.cpp` | nuevo driver SOLAR |
| `src/core/sensors/SensorFactory.cpp` | registro `"SOLAR"` |

## v1.18.0

| Archivo | Cambio |
|---------|--------|
| `include/core/sensors/CoSensor.hpp` + `.cpp` | nuevo driver CO |
| `src/core/sensors/SensorFactory.cpp` | registro `"CO"` |

## v1.17.0

| Archivo | Cambio |
|---------|--------|
| `include/core/ShiftRegisterManager.hpp` + `.cpp` | nuevo (74HC595/74HC165) |
| `include/core/ConfigManager.hpp` + `.cpp` | sección `shift_register` |
| `src/core/web/HttpServer.*` | `/api/v1/shift` GET/POST |

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
