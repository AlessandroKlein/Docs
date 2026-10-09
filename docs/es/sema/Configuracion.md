---
tags:
  - sema
  - configuracion
---

# Configuración

> **Tipo:** Configuración | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

SEMA se configura con **un único documento JSON** (`schema_version: 1`) que se
persiste en la NVS del ESP32 y se aplica de forma **transaccional** (validar →
aplicar → persistir → rollback). Esta página explica dónde vive, cómo se edita,
qué se aplica en caliente y qué necesita reinicio. La lista completa de claves
está en [Referencia de configuración](Referencia-configuracion.md).

---

## 1. Dónde vive la configuración

| Aspecto | Valor real | Código |
|---------|------------|--------|
| Backend | NVS (`Preferences` de arduino-esp32) vía `NvsStore` | `src/core/storage/NvsStore.cpp`, `include/core/storage/Storage.hpp` |
| Namespace NVS | `sema` | `src/core/SemaCore.cpp:50` (`store_.begin("sema")`) |
| Clave de la config | `config` (un string con el JSON completo) | `src/core/ConfigManager.cpp:9` (`kConfigKey`) |
| Clave del layout | `layout` (JSON del dashboard Gridstack) | `ConfigManager::saveDashboardLayout()` (`ConfigManager.cpp:475`) |
| Clave del contador | `boots` (`uint32`, contador de reinicios) | `src/core/SemaCore.cpp:57-59` |
| Partición | `nvs`, `data/nvs`, offset `0x9000`, tamaño `0x5000` (20 KB) en las tres placas | `partitions_4mb.csv`, `partitions_8mb.csv`, `partitions_16mb.csv` |

```text
NVS (namespace "sema")
├── config   → JSON de configuración (schema_version = 1)
├── layout   → layout del dashboard (clave aparte, ver §2)
└── boots    → uint32 con la cantidad de arranques
```

!!! warning "`storage.backend` no elige dónde se guarda la config"
    La clave `storage.backend` (`"littlefs"` · `"flash"` · `"sd"`) **solo se
    valida y se persiste**: ningún módulo la lee para decidir el medio. La
    configuración **siempre** va a NVS. El histórico va a la microSD (`SD.begin`
    en `src/core/storage/HistoryStore.cpp:40`) y los eventos a LittleFS
    (`src/core/events/EventLog.cpp:18`).

## 2. Formato y tamaño

- Un solo objeto JSON; el parseo y la serialización usan un
  `DynamicJsonDocument` de **16 384 bytes** (`ConfigManager::serialize()` y
  `parseInto()`), así que el documento debe quedar cómodamente por debajo de ese
  tamaño.
- El layout del dashboard **no** está dentro del JSON de config: se guarda en la
  clave NVS `layout` para no exceder el límite de tamaño de una entrada NVS
  (comentario en `include/core/ConfigManager.hpp:252-254`).
- `schema_version` debe ser `1` (`SEMA_CONFIG_SCHEMA_VERSION` en
  `include/core/Version.hpp:17`). Si la clave falta, el parser asume `1`.
- **Todas las claves son opcionales al parsear**: la que falta toma su default
  (columna «Default» de [Referencia de configuración](Referencia-configuracion.md)).
  La única excepción es `station.id`, que no puede quedar vacío (§6).

## 3. Vías de edición

| Vía | Entrada | Autenticación | Alcance |
|-----|---------|---------------|---------|
| Web embebida | `/config/network`, `/config/system`, `/config/security`, `/config/wind`, `/config/sensors` | Login web (cookie `sema_auth`) o `X-API-Key` | Secciones puntuales |
| API (completa) | `PUT /api/v1/config` | Sesión o `X-API-Key` | Reemplaza **todo** el documento |
| API (parcial) | `POST /api/v1/config/network` | Sesión o `X-API-Key` | `network.*` (y **reinicia**) |
| API (parcial) | `POST /api/v1/config/system` | Sesión o `X-API-Key` | `system.timezone`, `system.ntp_server`, `system.units`, `system.lang`, `storage.sd_enabled`, `storage.sd_cs` |
| API (parcial) | `POST /api/v1/config/sensors` | Sesión o `X-API-Key` | Reemplaza `sensors[]` |
| API (parcial) | `POST /api/v1/config/io` | Sesión o `X-API-Key` | `mcp23s17_cs`, `mcp23s17_pins`, `shift_registers`, `spi_expanders`, `gpio[]` |
| API (parcial) | `POST /api/v1/config/buses` | Sesión o `X-API-Key` | `i2c_sda`, `i2c_scl`, `sd_cs`, `modbus.*`, `can.*`, `lora.*`, `zigbee.*`, `ethernet.*` |
| API (lectura) | `GET /api/v1/config` | Sesión o `X-API-Key` | Devuelve el JSON completo (incluye claves) |
| Restauración | `POST /api/v1/backup` | Sesión o `X-API-Key` | Mismo handler que `PUT /api/v1/config` |
| Escrituras indirectas | `POST /api/v1/security/keys`, `POST /api/v1/wind/north`, `POST /api/v1/wind/resistors`, `GET`/`POST /api/v1/dashboard/layout` | Sesión o `X-API-Key` | `security.extra_keys`, `system.wind_north_offset`, `system.wind_rpull`, `system.wind_resistors`, clave `layout` |

> La diferencia entre «sesión» y `X-API-Key` no es cosmética: `POST /api/v1/ota`
> **solo** acepta `X-API-Key` (ver [Seguridad](Seguridad.md) §3 y
> [OTA y actualización](OTA-y-Actualizacion.md) §3).

## 4. Escritura transaccional

`ConfigManager::apply()` (`src/core/ConfigManager.cpp:89-101`) implementa el
ciclo completo:

```text
POST/PUT (JSON)
   │
   ├─ parseInto(json, next)        ← falla si el JSON no parsea (HTTP 400)
   │
   ├─ validate(next)               ← 4 reglas; si falla → false (HTTP 400/500)
   │
   ├─ backup_ = config_            ← copia de seguridad en RAM
   ├─ config_  = next
   │
   └─ save()
        ├─ serialize() + NVS putString("config")
        │     └─ si NVS está lleno → store_.clear() + un reintento
        ├─ OK   → valid_ = true      → HTTP 200 {"ok":true}
        └─ FALLA→ config_ = backup_  → rollback → HTTP 500
```

Detalles verificados:

- **Parseo**: `parseInto()` devuelve `false` si `deserializeJson` falla
  (`ConfigManager.cpp:284-287`).
- **Validación**: ver §6.
- **Persistencia**: `save()` serializa y escribe la clave `config`. Si la
  escritura falla, asume NVS lleno/fragmentado, ejecuta `store_.clear()`
  (**borra `config`, `layout` y `boots`**) y reintenta **una sola vez**
  (`ConfigManager.cpp:78-85`).
- **Rollback**: si el segundo intento también falla, `config_` vuelve a la copia
  previa en RAM y la operación devuelve `false`. Ojo: si el `clear()` ya se
  ejecutó, la configuración anterior **ya no está en NVS**; el rollback solo
  protege la copia en RAM hasta el próximo reinicio.
- **Respuestas HTTP**: `PUT /api/v1/config` y `POST /api/v1/backup` devuelven
  `400 {"error":"invalid config"}` si el JSON no parsea o no valida; los
  endpoints parciales devuelven `400 {"error":"invalid json"}` o
  `400 {"error":"body required"}` y `500 {"error":"config apply failed"}` si el
  `apply()` falla.

## 5. Qué se aplica en caliente y qué necesita reinicio

### 5.1 Se aplica sin reiniciar (`PUT /api/v1/config`)

Al aceptar la config, `onConfigPut()` (`src/core/web/HttpServer.cpp:1871-1894`)
re-aplica en este orden:

| Orden | Llamada | Efecto | Código |
|:-----:|---------|--------|--------|
| 1 | `applySensors()` | Recrea el catálogo desde `sensors[]` y reconfigura las derivadas con `system.*` (`altitude`, veleta) | `SemaCore.cpp:270-315` |
| 2 | `applyRules()` | Recarga `rules[]` | `SemaCore.cpp:253-268` |
| 3 | `applyCalibrations()` | Recarga `calibrations[]` | `SemaCore.cpp:227-251` |
| 4 | `applyGpio()` | Reaplica `gpio[]` (GPIO nativo y MCP23017) | `SemaCore.cpp:317-319` |
| 5 | `applyPublishers()` | Reconfigura webhook y MQTT | `SemaCore.cpp:357-364` |
| 6 | `applyShift()` | Reaplica `shift_registers[]` **solo si `SEMA_USE_SHIFT=1`** | `SemaCore.cpp:321-325` |
| 7 | `applyModbus()` | Reaplica `modbus.*` | `SemaCore.cpp:328-330` |
| 8 | `applyCan()` | Reaplica `can.*` | `SemaCore.cpp:334-336` |
| 9 | `applyLora()` | Reaplica `lora.*` (libera el radio anterior) | `LoraManager.cpp:13-35` |
| 10 | `applyZigbee()` | Reaplica `zigbee.*` | `SemaCore.cpp:346-348` |
| 11 | `applyEthernet()` | Reaplica `ethernet.*` | `SemaCore.cpp:352-354` |

Además se leen **en cada petición o cada ciclo** (por eso un cambio tiene efecto
inmediato aunque no haya `apply*`):

| Clave | Por qué es inmediata |
|-------|----------------------|
| `station.id`, `station.name` | `onStatus()`/`onSystem()` leen `config().get()` en cada request |
| `security.*` | `authorized()`/`webAuthed()` leen la config en cada request |
| `storage.retention_days` | La poda horaria lee la config en cada pasada (`SemaCore.cpp:385`) |
| `system.units`, `system.lang` | Se leen al generar respuestas/páginas |

!!! warning "CAN en caliente"
    `CanManager::apply()` vuelve a llamar a `twai_driver_install()` sin
    `twai_driver_uninstall()` previo (`src/core/CanManager.cpp:21-38`); si el
    driver ya estaba instalado, la instalación falla y `ready_` queda en `false`
    (el bus deja de operar). Para cambiar `can.*` lo seguro es guardar y
    **reiniciar**.

### 5.2 Requiere reinicio

| Clave | Motivo | Código |
|-------|--------|--------|
| `network.mode`, `ssid`, `password`, `hostname`, `mdns`, `ip`, `gateway`, `subnet`, `dns` | `WiFiManager::begin()` corre una sola vez en `setup()` | `SemaCore.cpp:149-157` |
| `storage.sd_enabled`, `storage.sd_cs` | `history_.enableSd()` corre en `setup()` | `SemaCore.cpp:84-89` |
| `energy.rain_pin` | `enableRainWakeup()` (wake `ext0`) corre en `setup()` | `SemaCore.cpp:102-104`, `PowerManager.cpp:19-23` |
| `system.timezone`, `system.ntp_server` | `configTzTime()` corre en `setup()` | `SemaCore.cpp:161-162` |

`POST /api/v1/config/network` es el único endpoint parcial que **reinicia solo**:
responde `200`, espera 300 ms y llama a `ESP.restart()`
(`HttpServer.cpp:1524-1527`). El resto de los cambios «de reinicio» quedan
guardados y se aplican en el próximo arranque (la web lo avisa con
«Guardado (reiniciá para aplicar)» al habilitar la microSD).

### 5.3 Claves guardadas que hoy **no** tienen efecto

Estas claves se aceptan, se validan y se persisten, pero ningún módulo del
firmware las lee (❌ no implementado):

| Clave | Situación |
|-------|-----------|
| `system.log_level` | Solo se guarda/serializa; no hay logger que lo consuma |
| `storage.backend` | Solo se valida (`littlefs`/`flash`/`sd`); nadie lo lee |
| `sensors[].address`, `sensors[].bus`, `sensors[].uart`, `sensors[].uart_port` | `SensorFactory` no los usa (`src/core/sensors/SensorFactory.cpp`) |
| `mcp23s17_cs`, `mcp23s17_pins` | No hay driver MCP23S17 en `src/` (solo config + UI) |
| `shift_registers[]` | `SEMA_USE_SHIFT=0` por defecto (`include/hw/HwProfile.hpp:68-69`); sin driver activo |
| `spi_expanders[]` | No hay driver MAX14830 ni SC18IS602B en `src/` (solo config + UI) |
| `i2c_sda`, `i2c_scl` | El bus se abre con `SEMA_PIN_I2C_SDA`/`SEMA_PIN_I2C_SCL` (`SemaCore.cpp:112`) |
| `system.dashboard_layout` | Campo del struct `SystemConfig` que nunca se serializa ni se parsea |

## 6. Validación y recuperación

`ConfigManager::validate()` (`ConfigManager.cpp:103-118`) aplica **exactamente**
cuatro reglas:

1. `schema_version == 1`.
2. `station.id` con longitud mayor que cero.
3. `network.mode` ∈ {`"STA"`, `"AP"`}.
4. `storage.backend` ∈ {`"littlefs"`, `"flash"`, `"sd"`}.

No se validan rangos de pines, direcciones I²C, coherencia de canales ni
duplicados de `id`: un JSON sintácticamente válido y con esas cuatro condiciones
se acepta tal cual.

### Config corrupta o inválida

`ConfigManager::load()` (`ConfigManager.cpp:45-69`):

```text
NVS "config"
├── no existe / vacío ──────────────► defaults
├── deserialize() falla ────────────► defaults
├── validate() falla ───────────────► defaults
└── ok ─────────────────────────────► valid_ = true
```

Los defaults que se cargan en RAM son los del struct `Config` más los campos
fijados en `load()`: `station.id = "SEMA-001"`, `station.name = "Estación Norte"`,
`network.mode = "STA"`, `network.hostname = "sema-001"`, `network.mdns = true`,
`system.timezone = "America/Argentina/Buenos_Aires"`, `system.log_level = "INFO"`,
`storage.backend = "littlefs"`, `storage.retention_days = 30`.

Puntos clave:

- La NVS **no se sobrescribe** al cargar defaults: el equipo arranca con
  defaults en RAM y el JSON inválido sigue en NVS hasta el próximo `save()`.
- El único indicador es el log serie `Config válida: no`
  (`SemaCore.cpp:100`) y `GET /api/v1/system` → `config_schema`.
- ❌ **No hay migración de esquema**: no existe `ConfigManager::migrate()` en el
  código. Un JSON con `schema_version != 1` se descarta completo y se usan
  defaults (⚠️ contradice a `docs/VERSIONADO.md`, que menciona `migrate()`).
- Recuperación práctica: guardar de nuevo la config desde la web o `PUT
  /api/v1/config` con un JSON válido; para volver a fábrica, guardar los
  defaults o borrar la partición NVS por serie.

## 7. Diagnóstico de la configuración

| Consulta | Qué devuelve |
|----------|--------------|
| `GET /api/v1/config` | El JSON completo (incluye `api_key`, claves WiFi y contraseñas) — requiere auth |
| `GET /api/v1/system` | `config_schema`, `board`, `flash_mb`, `pins_from_file`, `sd_enabled`, `history_available`, `firmware_file` |
| `GET /api/v1/status` | `station`, `name`, `firmware`, `uptime_s` |
| `GET /api/v1/backup` | Config + metadatos `backup_*` (ver [Backup y restauración](Backup-y-restauracion.md)) |

---

## Ver también

- [Referencia de configuración](Referencia-configuracion.md) · [Variables modificables](Variables-modificables.md)
- [Seguridad](Seguridad.md) · [Backup y restauración](Backup-y-restauracion.md)
- [API REST](API-REST.md) · [Almacenamiento e histórico](Almacenamiento-e-historico.md)
