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

## Módulos

```text
core/       Types, PinMap, Version, PlatformTypes, CapabilityRegistry,
            ModuleRegistry, EventBus, Scheduler
config/     ConfigManager (NVS/JSON + token de API), Defaults
storage/    History, StorageManager (LittleFS/SPIFFS/SD)
hardware/   BusManager, HardwareManager, CanManager, SpiManager, AdcManager,
            ShiftRegister595/165, Mcp23017/23s17, ModbusRtu
sensors/    SHT31/AHT20, DS18B20, ADS1115, BH1750, SCD4x, caudal, tanque,
            lluvia, viento, pH, EC + SensorManager + registries
actuators/  ActuatorManager + ActuatorRegistry
control/    Climate, Irrigation, Lighting, Roof, Safety
network/    NetworkManager, MqttManager
api/        RestApi, WebSocketServer
system/     Watchdog, HealthMonitor, BootCounters, Logger, OtaManager, Device
```

## Jerarquía de control

```text
EMERGENCIA → SEGURIDAD → MANUAL → AUTOMÁTICO → PROGRAMACIÓN
```
