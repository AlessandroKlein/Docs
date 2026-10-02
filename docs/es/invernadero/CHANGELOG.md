# Registro de versiones (CHANGELOG)

> **Tipo:** Referencia | **Estado:** En desarrollo | **Fecha:** 2026-10-02

Historial completo de versiones del firmware. Formato [Keep a Changelog](https://keepachangelog.com/es/1.1.0/).

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
