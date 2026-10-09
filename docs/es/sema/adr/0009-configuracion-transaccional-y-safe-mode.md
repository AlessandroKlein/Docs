---
tags:
  - sema
  - adr
  - configuracion
---

# 0009. Configuración transaccional con rollback y Safe Mode

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0023, D-0024, D-0042

## Contexto

La configuración se edita desde el navegador, muchas veces a distancia. Un JSON
inválido, un pin equivocado o un NVS lleno pueden dejar la estación inaccesible. Hace
falta que aplicar configuración sea una operación **atómica**: o queda aplicada y
persistida, o no cambia nada.

## Decisión

**Configuración transaccional con rollback** (D-0023) sobre un esquema jerárquico
versionado `schema_version = 1` (D-0042):

```text
validar → aplicar en memoria → persistir en NVS → verificar
                                   │
                                   └── si falla: restaurar el estado anterior
```

- `ConfigManager::apply()` guarda un `backup_` antes de escribir y lo restaura si
  `save()` falla (`src/core/ConfigManager.cpp` §89-101).
- `validate()` rechaza `schema_version != 1`, `station.id` vacío, `mode` distinto de
  `STA`/`AP` y `storage.backend` fuera de `littlefs`/`flash`/`sd`.
- `applyJson()` parsea a una `Config` nueva y recién después aplica: un JSON malformado
  no toca la configuración viva.
- **Safe Mode** (D-0024): si la configuración es inválida o el arranque falla, la
  estación arranca en modo mínimo de recuperación (AP + web básica).
- Cada sección tiene defaults; los campos ausentes se completan con
  `doc["sección"]["clave"] | default`.

## Consecuencias

- ✅ Un `PUT /api/v1/config` inválido responde error y deja la estación funcionando.
- ✅ La config es autodescriptiva y versionable: `schema_version` habilita migraciones
  futuras sin romper instalaciones.
- ✅ Hot reload: tras aplicar, el Core re-aplica sensores, reglas, calibración, GPIO,
  publicadores y buses **sin reiniciar** (`HttpServer::onConfigPut()`, §1871-1894).
- ⚠️ **No existe `migrate()`**: hay un único `schema_version` (1) y no hay código de
  migración entre versiones de esquema.
- ⚠️ **Safe Mode no está implementado** como modo de arranque: si `load()` falla se
  usan defaults y se sigue; el AP de recuperación solo aparece cuando `network.mode`
  es distinto de `STA` o no hay SSID. La detección de arranque fallido (contador de
  reinicios) se registra (`boots` en NVS, `restartCount`) pero no cambia el modo.

## Ver también

- [Decisiones](../Decisiones.md) · [Configuración](../Configuracion.md) ·
  [Referencia de configuración](../Referencia-configuracion.md) · [Identidad y estados](../Identidad-y-estados.md)
