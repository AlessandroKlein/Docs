---
tags:
  - sema
  - versiones
  - compatibilidad
---

# Compatibilidad de versiones

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Qué versiona SEMA, cómo se comporta frente a una configuración, un backup o un
firmware de otra versión, y qué compatibilidad **no** está implementada. Todo lo
verificado apunta al archivo y la línea que lo decide.

## 1. Los cuatro números del firmware

`include/core/Version.hpp` (18 líneas):

| Constante | Valor en v1.103.0 | Qué versiona | Dónde se ve |
|-----------|-------------------|--------------|-------------|
| `SEMA_FW_VERSION` | `"1.103.0"` | Firmware (SemVer, **sin** `v`) | Banner serie, `/api/v1/status`, `/api/v1/system` (`firmware`), `/api/v1/diagnostics`, `/api/v1/backup` (`firmware`) |
| `SEMA_HW_VERSION` | `"rev0"` | Revisión del hardware (acumulativa) | Banner serie, `/api/v1/system` (`hw`), `/api/v1/diagnostics` (`hw`) |
| `SEMA_CONFIG_SCHEMA_VERSION` | `1` | Estructura del JSON de configuración | Banner serie, `/api/v1/system` (`config_schema`) |
| `SEMA_PROTOCOL_VERSION` | `1` | Protocolo con el Servidor Central | Banner serie, `/api/v1/system` (`protocol`) |

El prefijo `v` minúscula queda reservado para tags de git y releases de GitHub; el
firmware, el manifest y el CHANGELOG van sin `v` (`docs/VERSIONADO.md` §2).

### 1.1 Convención SemVer aplicada

`docs/VERSIONADO.md` fija la regla: MAJOR solo cuando algo **deja de funcionar**
para quien depende del firmware (el servidor central, las instalaciones ya
flasheadas o el cableado del usuario); MINOR para funcionalidad nueva
retrocompatible; PATCH para correcciones; `docs:`/refactor sin cambio visible **no**
genera release. El historial real: `CHANGELOG.md` (843 líneas) llega hasta
`## [1.103.0] - 2026-10-08`.

⚠️ **No implementado:** `docs/VERSIONADO.md` §6 describe un canal de actualización
(`stable` / `beta` / `development`) y el campo `update_channel`. En el código de
SEMA **no existe** esa cadena ni ese campo: `/api/v1/update/check` no filtra por
canal y siempre compara contra la rama `main`.

## 2. Esquema de configuración (`SEMA_CONFIG_SCHEMA_VERSION`)

### 2.1 Qué pasa hoy cuando el esquema no es 1

No hay migración. El único lugar donde se acepta o rechaza un esquema es
`ConfigManager::validate()`:

```cpp
// src/core/ConfigManager.cpp:103-106
bool ConfigManager::validate(const Config& c) const {
  if (c.schemaVersion != 1) {
    return false;
  }
  ...
```

| Escenario | Camino del código | Resultado real |
|-----------|-------------------|----------------|
| Arranque con NVS de esquema 1 | `load()` → `deserialize()` → `validate()` OK | Config cargada, `valid_ = true` |
| Arranque con NVS de esquema ≠ 1 | `load()` → `deserialize()` OK pero `validate()` falla | **Se descarta la config guardada y se aplican los defaults** (no se avisa al usuario más que por serie: `Config válida: no` antes de los defaults) |
| Arranque sin NVS | `load()` → defaults | `SEMA-001`, `Estación Norte`, STA, `sema-001`, mDNS, Buenos Aires, INFO, littlefs, 30 días |
| `PUT /api/v1/config` con `schema_version` ≠ 1 | `applyJson` → `parseInto` → `apply` → `validate` falla | HTTP 400 `{"error":"invalid config"}`; la config vigente **no** cambia |
| Config de esquema 1 a la que le faltan claves | `parseInto` aplica `doc["x"] \| default` a **cada** campo | Se completa con defaults: es un lector tolerante hacia adelante |

```text
❌ No existe migrate() en el firmware (grep de "migrate" en src/ e include/ → 0 coincidencias).
❌ No hay aviso al usuario por API cuando se descarta una config por esquema.
✅ El lector de JSON es tolerante a claves faltantes y a claves desconocidas.
```

### 2.2 Cómo se migraría (⚠️ propuesta, no implementada)

1. Subir `SEMA_CONFIG_SCHEMA_VERSION` a `2` en `include/core/Version.hpp`.
2. Agregar `bool migrate(uint32_t from, Config& c)` en `ConfigManager` y llamarlo
   **después** de `parseInto` y **antes** de `validate`, porque hoy `validate()`
   corta con cualquier esquema distinto de 1.
3. Con el lector tolerante actual, casi toda migración se resuelve renombrando o
   rellenando campos; conviene registrar un `Event` de tipo `System` con la
   migración aplicada.
4. `docs/VERSIONADO.md` §7 ya exige este paso como convención del proyecto (bajo el
   nombre `ConfigManager::migrate()`, con prefijo `GH_` en su redacción genérica).

## 3. Compatibilidad de backups

### 3.1 Formato real

`GET /api/v1/backup` (requiere autenticación) serializa la config y le **inyecta**
cuatro claves de metadatos (`HttpServer.cpp:1375-1399`):

| Clave | Valor | Uso |
|-------|-------|-----|
| *(toda la config)* | schema 1 | `"schema_version": 1` + `station`, `network`, `system`, `storage`, `security`, `energy`, `publishers`, `sensors[]`, `rules[]`, `calibrations[]`, `gpio[]`, `mcp23s17_cs`, `mcp23s17_pins[]`, `shift_registers[]`, `spi_expanders[]`, `modbus`, `can`, `lora`, `zigbee`, `ethernet`, `i2c_sda`, `i2c_scl` |
| `backup_format` | `"sema-backup"` | Marca de formato |
| `backup_version` | `1` | Versión del **formato de backup** (independiente de `schema_version`) |
| `firmware` | `SEMA_FW_VERSION` | Firmware que generó el backup (trazabilidad) |
| `timestamp` | `millis()/1000` | Uptime del equipo al exportar (**no** es fecha real) |

### 3.2 Restauración: qué se mira y qué se ignora

`POST /api/v1/backup` apunta al **mismo** handler que `PUT /api/v1/config`
(`onConfigPut`), que llama a `applyJson()`. Consecuencias verificadas:

| Aspecto | Comportamiento real |
|---------|---------------------|
| `backup_format`, `backup_version`, `firmware`, `timestamp` | **Se ignoran** al restaurar (comentario explícito en `HttpServer.cpp:1381`; `parseInto` solo lee claves conocidas) |
| `schema_version` ≠ 1 dentro del backup | Rechazado con HTTP 400: un backup de otro esquema **no migra** |
| Claves nuevas que el firmware no conoce | Ignoradas en silencio (restauración parcial) |
| Claves ausentes en el backup | Se completan con los defaults del código |
| `system.dashboard_layout` | ❌ **No viaja en el backup**: `serialize()` no emite `dashboard_layout` y el layout vive en la clave NVS separada `layout` (`saveDashboardLayout`). Restaurar un backup **no** restaura el dashboard |
| Claves `security.*` | Viajan en claro (JSON sin cifrar) |
| Efecto inmediato | Tras aplicar, `onConfigPut` re-aplica todo sin reiniciar: `applySensors`, `applyRules`, `applyCalibrations`, `applyGpio`, `applyPublishers`, `applyShift`, `applyModbus`, `applyCan`, `applyLora`, `applyZigbee`, `applyEthernet` |

```text
Compatibilidad real de backups:
  mismo backup_version + mismo schema_version  → ✅ restaura completo (menos el layout)
  backup_version distinto, schema igual        → ⚠️ se ignora la marca: puede quedar parcial
  schema_version distinto                      → ❌ HTTP 400, sin migración
```

Los backups son un JSON autodescriptivo que también sirve como documento de
configuración manual: ver [Referencia de configuración](Referencia-configuracion.md).

## 4. Compatibilidad de firmware (OTA)

`POST /api/v1/ota` (`onOtaUpload` + `onOta`) es el único camino de actualización
remota. Controles que **sí** existen:

| Control | Implementación | Efecto |
|---------|----------------|--------|
| Autorización | `X-API-Key` verificada al iniciar la subida (`otaAuthorized_`) | Sin clave válida no se escribe nada en la partición |
| Integridad | Cabecera `X-SHA256` (64 hex) comparada contra el SHA-256 calculado de la partición OTA (`otaPartitionSha256`) | No coincide → HTTP 400 `sha256 mismatch` y **no reinicia** |
| Rollback de particiones | Tablas `partitions_*.csv` con `app0`/`app1` + `otadata` | Permite volver a la imagen anterior a nivel bootloader |
| Publicación | `firmware_manifest.json` con `sha256` por board | Es la fuente del hash que debe mandar el cliente |

Controles que **no** existen:

| Falta | Consecuencia |
|-------|--------------|
| Verificación de board/modelo en el OTA | Nada impide subir un binario de otra board; sólo lo frenaría el hash (tamaños de partición distintos ⇒ hash distinto) |
| Firma digital del firmware | El SHA-256 es un checksum, no autenticación: quien controla el canal puede recalcularlo |
| Bloqueo de downgrade | Cualquier binario con hash válido se acepta, sea más nuevo o más viejo |
| Comparación SemVer en `/api/v1/update/check` | Usa desigualdad de cadenas: `resp["update"] = (ver != SEMA_FW_VERSION)`; una versión anterior también se reporta como `update: true` |
| Origen configurable del update | La URL está fija en el código: `raw.githubusercontent.com/AlessandroKlein/SEMA/refs/heads/main/firmware_manifest.json` |
| Verificación de `hw_version` en runtime | El manifest declara `hw_version: "rev0"`, pero el firmware nunca lo lee ni lo compara con `SEMA_HW_VERSION` |

Respuesta de `GET /api/v1/update/check`: `{"current":"1.103.0","latest":"<version del manifest>","update":<bool>,"url":"https://github.com/AlessandroKlein/SEMA/releases/tag/v<latest>"}`.
TLS del chequeo se hace con `client.setInsecure()` (sin validar el certificado).

⚠️ Detalle importante del hash del OTA: `otaPartitionSha256()` recorre **toda la
partición OTA** (`part->size`), no la longitud de la imagen. El `sha256` que publica
`firmware_manifest.json` es el del archivo `.bin`; por lo tanto el valor de
`X-SHA256` que espere el equipo debe ser el de la partición completa tal como la ve
el firmware, no necesariamente el del `.bin` descargado. Verificá el hash con el
equipo real antes de automatizar un despliegue.

## 5. `SEMA_PROTOCOL_VERSION`

| Aspecto | Estado real |
|---------|-------------|
| Dónde se define | `include/core/Version.hpp` = `1` |
| Dónde se expone | Banner serie y `GET /api/v1/system` → `"protocol": 1` |
| Negociación con el servidor | ❌ **No implementada**: no hay handshake, ni endpoints de versión de protocolo, ni validación de compatibilidad |
| `security.server_key` | Es solo una clave de autenticación alternativa (`authorized()`); no participa de ningún protocolo |
| Servidor Central | Fuera del alcance de SEMA (`docs/PENDIENTES.md` §2.2); la constante queda reservada para cuando exista |

```text
⚠️ SEMA_PROTOCOL_VERSION es hoy un valor informativo, no un contrato verificado.
```

## 6. Matriz de compatibilidad runtime ↔ firmware

Lo estrictamente verificable en el código y el manifest:

| Board ID (`SEMA_BOARD_ID`) | Entorno PlatformIO | Flash | Tabla de particiones | Ethernet nativo | Entrada en `firmware_manifest.json` | Nombre del binario |
|----------------------------|--------------------|------:|----------------------|:---------------:|:-----------------------------------:|--------------------|
| `esp32-wroom-4mb` | `esp32doit-devkit-v1` (default) | 4 MB | `partitions_4mb.csv` (`app0/app1` 0x1C0000, `spiffs` 0x70000) | Sí (`SEMA_NATIVE_ETH=1`) | ✅ `sema_1.103.0_esp32-wroom-4mb.bin` | `sema_<fw>_esp32-wroom-4mb.bin` |
| `esp32-s3-8mb` | `esp32-s3-devkitc-1` | 8 MB | `partitions_8mb.csv` (`app0/app1` 0x300000, `spiffs` 0x1F0000) | No (W5500 SPI) | ✅ `sema_1.103.0_esp32-s3-8mb.bin` | `sema_<fw>_esp32-s3-8mb.bin` |
| `esp32-wroom32u-16mb` | `esp32-wroom-32u` (PCB futura) | 16 MB | `partitions_16mb.csv` (`app0/app1` 0x300000, `spiffs` 0x9F0000) | Sí | ❌ **Sin entrada en el manifest** | No publicado |

Notas verificadas:

- `GET /api/v1/system` devuelve `board`, `flash_mb`, `pins_from_file`, `demo`,
  `native_eth`, `firmware_file` (`sema_<fw>_<board>.bin`) y `reserved_pins[]`; es la
  forma de saber **desde el equipo** qué binario corresponde.
- El firmware **no** consulta el manifest para auto-validarse; el manifest solo lo
  consume `/api/v1/update/check` (y el servidor/release).
- El manifest lista `filesystem` + `filesystem_sha256` por board (imagen LittleFS),
  además del `firmware` + `sha256`.
- Las tablas de particiones difieren en tamaño y offset: un binario de 8 MB no entra
  en una partición de 4 MB, así que la incompatibilidad de flash es física aunque el
  código no la valide.
- Usar `SEMA_PINS_FROM_FILE=1` cambia el mapeo de pines en runtime
  (`ConfigManager::applyHwProfile`): un backup exportado con `SEMA_PINS_FROM_FILE=0`
  y restaurado en un equipo con `=1` verá sobrescritos los pines de CAN, Modbus,
  Zigbee, LoRa y Ethernet.

```text
⚠️ La matriz "qué firmware corre en qué runtime" es conceptual: el código no
   valida board, chip, flash ni hw_version en ningún camino (arranque, OTA o
   restauración de config). Se deduce de SEMA_BOARD_ID + las tablas de particiones
   + el manifest, no de una comprobación en runtime.
```

## 7. Resumen de compatibilidad

| Cambio | ¿Rompe? | Mecanismo actual |
|--------|:-------:|------------------|
| Campo nuevo en la config | ❌ | `parseInto` aplica default por campo |
| Campo eliminado de la config | ❌ | La clave se ignora al leer |
| Cambio incompatible del JSON de config | ✅ | `validate()` rechaza el esquema ≠ 1 y cae a defaults (sin migración) |
| Backup de otra versión de firmware con el mismo schema | ❌ | Se ignoran `backup_*`/`firmware`/`timestamp` |
| Backup de otro schema | ✅ | HTTP 400 |
| Firmware nuevo en la misma board | ❌ | OTA con `X-SHA256` y particiones A/B |
| Firmware de otra board | ✅ (no validado) | Solo lo frenaría el hash; no hay chequeo de board |
| Downgrade de firmware | ⚠️ permitido | No hay bloqueo de versión |
| Protocolo con el Servidor Central | ⚠️ indefinido | `SEMA_PROTOCOL_VERSION` es informativo |

## 8. Referencias de historial

| Fuente | Contenido |
|--------|-----------|
| `CHANGELOG.md` (repo de código) | 843 líneas, formato Keep a Changelog; última entrada `[1.103.0] - 2026-10-08` |
| `docs/VERSIONADO.md` | Convención completa (SemVer, `v` minúscula, canales, fases `V8`/`V9`) |
| `firmware_manifest.json` | `version`, `hw_version`, `config_schema_version`, `protocol_version` y los binarios con SHA-256 |
| `.release/` | Binarios publicados por release (desde `1.27.0` hasta `1.103.0` en el repo local) |

⚠️ Discrepancia detectada en `docs/CONTINUACION.md`: la cabecera dice
`Releases: 89` y `Firmware: v1.103.0`, pero el §4 (checklist) sigue pidiendo
"52 releases" y un `firmware_manifest.json` con `version: 0.52.0`. El checklist está
desactualizado respecto del manifest real (`1.103.0`).

---

## Ver también

- [Referencia de código](Referencia-de-codigo.md) · [Referencia de configuración](Referencia-configuracion.md) · [OTA y actualización](OTA-y-Actualizacion.md) · [Backup y restauración](Backup-y-restauracion.md) · [Pruebas y validación](Pruebas-y-validacion.md) · [Registro de cambios](Registro-de-cambios.md) · [Evolución](Evolucion.md) · [Enumeraciones y tipos](Enumeraciones-y-tipos.md)
