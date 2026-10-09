---
tags:
  - sema
  - configuracion
---

# Variables modificables

> **Tipo:** Configuración | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Clasificación **real** (verificada contra el código) de cada grupo de variables
del JSON de configuración según lo que pasa al guardarlas: se aplican en
caliente, requieren reinicio o directamente no tienen efecto hoy. La definición
de cada clave está en [Referencia de configuración](Referencia-configuracion.md).

---

## 1. Resumen

| Grupo de claves | Al guardar con `PUT /api/v1/config` | Motivo |
|-----------------|-------------------------------------|--------|
| `sensors[]` | ✅ Caliente | `applySensors()` recrea el catálogo |
| `rules[]` | ✅ Caliente | `applyRules()` |
| `calibrations[]` | ✅ Caliente | `applyCalibrations()` |
| `gpio[]` | ✅ Caliente | `applyGpio()` |
| `publishers.*` | ✅ Caliente | `applyPublishers()` (webhook + MQTT) |
| `modbus.*` | ✅ Caliente | `applyModbus()` |
| `can.*` | ⚠️ Con reservas | `applyCan()` re-instala el driver TWAI sin desinstalarlo (§5) |
| `lora.*` | ✅ Caliente | `applyLora()` libera el radio anterior antes de recrearlo |
| `zigbee.*` | ✅ Caliente | `applyZigbee()` |
| `ethernet.*` | ✅ Caliente | `applyEthernet()` |
| `system.altitude`, `system.wind_*` | ✅ Caliente | Viajan dentro de `derived_.configure(system)` que llama `applySensors()` |
| `system.units`, `system.lang` | ✅ Caliente | Se leen al generar respuestas y páginas |
| `station.*` | ✅ Caliente | Se leen en cada request |
| `security.*` | ✅ Caliente | `authorized()`/`webAuthed()` leen la config en cada request |
| `storage.retention_days` | ✅ Caliente | La poda horaria lee la config en cada pasada |
| `network.*` | ❌ Reinicio | `WiFiManager::begin()` solo corre en `setup()` |
| `system.timezone`, `system.ntp_server` | ❌ Reinicio | `configTzTime()` solo corre en `setup()` |
| `storage.sd_enabled`, `storage.sd_cs` | ❌ Reinicio | `history_.enableSd()` solo corre en `setup()` |
| `energy.rain_pin` | ❌ Reinicio | `enableRainWakeup()` (wake `ext0`) solo corre en `setup()` |
| `system.log_level`, `storage.backend` | ⛔ Sin efecto | No hay consumidor |
| `i2c_sda`, `i2c_scl` | ⛔ Sin efecto | El bus usa `SEMA_PIN_I2C_SDA`/`SEMA_PIN_I2C_SCL` |
| `sensors[].address`, `sensors[].bus`, `sensors[].uart`, `sensors[].uart_port` | ⛔ Sin efecto | `SensorFactory` no los lee |
| `mcp23s17_cs`, `mcp23s17_pins`, `shift_registers[]`, `spi_expanders[]` | ⛔ Sin efecto | No hay driver (y `SEMA_USE_SHIFT=0` por defecto) |

!!! info "Los endpoints parciales tienen su propio alcance"
    `POST /api/v1/config/system` solo toca `timezone`, `ntp_server`, `units`,
    `lang`, `sd_enabled` y `sd_cs`; `POST /api/v1/config/buses` **no** aplica
    `baud`, `slave_id`, `register`, `count`, `speed` ni los parámetros de radio
    (ver [Referencia de configuración](Referencia-configuracion.md) §13).

## 2. Cómo se aplica el cambio

```text
PUT /api/v1/config
   │  applyJson() → parseInto() → validate() → save() (NVS)
   │
   └─ si OK, onConfigPut() re-aplica en caliente:
        applySensors()  → catálogo + derived_.configure(system)
        applyRules()
        applyCalibrations()
        applyGpio()
        applyPublishers()
        applyShift()    (no-op con SEMA_USE_SHIFT=0)
        applyModbus() / applyCan() / applyLora() / applyZigbee() / applyEthernet()
   │
   └─ si el JSON no valida → 400 y NO se aplica nada (rollback en RAM)
```

Los endpoints parciales hacen exactamente lo mismo pero con `next` partiendo de
la configuración vigente y modificando solo su sección; `POST
/api/v1/config/network` además reinicia el equipo al responder.

## 3. Variables críticas de identidad

| Variable | Efecto real | Verificación |
|----------|-------------|--------------|
| `station.id` | Se publica como `station_id` en mediciones, MQTT y webhook; **no puede quedar vacío** | `validate()` (`ConfigManager.cpp:107`), `onStatus()` |
| `station.name` | Nombre visible en la web y en `/api/v1/status` | `HttpServer.cpp:1346-1347` |
| `network.hostname` | Host mDNS (`http://<hostname>.local`); requiere reinicio | `SemaCore.cpp:152`, `SemaCore.cpp:222` |
| `security.username` | Usuario del login web; vacío = `admin` | `HttpServer.cpp:1318` |
| `security.password` | Si está vacía, **la web y los endpoints de sesión quedan abiertos** | `webAuthed()` (`HttpServer.cpp:1282-1284`) |

## 4. Variables de seguridad

| Variable | Efecto real |
|----------|-------------|
| `security.api_key` | Habilita `X-API-Key` para toda la API autenticada y **también** el acceso a las páginas web (`webAuthed()` acepta sesión o API key) |
| `security.server_key` | Segunda clave con la misma semántica; pensada para el Servidor Central |
| `security.extra_keys` | Mapa `{"nombre":"clave"}` (como **string** JSON) con claves adicionales revocables |
| `security.password` + `security.username` | Login web (cookie `sema_auth`, 1 h deslizante) |

!!! danger "Sin claves, la API queda abierta"
    Si `api_key`, `server_key` y `extra_keys` están **las tres** vacías,
    `authorized()` devuelve `true` sin mirar el header
    (`HttpServer.cpp:1469-1472`): cualquiera en la red puede escribir la config.
    Es el estado de fábrica pensado para la primera puesta en marcha.

## 5. Advertencias al modificar la configuración

1. **Los arrays se reemplazan enteros, no se mezclan.** `PUT /api/v1/config` con
   `sensors` de 2 ítems borra los otros; `POST /api/v1/config/sensors` hace
   `next.sensors.clear()` antes de reconstruir (`HttpServer.cpp:1545`).
2. **Guardar «Expansores y salidas» desde la web borra `gpio[]`.** La página
   embebida manda `gpio: []` en el body (`HttpServer.cpp:1271`) y el endpoint,
   al ver la clave presente, vacía la lista (`HttpServer.cpp:1621-1633`). Si
   tenés GPIO standalone, volvé a cargarlos o usá `PUT /api/v1/config`.
3. **`POST /api/v1/config/sensors` descarta `address`/`rom`/`bus`/`uart`/`uart_port`**
   (los deja en `0`/vacío) aunque la UI los envíe.
4. **`POST /api/v1/config/io` usa `latch` y `expander`**, mientras el JSON
   completo usa `latch_pin` y `expander_addr` (§13 de la referencia).
5. **`POST /api/v1/config/network` reinicia siempre** (responde `200` y a los
   300 ms llama a `ESP.restart()`, `HttpServer.cpp:1524-1527`). Guardá el resto
   de los cambios antes.
6. **`POST /api/v1/config/system` no reinicia**: `sd_enabled`, `sd_cs`,
   `timezone` y `ntp_server` quedan persistidos pero inactivos hasta el próximo
   arranque o `POST /api/v1/restart`.
7. **Cambiar la contraseña no cierra la sesión web abierta**: la cookie
   `sema_auth` sigue siendo válida hasta el timeout de 1 h o hasta `/logout`
   (el token depende del arranque, no de la contraseña).
8. **`can.*` en caliente puede dejar el bus inoperante**: `CanManager::apply()`
   no desinstala el driver antes de reinstalarlo
   (`src/core/CanManager.cpp:21-38`). Si cambiás CAN, reiniciá.
9. **`station.id` vacío invalida todo el documento**, no solo ese campo.
10. **`schema_version` distinto de `1` descarta la config completa** y el equipo
    arranca con defaults (no hay migración).
11. **`sd_enabled` con `sd_cs` mal elegido** puede chocar con los pines del bus
    SPI (`SEMA_SPI_SCK`/`MISO`/`MOSI`, reservados en `GET /api/v1/system`).
12. **Las claves del JSON viajan en claro**: `GET /api/v1/config` devuelve
    `api_key`, `server_key`, `password` y `mqtt_pass` sin enmascarar (solo exige
    autenticación). Ver [Seguridad](Seguridad.md) §5.

## 6. Semántica de `sensors[]`

- Cada elemento es una entrada del catálogo. **`enabled: false` (o ausente) =
  sensor apagado**: `applySensors()` lo saltea (`SemaCore.cpp:300-303`). Es un
  valor obligatorio en la práctica para que el sensor exista.
- `id` + `channel` forman la clave pública `<sensor_id>:<channel_id>` que usan
  las reglas y las calibraciones.
- Con `sensors[]` vacío se usa el **catálogo fijo** de seis sensores descrito en
  [Referencia de configuración](Referencia-configuracion.md) §11; con
  `SEMA_FIXED_HARDWARE=1` el catálogo fijo se usa siempre.
- `model` debe pertenecer al catálogo de 17 modelos; un modelo desconocido no se
  crea y deja un `Sensor desconocido: <id> (modelo <model>)` en el log serie.
- `scale`/`offset` se aplican al valor crudo (ejemplo: batería 12 V con divisor
  11:1 → `3.3·11/4095 = 0.00887`).
- `rom` solo aplica a DS18B20 (16 hex); vacío = autodetección en el bus 1-Wire.

## 7. Semántica de `gpio[]` y actuadores

| Caso | Configuración | Cómo se opera |
|------|---------------|---------------|
| Salida nativa | `mode: "output"`, `pin: 26`, `initial: 0`, `expander_addr: 0` | `POST /api/v1/gpio` con `{"pin":26,"value":1}` |
| Entrada con pull-up | `mode: "input_pullup"` | `GET /api/v1/gpio` devuelve `value` por pin |
| Salida por MCP23017 | `expander_addr: 32` (`0x20`), `pin` de `0` a `15` | Igual que la nativa: `GpioManager` enruta al expansor si `begin_I2C()` respondió |
| Estado inicial | `initial: 1` | Se escribe en `GpioManager::apply()` al aplicar la config |

- Si `expander_addr != 0` pero el MCP23017 no responde en esa dirección, el pin
  se ignora (⚠️ `mcpReady_ = false`, `src/core/GpioManager.cpp:23-52`).
- `POST /api/v1/gpio` escribe **sin verificar el modo**: un pin configurado como
  entrada igual recibe `digitalWrite()`.
- `GET /api/v1/gpio` y `POST /api/v1/gpio` existen siempre, pero `POST` requiere
  autenticación; `GET` es **público**.

## 8. Qué conviene reiniciar después de guardar

| Cambio | Acción recomendada |
|--------|--------------------|
| `network.*` | Automático (`POST /api/v1/config/network` reinicia) |
| `storage.sd_enabled`, `storage.sd_cs` | `POST /api/v1/restart` para que `enableSd()` corra |
| `system.timezone`, `system.ntp_server` | `POST /api/v1/restart` para re-sincronizar NTP |
| `energy.rain_pin` | `POST /api/v1/restart` para armar el wake `ext0` |
| `can.*` | `POST /api/v1/restart` |
| Restauración de backup completo | `POST /api/v1/restart` al terminar |

---

## Ver también

- [Configuración](Configuracion.md) · [Referencia de configuración](Referencia-configuracion.md)
- [Seguridad](Seguridad.md) · [Identidad y estados](Identidad-y-estados.md)
- [Almacenamiento e histórico](Almacenamiento-e-historico.md) · [Energía y consumo](Energia-y-consumo.md)
