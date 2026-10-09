---
tags:
  - sema
  - ota
---

# OTA y actualización

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Actualización de firmware por aire (OTA) con particiones A/B, manifiesto con
SHA-256 y las limitaciones reales del flujo implementado en
`src/core/web/HttpServer.cpp` (bloque `onOta`/`onOtaUpload`, líneas 1900-1986).
Para el detalle de credenciales, ver [Seguridad](Seguridad.md).

---

## 1. Particiones A/B reales

`platformio.ini` selecciona el CSV según el entorno
(`board_build.partitions`), y `board_build.filesystem = littlefs` hace que la
partición llamada `spiffs` contenga **LittleFS**.

| Entorno | Placa | CSV |
|---------|-------|-----|
| `esp32doit-devkit-v1` (default) | ESP32-WROOM 4 MB | `partitions_4mb.csv` |
| `esp32-s3-devkitc-1` | ESP32-S3 8 MB | `partitions_8mb.csv` |
| `esp32-wroom-32u` | ESP32-WROOM-32U 16 MB | `partitions_16mb.csv` |
| `demo` | Hereda el entorno de 4 MB | `partitions_4mb.csv` |

### 4 MB (`partitions_4mb.csv`)

| Partición | Tipo / Subtipo | Offset | Tamaño | Bytes |
|-----------|----------------|--------|--------|-------|
| `nvs` | data / nvs | `0x9000` | `0x5000` | 20 480 |
| `otadata` | data / ota | `0xE000` | `0x2000` | 8 192 |
| `app0` | app / ota_0 | `0x10000` | `0x1C0000` | 1 835 008 (1,75 MiB) |
| `app1` | app / ota_1 | `0x1D0000` | `0x1C0000` | 1 835 008 (1,75 MiB) |
| `spiffs` | data / spiffs (LittleFS) | `0x390000` | `0x70000` | 458 752 (448 KiB) |

### 8 MB (`partitions_8mb.csv`)

| Partición | Tipo / Subtipo | Offset | Tamaño | Bytes |
|-----------|----------------|--------|--------|-------|
| `nvs` | data / nvs | `0x9000` | `0x5000` | 20 480 |
| `otadata` | data / ota | `0xE000` | `0x2000` | 8 192 |
| `app0` | app / ota_0 | `0x10000` | `0x300000` | 3 145 728 (3 MiB) |
| `app1` | app / ota_1 | `0x310000` | `0x300000` | 3 145 728 (3 MiB) |
| `spiffs` | data / spiffs (LittleFS) | `0x610000` | `0x1F0000` | 2 031 616 (1,94 MiB) |

### 16 MB (`partitions_16mb.csv`)

| Partición | Tipo / Subtipo | Offset | Tamaño | Bytes |
|-----------|----------------|--------|--------|-------|
| `nvs` | data / nvs | `0x9000` | `0x5000` | 20 480 |
| `otadata` | data / ota | `0xE000` | `0x2000` | 8 192 |
| `app0` | app / ota_0 | `0x10000` | `0x300000` | 3 145 728 (3 MiB) |
| `app1` | app / ota_1 | `0x310000` | `0x300000` | 3 145 728 (3 MiB) |
| `spiffs` | data / spiffs (LittleFS) | `0x610000` | `0x9F0000` | 10 420 224 (9,94 MiB) |

```text
0x009000 ┌──────── nvs (config, layout, boots) ────────┐ 20 KB
0x00E000 ├──────── otadata (índice A/B) ──────────────┤ 8 KB
0x010000 ├──────── app0  (OTA_0, activa) ─────────────┤ 1,75 MB (4 MB) / 3 MB (8-16 MB)
         ├──────── app1  (OTA_1, la nueva) ───────────┤ igual tamaño
0x390000 ├──────── spiffs → LittleFS (eventos) ───────┤ 448 KB (solo en 4 MB)
```

- `otadata` es lo que decide qué partición arranca; `Update.end(true)` deja la
  nueva imagen como próxima a bootear.
- El **histórico en microSD** no es una partición: si no hay SD, no hay histórico.
- La OTA escribe **solo** la partición de app: la NVS (config) y la partición
  LittleFS (eventos) no se tocan.

## 2. Flujo de subida (`POST /api/v1/ota`)

```text
POST /api/v1/ota            multipart/form-data, campo "firmware"
   │
   ├─ UPLOAD_FILE_START → authorized()            ← solo X-API-Key
   │       ├─ no autorizado → no escribe nada
   │       └─ Update.begin(UPDATE_SIZE_UNKNOWN)
   ├─ UPLOAD_FILE_WRITE → Update.write(buf, size)
   ├─ UPLOAD_FILE_END   → Update.end(true)
   │       └─ si vino X-SHA256 (64 hex) → compara SHA-256 de la partición
   │
   └─ onOta()
        ├─ no autorizado       → 401 {"error":"unauthorized"}
        ├─ hash distinto       → 400 {"error":"sha256 mismatch"} (no reinicia)
        └─ ok                  → 200 {"ok":true} + delay(100) + ESP.restart()
```

| Elemento | Valor real |
|----------|------------|
| Endpoint | `POST /api/v1/ota` (`HttpServer.cpp:79`) |
| Campo del multipart | `firmware` (así lo manda la web embebida, `HttpServer.cpp:934`) |
| Autorización | `authorized()`: **solo** `X-API-Key`; la cookie de sesión no se evalúa |
| Verificación | Cabecera opcional `X-SHA256` con 64 caracteres hex |
| Alcance del hash | SHA-256 de la **partición OTA completa** (`part->size`) del próximo slot, leída en bloques de 1024 bytes |
| Respuestas | `200 {"ok":true}` · `400 {"error":"sha256 mismatch"}` · `401 {"error":"unauthorized"}` |

!!! warning "El SHA-256 de la partición no es el SHA-256 del `.bin`"
    `otaPartitionSha256()` recorre `off = 0 … part->size` (`HttpServer.cpp:1902-1925`),
    es decir **hasha la partición entera**, no el tamaño del archivo. El valor
    `sha256` de `firmware_manifest.json` es el hash del archivo `.bin` (se calcula
    con `Get-FileHash firmware.bin`, ver `docs/CONTINUACION.md` §3). Como el `.bin`
    es más chico que la partición, **copiar el hash del manifiesto en `X-SHA256`
    produce `400 sha256 mismatch`**. Para que coincida hay que hashear la imagen
    rellenada con `0xFF` hasta el tamaño de la partición (§5).

!!! warning "Un hash incorrecto no revierte el flasheo"
    `Update.end(true)` ya dejó la imagen nueva como partición de arranque antes de
    que se evalúe `X-SHA256`. Si el hash no coincide, SEMA responde `400` y **no
    reinicia**, pero el próximo reinicio (manual, por watchdog o por corte de
    energía) arranca igual con el firmware nuevo.

## 3. La subida desde la web falla si hay claves configuradas

⚠️ **Discrepancia entre la UI y el backend.** El `doOta()` de la página Sistema
(`HttpServer.cpp:934`) envía el binario con `XMLHttpRequest` y `FormData`, **sin
header `X-API-Key`**, pero `onOtaUpload()` solo acepta `authorized()`
(API key). Con `security.api_key` o `security.server_key` configuradas —o con
cualquier `extra_keys`— la carga desde la web queda en `401` y no se escribe
nada. Peor: el `onload` de la UI muestra igual «Flasheado. Reiniciando…» y
redirige a `/` a los 12 s.

Consecuencias prácticas:

- Con la configuración de fábrica (todas las claves vacías), la subida desde la
  web funciona.
- Con claves configuradas, usá `curl`/script con `X-API-Key` (§4) o dejá el OTA
  para el puerto serie (`pio run -t upload`).
- La cookie `sema_auth` **no** sirve para OTA: no es un bug de sesión, es que el
  handler no llama a `webAuthed()`.

## 4. Procedimiento de actualización

### Paso a paso (API, recomendado)

```bash
# 1) Ver qué versión hay publicada
curl -s http://sema-001.local/api/v1/update/check -H "X-API-Key: <api_key>"

# 2) Descargar el binario del release y verificar su SHA-256 contra el manifiesto
curl -LO https://github.com/AlessandroKlein/SEMA/releases/download/v1.103.0/sema_1.103.0_esp32-wroom-4mb.bin
sha256sum sema_1.103.0_esp32-wroom-4mb.bin     # comparar con firmware_manifest.json

# 3) Subir el binario (la cabecera X-SHA256 es opcional; ver §2 para el valor correcto)
curl -X POST http://sema-001.local/api/v1/ota \
  -H "X-API-Key: <api_key>" \
  -F "firmware=@sema_1.103.0_esp32-wroom-4mb.bin"
```

### Verificación posterior

| Consulta | Qué mirar |
|----------|-----------|
| `GET /api/v1/status` | `firmware` debe ser la versión nueva |
| `GET /api/v1/system` | `firmware_file`, `board`, `config_schema`, `reset_reason` (`SOFTWARE` si reinició la OTA) |
| `GET /api/v1/health` | `status`, `free_heap`, sensores online |
| `GET /api/v1/diagnostics` | Tareas, módulos, dispositivos I²C detectados |
| `GET /api/v1/config` | La configuración sigue igual (la OTA no toca la NVS) |
| `GET /api/v1/history?limit=10` | El histórico sigue disponible (si hay microSD) |

No hay endpoint de historial de versiones ni de último resultado de OTA: la única
evidencia es `firmware` en `/api/v1/status` y `reset_reason` en `/api/v1/system`.

## 5. Manifiesto y verificación de integridad

`firmware_manifest.json` (raíz del repo) describe la versión publicada y el
SHA-256 de cada binario y de su filesystem:

```json
{
  "name": "sema",
  "version": "1.103.0",
  "hw_version": "rev0",
  "config_schema_version": 1,
  "protocol_version": 1,
  "firmwares": [
    { "board": "esp32-wroom-4mb", "chip": "esp32",  "flash_mb": 4,
      "firmware": "sema_1.103.0_esp32-wroom-4mb.bin",
      "sha256": "a48e078b01a3ad68fe3810bf124a3a6a43d7dbbfd060e5e935cfc1ed6dc4ae0b",
      "filesystem": "sema_1.103.0_esp32-wroom-4mb_littlefs.bin",
      "filesystem_sha256": "e2a7768e68b459dfb1167ef7015e65e7ad84530c02450baa2091673083137346" },
    { "board": "esp32-s3-8mb", "chip": "esp32s3", "flash_mb": 8,
      "firmware": "sema_1.103.0_esp32-s3-8mb.bin",
      "sha256": "16282da1406d4c8d5d1ef291e761d4dc076c602e58af05285ec1cc4f4da50f56",
      "filesystem": "sema_1.103.0_esp32-s3-8mb_littlefs.bin",
      "filesystem_sha256": "a7e9997adf1c8129d8e22cd4b71cbc4f76a9eea2a1a66f210fca988d4ef698ad" }
  ]
}
```

- El manifiesto **no incluye** entrada para la placa de 16 MB
  (`esp32-wroom32u-16mb`), aunque `GET /api/v1/system` construye el nombre de
  archivo `sema_<version>_<board_id>.bin` para esa placa.
- El filesystem (`*_littlefs.bin`) **no se puede actualizar por `POST
  /api/v1/ota`**: ese endpoint escribe solo la partición de app. Cargarlo
  requiere serie (`pio run -t uploadfs`) y **borra los eventos** guardados en
  LittleFS.

### Calcular el hash que espera `X-SHA256`

```powershell
# Tamaño de la partición OTA según la placa: 0x1C0000 (4 MB) o 0x300000 (8/16 MB)
$partSize = 0x1C0000
$bin = [IO.File]::ReadAllBytes(".\sema_1.103.0_esp32-wroom-4mb.bin")
$img = New-Object byte[] $partSize
[Array]::Copy($bin, $img, $bin.Length)
for ($i = $bin.Length; $i -lt $partSize; $i++) { $img[$i] = 0xFF }   # el resto de la partición queda borrado
(Get-FileHash -InputStream ([IO.MemoryStream]::new($img)) -Algorithm SHA256).Hash.ToLower()
```

Ese valor (64 hex, minúsculas) es el que se manda en `X-SHA256`.

## 6. Comprobar si hay versión nueva (`GET /api/v1/update/check`)

| Aspecto | Valor real |
|---------|------------|
| Autenticación | `webAuthed()` (sesión o `X-API-Key`) |
| URL consultada | `https://raw.githubusercontent.com/AlessandroKlein/SEMA/refs/heads/main/firmware_manifest.json` |
| Cliente | `WiFiClientSecure` con **`setInsecure()`** (sin validación de certificado) |
| Timeout | 8 000 ms |
| Respuesta | `current`, `latest`, `update` (bool), `url` (release en GitHub) |
| Sin conexión / sin HTTP 200 | `latest: ""` y `update: false` (no hay campo de error) |

La comparación es de **desigualdad de cadena**: `update = (latest != current)`, no
un compare SemVer. Una versión publicada más vieja también marcaría `update:
true`. El firmware **no se descarga solo**: la consulta es informativa y la
descarga/subida es manual (web o `curl`).

## 7. Rollback

| Nivel | Estado real |
|-------|-------------|
| Particiones A/B | ✅ Existen (`ota_0`/`ota_1` + `otadata`); la imagen nueva se escribe en el slot libre |
| Rollback automático del bootloader | ❌ No implementado en el proyecto: no hay `esp_ota_mark_app_valid_cancel_rollback()` ni `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` en la configuración del repo, y no se versiona `sdkconfig` (la configuración del bootloader la aporta el framework prebuilt de Arduino-ESP32). No verificable desde el repositorio |
| Rollback manual | ✅ Volver a subir el `.bin` de la versión anterior por `POST /api/v1/ota`, o flashear por serie (`pio run -t upload`) |
| Recuperación | Si el firmware nuevo no arranca, el equipo queda en el bootloader; se recupera por USB-serie. La partición `otadata` guarda qué slot (`ota_0`/`ota_1`) debe arrancar, pero SEMA no expone ningún comando de rollback por API |

> El firmware anterior **no** se conserva accesible desde la API: aunque la
> partición `app0`/`app1` vieja siga con su contenido, no hay endpoint para
> pedir «volver a la anterior». Hay que tener el `.bin` a mano.

## 8. Actualización desde el Servidor Central

El Servidor Central puede disparar `POST /api/v1/ota` con la `server_key` como
`X-API-Key` (tiene la misma semántica que `api_key`). El flujo de particiones,
verificación y reinicio es idéntico al de §2. El Servidor Central es un proyecto
aparte, fuera del alcance de SEMA (`docs/CONTINUACION.md` §2).

---

## Ver también

- [Seguridad](Seguridad.md) · [Configuración](Configuracion.md)
- [Backup y restauración](Backup-y-restauracion.md) · [Solucion de problemas](Solucion-de-problemas.md)
- [Compatibilidad de versiones](Compatibilidad-de-versiones.md) · [Registro de cambios](Registro-de-cambios.md)
