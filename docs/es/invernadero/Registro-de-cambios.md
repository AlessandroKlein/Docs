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

## 2026-10-02 — Modularidad completa (v3.18.0 → v3.29.0)

### `include/core/PinConfig.hpp` + `src/core/PinConfig.cpp` (v3.18.0)
- **Qué:** nueva estructura `PinConfig` (pines + direcciones I²C en runtime) y
  `PinConfigManager` con persistencia NVS (`ghpins`).
- **Por qué:** permitir configurar el hardware sin recompilar (README §205/§263).
- **Cómo:** struct con defaults equivalentes a `PinMap`, serialización JSON y
  `get/set/reset` sobre `Preferences`.
- **Impacto:** `SensorManager`, `BusManager`, `HardwareManager` y `main.cpp` leen
  pines de NVS en lugar de `PinMap.hpp`.
- **Referencia:** v3.18.0.

### `include/web/WebAssets.hpp` (v3.19.0)
- **Qué:** `PINS_HTML` pasó de tabla de solo lectura a **formulario editable**.
- **Por qué:** que el usuario configure los pines desde el navegador.
- **Cómo:** formulario con `fetch` GET/PUT contra `/api/v1/pins`; deshabilita los
  inputs si `locked` (PCB fija).
- **Impacto:** requiere sesión de admin o token para guardar.
- **Referencia:** v3.19.0.

### `include/sensors/SensorRegistry.hpp` + `.cpp` (v3.20.0 → v3.23.0)
- **Qué:** `fromJson`, `load`, `save` (NVS `ghsensors`); `buildFromConfig` sigue
  poblando el catálogo desde los flags.
- **Por qué:** catálogo de sensores editable y persistido.
- **Cómo:** `fromJson` actualiza solo entradas existentes por `id` (no permite
  inyectar drivers inexistentes).
- **Impacto:** `PUT /api/v1/sensors/catalog` (protegido) edita
  `enabled/address/zone/bus_index/read_interval_ms`.
- **Referencia:** v3.20.0/v3.23.0.

### `src/sensors/SensorManager.cpp` (v3.21.0 → v3.23.0)
- **Qué:** las direcciones I²C y el `enabled` de **todos** los sensores se leen del
  catálogo (patrón `addr(id, def)` y `on(id, flag)`).
- **Por qué:** que el catálogo sea la única fuente de verdad del hardware.
- **Cómo:** lambdas al inicio de `begin()` y `update()` con fallback a `PinConfig`/
  `SystemConfig`.
- **Impacto:** cambiar una dirección por web cambia el driver real (tras reinicio).
- **Referencia:** v3.21.0/v3.22.0/v3.23.0.

### `include/core/Version.hpp` (v3.22.0)
- **Qué:** nuevo flag `GH_PINS_LOCKED` (0 = público, 1 = PCB fija).
- **Por qué:** proteger el cableado en placas fabricadas sin cerrar el proyecto.
- **Cómo:** `PUT /api/v1/pins` responde 403 y `/pins` deshabilita el formulario.
- **Impacto:** los sensores/actuadores siguen configurables.
- **Referencia:** v3.22.0.

### `include/hardware/HardwareManager.hpp` + `.cpp` (v3.24.0 → v3.26.0)
- **Qué:** catálogo de nodos editable/persistido (NVS `ghhw`); `fromJson` permite
  **agregar** nodos de tipos conocidos (HC595/HC165/MCP23017/MCP23S17/ADC).
- **Por qué:** configurar expansores desde la web.
- **Cómo:** `kind` validado con `hwKindFromString` y `bus` derivado del tipo.
- **Impacto:** `PUT /api/v1/hardware` protegido; el `kind` desconocido se ignora.
- **Referencia:** v3.24.0/v3.26.0.

### `src/main.cpp` (v3.25.0 → v3.29.0)
- **Qué:** pools de expansores instanciados desde el catálogo: `mcpPool[4]`,
  `spiPool[4]`, `adcPool[4]`, `input` (74HC165) y `SpiManager`.
- **Por qué:** que el catálogo no sea solo metadatos sino que cree los drivers.
- **Cómo:** `snapshot()` del `HardwareManager` + bucle por `kind` + `begin()` de cada
  driver; `ActuatorManager` recibe el pool MCP23017 completo.
- **Impacto:** `ActuatorManager::writeChannel` mapea canales 32..95 a
  device 0..3 / pin 0..15.
- **Referencia:** v3.25.0 – v3.29.0.

### `include/system/OtaManager.hpp` + `.cpp` (v3.16.0)
- **Qué:** `applyFromUrl(url, sha, Client*)` con GET HTTP manual sobre `Client*`.
- **Por qué:** `HTTPClient` solo acepta `WiFiClient`; `Update` es agnóstico.
- **Cómo:** parseo de URL + headers, soporte `Content-Length` y `chunked`.
- **Impacto:** OTA por HTTP funciona sobre WiFi **y** Ethernet (W5500).
- **Referencia:** v3.16.0.

### `include/sensors/ModbusGateway.hpp` + `.cpp` (v3.17.0)
- **Qué:** gateway RS485 con polling multi-esclavo por perfiles.
- **Por qué:** leer varios esclavos Modbus sin bloquear.
- **Cómo:** tabla de esclavos desde `ModbusProfileRegistry` + `tick()` con intervalo.
- **Impacto:** `GET /api/v1/modbus/gateway`; conversión de tipos con escala/offset.
- **Referencia:** v3.17.0.
