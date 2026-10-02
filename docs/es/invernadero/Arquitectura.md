---
tags:
  - invernadero
  - arquitectura
---

# Arquitectura

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

## Niveles

```text
                SERVIDOR CENTRAL (opcional)
                        │
               HTTPS / MQTT / API
                        │
                     ESP32   ← controlador de campo autónomo
                        │
          ┌─────────────┼─────────────┐
      Sensores      Actuadores     Red local
```

El ESP32 sigue funcionando si el servidor o la red desaparecen.

## Tareas FreeRTOS

```text
Core 0 — SensorTask (prio 3) → leer sensores (slots protegidos)
         ControlTask (prio 2) → seguridad + controladores + salidas
Core 1 — loop()               → red, MQTT, API, WebSocket, OTA
```

## Módulos (v3.29.0)

```text
core/       Types, PinConfig (pines en NVS), PinMap (referencia), Version,
            PlatformTypes, CapabilityRegistry, ModuleRegistry, EventBus, Scheduler
config/     ConfigManager (NVS/JSON + capas + migraciones + token de API), Defaults
storage/    History, StorageManager (LittleFS/SPIFFS/SD)
hardware/   BusManager, HardwareManager (catálogo + NVS), SpiManager, AdcManager,
            ShiftRegister595/165, Mcp23017, Mcp23s17, ModbusRtu, CanManager
sensors/    SensorManager + SensorRegistry (catálogo en NVS), ModbusProfileRegistry,
            ModbusGateway (RS485 multi-esclavo) y drivers:
            SHT31/AHT20, DS18B20, ADS1115, BH1750, SCD4x, caudal, tanque,
            lluvia, viento, pH, EC
actuators/  ActuatorManager (pool MCP23017 + canales) + ActuatorRegistry
control/    Climate, Irrigation, Lighting, Roof, Safety, RuleEngine, CalculatedVariables
network/    NetworkManager (WiFi/Ethernet), MqttManager, WeatherStation
api/        RestApi (48 endpoints), WebSocketServer
system/     Device, Watchdog, HealthMonitor, BootCounters, Logger, OtaManager, Diagnostics
web/        WebAssets (formularios HTML: dashboard y /pins)
```

### Persistencia en NVS (modularidad)

| Namespace | Contenido | Endpoint |
|-----------|-----------|----------|
| `ghpins` | Mapa de pines + direcciones I²C | `GET/PUT /api/v1/pins` |
| `ghsensors` | Catálogo de sensores | `GET/PUT /api/v1/sensors/catalog` |
| `ghhw` | Catálogo de expansores/nodos | `GET/PUT /api/v1/hardware` |
| (config) | `SystemConfig` completo | `GET/PUT /api/v1/config` |

## Jerarquía de control

```text
EMERGENCIA → SEGURIDAD → MANUAL → AUTOMÁTICO → PROGRAMACIÓN
```
