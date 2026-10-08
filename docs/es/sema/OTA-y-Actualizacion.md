---
tags:
  - sema
  - ota
---

# OTA y actualización

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.87.0

Actualización de firmware por aire (OTA) con particiones A/B y rollback.

## Particiones (`partitions.csv`)

```text
# app0 (OTA)      → partición activa A
# app1 (OTA)      → partición activa B (para el nuevo firmware)
# spiffs          → LittleFS (histórico, eventos)
```

- `board_build.partitions = partitions.csv` en `platformio.ini`.
- La partición `app0` es de 0x1C0000.

## Subir firmware

```http
POST /api/v1/ota
X-API-Key: <api_key>
Content-Type: multipart/form-data

(firmware.bin como "firmware")
```

- `Update.begin(UPDATE_SIZE_UNKNOWN)` + `Update.write(...)` + `Update.end(true)`.
- Tras aplicar, reinicia (`ESP.restart()`).

## Ejemplo con curl

```bash
curl -X POST http://sema-001.local/api/v1/ota \
  -H "X-API-Key: <api_key>" \
  -F "firmware=@.pio/build/esp32doit-devkit-v1/firmware.bin"
```

## Rollback

- El bootloader arranca la partición marcada como válida.
- Si el nuevo firmware no arranca, se puede volver a la partición anterior
  (mecanismo de rollback del bootloader de ESP32).

## Manifiesto (`firmware_manifest.json`)

Cada release incluye el SHA-256 del binario:

```json
{ "name": "sema", "version": "1.87.0", "firmware": "firmware.bin",
  "sha256": "bb8f82e3dc2be049a4a6c3bc78b056ae88c0a802dea81d7f088bc25abd217e38",
  "hw_version": "rev0", "config_schema_version": 1, "protocol_version": 1 }
```

## Verificación recomendada

```bash
sha256sum firmware.bin   # comparar con el manifiesto
```

## Actualización vía Servidor Central

El Servidor Central puede disparar `POST /api/v1/ota` usando `server_key`; el flujo
de particiones y rollback es el mismo.
