---
tags:
  - sema
  - configuracion
  - backup
---

# Backup y restauración

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

SEMA expone un respaldo **autodescriptivo** de la configuración en
`GET /api/v1/backup` y lo restaura con `POST /api/v1/backup` (que usa el mismo
handler que `PUT /api/v1/config`, `onConfigPut()`). Esta página documenta su
formato exacto, qué entra y qué no, y cómo migrar la configuración a otro
equipo.

---

## 1. Cómo funciona

| Operación | Endpoint | Handler | Auth |
|-----------|----------|---------|------|
| Descargar | `GET /api/v1/backup` | `onBackup()` (`HttpServer.cpp:1375-1399`) | `webAuthed()` (sesión o `X-API-Key`) |
| Restaurar | `POST /api/v1/backup` (body JSON) | `onConfigPut()` (`HttpServer.cpp:1861-1898`) | `webAuthed()` |

`onBackup()` serializa la configuración completa con `toJson()`, la vuelve a
parsear y le agrega cuatro campos de metadatos:

```text
{"backup_format":"sema-backup", "backup_version":1, "firmware":"1.103.0", "timestamp":<segundos de uptime>, ...config completa...}
```

| Campo | Valor real | Se usa al restaurar |
|-------|------------|---------------------|
| `backup_format` | `"sema-backup"` (fijo) | ❌ Se ignora |
| `backup_version` | `1` (fijo) | ❌ Se ignora |
| `firmware` | Versión que generó el backup (`SEMA_FW_VERSION`) | ❌ Se ignora |
| `timestamp` | `millis()/1000`: **segundos desde el arranque**, no época Unix | ❌ Se ignora |

!!! warning "`timestamp` no es una fecha"
    El campo se calcula con `millis()/1000` (`HttpServer.cpp:1395`), así que vale
    cero tras cada reinicio y crece con el uptime. No sirve para datar el backup:
    poné la fecha en el nombre del archivo al guardarlo.

No hay ninguna página ni botón para el backup en la web embebida: la única vía
es la API (`GET`/`POST /api/v1/backup`).

## 2. Qué incluye y qué no

### 2.1 Incluye

- **Todas** las claves de la configuración: `schema_version`, `station`,
  `network`, `system`, `storage`, `security`, `energy`, `publishers`, `sensors[]`,
  `rules[]`, `calibrations[]`, `gpio[]`, `mcp23s17_cs`, `mcp23s17_pins`,
  `shift_registers[]`, `spi_expanders[]`, `modbus`, `can`, `lora`, `zigbee`,
  `ethernet`, `i2c_sda`, `i2c_scl`.
- **Secretos en claro**: `security.api_key`, `security.server_key`,
  `security.extra_keys`, `security.password`, `network.password`,
  `publishers.mqtt_pass`. Igual que `GET /api/v1/config`, no hay enmascarado.

### 2.2 No incluye

| Elemento | Dónde vive | Cómo se respalda |
|----------|-----------|------------------|
| Histórico de mediciones | MicroSD (`HistoryStore`, `SD.open`) | Copiar el CSV/agregados de la SD |
| Eventos y alarmas | LittleFS, partición `spiffs` (`EventLog`) | Flashear/leer la partición, o exportar por `/api/v1/events` |
| Layout del dashboard | NVS, clave `layout` (separada de `config`) | `GET /api/v1/dashboard/layout` |
| Contador de reinicios | NVS, clave `boots` | No aplica |
| Mediciones en curso y estado de buses | RAM | No aplica |
| Perfil energético (`EnergyProfile`) | RAM (`PowerManager`), no se persiste | No aplica |
| Configuración del Servidor Central | Fuera de SEMA | No aplica |

> `POST /api/v1/backup` restaura **solo la configuración**. Un equipo restaurado
> queda sin histórico ni eventos: son datos locales del equipo, no configuración.

## 3. Estructura del archivo

Ejemplo verificado (config recortada: en un backup real van todas las claves de
[Referencia de configuración](Referencia-configuracion.md) §1):

```json
{
  "backup_format": "sema-backup",
  "backup_version": 1,
  "firmware": "1.103.0",
  "timestamp": 38421,
  "schema_version": 1,
  "station": { "id": "SEMA-001", "name": "Estación Norte" },
  "network": {
    "mode": "STA", "ssid": "MiRed", "password": "clave-wifi",
    "hostname": "sema-001", "mdns": true,
    "ip": "", "gateway": "", "subnet": "", "dns": ""
  },
  "system": {
    "timezone": "America/Argentina/Buenos_Aires", "ntp_server": "pool.ntp.org",
    "log_level": "INFO", "units": "metric", "lang": "es", "altitude": 25.0,
    "wind_north_offset": 0.0, "wind_direction_pin": 34, "wind_rpull": 10000.0,
    "wind_resistors": [33000.0, 8200.0, 1000.0, 2200.0, 3900.0, 16000.0, 120000.0, 64900.0]
  },
  "storage": { "backend": "littlefs", "retention_days": 30, "sd_enabled": true, "sd_cs": 4 },
  "security": {
    "api_key": "clave-web-local", "server_key": "clave-servidor-central",
    "username": "admin", "password": "clave-login",
    "extra_keys": "{\"Cliente 1\":\"2f9c1d4b7a0e83561cb2f47a9de01357\"}"
  },
  "energy": { "rain_pin": 4 },
  "publishers": {
    "webhook_url": "https://example.com/hook", "mqtt_host": "broker.local",
    "mqtt_port": 1883, "mqtt_topic": "sema/measurement",
    "mqtt_user": "sema", "mqtt_pass": "clave-mqtt"
  },
  "sensors": [
    { "id": "EXT", "model": "BME280", "enabled": true, "address": 0, "rom": "",
      "sda": 21, "scl": 22, "bus": 0, "uart": 0, "uart_port": 0, "pin": 0,
      "rx": 0, "tx": 0, "channel": "", "unit": "", "scale": 1.0, "offset": 0.0 }
  ],
  "rules": [
    { "name": "high_temp", "sensor_id": "EXT", "channel_id": "temperature", "op": "gt", "value": 40.0 }
  ],
  "calibrations": [
    { "sensor_id": "EXT", "channel_id": "temperature", "gain": 1.0, "offset": -0.5,
      "has_range": true, "min": -40.0, "max": 85.0 }
  ],
  "gpio": [
    { "id": "relay", "pin": 26, "mode": "output", "initial": 0, "expander_addr": 0 }
  ],
  "mcp23s17_cs": 0,
  "mcp23s17_pins": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
  "shift_registers": [],
  "spi_expanders": [],
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

## 4. Procedimiento: descargar y restaurar

### 4.1 Descargar

```bash
curl -s http://sema-001.local/api/v1/backup \
  -H "X-API-Key: <api_key>" \
  -o "sema-backup_SEMA-001_2026-10-08.json"
```

Con login web (cookie) también funciona, porque `GET /api/v1/backup` usa
`webAuthed()`:

```bash
curl -s -c cookies.txt -X POST http://sema-001.local/login \
  -d "username=admin&password=<clave-login>"
curl -s -b cookies.txt http://sema-001.local/api/v1/backup -o backup.json
```

### 4.2 Restaurar

```bash
curl -X POST http://sema-001.local/api/v1/backup \
  -H "X-API-Key: <api_key>" \
  -H "Content-Type: application/json" \
  --data-binary @sema-backup_SEMA-001_2026-10-08.json
# → {"ok":true}  |  400 {"error":"invalid config"}
```

Secuencia recomendada:

```text
1. GET /api/v1/backup del equipo origen (guardar el archivo)
2. Revisar/editar el JSON: station.id, station.name, pines, sd_cs, claves
3. Enviar el archivo con POST /api/v1/backup
4. POST /api/v1/restart           ← aplica red, SD, NTP y wake por lluvia
5. Verificar con GET /api/v1/config y GET /api/v1/system
6. Comprobar sensores (GET /api/v1/sensors) y SD (GET /api/v1/system → history_available)
```

!!! danger "Restaurar es reemplazar, no mezclar"
    `POST /api/v1/backup` parsea el body **desde cero**: toda clave ausente vuelve
    a su default y todo array ausente queda **vacío** (y con los arrays vacíos
    entran los catálogos por defecto de `sensors[]`, `rules[]` y
    `calibrations[]`). Mandar un JSON parcial no es un «patch»: es un reemplazo
    total. Para cambios puntuales usá `PUT /api/v1/config` sobre el JSON completo
    o los endpoints parciales.

### 4.3 Qué se aplica al restaurar

Igual que `PUT /api/v1/config` (ver [Configuración](Configuracion.md) §5):
sensores, reglas, calibraciones, GPIO, publicadores y los buses se re-aplican en
caliente; **red, microSD, zona horaria/NTP y `energy.rain_pin` requieren
reinicio**, por eso el paso 4 del procedimiento.

## 5. Compatibilidad entre versiones y cambios de esquema

| Caso | Comportamiento |
|------|----------------|
| `schema_version` distinto de `1` | ❌ La config **completa** se rechaza y el equipo arranca con defaults; no hay migración (`ConfigManager::validate()`) |
| `backup_version` distinto | ✅ Se ignora: no se valida ni se compara |
| Claves desconocidas en el archivo | ✅ Se ignoran (ArduinoJson solo lee las que conoce) |
| Claves ausentes | ⚠️ Vuelven a su default: restaurar un backup viejo puede resetear claves nuevas a su valor de fábrica |
| `station.id` vacío o ausente | ❌ Config inválida (`400 invalid config`) |
| Orden de las claves | ✅ Irrelevante |
| Placa distinta | ⚠️ Los pines del backup son los del equipo origen (ver §6) |

No hay número de versión de backup que se incremente ni verificación de
procedencia: un archivo de otra instalación se acepta tal cual.

## 6. Migrar la configuración a otro equipo

1. **Mismo firmware/modelo**: el documento es el mismo, pero conviene que la
   versión no sea radicalmente distinta (las claves ausentes toman defaults).
2. **Revisar los pines según la placa destino**:
   - `esp32doit-devkit-v1` (4 MB) y `esp32-s3-devkitc-1` (8 MB) tienen
     `SEMA_PINS_FROM_FILE=0`: los pines del backup se respetan.
   - `esp32-wroom-32u` (16 MB) tiene `SEMA_PINS_FROM_FILE=1`: `applyHwProfile()`
     **sobrescribe** los pines de `can`, `modbus`, `zigbee`, `lora` y `ethernet`
     con los de `HwProfile.hpp` (`ConfigManager.cpp:13-41`) y los del backup se
     descartan para esos buses.
   - `sd_cs` y los pines de sensores dependen del cableado real de cada equipo.
3. **Cambiar la identidad**: si los dos equipos quedan en la misma red o
   publican al mismo servidor, poné `station.id` y `station.name` distintos para
   no mezclar telemetría.
4. **Rotar credenciales si el equipo origen sigue en servicio**: el backup copia
   `api_key`, `server_key`, `extra_keys` y `password`; dos equipos con la misma
   clave comparten el poder de configuración.
5. **Revisar `network.*`**: un `hostname` repetido rompe el mDNS; una IP estática
   repetida rompe la red.
6. **Restaurar, reiniciar y verificar** (§4.2).
7. **No esperar datos históricos**: el histórico (SD) y los eventos (LittleFS)
   no viajan en el backup.

## 7. Verificación posterior a una restauración

| Chequeo | Cómo |
|---------|------|
| Config aplicada | `GET /api/v1/config` y comparar con el archivo restaurado |
| Identidad | `GET /api/v1/status` → `station`, `name`, `firmware` |
| Red | `GET /api/v1/network` → `mode`, `connected`, `ip` |
| Sensores | `GET /api/v1/sensors` → cada sensor esperado presente y `healthy` |
| Reglas y alarmas | `GET /api/v1/alarms` tras provocar un valor fuera de umbral |
| Publicadores | Publicar una medición de prueba y verificar webhook/MQTT |
| MicroSD | `GET /api/v1/system` → `sd_enabled` y `history_available` |
| GPIO | `GET /api/v1/gpio` y probar `POST /api/v1/gpio` en un relé |

---

## Ver también

- [Configuración](Configuracion.md) · [Referencia de configuración](Referencia-configuracion.md)
- [Variables modificables](Variables-modificables.md) · [Seguridad](Seguridad.md)
- [OTA y actualización](OTA-y-Actualizacion.md) · [Almacenamiento e histórico](Almacenamiento-e-historico.md)
