---
tags:
  - roadmap
---

# Mejoras y roadmap

> **Tipo:** Roadmap | **Estado:** Estable | **Fecha:** 2026-10-02

Detalle completo en [`docs/MEJORAS.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/MEJORAS.md).

## Estado (v3.29.0)

| Ítem | Estado |
|------|--------|
| Partición 8 MB, registros V8 REST, token API, health, boot counters | ✅ |
| Tareas FreeRTOS (SensorTask + ControlTask) | ✅ |
| V8.1 (SpiManager, 74HC165, MCP23S17, ADC) | ✅ |
| V8.4 (StorageManager LittleFS/SPIFFS/SD) | ✅ |
| V9 (Modbus profiles, capa CAN) | ✅ |
| Logger + EventBus + Scheduler | ✅ |
| Multi-board + página `/pins` | ✅ |
| Interfaz WiFi/Ethernet (W5500) + MQTT | ✅ |
| Configuración por capas (migraciones + merge) | ✅ |
| Servidor: rate limiting + CORS + Store & Forward | ✅ |
| Autodetección guiada (`/api/v1/detect`) | ✅ |
| Gateway RS485 (polling multi-esclavo) | ✅ v3.17.0 |
| OTA por HTTP sobre Ethernet (W5500) | ✅ v3.16.0 |
| **Modularidad completa (Tasmota-like)** | ✅ v3.18 → v3.29 |

## Modularidad completa (v3.18.0 → v3.29.0)

Todo el hardware se configura desde la web **sin recompilar**:

| Elemento | Versión | Endpoint | Persistencia |
|----------|---------|----------|--------------|
| Mapa de pines en NVS | v3.18.0 | `GET/PUT /api/v1/pins` | `ghpins` |
| Formulario web de pines | v3.19.0 | `/pins` | — |
| Catálogo de sensores editable | v3.20.0 | `PUT /api/v1/sensors/catalog` | `ghsensors` |
| Direcciones I²C desde el catálogo | v3.21.0 | idem | — |
| Bloqueo de pines para PCB fija | v3.22.0 | `GH_PINS_LOCKED` | — |
| `enabled` del catálogo (todos) | v3.23.0 | idem | — |
| Catálogo de expansores | v3.24.0 | `PUT /api/v1/hardware` | `ghhw` |
| MCP23017 desde el catálogo | v3.25.0 | idem | — |
| Pool MCP23017 + agregar nodos | v3.26.0 | idem | — |
| Pools SPI (MCP23S17 + ADC) | v3.27.0 | idem | — |
| 74HC165 desde el catálogo | v3.28.0 | idem | — |
| Asignación de canales (pool I²C) | v3.29.0 | — | — |

## Pendientes (no bloqueantes)

1. **Mapeo de canales configurable por actuador** — hoy el `ROLE_TABLE` tiene
   canales fijos (0..24); falta que la web elija canal/bus por actuador.
2. **HTTPS sobre W5500** — la librería `Ethernet` clásica no tiene TLS; requiere
   `W5500lwIP` o ETH nativo.
3. **Estación meteorológica por Ethernet** — aplicar el patrón GET manual sobre
   `Client*` a `WeatherStation` (diferido hasta cerrar la modularidad).
4. **Integración CAN de aplicación** — tras definir el hardware.
