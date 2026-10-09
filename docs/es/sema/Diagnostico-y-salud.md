---
tags:
  - sema
  - diagnostico
  - salud
---

# Diagnóstico y salud

> **Tipo:** API | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Esta página documenta qué mide SEMA sobre su propia salud, qué endpoints lo exponen
y cómo interpretar cada campo. El foco está en cuatro endpoints:
`/api/v1/health`, `/api/v1/diagnostics`, `/api/v1/system` y `/api/v1/capabilities`.

## 1. Los cuatro endpoints de diagnóstico

| Endpoint | Método | Autenticación | Buffer JSON | Handler |
|----------|--------|---------------|-------------|---------|
| `/api/v1/health` | GET | **No requiere** | 768 B | `HttpServer::onHealth()` — `HttpServer.cpp:1355` |
| `/api/v1/diagnostics` | GET | **No requiere** | 1 024 B | `HttpServer::onDiagnostics()` — `HttpServer.cpp:2026` |
| `/api/v1/system` | GET | **No requiere** | 2 048 B | `HttpServer::onSystem()` — `HttpServer.cpp:1401` |
| `/api/v1/capabilities` | GET | **No requiere** | 1 024 B | `HttpServer::onCapabilities()` — `HttpServer.cpp:1988` |
| `/api/v1/status` | GET | **No requiere** | 256 B | `HttpServer::onStatus()` — `HttpServer.cpp:1344` |
| `/api/v1/network` | GET | **No requiere** | 384 B | `HttpServer::onNetwork()` — `HttpServer.cpp:2003` |
| `/api/v1/energy` | GET | **No requiere** | 256 B | `HttpServer::onEnergy()` — `HttpServer.cpp:2017` |

⚠️ Nota de seguridad: estos cuatro endpoints **no llaman a `webAuthed()`**. Cualquiera
que llegue a la IP del equipo obtiene estado, diagnóstico, identidad y capacidades
sin credenciales, incluso con login configurado. Otros endpoints de solo lectura
(`/api/v1/gpio`) sí están abiertos, y los de escritura y los que exponen claves
(`/api/v1/config`) sí requieren autenticación. El criterio de `webAuthed()` es:
si `security.password` está vacío, **todo** queda abierto.

## 2. `GET /api/v1/health`

Devuelve el estado agregado de salud más el detalle de tareas vigiladas.

### 2.1 Estructura de la respuesta

```json
{
  "status": "HEALTHY",
  "uptime_s": 12345,
  "free_heap": 198456,
  "sensors": { "total": 6, "online": 6, "error": 0 },
  "tasks": [
    { "name": "core.heartbeat", "healthy": true },
    { "name": "sensors.read",   "healthy": true }
  ]
}
```

| Campo | Origen en el código | Significado |
|-------|---------------------|-------------|
| `status` | `HealthMonitor::status()` | `"HEALTHY"` \| `"DEGRADED"` \| `"ERROR"` |
| `uptime_s` | `HealthMonitor::uptimeSeconds()` = `(millis() - startMs_) / 1000` | Segundos desde `health_.begin()`, no desde el encendido real |
| `free_heap` | `ESP.getFreeHeap()` | Heap libre en el instante de la respuesta |
| `sensors.total` | `SensorManager::count()` | Sensores registrados (incluye los que nunca respondieron) |
| `sensors.online` | `SensorManager::onlineCount()` | Sensores cuyo `healthy()` devuelve `true` |
| `sensors.error` | `count() − onlineCount()` | Calculado en el handler, no es un contador interno |
| `tasks[]` | `HealthMonitor::taskName(i)` + `taskHealthy()` | Hasta 10 entradas; `healthy` compara el último latido contra el timeout |

Nota: `tasks[]` **no incluye el timeout** ni el tiempo desde el último latido. Para
saber por qué una tarea figura como no sana hay que conocer los timeouts del código
(30 s para `core.heartbeat`, 60 s para `sensors.read`; ver
[Tareas-y-concurrencia](Tareas-y-concurrencia.md)).

### 2.2 Estados de salud y su lógica exacta

`HealthMonitor::status()` (`src/core/HealthMonitor.cpp`, líneas 67-81) evalúa en
**este orden** y devuelve el primero que se cumple:

| Orden | Condición | Resultado |
|-------|-----------|-----------|
| 1 | `total > 0` **y** `online == 0` | `ERROR` — hay sensores configurados y ninguno responde |
| 2 | `total > 0` **y** `online < total` | `DEGRADED` — al menos un sensor no responde |
| 3 | `!heartbeatHealthy()` | `DEGRADED` — el heartbeat no latió en los últimos 30 000 ms |
| 4 | `!allTasksHealthy()` | `DEGRADED` — alguna tarea vigilada superó su timeout |
| 5 | ninguno de los anteriores | `HEALTHY` |

Consecuencias que conviene tener presentes:

- **Si no hay sensores registrados (`total == 0`), el estado nunca será `ERROR` ni
  `DEGRADED` por sensores**: cae a `HEALTHY` si los latidos están al día.
- **`sensors.read` siempre late.** El callback del `Scheduler` llama a
  `health_.taskHeartbeat("sensors.read")` al final del ciclo sin importar si algún
  sensor falló. Que `tasks[]` diga `healthy: true` **no** implica que los sensores
  midan: eso lo dice `sensors.online` / `status`.
- `HealthMonitor::tick()` solo actualiza `lastTickMs_`; no hace ninguna
  comprobación activa.
- `setSensorStats()` se llama **solo** dentro del ciclo `sensors.read`. Entre el
  arranque y la primera ejecución (10 s) los contadores quedan en 0.

### 2.3 Cómo interpretar `status`

| Valor | Qué está pasando | Qué mirar |
|-------|------------------|-----------|
| `HEALTHY` | Todos los sensores responden y los latidos están al día | — |
| `DEGRADED` con `sensors.online < sensors.total` | Uno o más sensores no responden, pero la adquisición sigue | `GET /api/v1/sensors` (`catalog[].healthy`), `i2c_devices` en `/diagnostics` |
| `DEGRADED` con `sensors` completo | El heartbeat o una tarea vigilada se atrasó | `tasks[]`, tiempo del último ciclo |
| `ERROR` | Ningún sensor responde | Cableado, bus I²C, alimentación; ver `detectedDevices` |

⚠️ No hay endpoint para reiniciar un contador de salud ni para forzar una
re-evaluación: `status()` es una función pura que se calcula en cada consulta.

## 3. `GET /api/v1/diagnostics`

Es el endpoint "todo en uno" para soporte. Combina firmware, memoria, motivo de
reinicio, almacenamiento, contadores y el inventario del bus I²C.

### 3.1 Estructura de la respuesta

```json
{
  "firmware": "1.103.0",
  "hw": "rev0",
  "uptime_s": 12345,
  "free_heap": 198456,
  "reset_reason": 1,
  "health": "HEALTHY",
  "history": { "entries": 1520, "max": 10000 },
  "tasks": 2,
  "modules": 0,
  "events": 37,
  "i2c_devices": [
    { "address": "0x76", "model": "BME280" },
    { "address": "0x44", "model": "SHT40" }
  ]
}
```

| Campo | Origen | Interpretación |
|-------|--------|----------------|
| `firmware` | `SEMA_FW_VERSION` | Versión del firmware (`include/core/Version.hpp`) |
| `hw` | `SEMA_HW_VERSION` | Revisión de hardware, hoy `"rev0"` |
| `uptime_s` | `millis() / 1000` | Segundos desde el encendido (a diferencia de `health.uptime_s`) |
| `free_heap` | `ESP.getFreeHeap()` | Heap libre actual |
| `reset_reason` | `esp_reset_reason()` como entero | Motivo del último reinicio (ver §3.3) |
| `health` | `HealthMonitor::status()` | Mismo valor que en `/health` |
| `history.entries` | `HistoryStore::count()` | Entradas de `/history.jsonl` contadas al abrir la SD o tras cada append |
| `history.max` | `HistoryStore::maxEntries()` | Tope de entradas, hoy **10 000** |
| `tasks` | `Scheduler::count()` | **Cantidad de tareas del planificador, hoy 2.** No son tareas FreeRTOS |
| `modules` | `ModuleRegistry::count()` | Módulos registrados. **Hoy siempre 0** (ver [Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md)) |
| `events` | `EventLog::events().size()` | Eventos en RAM, máximo 100 |
| `i2c_devices[]` | `SemaCore::detectedDevices()` | Resultado de `I2cScanner::scan()` **ejecutado una sola vez, en `setup()`** |

⚠️ Tres advertencias de interpretación importantes:

1. **`i2c_devices` es una foto del arranque.** `I2cScanner::scan()` se llama una
   única vez en `SemaCore::setup()` (`SemaCore.cpp:113`) y llena
   `detectedDevices_`. Si conectás un sensor después de encender, no aparece hasta
   reiniciar. Además, `history.entries` sin SD habilitada es 0 y no hay indicador
   de "SD ausente" en este endpoint: para eso hay que mirar
   `history_available` en `/api/v1/system`.
2. **`tasks: 2` no significa "2 tareas FreeRTOS".** Es la cantidad de callbacks
   registrados en el `Scheduler` (`core.heartbeat`, `sensors.read`).
3. **`modules: 0` es el valor correcto hoy**, no un error: no hay módulos
   registrados.

### 3.2 Qué más conviene consultar junto con esto

| Pregunta | Endpoint | Campo |
|----------|----------|-------|
| ¿Hay microSD y funciona? | `/api/v1/system` | `sd_enabled` (config) y `history_available` (resultado real de `enableSd`) |
| ¿Cuántos sensores responden? | `/api/v1/health` | `sensors.online` / `total` |
| ¿Qué sensores hay y de qué modelo? | `/api/v1/sensors` | `catalog[]` con `id`, `model`, `interface`, `healthy` |
| ¿La red está arriba? | `/api/v1/network` | `mode`, `connected`, `ip`, `rssi`, `ethernet.*` |
| ¿Cuántos reinicios acumulados? | `/api/v1/system` | `restart_count` |
| ¿Qué eventos/alarmas hubo? | `/api/v1/events`, `/api/v1/alarms` | arrays con `ts`, `type`, `source`, `rule`, `severity`, `value` |

### 3.3 `reset_reason`: cómo leer el entero

`esp_reset_reason()` devuelve un valor del enum `esp_reset_reason_t` del ESP-IDF.
Tanto `/api/v1/system` como `/api/v1/diagnostics` lo entregan **como número crudo**
(`static_cast<int>`), sin traducir ni agregar un nombre. La correspondencia:

| Valor | Nombre IDF | Qué significa | Qué hacer |
|-------|------------|---------------|-----------|
| 0 | `ESP_RST_UNKNOWN` | Motivo no determinable | — |
| 1 | `ESP_RST_POWERON` | Encendido o reset por pin EN | Arranque normal |
| 2 | `ESP_RST_EXT` | Reset externo | Revisar el hardware de reset |
| 3 | `ESP_RST_SW` | Reinicio por software (`ESP.restart()`) | Normal si reiniciaste desde la web (`/api/v1/restart`, OTA, cambio de red) |
| 4 | `ESP_RST_PANIC` | Excepción o **panic del TWDT** | Buscar bloqueos > 10 s: revisar webhook/MQTT lentos (ver [Tareas-y-concurrencia](Tareas-y-concurrencia.md)) |
| 5 | `ESP_RST_INT_WDT` | Watchdog de interrupciones | Tarea con interrupciones deshabilitadas demasiado tiempo |
| 6 | `ESP_RST_TASK_WDT` | Task watchdog | Igual que `PANIC`, pero disparado por el TWDT de tarea |
| 7 | `ESP_RST_WDT` | Otro watchdog | — |
| 8 | `ESP_RST_DEEPSLEEP` | Despertar de deep sleep | Esperado si usás `PowerManager::sleep()` |
| 9 | `ESP_RST_BROWNOUT` | Caída de tensión | Revisar alimentación y cable USB/fuente |
| 10 | `ESP_RST_SDIO` | Reset por SDIO | Poco probable en este hardware |

Relacionado: `/api/v1/energy` expone `wake_reason`, que es el valor de
`esp_sleep_get_wakeup_cause()` (motivo del **despertar**, no del reinicio). Son dos
cosas distintas y conviene no confundirlas.

En la práctica: `1` (power-on) y `3` (software) son normales; `4` (PANIC) o `6`
(task WDT) indican un bloqueo de más de 10 s en la tarea del loop; `9` (brownout)
apunta a alimentación. Los valores `5`, `7`, `8` y `10` corresponden a
interrupciones, otros watchdogs, deep sleep y SDIO, y son poco frecuentes acá.

## 4. `GET /api/v1/system`

Identidad, versiones y topología de hardware tal como quedó compilada.

| Campo | Origen | Notas |
|-------|--------|-------|
| `id`, `name` | `station.id`, `station.name` | Identidad de la estación (defaults `SEMA-001`, `Estación Norte`) |
| `firmware` | `SEMA_FW_VERSION` | `"1.103.0"` |
| `hw` | `SEMA_HW_VERSION` | `"rev0"` |
| `config_schema` | `SEMA_CONFIG_SCHEMA_VERSION` | `1` |
| `protocol` | `SEMA_PROTOCOL_VERSION` | `1` |
| `board` | `SEMA_BOARD_ID` | `"esp32-wroom-4mb"` / `"esp32-s3-8mb"` / `"esp32-wroom32u-16mb"` |
| `flash_mb` | `SEMA_FLASH_MB` | 4 / 8 / 16 |
| `pins_from_file` | `SEMA_PINS_FROM_FILE != 0` | Origen de pines por archivo |
| `demo` | `SEMA_DEMO != 0` | Modo demo activo |
| `native_eth` | `SEMA_NATIVE_ETH != 0` | 1 = MAC Ethernet nativa (LAN8720A por RMII) |
| `spi_sck`, `spi_miso`, `spi_mosi` | `SEMA_SPI_*` | Pines del bus SPI de Arduino (LoRa / SD) |
| `sd_cs` | `SEMA_PIN_SD_CS` | Chip-select de la microSD |
| `sd_enabled` | `storage.sdEnabled` | Lo que pide la configuración |
| `history_available` | `HistoryStore::sdEnabled()` | Lo que realmente logró inicializar la SD |
| `shift_enabled` | `SEMA_USE_SHIFT != 0` | Hoy **false** por defecto |
| `esp_temp` | `temperatureRead() − 10.0f` | Temperatura del sensor interno del SoC, con una corrección aproximada de −10 °C indicada en el propio código |
| `restart_count` | `SemaCore::restartCount()` | Contador `boots` persistido en NVS; se incrementa en cada `setup()` |
| `reset_reason` | `esp_reset_reason()` | Ver §3.3 |
| `wifi_ssid`, `wifi_ip`, `wifi_host`, `wifi_mdns` | WiFiManager + config | Estado de red |
| `reserved_pins[]` | Calculado en el handler | Pines no disponibles para sensores/salidas |
| `firmware_file` | `"sema_" + versión + "_" + board + ".bin"` | Nombre esperado del binario de OTA |

Sobre `reserved_pins[]`: siempre incluye los tres pines del bus SPI
(`SEMA_SPI_SCK`, `SEMA_SPI_MISO`, `SEMA_SPI_MOSI`). Si la board tiene Ethernet
nativo **y** `ethernet.enabled` está activo, agrega los 10 pines de la interfaz
RMII: `SEMA_PIN_ETH_MDC`, `MDC/MDIO`, `TXD0`, `TXD1`, `TX_EN`, `RXD0`, `RXD1`,
`CRS_DV`, `RX_ER` y `REF_CLK`.

Sobre `esp_temp`: es la lectura del sensor de temperatura interno del ESP32 con una
corrección fija de −10 °C. No es una medición calibrada; sirve para detectar
calentamiento, no como termómetro.

## 5. `GET /api/v1/capabilities`

Lista las capacidades declaradas por el firmware. El handler recorre el enum
`Capability::Count` y agrega el nombre de las que están activas
(`capabilityName()` en `include/core/Capability.hpp`):

```json
{ "capabilities": ["wifi","bluetooth","adc","dac","pcnt","ledc_pwm","i2c","spi","uart","can","rtc_gpio","deep_sleep","dual_core"] }
```

`SemaCore::setup()` declara **13** capacidades (`SemaCore.cpp:65-78`):
`WiFi`, `Bluetooth`, `Adc`, `Dac`, `Pcnt`, `LedcPwm`, `I2c`, `Spi`, `Uart`, `Can`,
`RtcGpio`, `DeepSleep`, `DualCore`.

Capacidades que el enum define y que **no** se declaran en este perfil:
`Ethernet` (aunque el hardware la tenga, no se marca como capacidad),
`Psram` (el ESP32 clásico no la tiene) e `Ieee802154` (Zigbee usa un coprocesador
externo CC2652P2 por UART, así que la radio 802.15.4 del ESP32 no se usa).

⚠️ La lista de capacidades es **estática**: se declara una vez en `setup()` con
valores fijos para el ESP32 clásico, ignorando `SEMA_BOARD_ID` y `SEMA_NATIVE_ETH`.
El comentario del código lo reconoce: "En una iteración posterior esto se carga
desde el Board/Chip Profile en lugar de declararse aquí" (`SemaCore.cpp:63-64`).
Es decir, en una placa S3 la respuesta de `/capabilities` seguiría siendo la del
ESP32 clásico.

## 6. Qué mide —y qué no mide— el `HealthMonitor`

`include/core/HealthMonitor.hpp` y `src/core/HealthMonitor.cpp` (92 líneas)
implementan un monitor deliberadamente simple.

### 6.1 Lo que mide

| Dimensión | Cómo | Fuente |
|-----------|------|--------|
| Liveness del ciclo principal | `tick()` guarda `lastTickMs_`; sano si latió en los últimos 30 000 ms | Scheduler, cada 5 s |
| Sensores en línea | `setSensorStats(online, total)` — simple copia de dos enteros | Scheduler, cada 10 s |
| Watchdog jerárquico por tarea | Hasta 10 entradas `{name, timeoutMs, lastMs}`; sano si `millis() − lastMs < timeoutMs` | `registerTask()` / `taskHeartbeat()` |

No mide heap, temperatura, uso de CPU, errores por bus ni latencias. Esos datos se
obtienen de `ESP.getFreeHeap()` y `temperatureRead()` directamente en
`/api/v1/diagnostics` y `/api/v1/system`.

### 6.2 Lo que no mide

| Dimensión ausente | Consecuencia |
|-------------------|--------------|
| Uso de CPU o tiempo por etapa | No se puede saber qué consume el ciclo desde la API |
| Heap mínimo histórico (watermark) | Solo hay heap instantáneo; un pico fugaz no se detecta |
| Contadores de error por sensor o por bus | `sensors.error` es una resta derivada, no un acumulador |
| Latencia de publicación (webhook/MQTT) | Los fallos de publicador no se reflejan en la salud |
| Estado del bus I²C en runtime | `i2c_devices` es del arranque |
| Persistencia de la salud | No se guarda histórico de estados de salud |

## 7. Flujo de diagnóstico recomendado

```text
1. GET /api/v1/system        → ¿qué firmware, qué placa, hay SD?, ¿cuántos reinicios?
2. GET /api/v1/diagnostics   → reset_reason + free_heap + history + i2c_devices
3. GET /api/v1/health        → status + sensores online/total + tareas sanas
4. GET /api/v1/sensors       → detalle por sensor (modelo, interfaz, healthy)
5. GET /api/v1/events        → eventos registrados (máx. 100)
6. GET /api/v1/alarms        → solo las alarmas disparadas por reglas
7. GET /api/v1/network       → WiFi/Ethernet y RSSI
8. POST /api/v1/restart      → reiniciar (requiere autenticación)
```

Caso típico — "las gráficas están vacías":

```text
/api/v1/system → "sd_enabled": true  y  "history_available": false
   ⇒ la config pide SD pero SD.begin() falló → revisar tarjeta, CS y bus SPI
/api/v1/system → "sd_enabled": false
   ⇒ la microSD no está habilitada en la configuración → histórico deshabilitado a propósito
/api/v1/diagnostics → "history": { "entries": 0, "max": 10000 }
   ⇒ coherente con lo anterior: no hay entradas guardadas
```

Caso típico — "el equipo se reinicia solo":

```text
/api/v1/system → "reset_reason": 4 (ESP_RST_PANIC) o 6 (ESP_RST_TASK_WDT)
   ⇒ bloqueo > 10 s en la tarea del loop. Sospechosos en orden de probabilidad:
     webhook HTTP lento (2 s por medición), broker MQTT ausente (15 s de socket),
     lectura/agregación del histórico sobre una SD lenta.
/api/v1/system → "reset_reason": 3 (ESP_RST_SW)
   ⇒ reinicio pedido por software: OTA, /api/v1/restart o cambio de config de red.
/api/v1/system → "reset_reason": 9 (ESP_RST_BROWNOUT)
   ⇒ problema de alimentación.
```

---

## Ver también

- [Arquitectura](Arquitectura.md) · [Tareas-y-concurrencia](Tareas-y-concurrencia.md)
- [Rendimiento-y-memoria](Rendimiento-y-memoria.md) · [API-REST](API-REST.md)
- [Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md) · [Solucion-de-problemas](Solucion-de-problemas.md)
- [Identidad-y-estados](Identidad-y-estados.md) · [Almacenamiento-e-historico](Almacenamiento-e-historico.md)
- [Guia-de-pines](Guia-de-pines.md) · [Compatibilidad](Compatibilidad.md)
