---
tags:
  - sema
  - configuracion
---

# Configuración

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.45.0

Cómo se configura SEMA.

## Almacenamiento

- La config se persiste en **NVS** (clave `config`), vía `ConfigManager` + `Storage`.
- Formato JSON, esquema `schema_version: 1`.
- El histórico y los eventos se guardan en **LittleFS** (JSONL).

## Cómo editarla

| Vía | Endpoint / UI |
|-----|---------------|
| API | `PUT /api/v1/config` |
| Dashboard | formulario "Configuración" (nombre, WiFi, hostname, claves) |
| Backup/restore | `GET`/`POST /api/v1/backup` |
| Servidor Central | `PUT /api/v1/config` con `server_key` |

## Transaccionalidad

1. `parseInto()` parsea el JSON.
2. `validate()` comprueba `schema_version`, `station.id` no vacío, `network.mode` y
   `storage.backend` válidos.
3. Si es válido, `save()` persiste; si falla, **rollback** a la config previa.

## Aplicación en caliente

Al aplicar (`PUT /config`), se re-aplican **sin reinicio**:

- Sensores (`applySensors`)
- Reglas (`applyRules`)
- Calibración (`applyCalibrations`)
- GPIO (`applyGpio`)
- Publicadores (`applyPublishers`)

## Respaldos

`GET /api/v1/backup` devuelve un JSON autodescriptivo:

```json
{ "backup_format": "sema-backup", "backup_version": 1,
  "firmware": "1.45.0", "timestamp": 1720000000,
  "...config completa..." }
```

Los campos `backup_*`, `firmware` y `timestamp` se ignoran al restaurar.

## Referencia completa

Ver [Referencia de configuración (JSON)](Referencia-configuracion.md) y
[Variables modificables](Variables-modificables.md).
