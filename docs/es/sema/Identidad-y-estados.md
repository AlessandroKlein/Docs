---
tags:
  - sema
  - identidad
---

# Identidad y estados

> **Tipo:** Referencia | **Estado:** Estable
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Identidad de la estación y **todos** los estados que el firmware expone o maneja:
salud, calidad de medición, energía, red, sesión web, eventos y capacidades.

## 1. Identidad de la estación

Fuentes: `station.id` y `station.name` en la config (`include/core/ConfigManager.hpp`),
los `#define` de `include/core/Version.hpp` y `include/hw/HwProfile.hpp`.

| Campo | Origen | Default / valor en v1.103.0 | Dónde se ve |
|-------|--------|------------------------------|-------------|
| `station.id` | config `station.id` | `"SEMA-001"` | `GET /api/v1/status` (`station`), `/api/v1/system` (`id`) |
| `station.name` | config `station.name` | `"Estación Norte"` | `status` (`name`), `system` (`name`) |
| `firmware` | `SEMA_FW_VERSION` | `"1.103.0"` | `status`, `system`, `diagnostics`, `backup` |
| `hw` | `SEMA_HW_VERSION` | `"rev0"` | `system`, `diagnostics` |
| `config_schema` | `SEMA_CONFIG_SCHEMA_VERSION` | `1` | `system` |
| `protocol` | `SEMA_PROTOCOL_VERSION` | `1` | `system` (protocolo con el Servidor Central) |
| `board` | `SEMA_BOARD_ID` | `esp32-wroom-4mb` · `esp32-wroom32u-16mb` · `esp32-s3-8mb` | `system` |
| `flash_mb` | `SEMA_FLASH_MB` | `4` · `16` · `8` | `system` |
| `firmware_file` | construido | `sema_1.103.0_<board>.bin` | `system` |
| `network.hostname` | config | `"sema-001"` | `system` (`wifi_host`), mDNS `<hostname>.local` |
| `restart_count` | NVS, clave `boots` | contador acumulativo | `system` |

La identidad se propaga por: `GET /api/v1/status`, `GET /api/v1/system`, el bloque
`station` de `GET /api/v1/config`, los metadatos de `GET /api/v1/backup`
(`backup_format`, `backup_version`, `firmware`, `timestamp`) y el prefijo serial del
arranque (`SEMA v1.103.0 (hw rev0, schema 1, protocol 1)`).

### 1.1 `station.id` en las mediciones

El Modelo Canónico tiene un campo `stationId` (`Measurement.hpp:29`) y los
publicadores lo emiten como `station_id`… ⚠️ **pero nadie lo asigna**: en todo el
repositorio no hay una sola escritura a `Measurement::stationId` (los únicos usos
son la declaración, `MqttPublisher.cpp:47` y `HttpPublisher.cpp:20`). En v1.103.0 el
campo viaja **siempre como cadena vacía** (`"station_id": ""`), tanto en MQTT como
en el webhook. Tampoco aparece en `GET /api/v1/sensors` ni en el WebSocket.

⚠️ **Consecuencia:** un receptor externo no puede distinguir de qué estación viene
una medición por el payload; hay que inferirlo del topic MQTT o de la URL del
webhook.

### 1.2 Identidad de hardware en tiempo de compilación

| Macro | Dónde | Efecto |
|-------|-------|--------|
| `BOARD_ESP32_WROOM` / `BOARD_ESP32_WROOM32U` / `BOARD_ESP32_S3` | `platformio.ini` (`build_flags`) | Fija `SEMA_BOARD_ID`, `SEMA_FLASH_MB` y `SEMA_NATIVE_ETH`; si no se define ninguna, `#error` |
| `SEMA_NATIVE_ETH` | `HwProfile.hpp` | `1` = LAN8720A por RMII; `0` = W5500 por SPI |
| `SEMA_PINS_FROM_FILE` | `HwProfile.hpp` (default `0`) | `1` = pines fijos de `HwProfile.hpp` y la web los ignora; `0` = pines por web |
| `SEMA_FIXED_HARDWARE` | `include/core/BoardProfile.hpp` (default `0`) | `1` = se ignora el catálogo `sensors[]` de la config y se usa el catálogo fijo |
| `SEMA_DEMO` | `HwProfile.hpp` (default `0`) | `1` = valores ficticios en `/api/v1/sensors` y `/api/v1/history` (entorno `demo`) |
| `SEMA_MODBUS_ISOLATED` | `HwProfile.hpp` (default `1`) | Transceiver RS485: `TD501D485H` (aislado) vs `SN65HVD75DR` |

La placa `esp32-wroom-32u` compila con `SEMA_PINS_FROM_FILE=1`; las otras tres, con
`0`. `GET /api/v1/system` informa el estado efectivo en `board`, `flash_mb`,
`pins_from_file`, `demo` y `native_eth`.

## 2. Versiones y compatibilidad

| Versión | Valor | Cambia cuando… |
|---------|-------|----------------|
| Firmware | `1.103.0` (SemVer, sin prefijo `v`) | Cada release |
| Hardware | `rev0` | Se revisa el diseño de la PCB |
| Esquema de configuración | `1` | Cambia la forma del JSON de config (se valida: otro valor ⇒ `validate()` falla ⇒ `PUT /api/v1/config` devuelve `400`) |
| Protocolo | `1` | Cambia el contrato con el Servidor Central |

El respaldo es autodescriptivo: `backup_format: "sema-backup"`,
`backup_version: 1`. Ver [Compatibilidad-de-versiones](Compatibilidad-de-versiones.md)
y [Backup-y-restauracion](Backup-y-restauracion.md).

## 3. Estados de salud (`HealthMonitor`)

Fuente: `include/core/HealthMonitor.hpp` + `src/core/HealthMonitor.cpp`.
Se expone en `GET /api/v1/health` (`status`) y `GET /api/v1/diagnostics` (`health`).

| Estado | Condición exacta (evaluada en este orden) |
|--------|-------------------------------------------|
| `ERROR` | `total > 0 && online == 0` — hay sensores configurados y **ninguno** responde |
| `DEGRADED` | `total > 0 && online < total` — alguno no responde |
| `DEGRADED` | Heartbeat vencido: `millis() - lastTickMs >= 30000` (el latido corre cada 5 s) |
| `DEGRADED` | Alguna tarea registrada superó su timeout (watchdog jerárquico) |
| `HEALTHY` | Ninguna de las anteriores |

Tareas registradas en v1.103.0 (`SemaCore::setup`):

| Tarea | Timeout | Late cada | Registrada en |
|-------|---------|-----------|---------------|
| `core.heartbeat` | **30.000 ms** | 5.000 ms | `SemaCore.cpp:180` |
| `sensors.read` | **60.000 ms** | 10.000 ms | `SemaCore.cpp:181` |

- `HealthMonitor::kMaxTasks = 10`: hasta 10 tareas monitoreadas.
- `taskHealthy(nombre)` devuelve `true` para una tarea **no registrada** (no se
  evalúa).
- `uptime_s` de `/api/v1/health` sale de `HealthMonitor` (`millis() - startMs_`), no
  del reloj NTP.
- Los contadores de sensores se actualizan en cada lectura
  (`health_.setSensorStats(onlineCount, count)`).
- El **watchdog de hardware** es independiente: `Watchdog::begin(10)` configura el
  TWDT de ESP-IDF con **10 s** y `loop()` lo alimenta en cada vuelta; si el loop se
  bloquea más de 10 s, el SoC se reinicia (no pasa por `HEALTHY/DEGRADED/ERROR`).
- ❌ No hay estados intermedios ni contadores de fallos por sensor: `health` es una
  foto instantánea, sin histéresis ni promedios.

## 4. Estados de sensores y calidad de medición

### 4.1 Salud por sensor

Cada entrada del catálogo de `GET /api/v1/sensors` lleva `healthy` (`bool`) y
`interface` (bus usado: `I2C`, `1-Wire`, `ADC`, `UART`, `PCNT`, `demo`…). El
`SensorManager` cuenta `onlineCount()` y `count()`; la diferencia es
`sensors.error` en `/api/v1/health`.

### 4.2 Quality Flags (`Quality`)

Ocho estados (`include/core/Measurement.hpp`), serializados con `qualityName()` y
leídos con `parseQuality()`:

| Enum | Nombre en JSON | Significado |
|------|----------------|-------------|
| `Valid` | `VALID` | Medición válida |
| `Invalid` | `INVALID` | Lectura inválida |
| `Stale` | `STALE` | Dato viejo (sin actualizar) |
| `Timeout` | `TIMEOUT` | El sensor no respondió a tiempo |
| `OutOfRange` | `OUT_OF_RANGE` | Fuera del rango de calibración |
| `CalibrationError` | `CALIBRATION_ERROR` | Error de calibración |
| `CommunicationError` | `COMMUNICATION_ERROR` | Error de bus/comunicación |
| `SensorDisconnected` | `SENSOR_DISCONNECTED` | Sensor ausente |

- `parseQuality()` devuelve `Valid` para cualquier cadena desconocida **y** para
  `nullptr` (no hay estado de error de parseo).
- Los valores por defecto de una `Measurement` nueva son `quality = Valid`,
  `sequence = 0`, `timestamp = 0`.
- `sequence` es un contador monotónico **por sensor**; las magnitudes derivadas y la
  entrada sintética `clock` publican `sequence = 0`.
- La calibración por canal (`CalibrationSpec`: `gain`, `offset`, `has_range`, `min`,
  `max`) se aplica antes de publicar; `has_range` con `min`/`max` es lo que produce
  `OUT_OF_RANGE`. Sin calibraciones configuradas, el firmware aplica por defecto
  `gain = 1.0`, `offset = 0`, rango `-40…85` a `EXT:temperature` e `INT:temperature`
  (`SemaCore::applyCalibrations`).

### 4.3 Detección I²C

En el arranque se escanea el bus I²C (`Wire.begin(SEMA_PIN_I2C_SDA, SEMA_PIN_I2C_SCL)`
→ `I2cScanner::scan`) y se guardan los dispositivos detectados con una **sugerencia
de modelo** por dirección. Se exponen en `GET /api/v1/diagnostics` como
`i2c_devices[]` (`address` en formato `"0x%02X"`, `model`). Es una detección
informativa: no crea sensores ni cambia la config.

## 5. Estados de energía

### 5.1 Perfiles (`EnergyProfile`)

| Enum | Nombre en JSON | Default |
|------|----------------|---------|
| `Performance` | `performance` | — |
| `Normal` | `normal` | ✅ (`profile_ = EnergyProfile::Normal`) |
| `LowPower` | `low_power` | — |
| `UltraLowPower` | `ultra_low_power` | — |

Se leen en `GET /api/v1/energy` (`profile`). ⚠️ **No hay endpoint ni página para
cambiar el perfil**: solo existe `PowerManager::setProfile()` como API interna, y
nadie lo llama. **Los perfiles no alteran ningún comportamiento del firmware** (no
modifican frecuencias, intervalos de sensores ni WiFi): son una etiqueta de estado.

### 5.2 Wake reasons (`esp_sleep_wakeup_cause_t`)

`PowerManager::wakeReason()` devuelve el valor crudo de
`esp_sleep_get_wakeup_cause()`, que se publica como número en `/api/v1/energy` y se
imprime por serial en el arranque (`Wake reason: N`).

| Valor | Constante | Origen |
|-------|-----------|--------|
| 0 | `ESP_SLEEP_WAKEUP_UNDEFINED` | No vino de deep sleep (arranque, reset, OTA) |
| 1 | `ESP_SLEEP_WAKEUP_ALL` | No es una causa real (se usa para deshabilitar fuentes) |
| 2 | `ESP_SLEEP_WAKEUP_EXT0` | Señal externa por RTC_IO — **el pluviómetro usa esta** (`enableRainWakeup` → `esp_sleep_enable_ext0_wakeup(pin, 1)`), nivel **HIGH** |
| 3 | `ESP_SLEEP_WAKEUP_EXT1` | Señal externa por RTC_CNTL |
| 4 | `ESP_SLEEP_WAKEUP_TIMER` | Timer RTC — **el que usa `PowerManager::sleep(s)`** |
| 5 | `ESP_SLEEP_WAKEUP_TOUCHPAD` | Touchpad |
| 6 | `ESP_SLEEP_WAKEUP_ULP` | Programa ULP |
| 7 | `ESP_SLEEP_WAKEUP_GPIO` | GPIO (solo light sleep en ESP32/S2/S3) |
| 8 | `ESP_SLEEP_WAKEUP_UART` | UART (solo light sleep) |
| 9 | `ESP_SLEEP_WAKEUP_WIFI` | WiFi (solo light sleep) |

⚠️ **El deep sleep no está cableado en el runtime:** `PowerManager::sleep()` existe
pero **no se llama desde ningún lado** (no hay tarea, ni endpoint, ni lógica de
ciclo de trabajo). Lo único que corre es `enableRainWakeup(config.energy.rainPin)`
en `setup()` **si `energy.rain_pin != 0`** (default `0`), que arma la fuente de
wakeup sin llegar a dormir nunca. En la práctica `wake_reason` es siempre `0`
salvo que se agregue el ciclo de sueño.

## 6. Estados de red

### 6.1 WiFi y Ethernet

| Campo (`GET /api/v1/network`) | Valores | Nota |
|-------------------------------|---------|------|
| `mode` | `"STA"` · `"AP"` | `"AP"` también cuando se pidió STA sin SSID |
| `connected` | `true` · `false` | En AP es **siempre `true`** |
| `ip` | IPv4 en texto | `localIP()` o `softAPIP()` |
| `rssi` | dBm (negativo) | **`0`** en modo AP |
| `ethernet.enabled` | `true` · `false` | `ethernet.enabled` de la config |
| `ethernet.connected` | `true` · `false` | `ETH.linkUp()` (nativo) o IP ≠ 0 (W5500) |
| `ethernet.ip` | IPv4 en texto | Vacío si no hay IP |

Estados adicionales sin campo propio: `wifi_mdns` y `wifi_host` en
`GET /api/v1/system`; el backoff de reconexión (2 s → 60 s) es interno y no se
expone. Ver [Conectividad-y-red](Conectividad-y-red.md).

### 6.2 Estados del bus de datos (CAN/LoRa/Zigbee/Modbus)

| Campo | Rutas | Valores |
|-------|-------|---------|
| `ready` | `/modbus`, `/can`, `/lora`, `/zigbee` | El driver/bus se inicializó correctamente |
| `received` | `/can`, `/lora`, `/zigbee` | Si en esa llamada apareció una trama/paquete/mensaje |
| `result` | `/modbus` | Código de ModbusMaster (`0` = éxito, `0xFF` = bus no listo) |
| `src` | `/zigbee` | Dirección corta de origen del último `AF_INCOMING_MSG` (`0` si no hay) |

No hay estado de error de bus (bus-off, CRC, watchdog del co-procesador) expuesto
por API. Ver [Comunicaciones-remotas](Comunicaciones-remotas.md).

## 7. Estados de sesión y autenticación (web)

| Situación | `webAuthed()` | Lo que ve el navegador |
|-----------|---------------|------------------------|
| `security.password` vacío | `true` (sin login configurado) | Dashboard completo con `window.SEMA_AUTH=true` |
| Sin sesión y con contraseña configurada | `false` | `/` sirve el **dashboard público** con `SEMA_AUTH=false` (menú y edición ocultos); `/sensors`, `/events` y `/config/*` sirven el **formulario de login**; la API responde `401` |
| Sesión válida (cookie `sema_auth`) | `true` | Dashboard completo. Expira a la hora, **deslizante** |
| Sesión expirada | `false` | Igual que "sin sesión" (`sessionStartMs_ = 0`) |
| `POST /login` con 5 fallos | — | Bloqueo de **60 s** → `429` |

Detalles de estado de la sesión: el token es de 16 hex aleatorios
(`esp_random` en `begin()`), viaja en `Cookie: sema_auth=<token>` con `HttpOnly` y
`SameSite=Strict`, y **no cambia** al iniciar/cerrar sesión (es el mismo de todo el
ciclo de arranque). `GET /logout` limpia la cookie con `Max-Age=0`. Ver
[Seguridad](Seguridad.md) y [API-REST](API-REST.md) §2.

## 8. Estados y severidades de evento

Fuente: `include/core/EventBus.hpp`. La API publica los **nombres**, no los enums:

| `EventType` | Nombre | | `Severity` | Nombre |
|-------------|--------|-|------------|--------|
| `Sensor` | `sensor` | | `Debug` | `DEBUG` |
| `Rain` | `rain` | | `Info` | `INFO` |
| `Lightning` | `lightning` | | `Notice` | `NOTICE` |
| `Battery` | `battery` | | `Warning` | `WARNING` |
| `Network` | `network` | | `Error` | `ERROR` |
| `Alarm` | `alarm` | | `Critical` | `CRITICAL` |
| `System` | `system` | | | |
| `Wake` | `wake` | | | |
| `Sleep` | `sleep` | | | |

Eventos que genera el firmware en v1.103.0:

| Evento | Tipo | Severidad | `source` | `value` | `rule` |
|--------|------|-----------|----------|---------|--------|
| Arranque | `system` | `INFO` | `core` | `0` | `boot` |
| Alarma por regla | `alarm` | `WARNING` | `sensorId` de la medición | `valor × 100` (entero) | nombre de la regla |

`EventLog` guarda hasta **100** eventos (buffer circular) y los persiste en
`/events.jsonl` (LittleFS). El único suscriptor de alarmas registrado imprime por
serial (`[ALARM] <regla> → <sensor> = <valor×100>`). ⚠️ Recordá que el `ts` de estos
eventos en la API es `millis()`, no época (ver [API-REST](API-REST.md) §6).

### 8.1 Reset reasons (`esp_reset_reason()`)

`GET /api/v1/system` (`reset_reason`) y `/api/v1/diagnostics` devuelven el entero.
La página `/config/system` lo traduce con esta tabla exacta:

| Valor | Nombre | Significado |
|-------|--------|-------------|
| 0 | `UNKNOWN` | Desconocido |
| 1 | `POWERON` | Encendido / power-on |
| 2 | `EXTERNAL` | Reset externo (`ESP_RST_EXT`) |
| 3 | `SOFTWARE` | Reset por software (`ESP.restart()`) |
| 4 | `PANIC` | Excepción / pánico |
| 5 | `INT_WDT` | Watchdog de interrupción |
| 6 | `TASK_WDT` | Watchdog de tareas (TWDT de 10 s de SEMA) |
| 7 | `WDT` | Otro watchdog |
| 8 | `DEEPSLEEP` | Despertar de deep sleep |
| 9 | `BROWNOUT` | Caída de tensión |
| 10 | `SDIO` | Reset por SDIO |

## 9. Estados de capacidades (`Capability`)

`GET /api/v1/capabilities` recorre el enum y devuelve las activas. Las 16
capacidades declaradas y su estado real en v1.103.0:

| Capacidad | Nombre JSON | ¿Activa? | | Capacidad | Nombre JSON | ¿Activa? |
|-----------|-------------|----------|-|-----------|-------------|----------|
| `WiFi` | `wifi` | ✅ | | `Uart` | `uart` | ✅ |
| `Bluetooth` | `bluetooth` | ✅ | | `Can` | `can` | ✅ |
| `Ethernet` | `ethernet` | ❌ | | `Psram` | `psram` | ❌ |
| `Adc` | `adc` | ✅ | | `RtcGpio` | `rtc_gpio` | ✅ |
| `Dac` | `dac` | ✅ | | `DeepSleep` | `deep_sleep` | ✅ |
| `Pcnt` | `pcnt` | ✅ | | `DualCore` | `dual_core` | ✅ |
| `LedcPwm` | `ledc_pwm` | ✅ | | `Ieee802154` | `ieee802154` | ❌ |
| `I2c` | `i2c` | ✅ | | | | |
| `Spi` | `spi` | ✅ | | | | |

⚠️ La lista se arma **a mano** en `SemaCore::setup()` con el perfil del ESP32
clásico: es idéntica en las tres placas (incluida la S3, que tiene W5500 pero
reporta `Ethernet` inactivo, y no reporta `psram` aunque la S3 tenga PSRAM). El
`Wire.begin()` tampoco usa los pines de la config: usa `SEMA_PIN_I2C_SDA`/`SCL`
(21/22) del `BoardProfile`.

## 10. Estados del dashboard (sesión web)

| Elemento | Comportamiento |
|----------|----------------|
| `window.SEMA_AUTH` | Inyectado por `onRoot()`: `true` con sesión (o sin login configurado), `false` sin sesión |
| `SEMA_AUTH === false` | Oculta la barra de navegación y la de edición; muestra solo tema + botón de login |
| Layout | Se guarda en NVS aparte (`saveDashboardLayout`); si no existe, el JS arma uno por defecto con la primera lectura y lo guarda |
| Auto-refresco | `/api/v1/status`, `/api/v1/system` y `/api/v1/sensors` cada **5 s**; gráficos cada **60 s** |
| Tema e idioma | `localStorage`: `sema_theme` (`light`/`dark`) y `sema_lang` (`es`/`en`) — del lado del cliente |

## 11. Tareas, módulos y eventos: recuento en `/api/v1/diagnostics`

| Campo | Origen | Valor típico |
|-------|--------|--------------|
| `tasks` | `Scheduler::count()` | **2** (`core.heartbeat`, `sensors.read`) |
| `modules` | `ModuleRegistry::count()` | **0** — `modules_.enableAll()` no registra ninguno |
| `events` | `EventLog::events().size()` | 1 (boot) + alarmas, tope 100 |
| `history.entries` / `history.max` | `HistoryStore::count()` / `maxEntries()` | `maxEntries` = **10000** |

## 12. Pendientes y limitaciones

| Tema | Estado |
|------|--------|
| `station_id` en mediciones y payloads | ⚠️ Campo siempre vacío |
| Endpoint/página para el perfil de energía | ❌ No existe (y los perfiles no cambian comportamiento) |
| Ciclo de deep sleep (`PowerManager::sleep`) | ❌ Definido pero nunca invocado |
| Estados de error de bus por API | ❌ No expuestos |
| Histéresis / histórico de salud | ❌ `health` es una foto instantánea |
| Refresh del token de sesión | ❌ Fijo por arranque |
| Capacidades por placa (Board/Chip Profile) | ⚠️ Lista fija del ESP32 clásico |
| Wake por `energy.rain_pin` | ⚠️ Se arma pero nunca se duerme |

---

## Ver también

- [API-REST](API-REST.md) · [Configuracion](Configuracion.md) · [Referencia-configuracion](Referencia-configuracion.md)
- [Diagnostico-y-salud](Diagnostico-y-salud.md) · [Enumeraciones-y-tipos](Enumeraciones-y-tipos.md) · [Compatibilidad-de-versiones](Compatibilidad-de-versiones.md)
- [Conectividad-y-red](Conectividad-y-red.md) · [Comunicaciones-remotas](Comunicaciones-remotas.md) · [MQTT-y-WebSocket](MQTT-y-WebSocket.md)
- [Seguridad](Seguridad.md) · [Energia-y-consumo](Energia-y-consumo.md) · [Alarmas-y-reglas](Alarmas-y-reglas.md)
