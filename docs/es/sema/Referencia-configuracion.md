---
tags:
  - sema
  - configuracion
  - referencia
---

# Referencia de configuración (JSON)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Referencia **exhaustiva** de todas las claves que `ConfigManager` lee y escribe
(`include/core/ConfigManager.hpp` + `src/core/ConfigManager.cpp`). Es el mismo
documento que se persiste en NVS (clave `config`) y el que se intercambia por
`GET`/`PUT /api/v1/config` y por `GET`/`POST /api/v1/backup`.

!!! info "Cómo leer las tablas"
    - **Clave**: nombre exacto en el JSON.
    - **Tipo**: tipo del valor tal como lo interpreta ArduinoJson.
    - **Default**: el valor que se usa cuando la clave **falta**. Es el default
      del parser (`doc["..."] | <valor>`), que es el que realmente ve el equipo
      cuando mandás un JSON incompleto o restaurás un backup.
    - **Oblig.**: `No` = opcional; `Sí` = obligatoria; `Cond.` = condicional.
    - Todo el JSON es un solo objeto; no hay claves de primer nivel repetidas.

---

## 1. Ejemplo completo verificado

```json
{
  "schema_version": 1,
  "station": { "id": "SEMA-001", "name": "Estación Norte" },
  "network": {
    "mode": "STA",
    "ssid": "MiRed",
    "password": "clave-wifi",
    "hostname": "sema-001",
    "mdns": true,
    "ip": "",
    "gateway": "",
    "subnet": "",
    "dns": ""
  },
  "system": {
    "timezone": "America/Argentina/Buenos_Aires",
    "ntp_server": "pool.ntp.org",
    "log_level": "INFO",
    "units": "metric",
    "lang": "es",
    "altitude": 25.0,
    "wind_north_offset": 0.0,
    "wind_direction_pin": 34,
    "wind_rpull": 10000.0,
    "wind_resistors": [33000.0, 8200.0, 1000.0, 2200.0, 3900.0, 16000.0, 120000.0, 64900.0]
  },
  "storage": { "backend": "littlefs", "retention_days": 30, "sd_enabled": true, "sd_cs": 4 },
  "security": {
    "api_key": "clave-web-local",
    "server_key": "clave-servidor-central",
    "username": "admin",
    "password": "clave-login",
    "extra_keys": "{\"Cliente 1\":\"2f9c1d4b7a0e83561cb2f47a9de01357\"}"
  },
  "energy": { "rain_pin": 4 },
  "publishers": {
    "webhook_url": "https://example.com/hook",
    "mqtt_host": "broker.local",
    "mqtt_port": 1883,
    "mqtt_topic": "sema/measurement",
    "mqtt_user": "sema",
    "mqtt_pass": "clave-mqtt"
  },
  "sensors": [
    { "id": "EXT", "model": "BME280", "enabled": true, "address": 0, "rom": "",
      "sda": 21, "scl": 22, "bus": 0, "uart": 0, "uart_port": 0, "pin": 0,
      "rx": 0, "tx": 0, "channel": "", "unit": "", "scale": 1.0, "offset": 0.0 },
    { "id": "SOIL", "model": "DS18B20", "enabled": true, "address": 0,
      "rom": "28FF64A1B2C3D4E5", "sda": 21, "scl": 22, "bus": 0, "uart": 0,
      "uart_port": 0, "pin": 4, "rx": 0, "tx": 0, "channel": "", "unit": "",
      "scale": 1.0, "offset": 0.0 },
    { "id": "PM", "model": "PMS5003", "enabled": true, "address": 0, "rom": "",
      "sda": 21, "scl": 22, "bus": 0, "uart": 0, "uart_port": 0, "pin": 0,
      "rx": 16, "tx": 17, "channel": "", "unit": "", "scale": 1.0, "offset": 0.0 },
    { "id": "BATT", "model": "ADC", "enabled": true, "address": 0, "rom": "",
      "sda": 21, "scl": 22, "bus": 0, "uart": 0, "uart_port": 0, "pin": 34,
      "rx": 0, "tx": 0, "channel": "voltage", "unit": "V", "scale": 0.00887, "offset": 0.0 }
  ],
  "rules": [
    { "name": "high_temp", "sensor_id": "EXT", "channel_id": "temperature", "op": "gt", "value": 40.0 }
  ],
  "calibrations": [
    { "sensor_id": "EXT", "channel_id": "temperature", "gain": 1.0, "offset": -0.5,
      "has_range": true, "min": -40.0, "max": 85.0 }
  ],
  "gpio": [
    { "id": "relay", "pin": 26, "mode": "output", "initial": 0, "expander_addr": 0 },
    { "id": "mcp-in", "pin": 3, "mode": "input_pullup", "initial": 0, "expander_addr": 32 }
  ],
  "mcp23s17_cs": 0,
  "mcp23s17_pins": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
  "shift_registers": [
    { "type": "74HC595", "latch_pin": 15, "pins": [1, 1, 0, 0, 0, 0, 0, 0] }
  ],
  "spi_expanders": [
    { "type": "SC18IS602B", "cs": 32 }
  ],
  "modbus": {
    "enabled": true, "rx": 16, "tx": 17, "de_re": 18, "uart": 0, "uart_port": 0,
    "baud": 9600, "slave_id": 1, "register": 0, "count": 4
  },
  "can": { "enabled": true, "tx": 5, "rx": 4, "speed": 500000 },
  "lora": {
    "enabled": true, "cs": 10, "rst": 32, "dio1": 26, "busy": 27,
    "frequency": 915.0, "bandwidth": 125.0, "spreading": 7, "coding_rate": 5, "tx_power": 14
  },
  "zigbee": { "enabled": false, "rx": 18, "tx": 19, "uart": 0, "uart_port": 0, "baud": 115200 },
  "ethernet": {
    "enabled": true, "mdc": 23, "mdio": 18, "phy_addr": 1, "power": -1,
    "cs": 5, "rst": -1, "irq": 4, "sck": 18, "miso": 19, "mosi": 21
  },
  "i2c_sda": 21,
  "i2c_scl": 22
}
```

## 2. Raíz

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `schema_version` | int | `1` | Sí | Esquema del documento; si no es `1` la config se descarta (§12) |
| `station` | objeto | — | No | Identidad lógica de la estación |
| `network` | objeto | — | No | WiFi, IP y mDNS |
| `system` | objeto | — | No | Hora, unidades, idioma, altitud y veleta |
| `storage` | objeto | — | No | Retención del histórico y microSD |
| `security` | objeto | — | No | Claves de API y login web |
| `energy` | objeto | — | No | Wake por lluvia |
| `publishers` | objeto | — | No | Webhook HTTP y MQTT |
| `sensors` | array | `[]` | No | Catálogo de sensores (§9); vacío = catálogo por defecto (§11) |
| `rules` | array | `[]` | No | Reglas de alarma (§9); vacío = regla por defecto (§11) |
| `calibrations` | array | `[]` | No | Calibración por canal (§9); vacío = calibración por defecto (§11) |
| `gpio` | array | `[]` | No | GPIO standalone y MCP23017 (§10) |
| `mcp23s17_cs` | int | `0` | No | CS SPI del expansor MCP23S17 (`0` = no usar) |
| `mcp23s17_pins` | array[16] | `[0,…,0]` | No | Modo de cada pin del MCP23S17: `0`=no usado, `1`=salida, `2`=entrada |
| `shift_registers` | array | `[]` | No | 74HC595/74HC165 encadenables (§10) |
| `spi_expanders` | array | `[]` | No | Expansores SPI MAX14830/SC18IS602B (§10) |
| `modbus` | objeto | — | No | RS485/Modbus RTU |
| `can` | objeto | — | No | CAN (TWAI) |
| `lora` | objeto | — | No | SX1262 |
| `zigbee` | objeto | — | No | CC2652P2 por UART |
| `ethernet` | objeto | — | No | LAN8720A (RMII) o W5500 (SPI) |
| `i2c_sda` | int | `21` | No | SDA del bus I²C compartido (hoy informativo, §13) |
| `i2c_scl` | int | `22` | No | SCL del bus I²C compartido (hoy informativo, §13) |

## 3. `station`

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `id` | str | `"SEMA-001"` | **Sí** | Identificador lógico; se publica como `station_id`. Vacío ⇒ `validate()` rechaza toda la config |
| `name` | str | `"Estación Norte"` | No | Nombre visible en la web, `/api/v1/status` y `/api/v1/system` |

## 4. `network`

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `mode` | enum | `"STA"` | No | `"STA"` (cliente) · `"AP"` (punto de acceso). Cualquier otro valor ⇒ config inválida |
| `ssid` | str | `""` | No | SSID de la red WiFi |
| `password` | str | `""` | No | Clave WiFi |
| `hostname` | str | `"sema-001"` | No | Nombre de host; se usa para mDNS (`http://<hostname>.local`) |
| `mdns` | bool | `true` | No | Habilitar mDNS |
| `ip` | str | `""` | No | IP estática; vacío = DHCP |
| `gateway` | str | `""` | No | Puerta de enlace (solo con IP estática) |
| `subnet` | str | `""` | No | Máscara de red (solo con IP estática) |
| `dns` | str | `""` | No | Servidor DNS (solo con IP estática) |

Requiere reinicio (ver [Configuración](Configuracion.md) §5.2).

## 5. `system`

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `timezone` | str | `"America/Argentina/Buenos_Aires"` | No | Zona horaria IANA; se pasa a `configTzTime()` en el arranque |
| `ntp_server` | str | `"pool.ntp.org"` | No | Servidor NTP primario (el secundario es `time.nist.gov`) |
| `log_level` | enum | `"INFO"` | No | Se persiste; **ningún módulo lo consume** |
| `units` | enum | `"metric"` | No | `"metric"` · `"imperial"`; afecta a las respuestas de la API |
| `lang` | enum | `"es"` | No | `"es"` · `"en"`; idioma de la web embebida |
| `altitude` | float | `0.0` | No | Altitud en metros (QNH y altitud barométrica) |
| `wind_north_offset` | float | `0.0` | No | Grados de corrección del norte de la veleta; lo escribe `POST /api/v1/wind/north` |
| `wind_direction_pin` | int | `0` | No | GPIO ADC de la veleta WH-SP-WD (`0` = sin veleta) |
| `wind_rpull` | float | `10000.0` | No | Resistencia de pull-up de la veleta, en Ω |
| `wind_resistors` | array[8] | `[33000, 8200, 1000, 2200, 3900, 16000, 120000, 64900]` | No | Resistencias de la rosa de los vientos en orden del datasheet: N, NE, E, SE, S, SO, O, NO (Ω) |

Los campos de la veleta y `altitude` se aplican en caliente porque
`applySensors()` llama a `derived_.configure(system)` (`SemaCore.cpp:271`).
`timezone` y `ntp_server`, en cambio, requieren reinicio.

## 6. `storage`

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `backend` | enum | `"littlefs"` | No | `"littlefs"` · `"flash"` · `"sd"`. Solo se valida; ningún módulo lo lee |
| `retention_days` | uint32 | `30` | No | Días de retención del histórico; la poda lo lee en cada pasada |
| `sd_enabled` | bool | `false` | No | Habilita la microSD (SPI) para el histórico. Se aplica **solo en el arranque** |
| `sd_cs` | int | `4` | No | Chip-select de la microSD (`SEMA_PIN_SD_CS`). Se aplica **solo en el arranque** |

## 7. `security`

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `api_key` | str | `""` | No | Clave de la web/API local (`X-API-Key`). Vacía = sin clave |
| `server_key` | str | `""` | No | Clave del Servidor Central → SEMA (misma semántica que `api_key`) |
| `username` | str | `""` | No | Usuario del login web; vacío = `admin` |
| `password` | str | `""` | No | Contraseña del login web; vacía = **sin login** (web abierta) |
| `extra_keys` | str (JSON) | `""` | No | **String** que contiene un objeto `{"nombre":"clave"}` con claves API adicionales revocables |

!!! warning "`extra_keys` es un string, no un objeto"
    El valor es texto JSON escapado, tal como lo escribe
    `POST /api/v1/security/keys` (`HttpServer.cpp:1773-1775`). Si mandás un objeto
    real, el parser lo ignora (queda vacío) y perdés las claves adicionales.
    Las claves de este mapa **solo** sirven para `X-API-Key`; no habilitan el login web.

## 8. `energy` y `publishers`

`energy`:

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `rain_pin` | int | `0` | No | GPIO del pluviómetro para wake `ext0` (`0` = deshabilitado). Se aplica **solo en el arranque** |

`publishers`:

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `webhook_url` | str | `""` | No | URL del webhook HTTP; vacío = deshabilitado |
| `mqtt_host` | str | `""` | No | Host del broker MQTT; vacío = deshabilitado |
| `mqtt_port` | uint16 | `1883` | No | Puerto del broker MQTT |
| `mqtt_topic` | str | `"sema/measurement"` | No | Tópico de publicación |
| `mqtt_user` | str | `""` | No | Usuario MQTT; vacío = sin autenticación |
| `mqtt_pass` | str | `""` | No | Contraseña MQTT |

## 9. `sensors[]`, `rules[]`, `calibrations[]`

`sensors[]` (`SensorSpec`):

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `id` | str | `""` | Cond. | Id lógico del sensor; es la clave de `sensor_id` en reglas y calibraciones |
| `model` | str | `""` | Cond. | Modelo del catálogo (`BME280`, `SHT40`, `SHT31`, `BMP280`, `DS18B20`, `BH1750`, `AHT20`, `ADC`, `PCNT`, `VEML6075`, `SCD30`, `SGP30`, `PMS5003`, `AS3935`, `ADS1115`, `CO`, `SOLAR`). Modelo desconocido = se descarta con log serie |
| `enabled` | bool | `false` | No | **`false` = sensor apagado**: `applySensors()` lo saltea aunque esté en la lista |
| `address` | int | `0` | No | Dirección I²C (`0` = default del driver). Guardado, pero `SensorFactory` no lo usa |
| `rom` | str | `""` | No | ROM 1-Wire del DS18B20 (16 hex); vacío = autodetección |
| `sda` | int | `21` | No | Pin SDA del sensor I²C |
| `scl` | int | `22` | No | Pin SCL del sensor I²C |
| `bus` | int | `0` | No | `0` = I²C nativo; distinto de `0` = CS del SC18IS602B. Guardado, no aplicado |
| `uart` | int | `0` | No | `0` = UART nativa; distinto de `0` = CS del MAX14830. Guardado, no aplicado |
| `uart_port` | int | `0` | No | Puerto UART del MAX14830 (`0`…`3`). Guardado, no aplicado |
| `pin` | int | `0` | Cond. | Pin para 1-Wire/ADC/PCNT/CO/SOLAR; **índice de canal** en ADS1115 |
| `rx` | int | `0` | Cond. | Pin RX para sensores UART (PMS5003) |
| `tx` | int | `0` | Cond. | Pin TX para sensores UART (PMS5003) |
| `channel` | str | `""` | Cond. | Magnitud para sensores genéricos (ADC/PCNT/ADS1115), p. ej. `"voltage"` |
| `unit` | str | `""` | Cond. | Unidad publicada, p. ej. `"V"` |
| `scale` | float | `1.0` | No | Factor de escala aplicado al valor crudo |
| `offset` | float | `0.0` | No | Offset sumado al valor escalado |

`rules[]` (`RuleSpec`):

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `name` | str | `""` | No | Nombre de la regla; se usa como `id` en el evento de alarma |
| `sensor_id` | str | `""` | No | Sensor objetivo; `""` = cualquier sensor |
| `channel_id` | str | `""` | No | Magnitud a vigilar (`"temperature"`, `"humidity"`, `"pressure"`, `"light"`, `"co2"`, `"pm25"`, `"uvi"`, `"wind_speed"`, `"rain_rate"`, …) |
| `op` | enum | `"gt"` | No | `"gt"` · `"lt"` · `"ge"` · `"le"`; un valor desconocido cae a `gt` (`parseRuleOp`) |
| `value` | float | `0.0` | No | Umbral de comparación |

`calibrations[]` (`CalibrationSpec`):

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `sensor_id` | str | `""` | No | Sensor calibrado; se concatena como `"<sensor_id>:<channel_id>"` |
| `channel_id` | str | `""` | No | Canal calibrado |
| `gain` | float | `1.0` | No | Ganancia multiplicativa |
| `offset` | float | `0.0` | No | Offset aditivo |
| `has_range` | bool | `false` | No | Si la calibración define rango válido |
| `min` | float | `0.0` | No | Mínimo del rango válido |
| `max` | float | `0.0` | No | Máximo del rango válido |

## 10. `gpio[]`, expansores y shift registers

`gpio[]` (`GpioSpec`):

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `id` | str | `""` | No | Id lógico del pin |
| `pin` | int | `0` | No | GPIO nativo, o pin `0`…`15` del MCP23017 cuando `expander_addr != 0` |
| `mode` | enum | `"input"` | No | `"output"` · `"input"` · `"input_pullup"` · `"input_pulldown"` (en `POST /api/v1/config/io` el default es `"output"`) |
| `initial` | int | `0` | No | Estado inicial de las salidas (`0`/`1`); se escribe al aplicar |
| `expander_addr` | int | `0` | No | `0` = pin nativo; distinto de `0` = dirección I²C del MCP23017 (p. ej. `32` = `0x20`) |

`shift_registers[]` (`ShiftRegisterConfig`):

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `type` | enum | `"74HC595"` | No | `"74HC595"` (salida) · `"74HC165"` (entrada) |
| `latch_pin` | int | `0` | No | RCLK (595) / SH-LD (165). `POST /api/v1/config/io` usa el nombre `latch` |
| `pins` | array[8] | `[0,…,0]` | No | Uso de Q0…Q7 / D0…D7: `0`=no usado, `1`=usado |

`spi_expanders[]` (`SpiExpanderConfig`):

| Clave | Tipo | Default | Oblig. | Descripción |
|-------|------|---------|:------:|-------------|
| `type` | enum | `""` | No | `"MAX14830"` (UART por SPI) · `"SC18IS602B"` (I²C por SPI) |
| `cs` | int | `0` | No | Chip-select SPI (`0` = no usar) |

`mcp23s17_cs` / `mcp23s17_pins` están en la tabla de raíz (§2): el CS es un
escalar y los modos son un array fijo de 16 valores.

⚠️ MCP23S17, MAX14830, SC18IS602B y los shift registers **no tienen driver** en
`src/`: son configuración + UI (ver [Configuración](Configuracion.md) §5.3 y
[Expansores de entrada/salida](Expansores-de-entrada-salida.md)).

## 11. Defaults que se aplican cuando los arrays están vacíos

| Array vacío | Qué usa el firmware | Código |
|-------------|---------------------|--------|
| `sensors: []` | Catálogo fijo: `EXT` BME280, `INT` SHT40, `SOIL` DS18B20, `LUX` BH1750, `AUX` AHT20 y `BATT` (ADC, `scale = 3.3·11/4095`, canal `voltage`, unidad `V`), con los pines de `HwProfile`/`BoardProfile` | `SemaCore.cpp:282-297` |
| `rules: []` | Una regla: `high_temp`, `EXT`, `temperature`, `gt`, `40.0` | `SemaCore.cpp:255-256` |
| `calibrations: []` | `EXT:temperature` e `INT:temperature`, gain `1.0`, offset `0.0`, rango `-40`…`85` | `SemaCore.cpp:229-238` |
| `gpio: []` | Sin GPIO gestionados por `GpioManager` | `SemaCore.cpp:317-319` |
| `shift_registers: []` | Sin shift registers | `SemaCore.cpp:321-325` |
| `spi_expanders: []` | Sin expansores SPI | (sin consumidor) |

Con `SEMA_FIXED_HARDWARE=1` (`include/core/BoardProfile.hpp:20-21`, hoy `0` en
los cuatro entornos) el catálogo fijo se usa **aunque** `sensors[]` tenga
contenido.

## 12. Reglas de validación

`ConfigManager::validate()` (`ConfigManager.cpp:103-118`) acepta o rechaza el
documento entero con **cuatro** condiciones:

| # | Regla | Si falla |
|:-:|-------|----------|
| 1 | `schema_version == 1` | Config inválida |
| 2 | `station.id.length() > 0` | Config inválida |
| 3 | `network.mode` ∈ {`"STA"`, `"AP"`} | Config inválida |
| 4 | `storage.backend` ∈ {`"littlefs"`, `"flash"`, `"sd"`} | Config inválida |

No hay validación de rangos de pines, direcciones I²C, `id` duplicados,
coherencia de `op` ni de canales. Un JSON con `schema_version: 2` (u otro valor)
**no migra**: se descarta y el equipo arranca con defaults en RAM
(ver [Configuración](Configuracion.md) §6).

## 13. Diferencias de nombres entre endpoints parciales

Los endpoints parciales leen un subconjunto con nombres propios; si automatizás
contra ellos, usá el nombre que corresponde a cada uno:

| Campo | JSON completo (`PUT /api/v1/config`, backup) | `POST /api/v1/config/io` |
|-------|---------------------------------------------|--------------------------|
| Latch del shift register | `latch_pin` | `latch` |
| Expansor del GPIO | `expander_addr` | `expander` |
| Tipo de shift register | `type` (default `"74HC595"`) | `type` (default `"74HC595"`) |
| Tipo de expansor SPI | `type` (default `""`) | `type` (default `""`) |

Además, `POST /api/v1/config/sensors` **no** lee `address`, `rom`, `bus`,
`uart` ni `uart_port`: esos campos quedan en `0`/vacío aunque el JSON los traiga
(`HttpServer.cpp:1547-1564`). Y `POST /api/v1/config/buses` no lee
`modbus.baud`, `modbus.slave_id`, `modbus.register`, `modbus.count`,
`can.speed`, `lora.frequency`, `lora.bandwidth`, `lora.spreading`,
`lora.coding_rate`, `lora.tx_power` ni `zigbee.baud`: para esos valores hay que
usar `PUT /api/v1/config` (`HttpServer.cpp:1657-1697`).

## 14. Claves que se aceptan pero no se aplican

`system.log_level`, `storage.backend`, `sensors[].address`, `sensors[].bus`,
`sensors[].uart`, `sensors[].uart_port`, `mcp23s17_cs`, `mcp23s17_pins`,
`shift_registers[]` (con `SEMA_USE_SHIFT=0`), `spi_expanders[]`, `i2c_sda` e
`i2c_scl` se guardan y se devuelven, pero hoy ningún módulo las lee. El detalle
está en [Configuración](Configuracion.md) §5.3.

---

## Ver también

- [Configuración](Configuracion.md) · [Variables modificables](Variables-modificables.md)
- [Backup y restauración](Backup-y-restauracion.md) · [API REST](API-REST.md)
- [Buses y periféricos](Buses-y-perifericos.md) · [Sensores](Sensores.md) · [Alarmas y reglas](Alarmas-y-reglas.md)
