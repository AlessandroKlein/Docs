# Registro de cambios por archivo

> **Tipo:** Referencia | **Estado:** En desarrollo | **Fecha:** 2026-10-02

Historial de modificaciones por archivo (qué / por qué / cómo / impacto). Se
actualiza en cada commit que cambie comportamiento.

## 2026-10-02 — Multi-board y página de pines

### `platformio.ini`
- **Qué:** se añadieron entornos para ESP32 / S2 / S3 / C3 / C6 con `default_envs`.
- **Por qué:** soportar multi-board (README §207) desde un único código fuente.
- **Cómo:** sección `[platformio]` + base `[env]` compartida + un `[env:<board>]` por target.
- **Impacto:** `pio run` sigue compilando el target por defecto; los demás se compilan con `-e`.
- **Referencia:** v3.12.0.

### `src/hardware/CanManager.cpp`
- **Qué:** guardas `CONFIG_IDF_TARGET_*` para el driver TWAI.
- **Por qué:** el periférico TWAI solo existe en ESP32/S2/S3 (C3/C5/C6 no lo tienen).
- **Cómo:** `#if GH_HAS_TWAI` con stubs inertes en targets sin CAN.
- **Impacto:** compilación multi-board sin romper CAN.
- **Referencia:** v3.12.0.

## 2026-10-02 — Tareas FreeRTOS separadas

### `src/main.cpp`
- **Qué:** `automationTask` dividida en `sensorTask` + `controlTask`.
- **Por qué:** principio de no bloqueo (SEMA §202-203).
- **Cómo:** semáforo `sensorReady`; prioridad 3 (sensores) y 2 (control).
- **Impacto:** `include/sensors/SensorManager.hpp` (getters protegidos).
- **Referencia:** v3.10.0.

### `include/sensors/SensorManager.hpp`
- **Qué:** se añadió `valueAt()` y se protegieron los getters.
- **Por qué:** evitar carrera de datos entre sensorTask y controlTask.
- **Cómo:** `xSemaphoreTake/Give` alrededor de la lectura.
- **Impacto:** sin cambio de firma en controladores.
- **Referencia:** v3.10.0.
