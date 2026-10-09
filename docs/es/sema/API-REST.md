---
tags:
  - sema
  - api
---

# API REST

> **Tipo:** API | **Estado:** Estable
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Referencia **exhaustiva** de la API HTTP local de SEMA. Todo lo que sigue está
verificado contra `src/core/web/HttpServer.cpp` (2637 líneas) y
`include/core/web/HttpServer.hpp`, que es donde se registran las rutas
(`HttpServer::begin`, líneas 52–121).

## 1. Consideraciones generales

| Dato | Valor |
|------|-------|
| Servidor | `WebServer` de Arduino-ESP32 (`::WebServer server_`) |
| Puerto HTTP | **80** (el `WebServer` de Arduino siempre usa el 80) |
| Base de las APIs | `/api/v1` |
| Codificación | JSON UTF-8 en cuerpo de respuesta (`application/json`) |
| Cuerpo de entrada | `server_.arg("plain")` → JSON crudo, sin `form-urlencoded` |
| Cabeceras recolectadas | `X-API-Key` y `Cookie` (`server_.collectHeaders`) |
| Timeout de sesión | **3.600.000 ms (1 h)**, *deslizante* (se refresca en cada request válido) |
| 404 | `onNotFound()` → `404 {"error":"not found"}` |
| Modo demo | `-D SEMA_DEMO=1` reemplaza `/api/v1/sensors` y `/api/v1/history` por datos ficticios |

El dashboard embebido es HTML servido por el mismo servidor (ver §14). Todo el
frontend (JS/CSS de Gridstack) está **embebido en el firmware**: no hay assets
externos ni filesystem web.

## 2. Autenticación

Hay **dos mecanismos independientes** y tres funciones de control
(`HttpServer.cpp:1280–1301` y `1465–1494`):

| Función | Acepta | Regla |
|---------|--------|-------|
| `authorized()` | `X-API-Key` | Compara contra `security.api_key`, `security.server_key` y los valores de `security.extra_keys`. Si las **tres** están vacías → devuelve `true` (primera configuración) |
| `sessionAuthorized()` | `Cookie: sema_auth=<token>` | Token de 16 hex aleatorios (`esp_random`) generado en `begin()`; expira a la hora; se refresca al validar |
| `webAuthed()` | cookie **o** `X-API-Key` | `sessionAuthorized() \|\| authorized()`. Si `security.password` está vacío → `true` |

Consecuencias prácticas:

- **`server_key`** (clave del Servidor Central) es aceptada como `X-API-Key` en
  **todas** las rutas que pasan por `webAuthed()`, incluidas las de escritura de
  configuración.
- Las rutas de **solo lectura** de estado (`status`, `health`, `system`,
  `capabilities`, `network`, `energy`, `diagnostics`, `sensors`, `history`,
  `events`, `alarms`, `gpio` GET, `shift` GET, `modbus`, `can` GET, `lora` GET,
  `zigbee` GET) **no llaman a ningún control de acceso**: son públicas.
- `POST /api/v1/ota` es la **única** ruta que usa `authorized()` en vez de
  `webAuthed()`: **la cookie de sesión no sirve para OTA**. Ver §12.

### 2.1 Login y rate limiting

| Ruta | Método | Cuerpo | Respuestas |
|------|--------|--------|------------|
| `/login` | GET | — | `200` con el formulario HTML |
| `/login` | POST | `application/x-www-form-urlencoded`: `username`, `password` | `302` + `Set-Cookie: sema_auth=<token>; Path=/; HttpOnly; SameSite=Strict` si coincide; `401` con el formulario si no; `429 text/plain` si está bloqueado |
| `/logout` | GET | — | `302` a `/` y `Set-Cookie: sema_auth=; Path=/; Max-Age=0; SameSite=Strict` |

- Usuario esperado: `security.username` o `"admin"` si está vacío.
- Contraseña esperada: `security.password`. **Si está vacía no se puede iniciar
  sesión** (`ok` exige `sec.password.length() > 0`): sin contraseña el acceso web
  ya está abierto por `webAuthed()`.
- Rate limiting (D-0048): a los **5 intentos fallidos** se bloquea el login
  durante **60 s** (`lockoutUntilMs_ = now + 60000`); el contador se resetea al
  bloquear. La respuesta del bloqueo es `429 "Demasiados intentos. Reintentá más tarde."`.

## 3. Mapa completo de rutas

`A` = autenticación. `pública` = sin control; `web` = `webAuthed()` (cookie o
`X-API-Key`); `key` = `authorized()` (solo `X-API-Key`).

| Método | Ruta | A | Área | Condición |
|--------|------|---|------|-----------|
| GET | `/api/v1/status` | pública | estado | — |
| GET | `/api/v1/health` | pública | estado | — |
| GET | `/api/v1/system` | pública | sistema | — |
| GET | `/api/v1/capabilities` | pública | sistema | — |
| GET | `/api/v1/network` | pública | red | — |
| GET | `/api/v1/energy` | pública | energía | — |
| GET | `/api/v1/diagnostics` | pública | diagnóstico | — |
| GET | `/api/v1/sensors` | pública | sensores | — |
| GET | `/api/v1/history` | pública | histórico | — |
| GET | `/api/v1/events` | pública | eventos | — |
| GET | `/api/v1/alarms` | pública | eventos | — |
| GET | `/api/v1/gpio` | pública | GPIO | — |
| POST | `/api/v1/gpio` | web | GPIO | — |
| GET | `/api/v1/shift` | pública | expansores | — |
| POST | `/api/v1/shift` | web | expansores | — |
| GET | `/api/v1/config` | web | configuración | — |
| PUT | `/api/v1/config` | web | configuración | — |
| POST | `/api/v1/config/network` | web | configuración | — |
| POST | `/api/v1/config/system` | web | configuración | — |
| POST | `/api/v1/config/sensors` | web | configuración | — |
| POST | `/api/v1/config/io` | web | configuración | — |
| POST | `/api/v1/config/buses` | web | configuración | — |
| GET | `/api/v1/wifi/scan` | web | red | — |
| POST | `/api/v1/security/keys` | web | seguridad | — |
| GET | `/api/v1/update/check` | web | OTA | — |
| GET | `/api/v1/backup` | web | backup | — |
| POST | `/api/v1/backup` | web | backup | alias de `PUT /config` |
| POST | `/api/v1/restart` | web | sistema | — |
| POST | `/api/v1/ota` | key | OTA | — |
| POST | `/api/v1/wind/north` | web | veleta | — |
| POST | `/api/v1/wind/resistors` | web | veleta | — |
| GET | `/api/v1/dashboard/layout` | web | dashboard | — |
| POST | `/api/v1/dashboard/layout` | web | dashboard | — |
| GET | `/api/v1/modbus` | pública | bus Modbus | `SEMA_USE_MODBUS` |
| GET | `/api/v1/can` | pública | bus CAN | `SEMA_USE_CAN` |
| POST | `/api/v1/can` | web | bus CAN | `SEMA_USE_CAN` |
| GET | `/api/v1/lora` | pública | radio LoRa | `SEMA_USE_LORA` |
| POST | `/api/v1/lora` | web | radio LoRa | `SEMA_USE_LORA` |
| GET | `/api/v1/zigbee` | pública | radio Zigbee | `SEMA_USE_ZIGBEE` |
| POST | `/api/v1/zigbee` | web | radio Zigbee | `SEMA_USE_ZIGBEE` |
| GET | `/gridstack.min.css` | pública | assets | gzip |
| GET | `/gridstack-all.min.js` | pública | assets | gzip |

Las cuatro rutas de bus están envueltas en `#if SEMA_USE_MODBUS|CAN|LORA|ZIGBEE`
(`HttpServer.cpp:106–120`): si el flag vale `0`, la ruta **no existe** y responde
`404 {"error":"not found"}`. `SEMA_USE_SHIFT` **no** condiciona `/api/v1/shift`.
Las cinco placas de `platformio.ini` compilan con los cuatro flags en `1`.

## 4. Estado y sistema

### 4.1 `GET /api/v1/status`

Buffer `DynamicJsonDocument(256)`. Es la identidad mínima de la estación.

```json
{ "station": "SEMA-001", "name": "Estación Norte", "firmware": "1.103.0", "uptime_s": 12345 }
```

### 4.2 `GET /api/v1/health`

Buffer `768`. `sensors.error` es `count() - onlineCount()`.

```json
{
  "status": "HEALTHY",
  "uptime_s": 120,
  "free_heap": 210000,
  "sensors": { "total": 6, "online": 6, "error": 0 },
  "tasks": [
    { "name": "core.heartbeat", "healthy": true },
    { "name": "sensors.read", "healthy": true }
  ]
}
```

`status` ∈ `HEALTHY` · `DEGRADED` · `ERROR` (criterios en
[Identidad-y-estados](Identidad-y-estados.md)). Las tareas listadas son las
registradas en el `HealthMonitor`: en v1.103.0 son solo `core.heartbeat`
(timeout 30 s) y `sensors.read` (timeout 60 s).

### 4.3 `GET /api/v1/system`

Buffer `2048`. Es el endpoint de identidad más completo.

```json
{
  "id": "SEMA-001",
  "name": "Estación Norte",
  "firmware": "1.103.0",
  "hw": "rev0",
  "config_schema": 1,
  "protocol": 1,
  "board": "esp32-wroom-4mb",
  "flash_mb": 4,
  "pins_from_file": false,
  "demo": false,
  "native_eth": true,
  "spi_sck": 14,
  "spi_miso": 12,
  "spi_mosi": 15,
  "sd_cs": 4,
  "sd_enabled": false,
  "history_available": false,
  "shift_enabled": false,
  "esp_temp": 41.2,
  "restart_count": 12,
  "reset_reason": 1,
  "wifi_ssid": "MiRed",
  "wifi_ip": "192.168.1.50",
  "wifi_host": "sema-001",
  "wifi_mdns": true,
  "reserved_pins": [14, 12, 15],
  "firmware_file": "sema_1.103.0_esp32-wroom-4mb.bin"
}
```

| Campo | Origen |
|-------|--------|
| `board` / `flash_mb` / `native_eth` | `SEMA_BOARD_ID`, `SEMA_FLASH_MB`, `SEMA_NATIVE_ETH` (`include/hw/HwProfile.hpp`) |
| `spi_sck` / `spi_miso` / `spi_mosi` | `SEMA_SPI_SCK`, `SEMA_SPI_MISO`, `SEMA_SPI_MOSI` (14/12/15 en WROOM; 12/13/11 en S3) |
| `esp_temp` | `temperatureRead() - 10.0f` (corrección aproximada del sensor interno) |
| `reset_reason` | `esp_reset_reason()` como entero (0 `UNKNOWN`, 1 `POWERON`, 3 `SW`, 4 `PANIC`, …) |
| `history_available` | `core_->history().sdEnabled()` |
| `reserved_pins` | Siempre SCK/MISO/MOSI; además los 10 pines RMII **solo** si `SEMA_NATIVE_ETH && SEMA_USE_ETHERNET` y `ethernet.enabled` |
| `firmware_file` | `"sema_" + SEMA_FW_VERSION + "_" + SEMA_BOARD_ID + ".bin"` |

### 4.4 `GET /api/v1/capabilities`

Buffer `1024`. Recorre el enum `Capability` y agrega el nombre de las activas.

```json
{ "capabilities": ["wifi","bluetooth","adc","dac","pcnt","ledc_pwm","i2c","spi","uart","can","rtc_gpio","deep_sleep","dual_core"] }
```

⚠️ **Discrepancia conocida:** `Capability::Ethernet`, `Capability::Psram` y
`Capability::Ieee802154` **nunca se activan**: `SemaCore::setup()` declara el
perfil base del ESP32 clásico con una lista fija (`SemaCore.cpp:65–78`). En la
placa S3 (W5500 por SPI) la lista devuelta es idéntica. `capabilityName()`
devuelve `"ethernet"`, `"psram"` e `"ieee802154"`, pero no se emiten hoy.

### 4.5 `GET /api/v1/network`

Buffer `384`.

```json
{ "mode": "STA", "connected": true, "ip": "192.168.1.50", "rssi": -55,
  "ethernet": { "enabled": false, "connected": false, "ip": "" } }
```

`mode` es `"AP"` cuando `WiFiManager::isAp()` es verdadero — es decir, también
cuando se pidió `STA` pero **el SSID está vacío** (cae a AP de emergencia). En AP
`connected` siempre es `true` y `rssi` vale `0`.

### 4.6 `GET /api/v1/energy`

Buffer `256`. `wake_reason` es numérico (`esp_sleep_get_wakeup_cause()`).

```json
{ "profile": "normal", "wake_reason": 0 }
```

`profile` ∈ `performance` · `normal` · `low_power` · `ultra_low_power`
(`energyProfileName`). El default de arranque es `normal`; **no hay endpoint para
cambiarlo** (solo `PowerManager::setProfile()` interno). `wake_reason` es el
`esp_sleep_wakeup_cause_t` crudo: 0 `UNDEFINED`, 2 `EXT0`, 3 `EXT1`, 4 `TIMER`,
5 `TOUCHPAD`, 6 `ULP`, 7 `GPIO`, 8 `UART`, 9 `WIFI` (ver
[Identidad-y-estados](Identidad-y-estados.md) §5).

### 4.7 `GET /api/v1/diagnostics`

Buffer `1024`.

```json
{
  "firmware": "1.103.0", "hw": "rev0", "uptime_s": 3600, "free_heap": 198432,
  "reset_reason": 1, "health": "HEALTHY",
  "history": { "entries": 1200, "max": 10000 },
  "tasks": 2, "modules": 0, "events": 7,
  "i2c_devices": [ { "address": "0x76", "model": "BME280" } ]
}
```

`tasks` es la cantidad de tareas del `Scheduler`, `modules` la del
`ModuleRegistry` (vacío en v1.103.0: `modules_.enableAll()` no registra ninguno) y
`events` el tamaño del `EventLog`. `i2c_devices` viene del escaneo I²C del
arranque (`detectedDevices()`), con la dirección formateada `"0x%02X"`.

## 5. Sensores e histórico

### 5.1 `GET /api/v1/sensors`

Buffer `8192`. Devuelve catálogo + mediciones actuales + magnitudes derivadas +
la entrada sintética del reloj.

```json
{
  "catalog": [ { "id": "EXT", "model": "BME280", "interface": "I2C", "healthy": true } ],
  "measurements": [
    { "sensor_id": "EXT", "channel_id": "temperature", "measurement": "temperature",
      "value": 23.4, "unit": "°C", "quality": "VALID", "sequence": 123 },
    { "sensor_id": "EXT", "channel_id": "dew_point", "measurement": "dew_point",
      "value": 11.2, "unit": "°C", "quality": "VALID", "sequence": 0 },
    { "sensor_id": "clock", "channel_id": "0", "measurement": "clock",
      "value": 1760000000, "unit": "epoch", "quality": "VALID", "sequence": 0 }
  ],
  "units": "metric"
}
```

- `unit` se convierte según `system.units` (`DerivedCalculator::convertUnit`): con
  `"imperial"` la temperatura sale en `°F`, el viento en `mph`, etc.
- `quality` usa `qualityName(Quality)` (ver [Identidad-y-estados](Identidad-y-estados.md)).
- Con `SEMA_DEMO=1` la respuesta completa es ficticia (9 sensores `interface:"demo"`
  y ~18 magnitudes + derivadas), y `sequence` de las derivadas es `0`.

### 5.2 `GET /api/v1/history`

| Query | Valores | Default | Notas |
|-------|---------|---------|-------|
| `limit` | 1–**3000** | `50` | Fuera de rango se ignora y queda 50 |
| `series` | `recent` \| `aggregated` | `recent` | `aggregated` lee los promedios horarios |
| `format` | `json` \| `csv` | `json` | `csv` agrega `Content-Disposition: attachment; filename=sema_history.csv` y `Content-Type: text/csv` |

```json
{ "history": [ { "ts": 1760000000, "sensor": "EXT", "channel": "temperature",
                 "measurement": "temperature", "value": 23.4, "unit": "°C",
                 "quality": "VALID", "seq": 123 } ] }
```

CSV real (cabecera + filas, `value` con 4 decimales):

```csv
ts,sensor,channel,measurement,value,unit,quality,seq
1760000000,EXT,temperature,temperature,23.4000,°C,VALID,123
```

El dashboard usa `GET /api/v1/history?limit=3000` y el botón "CSV histórico" usa
`?limit=3000&format=csv`. Si la microSD está deshabilitada, la lista vuelve vacía
(no se usa memoria interna). En `SEMA_DEMO=1` se devuelven 320 muestras ficticias
y **se ignoran `limit`, `series` y `format`**.

## 6. Eventos y alarmas

### 6.1 `GET /api/v1/events`

Buffer `8192`. Todo el `EventLog` (máximo **100** entradas).

```json
{ "events": [ { "ts": 123456, "type": "alarm", "source": "EXT",
                "rule": "high_temp", "severity": "WARNING", "value": 4150 } ] }
```

⚠️ `ts` es **`millis()` (uptime en ms)**, no epoch: `Event.timestampMs` se asigna
con `millis()` (`RuleEngine.cpp:35`, `SemaCore.cpp:210`).
`value` es `int32_t` y, para alarmas disparadas por reglas, vale
**`valor_medido × 100`** (`static_cast<int32_t>(m.value * 100.0f)`): `4150` = 41,50.
`type` ∈ `sensor` · `rain` · `lightning` · `battery` · `network` · `alarm` ·
`system` · `wake` · `sleep`; `severity` ∈ `DEBUG` · `INFO` · `NOTICE` · `WARNING`
· `ERROR` · `CRITICAL`. `rule` es el `correlationId` (el nombre de la regla).

### 6.2 `GET /api/v1/alarms`

Buffer `8192`. Igual que `events` pero filtrando `type == "alarm"`; **no incluye
la clave `type`**:

```json
{ "alarms": [ { "ts": 123456, "source": "EXT", "rule": "high_temp",
                "severity": "WARNING", "value": 4150 } ] }
```

## 7. Configuración

### 7.1 `GET /api/v1/config` (web)

Devuelve el JSON completo de configuración **incluidas las claves**
(`security.api_key`, `security.server_key`, `security.password`). Buffer `16384` en
`ConfigManager::serialize`. Claves de primer nivel reales (el detalle campo por campo
está en [Referencia-configuracion](Referencia-configuracion.md)):

`schema_version` · `station` · `network` · `system` · `storage` · `security` ·
`energy` · `publishers` · `rules` · `calibrations` · `sensors` · `gpio` ·
`mcp23s17_cs` · `mcp23s17_pins` · `shift_registers` · `spi_expanders` · `modbus` ·
`can` · `lora` · `zigbee` · `ethernet` · `i2c_sda` · `i2c_scl`.

Esqueleto de la respuesta (valores = defaults de fábrica):

```json
{
  "schema_version": 1,
  "station": { "id": "SEMA-001", "name": "Estación Norte" },
  "network": { "mode": "STA", "ssid": "", "password": "", "hostname": "sema-001",
               "mdns": true, "ip": "", "gateway": "", "subnet": "", "dns": "" },
  "system": { "timezone": "America/Argentina/Buenos_Aires", "ntp_server": "pool.ntp.org",
              "log_level": "INFO", "units": "metric", "lang": "es", "altitude": 0,
              "wind_north_offset": 0, "wind_direction_pin": 0, "wind_rpull": 10000,
              "wind_resistors": [8 valores] },
  "storage": { "backend": "littlefs", "retention_days": 30, "sd_enabled": false, "sd_cs": 4 },
  "security": { "api_key": "", "server_key": "", "username": "", "password": "", "extra_keys": "{}" },
  "energy": { "rain_pin": 0 },
  "publishers": { "webhook_url": "", "mqtt_host": "", "mqtt_port": 1883,
                  "mqtt_topic": "sema/measurement", "mqtt_user": "", "mqtt_pass": "" },
  "rules": [], "calibrations": [], "sensors": [], "gpio": [],
  "mcp23s17_cs": 0, "mcp23s17_pins": [16 valores],
  "shift_registers": [], "spi_expanders": [],
  "modbus": { "enabled": false, "rx": 16, "tx": 17, "de_re": 0, "uart": 0, "uart_port": 0,
              "baud": 9600, "slave_id": 1, "register": 0, "count": 4 },
  "can": { "enabled": false, "tx": 5, "rx": 4, "speed": 500000 },
  "lora": { "enabled": false, "cs": 5, "rst": 14, "dio1": 26, "busy": 27,
            "frequency": 915, "bandwidth": 125, "spreading": 7, "coding_rate": 5, "tx_power": 14 },
  "zigbee": { "enabled": false, "rx": 16, "tx": 17, "uart": 0, "uart_port": 0, "baud": 115200 },
  "ethernet": { "enabled": false, "mdc": 23, "mdio": 18, "phy_addr": 1, "power": -1,
                "cs": 5, "rst": -1, "irq": 4, "sck": 18, "miso": 19, "mosi": 21 },
  "i2c_sda": 21, "i2c_scl": 22
}
```

`rules` y `calibrations` quedan vacíos por defecto, pero **no** están vacíos en
runtime: sin reglas configuradas el núcleo carga en memoria
`high_temp` (`EXT`/`temperature`/`gt`/`40`) y sin calibraciones aplica
`gain 1.0 / offset 0 / rango -40…85` a `EXT:temperature` e `INT:temperature`
(`SemaCore::applyRules` / `applyCalibrations`); esos defaults **no** se reflejan en
este JSON hasta que se guardan.

El layout del dashboard **no** está en este JSON: vive en una clave NVS aparte
(§13).

### 7.2 `PUT /api/v1/config` (web)

Aplica un JSON completo de forma **transaccional** (`ConfigManager::applyJson` →
`validate()` + `save()` con rollback).

```
PUT /api/v1/config
Content-Type: application/json
Cookie: sema_auth=<token>

{ "schema_version": 1, "station": { "id": "SEMA-001", "name": "Norte" }, ... }
```

| Respuesta | Cuándo |
|-----------|--------|
| `200 {"ok":true}` | Aplicado y persistido |
| `400 {"error":"body required"}` | Sin cuerpo |
| `400 {"error":"invalid config"}` | JSON inválido o `validate()` falla |
| `401 {"error":"unauthorized"}` | Sin sesión ni `X-API-Key` |

Al aplicarse, el handler re-aplica en caliente: sensores, reglas, calibraciones,
GPIO, publicadores, shift y —según flags— Modbus, CAN, LoRa, Zigbee y Ethernet
(`HttpServer.cpp:1871–1897`). **Nunca reinicia.**

Reglas de validación (`ConfigManager::validate`): `schema_version == 1`,
`station.id` no vacío, `network.mode` ∈ `STA`/`AP`,
`storage.backend` ∈ `littlefs`/`flash`/`sd`. Cualquier otra clave se ignora
silenciosamente o toma su default (`|` de ArduinoJson).

### 7.3 `POST /api/v1/config/network` (web)

| Campo | Default si falta |
|-------|------------------|
| `mode` | `"STA"` |
| `ssid`, `password`, `hostname`, `ip`, `gateway`, `subnet`, `dns` | `""` |
| `mdns` | `true` |

```json
{ "mode": "STA", "ssid": "MiRed", "password": "secreto", "hostname": "sema-001", "mdns": true }
```

Respuestas: `200 {"ok":true}` · `400 {"error":"body required"}` ·
`400 {"error":"invalid json"}` · `401` · `500 {"error":"config apply failed"}`.
⚠️ **Reinicia sola**: tras responder hace `delay(300)` y `ESP.restart()`. También
se reinicia si solo cambiaste `hostname` o `mdns` (la página de red usa este mismo
endpoint para mDNS). Buffer de parseo: `512`.

### 7.4 `POST /api/v1/config/system` (web)

Buffer `512`. Claves aceptadas y defaults: `timezone` →
`"America/Argentina/Buenos_Aires"`, `ntp_server` → `"pool.ntp.org"`, `units` →
`"metric"`, `lang` → `"es"`, `sd_enabled` → `false`, `sd_cs` → `4`. **No reinicia**;
`sd_enabled` se aplica recién al reiniciar (`HistoryStore::enableSd` corre en
`setup()`), y la propia web avisa "Guardado (reiniciá para aplicar)".

```json
{ "timezone": "America/Argentina/Buenos_Aires", "ntp_server": "pool.ntp.org" }
```

### 7.5 `POST /api/v1/config/sensors` (web)

Buffer `8192`. El array `sensors` **reemplaza** la lista completa
(`next.sensors.clear()`). Claves por sensor y defaults:

| Clave | Default | Campo |
|-------|---------|-------|
| `id` | `""` | id lógico |
| `model` | `""` | modelo del catálogo (17 modelos) |
| `enabled` | `false` | habilitado |
| `address` | `0` | dirección I²C |
| `rom` | `""` | ROM 1-Wire (16 hex) |
| `sda` / `scl` | `21` / `22` | pines I²C |
| `pin` | `0` | GPIO genérico |
| `rx` / `tx` | `0` / `0` | UART |
| `channel` | `""` | canal lógico |
| `unit` | `""` | unidad |
| `scale` / `offset` | `1.0` / `0.0` | escalado |

```json
{ "sensors": [ { "id": "EXT", "model": "BME280", "enabled": true,
                 "address": 118, "sda": 21, "scl": 22,
                 "channel": "temperature", "unit": "°C", "scale": 1, "offset": 0 } ] }
```

Respuestas: `200 {"ok":true}` (llama a `core_->applySensors()` en caliente) ·
`400` · `401` · `500`. ⚠️ Las claves `bus`, `uart` y `uart_port` (expansores
SC18IS602B/MAX14830) **se leen** de la config y la web **sí las envía**, pero este
handler **no** las escribe: guardar sensores desde la web las pierde (vuelven a `0`).
Ver §11 y [Comunicaciones-remotas](Comunicaciones-remotas.md).

### 7.6 `POST /api/v1/config/io` (web)

Buffer `4096`. Bloque `mcp23s17_cs` + `mcp23s17_pins[16]`,
`shift_registers[]` (solo si `SEMA_USE_SHIFT`, que está **en 0** en todas las
placas), `spi_expanders[]` (`type`, `cs`) y `gpio[]` (`id`, `pin`, `mode`,
`initial`, `expander`). Al responder OK llama a `applyGpio()` y `applyShift()`.

```json
{ "mcp23s17_cs": 0, "mcp23s17_pins": [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
  "shift_registers": [], "spi_expanders": [ { "type": "MAX14830", "cs": 15 } ],
  "gpio": [ { "id": "relay", "pin": 26, "mode": "output", "initial": 0, "expander": 0 } ] }
```

### 7.7 `POST /api/v1/config/buses` (web)

Buffer `4096`. Sub-objetos y claves (con defaults):

| Bloque | Claves |
|--------|--------|
| raíz | `i2c_sda` (21), `i2c_scl` (22), `sd_cs` (4) |
| `modbus` | `enabled` (false), `rx` (16), `tx` (17), `de_re` (0), `uart` (0), `uart_port` (0) |
| `can` | `enabled` (false), `tx` (5), `rx` (4) |
| `lora` | `enabled` (false), `cs` (10), `rst` (14), `dio1` (26), `busy` (27) |
| `zigbee` | `enabled` (false), `rx` (18), `tx` (19), `uart` (0), `uart_port` (0) |
| `ethernet` | `enabled` (false), `mdc` (23), `mdio` (18), `cs` (5) |

```json
{ "i2c_sda": 21, "i2c_scl": 22, "sd_cs": 4,
  "modbus": { "enabled": true, "rx": 16, "tx": 17, "de_re": 4, "uart": 0, "uart_port": 0 },
  "can": { "enabled": false, "tx": 5, "rx": 4 },
  "lora": { "enabled": false, "cs": 10, "rst": 14, "dio1": 26, "busy": 27 },
  "zigbee": { "enabled": false, "rx": 18, "tx": 19, "uart": 0, "uart_port": 0 },
  "ethernet": { "enabled": false, "mdc": 23, "mdio": 18, "cs": 5 } }
```

⚠️ Este handler **solo persiste**: no llama a `applyModbus()`, `applyCan()`,
`applyLora()`, `applyZigbee()` ni `applyEthernet()`. Los buses no se configuran
por esta vía: la web guarda y después dispara `POST /api/v1/restart`. Además
**no** expone `baud`, `slave_id`, `register`, `count` (Modbus), `speed` (CAN),
`frequency`/`bandwidth`/`spreading`/`coding_rate`/`tx_power` (LoRa), `baud`
(Zigbee) ni `phy_addr`/`power`/`rst`/`irq`/`sck`/`miso`/`mosi` (Ethernet): esos
valores solo se cambian con `PUT /api/v1/config`. Ver
[Comunicaciones-remotas](Comunicaciones-remotas.md).

## 8. Red y WiFi

### 8.1 `GET /api/v1/wifi/scan` (web)

Buffer `8192`. Ejecuta `WiFi.scanNetworks()`; si devuelve `< 0` espera 200 ms y
reintenta una vez; si vuelve a fallar, la lista queda vacía (no hay error HTTP).
Devuelve como máximo **40** redes y borra los resultados con `WiFi.scanDelete()`.

```json
{ "networks": [ { "ssid": "MiRed", "rssi": -55, "secure": true } ] }
```

`rssi` en dBm; `secure` es `encryptionType(i) != WIFI_AUTH_OPEN`. La web ordena
por `rssi` descendente del lado del cliente. En modo AP el escaneo puede devolver
una lista vacía según el estado del radio.

## 9. Seguridad

### 9.1 `POST /api/v1/security/keys` (web)

Buffer `1024`. Genera y revoca claves adicionales (`security.extra_keys`, un JSON
`{"nombre":"clave"}`).

```json
{ "action": "generate", "name": "Cliente 1" }
```

| Respuesta | Caso |
|-----------|------|
| `200 {"ok":true,"name":"Cliente 1","key":"a1b2…"}` | `generate`: clave de **32 hex** (16 bytes de `esp_random`) |
| `200 {"ok":true,"name":"Cliente 1"}` | `revoke` (sin `key`) |
| `400 {"error":"action/name required"}` | Falta `action` o `name` |
| `400 {"error":"unknown action"}` | `action` distinto de `generate`/`revoke` |
| `500 {"error":"extra_keys parse"}` | `extra_keys` corrupto en NVS |
| `500 {"error":"config apply failed"}` | No se pudo persistir |

No hay endpoint dedicado para `api_key`, `server_key`, `username` ni `password`:
se editan con `PUT /api/v1/config` (así lo hace la página `/config/security`).

## 10. GPIO y expansores

| Ruta | Cuerpo | Respuesta |
|------|--------|-----------|
| `GET /api/v1/gpio` | — | `{"gpio":[{"id":"relay","pin":26,"mode":"output","value":0}]}` |
| `POST /api/v1/gpio` (web) | `{"pin":26,"value":1}` | `200 {"ok":true}` · `400 {"error":"missing body"}` · `400 {"error":"invalid json"}` |
| `GET /api/v1/shift` | — | `{"value":123}` |
| `POST /api/v1/shift` (web) | `{"value":255}` | `200 {"ok":true}` · `400 {"error":"missing body"}` · `400 {"error":"invalid json"}` |

`GET /api/v1/gpio` solo lista los pines declarados en `gpio[]` (con su `mode` y la
lectura actual de `GpioManager::read`). `POST` escribe **sin validar** que el pin
esté declarado: `value` es `int` y `pin` es `uint8_t` (un pin > 255 se trunca).
`POST /api/v1/shift` escribe el byte en el 74HC595 y `GET` lo lee del 74HC165 o
devuelve el último valor escrito. El driver de shift registers está en **beta**:
`SEMA_USE_SHIFT` vale `0` por defecto (`HwProfile.hpp:68`), así que
`shift_enabled` en `/api/v1/system` es `false` aunque las rutas respondan.

## 11. Buses Modbus / CAN / LoRa / Zigbee

### 11.1 `GET /api/v1/modbus` (`SEMA_USE_MODBUS`)

Buffer `1024`. **Dispara una lectura** Modbus RTU (`readHoldingRegisters`) y
devuelve el resultado.

```json
{ "ready": true, "result": 0, "values": [ 512, 300, 0, 0 ] }
```

- `ready`: el bus se configuró (`apply()` con `enabled: true`).
- `result`: código de error de `ModbusMaster` — `0` = `ku8MBSuccess`; si el bus no
  está listo, `read()` corta con `0xFF` antes de tocar el puerto.
- `values`: `registerCount` registros de 16 bits crudos (sin escala) desde
  `registerAddr`.

Parámetros (solo por `PUT /api/v1/config` → `modbus`): `baud` (9600), `slave_id`
(1), `register` (0), `count` (4). Es **maestro de un único esclavo**.

### 11.2 `GET` / `POST /api/v1/can` (`SEMA_USE_CAN`)

`GET` (público, buffer `256`) intenta recibir **una** trama y no bloquea:

```json
{ "ready": true, "received": true, "id": 291, "extd": false, "data": [1,2,3,4,5,6,7,8] }
```

Sin trama: `{ "ready": true, "received": false }`.

`POST` (web) transmite:

```json
{ "id": 291, "extd": false, "data": [1,2,3,4,5,6,7,8] }
```

| Respuesta | Caso |
|-----------|------|
| `200 {"ok":true}` | `twai_transmit` OK (timeout 1000 ms) |
| `500 {"error":"send failed"}` | No listo o error de transmisión |
| `400 {"error":"missing body"}` / `400 {"error":"invalid json"}` | Cuerpo ausente o inválido |

El `data` se recorta a **8 bytes** (`dlc`). El baudrate y los pines salen de la
config (`speed` default 500000).

### 11.3 `GET` / `POST /api/v1/lora` (`SEMA_USE_LORA`)

`GET` (público, buffer `512`) intenta leer un paquete:

```json
{ "ready": true, "received": true, "data": [72, 111, 108, 97] }
```

`POST` (web) transmite un array de bytes (`data`, máximo **64**):

```json
{ "data": [72, 111, 108, 97] }
```

`200 {"ok":true}` o `500 {"error":"send failed"}` (además de `400`/`401`).
`data` vacío o `len == 0` → `send()` devuelve `false` → `500`.

### 11.4 `GET` / `POST /api/v1/zigbee` (`SEMA_USE_ZIGBEE`)

`GET` (público, buffer `512`) consume el último mensaje recibido del
co-procesador:

```json
{ "ready": true, "received": true, "src": 4660, "data": [1, 2, 3] }
```

`src` es la dirección corta de origen (`lastSrc()`, en decimal) y vale `0` si no
hay mensaje.

`POST` (web) envía un `AF_DATA_REQUEST`:

```json
{ "destination": 4660, "data": [1, 2, 3] }
```

`data` máximo **110** bytes; `destination` es `uint16` (endpoint destino/fuente
fijos en `0x01`, cluster `0x0001`). Respuestas `200 {"ok":true}` /
`500 {"error":"send failed"}` / `400` / `401`.

## 12. OTA, backup, reinicio y actualización

### 12.1 `POST /api/v1/ota` (solo `X-API-Key`)

```
POST /api/v1/ota
X-API-Key: <api_key|server_key|extra_key>
Content-Type: multipart/form-data
X-SHA256: <64 hex opcional>

<form-data con el .bin>
```

Es una ruta mixta: `server_.on("/api/v1/ota", HTTP_POST, onOta, onOtaUpload)`. El
upload corre primero (`onOtaUpload`) y luego el handler de cierre (`onOta`).

| Respuesta | Caso |
|-----------|------|
| `200 {"ok":true}` (y reinicio tras 100 ms) | Firmware escrito y SHA correcto/ausente |
| `401 {"error":"unauthorized"}` | `authorized()` falso |
| `400 {"error":"sha256 mismatch"}` | `X-SHA256` presente (64 hex) y el SHA-256 de la partición no coincide |

Comportamiento real: la autorización se evalúa **al iniciar** el upload; si falla,
no se escribe nada. La verificación SHA-256 se calcula leyendo la partición de
actualización completa con mbedTLS. Sin cabecera `X-SHA256` (o con longitud ≠ 64)
no hay verificación y el firmware se aplica igual.

⚠️ **Discrepancia funcional:** como esta ruta usa `authorized()` y no
`webAuthed()`, **la cookie de sesión no alcanza**. La página `/config/system`
sube el firmware con `XMLHttpRequest` + `FormData` (campo `firmware`) **sin**
enviar `X-API-Key`, con lo que la subida desde la web solo funciona mientras
`api_key`, `server_key` y `extra_keys` estén todas vacías. Con una clave
configurada, la web muestra "Subiendo N%" y el servidor responde `401`.

### 12.2 `GET /api/v1/backup` (web)

Buffer `8192`. Config completa + metadatos autodescriptivos:

```json
{ "schema_version": 1, "station": { "...": "..." },
  "backup_format": "sema-backup", "backup_version": 1,
  "firmware": "1.103.0", "timestamp": 12345 }
```

`timestamp` es `millis()/1000` (uptime, no epoch). Los campos `backup_*`,
`firmware` y `timestamp` **se ignoran al restaurar**. Errores:
`401 {"error":"unauthorized"}` · `500 {"error":"serialization failed"}` ·
`500 {"error":"internal"}`.

### 12.3 `POST /api/v1/backup` (web)

Alias exacto de `PUT /api/v1/config`: restaura la configuración. Mismas
respuestas (`200 {"ok":true}` / `400 {"error":"invalid config"}`).

### 12.4 `POST /api/v1/restart` (web)

Sin cuerpo. Responde `200 {"ok":true}` y reinicia tras 100 ms.

### 12.5 `GET /api/v1/update/check` (web)

Buffer `512`. Consulta `https://raw.githubusercontent.com/AlessandroKlein/SEMA/refs/heads/main/firmware_manifest.json`
con `WiFiClientSecure::setInsecure()` y timeout de 8 s.

```json
{ "current": "1.103.0", "latest": "1.104.0", "update": true,
  "url": "https://github.com/AlessandroKlein/SEMA/releases/tag/v1.104.0" }
```

- Si la descarga falla, `latest` queda `""` y `update` en `false` (HTTP `200`
  igual: la web muestra "No se pudo consultar GitHub").
- `setInsecure()` se usa **solo** para leer la versión; el OTA real no valida
  certificado, valida SHA-256 opcional.
- Esta ruta abre una conexión TLS: **sin Internet** tarda hasta 8 s y devuelve
  `latest: ""`.

## 13. Dashboard

| Ruta | Cuerpo | Respuesta |
|------|--------|-----------|
| `GET /api/v1/dashboard/layout` (web) | — | `{"layout":"[{\"x\":0,…}]"}`; si no hay layout guardado, `"[]"` |
| `POST /api/v1/dashboard/layout` (web) | JSON de Gridstack (array crudo) | `200 {"ok":true}` · `400 {"error":"missing body"}` · `500 {"error":"layout save failed"}` |

El layout se guarda en una clave NVS propia (no dentro del JSON de config) para no
exceder el tamaño de una entrada NVS. El `GET` devuelve el layout como **string**
dentro del JSON, no como array.

Los assets del dashboard están embebidos y comprimidos en el firmware
(`src/core/web/GridstackAssets.h`):

| Ruta | Estado | Cabeceras |
|------|--------|-----------|
| `GET /gridstack.min.css` | `200` | `Content-Encoding: gzip`, `Cache-Control: max-age=3600`, `text/css` |
| `GET /gridstack-all.min.js` | `200` | `Content-Encoding: gzip`, `Cache-Control: max-age=3600`, `application/javascript` |

Ambas rutas son públicas y usan `send_P` (contenido en flash). No hay `/favicon.ico`
ni ningún otro asset: cualquier otra ruta cae en `onNotFound` → `404` JSON.

## 14. Páginas HTML

Todas devuelven `200 text/html`. Salvo `/login`, sirven la **pantalla de login** si
`webAuthed()` es falso en vez de un `401`.

| Ruta | Contenido |
|------|-----------|
| `GET /` | Dashboard Gridstack. Inyecta `window.SEMA_AUTH=<true\|false>`; sin sesión oculta el menú y los controles de edición. Refresca `/api/v1/sensors` cada **5 s** y redibuja gráficos cada **60 s** |
| `GET /sensors` | Vista de sensores (refresco cada 5 s) |
| `GET /events` | Vista de eventos y alarmas (refresco cada 5 s) |
| `GET /config/network` | WiFi (STA/AP), IP estática, mDNS, escaneo |
| `GET /config/security` | Nombre de estación, login, claves API |
| `GET /config/system` | Estado, NTP/zona, unidades, idioma, microSD, OTA, CSV, reinicio |
| `GET /config/wind` | Veleta: resistencias y calibración de norte |
| `GET /config/sensors` | Catálogo de sensores, pines de buses, expansores |

Las páginas usan i18n propio (`es`/`en`) en `localStorage` (`sema_lang`) y tema
claro/oscuro (`sema_theme`).

## 15. Códigos de error

| Código | Cuerpo típico | Significado |
|--------|---------------|-------------|
| `200` | `{"ok":true}` / payload | Éxito |
| `302` | — | Login/logout correcto (con `Location`) |
| `400` | `{"error":"body required"}` | Falta el cuerpo (`plain`) |
| `400` | `{"error":"invalid json"}` | JSON malformado |
| `400` | `{"error":"invalid config"}` | `PUT /config` o `/backup`: validación fallida |
| `400` | `{"error":"missing body"}` | Igual que `body required` en GPIO/shift/wind CAN/LoRa/Zigbee |
| `400` | `{"error":"action/name required"}` · `unknown action` | `/security/keys` |
| `400` | `{"error":"wind direction not configured"}` | `/wind/north` con `wind_direction_pin = 0` |
| `400` | `{"error":"sha256 mismatch"}` | OTA con SHA incorrecto |
| `401` | `{"error":"unauthorized"}` | `webAuthed()`/`authorized()` falso |
| `401` | HTML del formulario | `POST /login` con credenciales incorrectas |
| `404` | `{"error":"not found"}` | Ruta inexistente o bus no compilado |
| `429` | `text/plain` | Login bloqueado 60 s por 5 fallos |
| `500` | `{"error":"config apply failed"}` | Falló la persistencia de la config |
| `500` | `{"error":"serialization failed"}` / `internal` | `/config` o `/backup` |
| `500` | `{"error":"layout save failed"}` | NVS rechazó el layout |
| `500` | `{"error":"send failed"}` | Transmisión CAN/LoRa/Zigbee fallida |

## 16. Límites de implementación

| Aspecto | Valor real |
|---------|------------|
| Buffer del histórico | `limit` máximo **3000** entradas por request |
| Buffer de `EventLog` | **100** eventos |
| `HistoryStore::maxEntries` | **10000** |
| Redes por escaneo | **40** |
| Bytes por trama CAN | **8** |
| Bytes por paquete LoRa | **64** (`GET` y `POST`) |
| Bytes por mensaje Zigbee | **110** |
| Claves extra | clave de **32 hex** |
| Timeout TLS de `update/check` | **8000 ms** |
| Timeout HTTP del webhook | **2000 ms** (ver [MQTT-y-WebSocket](MQTT-y-WebSocket.md)) |
| Documentos JSON | `256`–`16384` según handler (`/sensors` 8192, `/history` 16384, `/config` 16384) |

❌ **No implementado:** paginación, filtros por rango de fechas en `/history`,
autenticación por roles, HTTPS local, CORS, `OPTIONS`, `DELETE`, endpoint para
cambiar el perfil de energía, endpoint para `altitude`/`rain_pin`/`rules`/
`calibrations` (solo vía `PUT /api/v1/config`), y descubrimiento automático
(mDNS anuncia `_http._tcp` en el puerto 80 pero no hay `_sema._tcp`).

---

## Ver también

- [Configuracion](Configuracion.md) · [Referencia-configuracion](Referencia-configuracion.md) · [Variables-modificables](Variables-modificables.md)
- [Seguridad](Seguridad.md) · [OTA-y-Actualizacion](OTA-y-Actualizacion.md) · [Backup-y-restauracion](Backup-y-restauracion.md)
- [Conectividad-y-red](Conectividad-y-red.md) · [Comunicaciones-remotas](Comunicaciones-remotas.md) · [MQTT-y-WebSocket](MQTT-y-WebSocket.md)
- [Identidad-y-estados](Identidad-y-estados.md) · [Enumeraciones-y-tipos](Enumeraciones-y-tipos.md) · [Almacenamiento-e-historico](Almacenamiento-e-historico.md)
