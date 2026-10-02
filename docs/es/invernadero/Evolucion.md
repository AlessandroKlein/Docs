# Evolución (V8 / V9 / V10)

> **Tipo:** Roadmap | **Estado:** Estable | **Fecha:** 2026-10-02

Objetivo: que el firmware pase de representar **una instalación** a representar
**una plataforma**.

```text
FIRMWARE       → "qué puede hacer"     (capacidades)
CONFIGURACIÓN  → "qué hardware hay"    (instalación)
AUTOMATIZACIÓN → "qué debe hacer"      (comportamiento)
```

## Estado (v3.29.0)

| Área | Estado |
|------|--------|
| Firmware de campo | ✅ v3.29.0 |
| Servidor central | ✅ entregado (rate limiting + CORS + Store & Forward) |
| V8 base + registros REST | ✅ |
| V8.1 (SpiManager, 74HC165, MCP23S17, ADC) | ✅ |
| V8.4 StorageManager (LittleFS/SPIFFS/SD) | ✅ |
| V9 Modbus profiles + capa CAN | ✅ |
| Interfaz WiFi/Ethernet intercambiable (W5500) | ✅ v3.13.0 |
| Multi-board + página `/pins` | ✅ |
| OTA por HTTP sobre Ethernet (W5500) | ✅ v3.16.0 |
| Gateway RS485 (polling multi-esclavo) | ✅ v3.17.0 |
| Configuración por capas: merge/migraciones | ✅ v3.14.0 (schema 2) |
| Store & Forward (servidor) | ✅ |
| **Modularidad completa** (pines/sensores/expansores por web) | ✅ v3.18 → v3.29 |

## Pendientes

| Área | Estado |
|------|--------|
| HTTPS sobre W5500 (TLS) | ⚠️ requiere `W5500lwIP` o ETH nativo |
| Mapeo de canales configurable por actuador | ❌ pendiente |
| Estación meteorológica por Ethernet | ❌ pendiente |
| Integración CAN de aplicación | ❌ pendiente |

## Camino recorrido

```text
v3.0  → plataforma base (config, sensores, actuadores, control, red, OTA)
v3.1  → identidad, estados, Modbus, simulación
v3.7  → OTA remota con SHA-256 + rollback
v3.8  → V8: registros (capability/module/sensor/actuator)
v3.9  → partición 8 MB, REST V8, token de API, health, CAN
v3.10 → FreeRTOS (SensorTask/ControlTask), V8.1, V8.4, V9, StorageManager
v3.11 → Logger, EventBus, Scheduler, config por capas
v3.12 → multi-board + página /pins
v3.13 → interfaz WiFi/Ethernet (W5500)
v3.14 → configuración por capas (migraciones + merge), schema 2
v3.15 → autodetección guiada I²C
v3.16 → OTA por HTTP sobre Ethernet
v3.17 → gateway RS485 (polling multi-esclavo)
v3.18 → mapa de pines en NVS (PinConfig)
v3.19 → formulario web de pines
v3.20 → catálogo de sensores editable
v3.21 → direcciones I²C desde el catálogo
v3.22 → bloqueo de pines (PCB) + enabled del catálogo
v3.23 → el catálogo controla todos los sensores
v3.24 → catálogo de expansores
v3.25 → MCP23017 desde el catálogo
v3.26 → pool MCP23017 + agregar nodos
v3.27 → pools SPI (MCP23S17 + ADC)
v3.28 → 74HC165 desde el catálogo
v3.29 → asignación de canales (pool I²C)
```
