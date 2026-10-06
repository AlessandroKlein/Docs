---
tags:
  - sema
  - arquitectura
---

# Arquitectura

> **Tipo:** Concepto | **Estado:** Planificación | **Fecha:** 2026-10-03 | **Firmware:** v1.12.0

Arquitectura de SEMA consolidada a partir del `README.md`. Es la referencia para
implementar el firmware de forma modular.

## 1. Principios

```text
SENSOR ≠ FUNCIÓN
GPIO   ≠ SENSOR
BUS    ≠ SENSOR
MODELO ≠ MAGNITUD
HARDWARE ≠ CONFIGURACIÓN
```

- El hardware define las capacidades; la configuración web define cómo se usan.
- Simple por fuera, modular por dentro.
- El Core nunca depende de módulos opcionales.
- Una falla opcional no detiene la adquisición.

## 2. Arquitectura conceptual

```text
                    SEMA CORE
                        │
        ┌───────────────┼────────────────┐
        │               │                │
    Hardware         Sensors          Services
        │               │                ├── Web
        │               │                ├── API
        │               │                ├── MQTT
        │               │                ├── OTA
        │               │                └── Storage
        │               │
        ├── GPIO        ├── Temperature
        ├── I²C         ├── Humidity
        ├── SPI         ├── Pressure
        ├── UART        ├── Light / UV
        ├── ADC         ├── CO₂ / PM
        ├── RS485       ├── Wind / Rain
        ├── CAN         ├── Solar / Lightning
        ├── 1-Wire      └── Soil / Energy
        └── Expansores
```

## 2.1 Compatibilidad por perfiles (HAL)

La compatibilidad entre variantes de ESP32 **no se resuelve con `#ifdef`
repartidos**:

```text
SEMA Application → SEMA Core
                        ├── Capability Manager
                        ├── Resource Manager
                        ├── Runtime Manager
                        └── Hardware Abstraction Layer (HAL)
                                        │
                                        ▼
                                 Board/Chip Profile
```

- **Capability Manager** — qué puede hacer la plataforma (ADC, PCNT, PSRAM, SMP,
  Wi-Fi, 802.15.4, CAN/TWAI, …).
- **Resource Manager** — asigna y valida recursos detectando conflictos.
- **Runtime Manager** — tareas y afinidad (`AUTO` por defecto) sobre FreeRTOS.
- **HAL + Board/Chip Profile** — única frontera con el hardware concreto.

## 3. Estructura de carpetas

```text
SEMA/
├── include/core/          ← interfaces y cabeceras del Core
├── src/
│   ├── main.cpp           ← pequeño: initialize → register → start → loop
│   ├── core/              ← registry, event bus, config, health
│   ├── config/            ← configuración persistente (NVS) y validación
│   ├── hardware/          ← GPIO, ADC y periféricos internos
│   ├── buses/             ← I²C, SPI, UART, 1-Wire, RS485, CAN
│   ├── sensors/           ← drivers de sensores (abstracción por magnitud)
│   ├── actuators/         ← salidas digitales/PWM
│   ├── communications/    ← MQTT, HTTP, WebSocket, publicadores externos
│   ├── storage/           ← LittleFS/SD, histórico, eventos
│   ├── energy/            ← batería/panel, sleep, wake
│   ├── alarms/            ← alarmas y umbrales
│   ├── diagnostics/       ← salud, logs, inventario, métricas
│   ├── web/               ← servidor web local
│   ├── api/               ← endpoints REST
│   ├── ota/               ← actualización de firmware
│   └── modules/           ← módulos opcionales (LoRa, Zigbee, …)
└── docs/
```

## 4. Módulos del Core

```text
CORE
 ├── Sensor Engine (medición + validación + calibración)
 ├── Configuration (persistencia, esquema versionado, migraciones)
 ├── Capability / Resource / Runtime Manager + HAL
 ├── Event Bus / Event Manager
 ├── Scheduler (tareas por capacidades)
 ├── Storage API (NVS / Flash / LittleFS / SD / externo)
 ├── Web Server / API / WebSocket
 └── Diagnostics (health, watchdog, observabilidad)

CAPACIDADES / INTERFACES (implementaciones, no módulos que contaminan el Core)
 ├── MQTT, HTTP, LoRa, Zigbee, RS485, CAN, OTA
 ├── Weather, Air Quality, Lightning, Soil, Energy
```

## 5. Ciclo de vida de módulos

```text
AVAILABLE → INSTALLED → CONFIGURED → ENABLED → RUNNING
RUNNING   → DISABLED  → UNINSTALLED

Errores: INSTALL_ERROR · CONFIG_ERROR · RUNTIME_ERROR · UPDATE_ERROR
```

## 6. Flujo de datos

```text
MEDIR → VALIDAR → PROCESAR → ALMACENAR → PUBLICAR → SINCRONIZAR → DORMIR
                                                                      │
                                              DESPERTAR POR EVENTO ←──┘
```

## 7. Event Bus

```text
RAIN_START → Storage, Alarm, MQTT, Webhook, Dashboard, Wake Manager
```

Eventos: `SensorEvent · RainEvent · LightningEvent · BatteryEvent ·
NetworkEvent · AlarmEvent · SystemEvent · WakeEvent · SleepEvent`.

## 8. Scheduler y recursos dinámicos

Las tareas se crean **según los módulos habilitados**: sin LoRa no hay tarea
LoRa, sin SD no hay tarea SD. Esto reduce RAM, CPU y consumo.

## 9. Prioridades y no bloqueo

```text
P1 Core / seguridad / watchdog
P2 Adquisición de sensores
P3 Procesamiento y almacenamiento
P4 Alarmas y eventos
P5 API local
P6 Comunicación
P7 Servicios externos
```

Ningún servicio externo puede bloquear `SensorTask`, `MeasurementTask`,
`StorageTask` ni el `Watchdog`.

## 10. Niveles de visualización

```text
Nivel 1 — Estación autónoma (web local, sin servidor)
Nivel 2 — Estación + servicios externos (MQTT, ThingSpeak, Windy, …)
Nivel 3 — Sistema centralizado (Servidor Central opcional)
```

## 11. Definition of Done

Una versión funcional requiere, como mínimo: Core, configuración persistente, web
local, API, dashboard modular, sistema de módulos y sensores, catálogo, detección
I²C/1-Wire, configuración GPIO/ADC, expansores, RS485/Modbus, CAN, LoRa, Zigbee,
medición energética, almacenamiento, histórico, alarmas, diagnóstico, calibración,
validación de configuración, backup, importación/exportación, OTA, seguridad,
watchdog y documentación.

## 12. Definiciones de Fase 1

### Endpoints REST mínimos

```text
GET  /api/v1/status · /health · /system · /config · /diagnostics
PUT  /api/v1/config   (transaccional, autenticado)
POST /api/v1/restart  (autenticado)
```

### Modelo Canónico

```json
{
  "station_id": "SEMA-001",
  "sensor_id": "TEMP_EXT",
  "measurement": "temperature",
  "value": 24.7,
  "unit": "degC",
  "timestamp": "2026-09-30T15:00:00Z",
  "quality": "VALID"
}
```

### Storage API

```text
Storage API → NVS (config) · LittleFS (histórico/eventos/logs) · SD (opcional)
```
