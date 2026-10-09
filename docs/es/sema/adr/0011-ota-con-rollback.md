---
tags:
  - sema
  - adr
  - ota
---

# 0011. OTA con particiones redundantes y rollback

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0031, D-0049

## Contexto

Una estación instalada en el campo no se puede descolgar para reflashear por USB. Una
actualización que falle a mitad de camino, o un firmware que arranque mal, dejaría el
equipo inaccesible sin visita técnica.

## Decisión

**OTA con rollback** (D-0031, D-0049) sobre las particiones redundantes de ESP-IDF:

```text
partitions_4mb.csv
 nvs      0x9000   0x5000
 otadata  0xe000   0x2000
 app0    0x10000  0x1C0000   ← aplicación A
 app1    0x1D0000 0x1C0000   ← aplicación B
 spiffs  0x390000 0x70000
```

- La actualización se escribe en la partición **libre** (`esp_ota_get_next_update_partition`)
  y `otadata` decide cuál arranca.
- La actualización se **verifica antes de marcar el firmware como válido**: si el
  cliente envía `X-SHA256`, el handler recalcula el SHA-256 de la partición escrita y
  responde `400 {"error":"sha256 mismatch"}` sin reiniciar cuando no coincide.
- El firmware se arranca recién si la verificación pasa; si no, se conserva la versión
  anterior.

Particiones por tamaño de flash: `partitions_4mb.csv` (WROOM),
`partitions_8mb.csv` (S3), `partitions_16mb.csv` (WROOM-32U).

## Consecuencias

- ✅ Un OTA fallido no deja la estación sin firmware: siempre queda la copia previa.
- ✅ El binario y su hash se publican por release: `firmware_manifest.json` lista
  `board`, `chip`, `flash_mb`, `firmware`, `sha256`, `filesystem` y
  `filesystem_sha256`.
- ✅ `GET /api/v1/update/check` compara `SEMA_FW_VERSION` contra el `version` del
  manifiesto en `raw.githubusercontent.com/AlessandroKlein/SEMA/refs/heads/main` y
  devuelve `current`, `latest`, `update` y `url` del release.
- ⚠️ El check de actualización usa `WiFiClientSecure::setInsecure()` (sin validar
  certificado); el comentario del código lo justifica porque el OTA real se valida por
  SHA-256, pero el **manifiesto** puede ser falsificado y mostrar una versión falsa.
- ⚠️ **OTA firmado pendiente**: no hay firma criptográfica del binario ni verificación
  de origen; el SHA-256 es opcional y lo aporta el cliente.
- ⚠️ El rollback automático depende del mecanismo de ESP-IDF y del `otadata`; no hay
  contador de arranques fallidos propio ni "modo recuperación" explícito.

## Ver también

- [Decisiones](../Decisiones.md) · [OTA y actualización](../OTA-y-Actualizacion.md) ·
  [Backup y restauración](../Backup-y-restauracion.md) · [Compilación y flasheo](../Compilacion-y-flasheo.md)
