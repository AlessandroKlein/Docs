---
tags:
  - sema
  - adr
  - diagnostico
---

# 0012. Watchdog jerárquico, Health Monitor y diagnóstico

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0019, D-0020, D-0034

## Contexto

Un equipo de campo sin pantalla ni consola solo puede diagnosticarse por red. Hace
falta (a) que un bloqueo no deje la estación colgada para siempre y (b) que un operador
pueda ver **qué** subsistema está fallando sin estar presente.

## Decisión

1. **D-0019 — Watchdog jerárquico**: un watchdog de sistema (TWDT) más un watchdog por
   tarea; cada tarea registra su timeout y "late" en cada ejecución.
2. **D-0020 — Health Monitor**: estado de salud recuperable por API y diagnóstico.
3. **D-0034 — Diagnóstico y observabilidad**: logs estructurados
   (`timestamp, level, module, event, message, code`), métricas y health.

Implementación verificable:

- `Watchdog::begin(10)` llama a `esp_task_wdt_init(10, true)` y agrega `loopTask`;
  `SemaCore::loop()` hace `watchdog_.feed()` en cada vuelta (`src/core/SemaCore.cpp`).
- `HealthMonitor` registra dos tareas vigiladas: `core.heartbeat` (timeout 30 s) y
  `sensors.read` (60 s); `status()` devuelve `HEALTHY`, `DEGRADED` o `ERROR` según
  sensores online/total, heartbeat y tareas estancadas.
- **D-0034** se expone en `GET /api/v1/health` (estado, uptime, `free_heap`, sensores
  total/online/error y salud por tarea) y `GET /api/v1/diagnostics` (firmware, reset
  reason, entradas y máximo del histórico, tasks, módulos, eventos y dispositivos I²C
  detectados).

## Consecuencias

- ✅ Si `loop()` se bloquea más de 10 s, el TWDT reinicia el SoC.
- ✅ `restartCount` (contador `boots` en NVS) y `reset_reason` permiten distinguir un
  reinicio por watchdog de uno manual.
- ✅ El estado de salud se degrada antes de fallar: un sensor caído da `DEGRADED`, no
  `ERROR`.
- ⚠️ El "watchdog jerárquico por tarea" es **lógico**: como no hay tareas de FreeRTOS
  (ADR 0008), `HealthMonitor` solo **reporta** la tarea estancada; no la reinicia ni
  alimenta el TWDT por tarea.
- ⚠️ **Logs estructurados pendientes**: hoy la salida es `Serial.printf` con texto
  libre (por ejemplo `[cfg]`, `[ALARM]`, `I²C 0x76 → BME280`); no hay JSON de log ni
  `code` de error.
- ⚠️ `system.log_level` se guarda y se muestra en la web, pero no filtra la salida
  serial.

## Ver también

- [Decisiones](../Decisiones.md) · [Diagnóstico y salud](../Diagnostico-y-salud.md) ·
  [Solución de problemas](../Solucion-de-problemas.md) · [Identidad y estados](../Identidad-y-estados.md)
