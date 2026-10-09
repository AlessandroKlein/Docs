---
tags:
  - sema
  - pruebas
  - validacion
---

# Pruebas y validación

> **Tipo:** Referencia | **Estado:** En desarrollo | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Cómo se comprueba hoy que SEMA funciona, qué **no** existe todavía y cómo se
implementaría la cobertura automatizada. Todo lo afirmado como existente se
verificó en el código; lo que falta está marcado explícitamente.

## 1. Estado actual: no hay tests automatizados

Verificado en el repo local el 2026-10-08:

| Comprobación | Comando / ruta | Resultado |
|--------------|----------------|-----------|
| Contenido de `test/` | `Get-ChildItem test -Recurse` | **Solo `test/README`** (529 B, el texto de plantilla de PlatformIO Unit Testing) |
| Tests escritos | `test/**/*.cpp` | **Ninguno** |
| Configuración de test runner | `Select-String platformio.ini -Pattern test` | **Sin coincidencias**: no hay `test_dir`, `test_build_src`, `test_filter` ni entorno `native` |
| Integración continua | `.github/` en el repo de código | **No existe** (el CI vive solo en el repo Docs, para publicar el sitio) |
| Menciones de testing en la spec | `README.md` | Sin `pio test`, `Unity` ni "PlatformIO Unit" |

```text
❌ No hay tests unitarios ni de integración.
❌ No hay CI que compile el firmware en cada push.
❌ No hay mocks de Arduino/ESP-IDF ni entorno nativo.
⚠️ test/README es la plantilla que genera PlatformIO, no documentación de SEMA.
```

Conclusión: **la validación de SEMA es hoy manual** (§2). Lo único "automático" es
la verificación de integridad del OTA y las validaciones en runtime (§3).

## 2. Cómo se valida en la práctica

### 2.1 Compilación

| Paso | Comando (documentado en `docs/GUIA.md` §4 y `docs/CONTINUACION.md` §3) | Criterio |
|------|------------------------------------------------------------------------|----------|
| Compilar | `pio run -e esp32doit-devkit-v1` | `SUCCESS` |
| Compilar por board | `pio run -e esp32-s3-devkitc-1` · `pio run -e esp32-wroom-32u` | `SUCCESS` en cada entorno |
| Compilar demo | `pio run -e demo` | `SUCCESS` (extiende `esp32doit-devkit-v1` + `-D SEMA_DEMO=1`) |
| Flashear | `pio run -e esp32doit-devkit-v1 -t upload` | Binario en `.pio/build/<env>/firmware.bin` |

`docs/CONTINUACION.md` §4 registra el criterio histórico "Flash ≈63 %, RAM ≈16 %".
⚠️ **No verificable en esta revisión**: exige compilar y la tarea no incluye esa
restricción de recursos en el repo. Tomarlo como referencia, no como umbral.

Entornos disponibles (`platformio.ini`): `esp32doit-devkit-v1` (default, 4 MB),
`esp32-s3-devkitc-1` (8 MB), `esp32-wroom-32u` (16 MB, `SEMA_PINS_FROM_FILE=1`) y
`demo`.

### 2.2 Revisión del log serie (115200 baudios)

`SemaCore::setup()` imprime, en orden: banner
`SEMA v<fw> (hw <hw>, schema <n>, protocol <n>)`, estación, `Config válida: sí|no`,
`Wake reason: <n>`, capacidades (`ADC/PCNT/DualCore/CAN`), el resultado del escaneo
I²C línea por línea (`I²C 0x<addr> → <modelo|desconocido>`), y al final
`Sensores: N registrados, M activos`, `Web local: http://<ip>/`, `mDNS: http://<host>.local/`
y `Módulos registrados: N`.

Son los chequeos manuales más baratos: config válida, sensores que responden
(`M` = `Sensor::healthy()`), módulos detectados en el bus y dirección web real.
Si `storage.sdEnabled` está activo, también se ve `MicroSD: conectada|no detectada (CS=<pin>)`.

### 2.3 Endpoints de diagnóstico

| Endpoint | Qué devuelve (verificado en `HttpServer.cpp`) |
|----------|-----------------------------------------------|
| `GET /api/v1/health` | `status` (`HEALTHY`/`DEGRADED`/`ERROR`), `uptime_s`, `free_heap`, `sensors{total,online,error}` y `tasks[]{name,healthy}` |
| `GET /api/v1/diagnostics` | `firmware`, `hw`, `uptime_s`, `free_heap`, `reset_reason`, `health`, `history{entries,max}`, `tasks`, `modules`, `events`, `i2c_devices[]{address,model}` |
| `GET /api/v1/system` | Identidad y versiones: `firmware`, `hw`, `config_schema`, `protocol`, `board`, `flash_mb`, `pins_from_file`, `demo`, `native_eth`, pines SPI, `sd_cs`, `sd_enabled`, `history_available`, `shift_enabled`, `esp_temp`, `restart_count`, `reset_reason`, `wifi_*`, `firmware_file`, `reserved_pins[]` |
| `GET /api/v1/capabilities` | Lista de capacidades declaradas (13 de 16) |
| `GET /api/v1/sensors` | Catálogo + últimas mediciones (con conversión de unidades) |
| `GET /api/v1/network` | Modo, conexión, IP, RSSI y estado de Ethernet |
| `GET /api/v1/energy` | `profile` y `wake_reason` |
| `GET /api/v1/history` · `/events` · `/alarms` | Histórico, eventos y alarmas |
| `GET /api/v1/status` | Resumen mínimo (estación, nombre, firmware, uptime) |

`/api/v1/health` se alimenta de tres fuentes reales: `HealthMonitor::status()`,
`SensorManager::onlineCount()` y el watchdog por tarea (§3).

### 2.4 Modo demo (`-D SEMA_DEMO=1`, entorno `demo`)

Sirve para validar la web completa sin sensores conectados:

| Efecto | Dónde | Detalle |
|--------|-------|---------|
| Catálogo sintético en `/api/v1/sensors` | `HttpServer.cpp:2058-2117` | 9 entradas (`ext` BME280, `int` SHT40, `co2` SCD30, `pm` PMS5003, `uv` VEML6075, `wind` WH-SP-WD, `rain` RG-9, `batt` ESP32-ADC, `solar` SOLAR) con `interface = "demo"` y `healthy = true` |
| Mediciones con onda senoidal | idem | 17 magnitudes (temperatura, humedad, presión, UVI, lux, CO₂, PM2.5, PM10, viento + ráfaga + dirección, lluvia, tensión de batería, radiación solar, clock) dentro del rango típico |
| Histórico sintético | `HttpServer.cpp:2294` | `/api/v1/history` genera series en vez de leer la SD |
| MCP23S17 forzado | `ConfigManager.cpp:385-391` | Si `mcp23s17_cs == 0`, lo pone en 5 con los 16 pines como salidas |

`GET /api/v1/system` expone `"demo": true` para no confundir un equipo demo con uno real.

### 2.5 Validaciones que el firmware hace solo (en runtime)

| Validación | Implementación | Qué hace ante el fallo |
|------------|----------------|------------------------|
| Esquema y campos de config | `ConfigManager::validate()` (`schemaVersion == 1`, `station.id` no vacío, `network.mode ∈ {STA, AP}`, `storage.backend ∈ {littlefs, flash, sd}`) | `apply()` devuelve `false`, **rollback** a la config previa; `applyJson()` → HTTP 400 `invalid config` |
| Escritura en NVS | `ConfigManager::save()` | Ante `NOT_ENOUGH_SPACE`: `store_.clear()` y reintento único (se pierde la config y el layout) |
| Rango de calibración | `applyCalibration()` | Marca la medición como `Quality::OutOfRange` |
| Salud del sistema | `HealthMonitor::status()` | `ERROR` si hay sensores y ninguno responde; `DEGRADED` si falta alguno, el heartbeat supera 30 s o una tarea venció |
| Loop bloqueado | `Watchdog::begin(10)` (`esp_task_wdt`) | Reinicia el SoC (panic handler del TWDT) |
| Tareas estancadas | `HealthMonitor::registerTask` + `taskHeartbeat` | `taskHealthy()` / `status()` pasan a degradado (no reinicia) |
| Integridad del OTA | `onOtaUpload` + `otaPartitionSha256` | Con cabecera `X-SHA256` (64 hex): si no coincide, `otaShaOk_ = false` → HTTP 400 `sha256 mismatch` y **no reinicia**. El hash se calcula sobre **toda la partición OTA** (`part->size`), no sobre la imagen: no es directamente el `sha256` del `.bin` del manifest |
| Publicación de reglas | `RuleEngine::evaluate` | Emite `Event` de alarma; no hay histéresis (ver §4) |

⚠️ El OTA solo verifica SHA-256 si el cliente manda `X-SHA256`; **no** hay firma
digital ni verificación de board (§4.3).

## 3. Qué falta

| Falta | Impacto | Vía de solución |
|-------|---------|-----------------|
| Tests unitarios | Ninguna fórmula ni parser tiene red de seguridad | Unity + PlatformIO Test Runner (§4) |
| Tests de integración en placa | Los buses (I²C, 1-Wire, UART, RS485, CAN, LoRa, Zigbee) solo se prueban a mano con hardware | Tests "on-target" con `pio test -e <board>` + hardware-in-the-loop |
| CI de compilación | Un cambio que rompe otra board se descubre al flashear | Workflow que corra `pio run` en las 4 configuraciones |
| Cobertura de código | No se mide | `--coverage` de PlatformIO sobre el entorno nativo |
| Tests de contrato de API | Las 53 rutas y sus JSON cambian sin test que avise | Tests de integración HTTP (cliente que pega a un equipo real o a un simulador) |
| Tests de migración de config | No hay migración (ver [Compatibilidad de versiones](Compatibilidad-de-versiones.md)) | Definir `migrate()` y sus tests |
| Umbrales de recursos en CI | El "≈63 % flash / ≈16 % RAM" es manual | `pio run -t size` + comparación automática |

## 4. Cómo se implementaría (Unity + PlatformIO)

Nada de esto existe hoy: es la propuesta concreta, paso a paso, usando las piezas
que PlatformIO ya trae.

### 4.1 Entorno nativo para lógica pura

```ini
[env:test]
platform = native
test_framework = unity
test_build_src = true
build_flags = -D UNIT_TEST
```

Con `test_build_src = true`, PlatformIO compila `src/` (excluyendo `main.cpp`) junto
con los tests. Los archivos que dependen de Arduino/ESP-IDF necesitan stubs
(`ArduinoFake`, `WString` o un `include/` de test con `String`, `millis()`,
`analogRead()`, `Wire`, `Preferences`, `SD`, `LittleFS`).

### 4.2 Candidatos a test unitario (lógica sin hardware)

| Objetivo | Archivo a testear | Casos sugeridos |
|----------|-------------------|-----------------|
| `DerivedEngine::dewPoint/heatIndex/saturationVaporPressure/absoluteHumidity` | `src/core/derived/DerivedEngine.cpp` | Valores de tabla (p. ej. 25 °C/50 % → rocío ≈13.9 °C), humedad 0 % y 100 %, temperaturas negativas |
| `DerivedCalculator::convertUnit` | `src/core/derived/DerivedCalculator.cpp` | Idempotencia con `imperial=false`; 0 °C = 32 °F; 1013.25 hPa ≈29.92 inHg; `rain_rate` → `in/h` |
| `DerivedCalculator::windVaneRawAngle/windDirection` | idem | Las 16 posiciones de la tabla de resistencias; normalización con `windNorthOffset` negativo y > 360 |
| `I2cScanner::modelForAddress` | `src/core/sensors/I2cScanner.cpp` | Las 10 direcciones mapeadas y una desconocida (`""`) |
| `parseQuality`/`qualityName`, `parseEventType`/`eventTypeName`, `parseSeverity`/`severityName`, `parseRuleOp` | headers `Measurement.hpp`, `EventBus.hpp`, `alarms/Rule.hpp` | Round-trip de todos los valores; `nullptr`; cadenas inválidas |
| `CapabilityManager` | `src/core/CapabilityManager.cpp` | Idempotencia de `set`, las 16 capacidades, `set(c, false)` |
| `ModuleRegistry` | `src/core/ModuleRegistry.cpp` | `id` duplicado → `false`; orden del ciclo `install→configure→enable→start`; `loopAll` solo en `Enabled`/`Running` |
| `RuleEngine` | `src/core/alarms/RuleEngine.cpp` | Los 4 operadores; `sensorId` vacío = comodín; `channelId` distinto no matchea; valor de evento `value*100` |
| `applyCalibration` | `src/core/Calibration.cpp` | `enabled=false` no toca el valor; `hasRange` y límites inclusivos/exclusivos |
| `ConfigManager::validate` + `applyJson` | `src/core/ConfigManager.cpp` | `schemaVersion != 1`; modos de red inválidos; `FakeKeyValueStore` que falle para provocar rollback |
| `EventLog` (serialización) | `src/core/events/EventLog.cpp` | Round-trip JSON de un `Event` (requiere mock de LittleFS) |

`sensors/`, `SemaCore`, `HttpServer`, `WiFiManager` y los gestores de bus **no** son
candidatos al entorno nativo sin un mock grande: van a test on-target o a pruebas
manuales.

### 4.3 Estructura propuesta de tests

```text
test/
├── test_derived/          # fórmulas derivadas y unidades
│   └── test_main.cpp
├── test_types/            # enums, parse*, calidad
├── test_alarms/           # RuleEngine + Calibration
├── test_config/           # validate + applyJson + rollback (FakeKeyValueStore)
└── test_on_target/        # #if defined(ARDUINO): escaneo I²C, NVS real, LittleFS
```

Ejecución:

```bash
pio test -e test                                  # lógica pura en el host
pio test -e esp32doit-devkit-v1 --test-port COM3 # on-target con hardware
```

### 4.4 CI propuesto (no existe)

```yaml
# .github/workflows/build.yml  (propuesta, NO está en el repo)
- pio run -e esp32doit-devkit-v1
- pio run -e esp32-s3-devkitc-1
- pio run -e esp32-wroom-32u
- pio run -e demo
- pio test -e test
```

Repositorio de referencia: el CI real del proyecto Docs usa `mkdocs build`; el repo
de firmware no tiene workflows.

## 5. Checklist de validación manual sugerido

| # | Chequeo | Cómo |
|--:|---------|------|
| 1 | Compila en las 4 configuraciones | `pio run -e …` → `SUCCESS` |
| 2 | El log de arranque muestra `Config válida: sí` | Serie 115200 |
| 3 | Los sensores esperados aparecen como activos | `Sensores: N registrados, M activos` con `M` = los conectados |
| 4 | Escaneo I²C coherente | Líneas `I²C 0x… → …` del arranque vs. hardware real |
| 5 | Red y web | `http://<ip>/` responde; `http://<host>.local/` si mDNS |
| 6 | Salud | `GET /api/v1/health` → `HEALTHY` con sensores conectados |
| 7 | Diagnóstico | `GET /api/v1/diagnostics` → `free_heap` > 0, `i2c_devices` correctos |
| 8 | Versiones correctas | `GET /api/v1/system` → `firmware`, `config_schema`, `protocol`, `board` |
| 9 | Escritura de config | `PUT /api/v1/config` con `X-API-Key` → `{"ok":true}` y persiste tras reiniciar |
| 10 | Backup/restauración | `GET /api/v1/backup` → editar → `POST /api/v1/backup` → `{"ok":true}` |
| 11 | OTA | `POST /api/v1/ota` con `X-API-Key` + `X-SHA256` → reinicia y reporta la versión nueva |
| 12 | Web completa sin hardware | `pio run -e demo -t upload` y recorrer las páginas |
| 13 | Alarma dispara | Forzar una medición fuera del umbral de `rules[]` y ver `[ALARM]` por serie |
| 14 | Watchdog | Bloquear el loop > 10 s y confirmar el reinicio (y que sube `restart_count`) |

---

## Ver también

- [Referencia de código](Referencia-de-codigo.md) · [Referencia de API interna](Referencia-API-interna.md) · [Diagnóstico y salud](Diagnostico-y-salud.md) · [Compatibilidad de versiones](Compatibilidad-de-versiones.md) · [Guía de desarrollo](Guia-de-desarrollo.md) · [Solucion de problemas](Solucion-de-problemas.md) · [API REST](API-REST.md) · [Tareas y concurrencia](Tareas-y-concurrencia.md)
