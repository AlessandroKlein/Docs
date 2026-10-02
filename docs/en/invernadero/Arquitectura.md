# Architecture

> **Type:** Embedded | **Status:** Stable | **Date:** 2026-10-02

## Layers

```text
                CENTRAL SERVER (optional)
                        │
               HTTPS / MQTT / API
                        │
                     ESP32   ← autonomous field controller
                        │
          ┌─────────────┼─────────────┐
      Sensors      Actuators     Local network
```

The ESP32 keeps working if the server or the network disappears.

## FreeRTOS tasks

```text
Core 0 — SensorTask (prio 3) → read sensors (protected slots)
         ControlTask (prio 2) → safety + controllers + outputs
Core 1 — loop()               → network, MQTT, API, WebSocket, OTA
```

## Modules

```text
core/       Types, PinMap, Version, PlatformTypes, CapabilityRegistry,
            ModuleRegistry, EventBus, Scheduler
config/     ConfigManager (NVS/JSON + API token), Defaults
storage/    History, StorageManager (LittleFS/SPIFFS/SD)
hardware/   BusManager, HardwareManager, CanManager, SpiManager, AdcManager,
            ShiftRegister595/165, Mcp23017/23s17, ModbusRtu
sensors/    SHT31/AHT20, DS18B20, ADS1115, BH1750, SCD4x, flow, tank,
            rain, wind, pH, EC + SensorManager + registries
actuators/  ActuatorManager + ActuatorRegistry
control/    Climate, Irrigation, Lighting, Roof, Safety
network/    NetworkManager (WiFi/Ethernet), MqttManager
api/        RestApi, WebSocketServer
system/     Watchdog, HealthMonitor, BootCounters, Logger, OtaManager, Device
```

## Control hierarchy

```text
EMERGENCY → SAFETY → MANUAL → AUTOMATIC → SCHEDULE
```
