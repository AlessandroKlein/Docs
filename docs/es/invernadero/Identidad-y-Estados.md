---
tags:
  - invernadero
  - sistema
---

# Identidad y estados

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

## 1. Identidad

- **UID permanente**: `ESP32-XXXXXX` (24 bits bajos de la MAC).
- **device_id** (`GH001`) y **greenhouse_id** (`GREENHOUSE-001`).
- **Perfil de hardware**: `ESP32-GH-V1`; **firmware** `3.29.0`; **hardware** `rev0`.
- **config_schema** 2, **protocol** 1.

## 2. Capacidades

`WIFI, I2C, SPI, ONEWIRE, ADC, GPIO, RS485, MODBUS, PWM, OTA`
(en `GET /api/v1/capabilities`).

## 3. Máquina de estados

```text
BOOTING → INITIALIZING → SELF_TEST → NETWORK → RUN
         (y SYNC / DEGRADED / ERROR / MAINTENANCE / UPDATING / RECOVERY)
```

## 4. Causa de reinicio

`POWER_ON | SOFTWARE_RESET | WATCHDOG | BROWNOUT | PANIC | OTA | FACTORY_RESET | UNKNOWN`.

## 5. Calidad de datos

`GOOD | WARNING | INVALID | TIMEOUT | OUT_OF_RANGE | CALIBRATION | DISCONNECTED`.

## 6. Salud y contadores de reinicio

- `GET /api/v1/health`: heartbeat por tarea, stack HWM, heap, uptime.
- `GET /api/v1/boot`: `boot_count`, `watchdog_count`, `brownout_count`, `panic_count`,
  `soft_reset_count`, `ota_count`, `factory_count`.

## 7. Logs estructurados

`GET /api/v1/logs`: nivel (`TRACE..CRITICAL`), módulo, mensaje.
