---
tags:
  - sema
  - metricas
---

# Estadísticas y métricas

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Dos familias de métricas: las **del proyecto** (tamaño del código, releases,
artefactos) y las que el **firmware expone en ejecución** para monitoreo y
diagnóstico.

## 1. Métricas del proyecto (v1.103.0)

| Métrica | Valor |
|---------|-------|
| Versión de firmware | `1.103.0` |
| Revisión de hardware | `rev0` |
| `SEMA_CONFIG_SCHEMA_VERSION` | `1` |
| `SEMA_PROTOCOL_VERSION` | `1` |
| Rango de desarrollo | `v0.1.0` (2026-10-03) → `v1.103.0` (2026-10-08) |
| Entradas en el CHANGELOG | 160 |
| Tags semver / releases en GitHub | 159 / 159 |
| Archivos de código (`.cpp`/`.h`/`.hpp`) | 111 (112 contando `include/README`) |
| Líneas de código | 10 952 (10 989 contando `include/README`) |
| Rutas HTTP registradas | 53 |
| Modelos de sensor | 17 |
| Entornos PlatformIO | 4 (`esp32doit-devkit-v1`, `esp32-s3-devkitc-1`, `esp32-wroom-32u`, `demo`) |

> El repositorio avanza a un ritmo de un release por incremento lógico: casi cada
> commit relevante genera versión, tag y release (ver [Versionado](../inicio/Versionado.md)).

## 2. Tamaño del código por área

| Área | Archivos | Líneas |
|------|:--------:|-------:|
| Núcleo (`src/core/*.cpp` + `include/core/*.hpp`) | 40 | 3 042 |
| Web y dashboard (`core/web`) | 3 | 4 205 |
| Sensores (`core/sensors`) | 41 | 1 955 |
| Almacenamiento (`core/storage`) | 5 | 462 |
| Magnitudes derivadas (`core/derived`) | 4 | 400 |
| Eventos y alarmas (`core/events`, `core/alarms`) | 5 | 268 |
| Publicadores (`core/publishers`) | 7 | 234 |
| Red (`core/network`) | 2 | 124 |
| Runtime (`core/runtime`) | 2 | 68 |
| Perfil de hardware (`include/hw`) | 1 | 176 |
| `src/main.cpp` | 1 | 18 |
| **Total** | **111** | **10 952** |

Archivos más grandes (líneas):

| Archivo | Líneas | Qué contiene |
|---------|-------:|--------------|
| `src/core/web/HttpServer.cpp` | 2 637 | Servidor web, 53 rutas, dashboard y páginas de config embebidas |
| `src/core/ConfigManager.cpp` | 483 | Defaults, parseo, serialización, validación y schema |
| `src/core/SemaCore.cpp` | 392 | Arranque, tareas, lazo de medición, poda y agregación |
| `src/core/storage/HistoryStore.cpp` | 304 | Histórico JSONL, rotación, retención y agregados |
| `include/core/ConfigManager.hpp` | 269 | Estructuras de configuración (fuente de las claves JSON) |
| `src/core/derived/DerivedCalculator.cpp` | 224 | Fórmulas de magnitudes derivadas |
| `src/core/web/GridstackAssets.h` | ~1 460 | Gridstack CSS/JS embebidos en PROGMEM (gzip) |

## 3. Artefactos de firmware

`firmware_manifest.json` publica un binario por placa con su SHA-256:

| Board | Chip | Flash | `firmware.bin` |
|-------|------|:-----:|---------------:|
| `esp32-wroom-4mb` | ESP32 | 4 MB | 1.390 MB |
| `esp32-s3-8mb` | ESP32-S3 | 8 MB | 1.335 MB |
| `demo` (mismo WROOM, `-D SEMA_DEMO=1`) | ESP32 | 4 MB | 1.391 MB |

Particiones (`partitions_4mb.csv`, 4 MB): `nvs` 0x5000 · `otadata` 0x2000 ·
`app0` 0x1C0000 · `app1` 0x1C0000 · `spiffs` 0x70000.

**Compilación real de `esp32doit-devkit-v1` (v1.103.0, 2026-10-08):**

| Métrica | Valor |
|---------|-------|
| Flash (app) | 1 450 949 B de 1 835 008 B → **79,1 %** |
| RAM estática | 56 136 B de 327 680 B → **17,1 %** |
| Flash libre en `app0` | ≈ 384 KB |
| Tiempo de compilación | 38,99 s |

> Los valores de ≈63 % / ≈16 % que cita `docs/GUIA.md` corresponden a una revisión
> anterior: la medición vigente es la de arriba. El detalle está en
> [Rendimiento y memoria](Rendimiento-y-memoria.md).

## 4. Métricas en ejecución

### 4.1 `GET /api/v1/diagnostics`

| Métrica | Descripción |
|---------|-------------|
| `firmware` | Versión del firmware |
| `hw` | Revisión de hardware |
| `uptime_s` | Segundos encendido |
| `free_heap` | Heap libre (bytes) |
| `reset_reason` | Código de la última causa de reinicio (`esp_reset_reason()`) |
| `health` | Estado de salud agregado |
| `history.entries` / `history.max` | Entradas actuales y límite del histórico |
| `tasks` | Tareas registradas en el `Scheduler` |
| `modules` | Módulos registrados |
| `events` | Eventos en memoria |
| `i2c_devices[]` | Dispositivos I²C detectados (dirección + modelo sugerido) |

### 4.2 `GET /api/v1/health`

| Métrica | Descripción |
|---------|-------------|
| `status` | `HEALTHY` · `DEGRADED` · `ERROR` |
| `uptime_s` | Uptime |
| `free_heap` | Heap libre |
| `sensors.total` | Sensores registrados |
| `sensors.online` | Sensores que responden |
| `sensors.error` | Sensores con error (`total − online`) |

### 4.3 Estado de salud (`HealthMonitor`)

| Estado | Criterio |
|--------|----------|
| `HEALTHY` | Heartbeat al día y todos los sensores online |
| `DEGRADED` | Algún sensor offline o heartbeat retrasado |
| `ERROR` | Heartbeat perdido (bloqueo) o fallo grave |

### 4.4 Causas de reinicio (`esp_reset_reason()`)

| Código | Significado |
|:------:|-------------|
| 1 | Power-on |
| 2 | Reset externo (pin EN) |
| 3 | Software reset (`ESP.restart()`) |
| 4 | Exception / panic |
| 5 | Interrupt watchdog |
| 6 | Task watchdog |
| 7 | Other watchdog |
| 8 | Deep-sleep wake |

### 4.5 Perfiles energéticos (`PowerManager`)

| Perfil | Nombre en la API |
|--------|------------------|
| Performance | `performance` |
| Normal (default) | `normal` |
| Low power | `low_power` |
| Ultra low power | `ultra_low_power` |

## 5. Límites del sistema

| Límite | Valor | Fuente |
|--------|-------|--------|
| Entradas de histórico | 10 000 por defecto (`maxEntries_`) | `HistoryStore.hpp` |
| Entradas de `EventLog` | 100 por defecto | `EventLog.hpp` |
| Retención de histórico | `storage.retention_days` (0 = deshabilitada) | `SemaCore`, `HistoryStore` |
| Agregación del histórico | buckets de 3 600 s invocados desde `SemaCore` | `SemaCore.cpp` |
| Documento JSON de configuración | hasta 16 384 B (`DynamicJsonDocument`) | `ConfigManager.cpp` |
| Sesión web | 3 600 000 ms (1 h, deslizante) | `HttpServer.cpp` |
| Rate limit de login | 5 intentos / 60 s | `HttpServer.cpp` |
| Canales MCP23017 | 16 (2 puertos × 8) | `ConfigManager.hpp` (`pinModes[16]`) |
| Salidas 74HC595 | 8 por integrado | `ConfigManager.hpp` (`pinModes[8]`) |
| Tabla de la veleta | 8 resistencias | `ConfigManager.hpp` (`windResistors[8]`) |
| Puertos UART del MAX14830 | 4 (0–3) | `ConfigManager.hpp` (`uartPort`) |

## 6. Uso típico

```bash
curl http://sema-001.local/api/v1/diagnostics
curl http://sema-001.local/api/v1/health
curl http://sema-001.local/api/v1/system
```

---

## Ver también

- [Rendimiento y memoria](Rendimiento-y-memoria.md) · [Diagnóstico y salud](Diagnostico-y-salud.md)
- [CHANGELOG](CHANGELOG.md) · [Registro de cambios](Registro-de-cambios.md) · [Evolución](Evolucion.md)
