---
tags:
  - sema
  - adr
  - runtime
---

# 0008. Runtime sobre FreeRTOS con SMP adaptativo y afinidad `AUTO`

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0012, D-0013, D-0014, D-0039, D-0052, D-0053

## Contexto

El ESP32 tiene dos núcleos y un FreeRTOS con SMP; otras variantes (S2, C3, C6) tienen
uno solo. Fijar tareas a un núcleo concreto o depender de dos núcleos rompe la
portabilidad. Además, exponer prioridades de FreeRTOS al usuario final convierte la
configuración en un campo minado.

## Decisión

FreeRTOS es el runtime (D-0012), pero **SEMA no se acopla a su API**: hay una capa de
abstracción RT propia (D-0039, `include/core/runtime/Task.hpp`). Reglas:

- **SMP adaptativo** (D-0013, D-0038): se aprovechan los dos núcleos cuando existen,
  sin depender de ellos.
- **Task Affinity `AUTO`** por defecto (D-0014, D-0053): `xTaskCreate` con
  `tskNO_AFFINITY`; el pinning solo se justifica por hardware, ISR, latencia o
  benchmark, y se documenta.
- **Perfiles de runtime**, no prioridades libres (D-0052): críticas → alta;
  adquisición → media/alta; comunicaciones → media; UI/logging → baja.
- La configuración avanzada de tasks queda en **Expert Mode** (D-0040); el usuario
  normal trabaja con magnitudes.

## Consecuencias

- ✅ El mismo código compila y corre en ESP32 single-core y multicore.
- ✅ El orden de prioridades del README (P1 core/seguridad/watchdog … P7 servicios
  externos) describe la **intención de diseño**.
- ⚠️ **Estado real v1.103.0**: el firmware es **mono-hilo**. No hay ninguna
  `xTaskCreate` en uso fuera de `src/core/runtime/Task.cpp`, que hoy no se instancia
  desde ningún módulo. Todo corre dentro de `loop()` de Arduino con un `Scheduler`
  cooperativo por `millis()` (`include/core/Scheduler.hpp`): `core.heartbeat` cada
  5 s y `sensors.read` cada 10 s (`src/core/SemaCore.cpp` §180-198).
- ⚠️ Por lo mismo, tampoco hay semáforos ni colas en uso: la "concurrencia" es
  secuencial y el `Watchdog` (10 s, `esp_task_wdt`) solo vigila `loopTask`.
- ⚠️ El `ModuleRegistry::loopAll()` y el `HttpServer::loop()` corren en el mismo hilo:
  una respuesta HTTP lenta retrasa el ciclo de sensores.

## Ver también

- [Decisiones](../Decisiones.md) · [Tareas y concurrencia](../Tareas-y-concurrencia.md) ·
  [Rendimiento y memoria](../Rendimiento-y-memoria.md) · [Módulos y ciclo de vida](../Modulos-y-ciclo-de-vida.md)
