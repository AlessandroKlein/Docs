---
tags:
  - roadmap
---

# Mejoras y roadmap

> **Tipo:** Roadmap | **Estado:** En desarrollo | **Fecha:** 2026-10-02

Detalle completo en [`docs/MEJORAS.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/MEJORAS.md).

## Estado

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
| HTTP/OTA sobre Ethernet | ⚠️ bloqueado (§10) |

## Prioridad sugerida

1. Gateway RS485 (polling multi-esclavo por perfiles Modbus).
2. HTTP/OTA sobre Ethernet (cliente HTTP propio o ETH nativo).
3. Integración CAN de aplicación.
