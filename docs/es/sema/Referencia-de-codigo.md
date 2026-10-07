---
tags:
  - sema
  - referencia
  - codigo
---

# Referencia de código

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.38.0

Mapa del código fuente de SEMA (PlatformIO + Arduino framework, ESP32).

## Estructura de directorios

```text
SEMA/
├── platformio.ini                 # board, particiones y lib_deps
├── partitions.csv                 # OTA A/B + spiffs
├── firmware_manifest.json         # versión + SHA-256 del firmware
├── CHANGELOG.md
├── include/
│   └── core/
│       ├── SemaCore.hpp           # núcleo (orquestador singleton)
│       ├── ConfigManager.hpp      # config schema=1 (transaccional)
│       ├── EventBus.hpp           # pub/sub tipado
│       ├── Measurement.hpp        # modelo canónico
│       ├── Capability.hpp/.hpp    # capacidades
│       ├── Version.hpp            # SEMA_FW_VERSION / HW / SCHEMA / PROTOCOL
│       ├── BoardProfile.hpp       # pines fijos (SEMA_FIXED_HARDWARE)
│       ├── Time.hpp               # nowEpoch()
│       ├── GpioManager.hpp        # GPIO standalone + MCP23017
│       ├── Watchdog.hpp           # esp_task_wdt
│       ├── HealthMonitor.hpp      # estado HEALTHY/DEGRADED/ERROR
│       ├── PowerManager.hpp       # deep sleep + wake por lluvia
│       ├── Scheduler.hpp          # tareas periódicas
│       ├── ModuleRegistry.hpp     # módulos loop
│       ├── Calibration.hpp
│       ├── alarms/
│       │   ├── Rule.hpp           # RuleOp + Rule
│       │   └── RuleEngine.hpp     # evalúa reglas
│       ├── events/
│       │   └── EventLog.hpp       # eventos JSONL persistentes
│       ├── sensors/
│       │   ├── Sensor.hpp         # interfaz
│       │   ├── SensorManager.hpp  # registro + lectura
│       │   ├── SensorFactory.hpp  # model → driver
│       │   ├── I2cScanner.hpp     # detección I²C
│       │   └── <Driver>Sensor.hpp # 15 drivers
│       ├── derived/
│       │   └── DerivedEngine.hpp  # dew point, heat index, etc.
│       ├── network/
│       │   └── WiFiManager.hpp    # STA/AP + mDNS + reconexión
│       ├── publishers/
│       │   ├── Publisher.hpp      # interfaz
│       │   ├── PublisherManager.hpp
│       │   ├── HttpPublisher.hpp  # webhook
│       │   └── MqttPublisher.hpp  # MQTT
│       ├── storage/
│       │   ├── Storage.hpp        # KeyValueStore (NVS)
│       │   ├── NvsStore.hpp
│       │   └── HistoryStore.hpp   # histórico JSONL + rotación
│       └── web/
│           └── HttpServer.hpp     # WebServer + WebSocket
└── src/
    ├── main.cpp                   # pequeño (DESIGN-SYSTEM §102)
    └── core/                      # implementaciones (.cpp)
```

## Componentes clave

| Clase | Responsabilidad |
|-------|-----------------|
| `SemaCore` | Singleton que inicializa y conduce todos los servicios |
| `ConfigManager` | Config transaccional con validación + rollback (NVS) |
| `EventBus` | Pub/sub tipado por `EventType` |
| `SensorManager` | Registra drivers y orquesta la lectura periódica |
| `SensorFactory` | Mapea `model` (string) → driver concreto |
| `RuleEngine` | Evalúa reglas (`gt`/`lt`/`ge`/`le`) y emite alarmas |
| `DerivedEngine` | Derivadas: punto de rocío, índice de calor, etc. |
| `HistoryStore` | Histórico JSONL con rotación (conserva la mitad) |
| `EventLog` | Eventos JSONL persistentes con rotación |
| `WiFiManager` | STA/AP, mDNS, reconexión con backoff |
| `HttpServer` | API REST + WebSocket + dashboard + login |
| `PowerManager` | Deep sleep + wake por timer/lluvia |
| `GpioManager` | GPIO standalone (nativo + MCP23017) |
| `Watchdog` | `esp_task_wdt` jerárquico |
| `HealthMonitor` | Heartbeat + estado de salud |

## Versiones (include/core/Version.hpp)

| Macro | Valor (v1.38.0) | Significado |
|-------|-----------------|-------------|
| `SEMA_FW_VERSION` | `"1.38.0"` | Versión SemVer del firmware (sin `v`) |
| `SEMA_HW_VERSION` | `"rev0"` | Revisión del hardware |
| `SEMA_CONFIG_SCHEMA_VERSION` | `1` | Esquema de configuración |
| `SEMA_PROTOCOL_VERSION` | `1` | Protocolo con el Servidor Central |

## Pines fijos (include/core/BoardProfile.hpp)

| Macro | Default | Uso |
|-------|---------|-----|
| `SEMA_FIXED_HARDWARE` | `0` | `1` = PCB propio (pines fijos, ignora `sensors[]`) |
| `SEMA_PIN_I2C_SDA` | `21` | SDA I²C |
| `SEMA_PIN_I2C_SCL` | `22` | SCL I²C |
| `SEMA_PIN_ONEWIRE` | `4` | Bus 1-Wire |
| `SEMA_PIN_BATTERY_ADC` | `34` | ADC de batería |
