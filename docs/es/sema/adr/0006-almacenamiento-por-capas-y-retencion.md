---
tags:
  - sema
  - adr
  - almacenamiento
---

# 0006. Almacenamiento por capas y retención por niveles

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.114.1 | **Decisión origen:** D-0032, D-0046, D-0057

## Contexto

La estación guarda cosas de naturaleza muy distinta: configuración crítica (no puede
corromperse), histórico de mediciones (crece sin límite), eventos/alarmas (pocos y
valiosos) y logs. Un único medio o formato para todo complica la retención y expone la
configuración a un borrado masivo. Además, la microSD no puede ser un requisito: la
estación tiene que funcionar sin ella.

## Decisión

**Almacenamiento por capas** (D-0032, D-0046) detrás de una abstracción del Core, con
**retención por niveles** (D-0057):

```text
Storage API (include/core/storage/Storage.hpp)
 ├── NVS (Preferences)        → contadores de arranque
 ├── LittleFS (flash)         → configuración (/config.json) + eventos/alarmas (/events.jsonl)
 └── microSD (SPI, opcional)  → histórico de mediciones (/history.jsonl + .agg)
```

- `KeyValueStore` es la frontera abstracta (`begin`, `getString/putString`,
  `getUInt/putUInt`, `clear`); su única implementación hoy es `NvsStore` (usada
  para `boots` y como fallback de lectura de config). Desde v1.114.0 la config se
  escribe en LittleFS `/config.json` (D-0046).
- Retención: se conserva alta resolución reciente y lo viejo se agrega a promedios
  horarios en el archivo `.agg` en lugar de descartarse
  (`HistoryStore::aggregate()`, invocado cada hora desde `SemaCore::loop()`).

## Consecuencias

- ✅ `storage.retention_days` (default `30`) fija la ventana de retención.
- ✅ La configuración sobrevive a un borrado del histórico: viven en medios distintos.
- ⚠️ **El histórico se guarda solo en microSD**: `HistoryStore::append()` devuelve
  `false` si `enableSd()` no tuvo éxito. Con `storage.sd_enabled = false` (default) el
  endpoint `/api/v1/history` responde vacío. Las mediciones **no** se persisten en
  flash interna.
- ⚠️ `storage.backend` (`"littlefs" | "flash" | "sd"`) se parsea y valida, pero **no
  selecciona** el backend en runtime: el histórico va a SD y los eventos a LittleFS,
  sin importar su valor.
- ⚠️ La config ya no se guarda en NVS (D-0046): `ConfigManager::save()` escribe
  `/config.json` en LittleFS. La NVS solo conserva `boots`.

## Ver también

- [Decisiones](../Decisiones.md) · [Almacenamiento e histórico](../Almacenamiento-e-historico.md) ·
  [Backup y restauración](../Backup-y-restauracion.md) · [Referencia de configuración](../Referencia-configuracion.md)
