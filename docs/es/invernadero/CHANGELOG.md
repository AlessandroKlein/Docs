# Registro de versiones (CHANGELOG)

> **Tipo:** Referencia | **Estado:** En desarrollo | **Fecha:** 2026-10-02

Historial completo de versiones del firmware. Formato [Keep a Changelog](https://keepachangelog.com/es/1.1.0/).

## [3.29.0] - 2026-10-02 — Asignación de canales (pool MCP23017)
- `ActuatorManager` mapea canales 32..95 → device 0..3 / pin 0..15 del pool I²C.

## [3.28.0] — 74HC165 desde el catálogo
- `ShiftRegister165` instanciado por nodo `HC165`; pines `hc165_*` en NVS.

## [3.27.0] — Pools SPI
- `Mcp23s17` y `AdcManager` (MCP3208) instanciados desde el catálogo; `SpiManager` desde NVS.

## [3.26.0] — Pool MCP23017 + agregar nodos
- Pool de MCP23017 (hasta 4); `PUT /api/v1/hardware` permite agregar nodos.

## [3.25.0] — MCP23017 desde el catálogo
- El driver MCP23017 se instancia desde el nodo `mcp23017-0` (address + enabled).

## [3.24.0] — Catálogo de expansores
- `HardwareManager` editable/persistido; `PUT /api/v1/hardware`.

## [3.23.0] — Catálogo controla todo
- El `enabled` del catálogo controla el reporte de **todos** los sensores.

## [3.22.0] — Bloqueo de pines + catálogo
- `GH_PINS_LOCKED` (0 público / 1 PCB); `enabled` del catálogo en los I²C.

## [3.21.0] — Direcciones I²C desde el catálogo
- SHT31/AHT20/ADS1115/BH1750/SCD41 leen su dirección del catálogo.

## [3.20.0] — Catálogo de sensores editable
- `SensorRegistry` editable/persistido (`PUT /api/v1/sensors/catalog`).

## [3.19.0] — Formulario web de pines
- Página `/pins` editable (carga/guarda el mapa de pines en NVS).

## [3.18.0] — Mapa de pines en NVS
- `PinConfig` + `PinConfigManager`; `GET/PUT /api/v1/pins`.

## [3.17.0] — Gateway RS485
- `ModbusGateway` (polling multi-esclavo por perfiles) + `GET /api/v1/modbus/gateway`.

## [3.16.0] — OTA HTTP sobre Ethernet
- GET manual sobre `Client*` + `Update.write` (W5500); soporta chunked.

## [3.15.0] - 2026-10-02
### Added
- Autodetección guiada de dispositivos I²C (`GET /api/v1/detect`).
### Changed
- `docs/MEJORAS.md`: HTTP/OTA sobre Ethernet registrado como bloqueo (§10).

## [3.14.0] - 2026-10-02
### Added
- Configuración por capas: `schema_version` + migraciones incrementales + merge profundo. Esquema v2.

## [3.13.0] - 2026-10-02
### Added
- Interfaz de red intercambiable WiFi/Ethernet (W5500): conexión + MQTT.
- Configuración Ethernet (`net_interface`, `eth_*`).

## [3.12.0] - 2026-10-02
### Added
- Multi-board (ESP32/S2/S3/C3/C6) y página `/pins`.
- Pines SPI nativos + nota de strapping.
### Removed
- `.clinerules`.

## [3.11.0] - 2026-10-02
### Added
- `Logger` estructurado, `EventBus`, `Scheduler` por capacidades.
- SD como backend de `StorageManager`.
- Tipos de configuración por capas (`ConfigLayer`).
- `docs/ESTANDAR-DOCUMENTACION.md`.

## [3.10.0] - 2026-10-02
### Added
- División de tareas FreeRTOS (`SensorTask` + `ControlTask`) con accesores protegidos.
- V8.1: `SpiManager`, `ShiftRegister165`, `Mcp23s17`, `AdcManager`.
- V8.4: `StorageManager` (LittleFS/SPIFFS).
- V9: `ModbusProfileRegistry` + capa de aplicación CAN.
- Tabla de datos responsive + `frontend-preview.html` actualizado.

## [3.9.0] - 2026-10-02
### Added
- Partición de 8 MB por defecto.
- Registros V8 por REST (`/modules`, `/buses`, `/hardware`, `/sensors/catalog`, `/actuators/catalog`).
- Token de API (web + endpoints) para control desde el servidor.
- `HealthMonitor`, `BootCounters`, watchdog de tareas.
- `CanManager` (TWAI).
- `docs/MEJORAS.md`.

## [3.8.0] - 2026-10-02
### Added
- Base V8: `PlatformTypes`, `CapabilityRegistry`, `ModuleRegistry`, `BusManager`, `HardwareManager`, `SensorRegistry`, `ActuatorRegistry`.

## [3.7.0] - 2026-09-29
### Added
- OTA remota desde el servidor central (SHA-256 + rollback) vía comando MQTT.

## [3.6.0] - 2026-09-29
### Added
- Estación meteorológica externa (HTTP + JSON configurable).

## [3.5.0] - 2026-09-28
### Added
- Tema claro/oscuro en la web local.

## [3.4.0] - 2026-09-28
### Added
- Autenticación local de la web (admin + token de sesión).

## [3.3.0] - 2026-09-28
### Added
- Variables calculadas (VPD, punto de rocío) y motor de reglas configurable.

## [3.2.0] - 2026-09-28
### Added
- Identificación de hardware, escaneo WiFi, AP identificable, export/import.

## [3.1.x] - 2026-09-28
### Added
- Identidad permanente (UID), máquina de estados, calidad de datos, versionado de configuración + rollback, RS485/Modbus, modo simulación, manifest OTA.

## [3.0.0] - 2026-09-28
### Added
- Plataforma modular base (config NVS/JSON, sensores, actuadores, control, red, API, WebSocket, OTA, watchdog).
