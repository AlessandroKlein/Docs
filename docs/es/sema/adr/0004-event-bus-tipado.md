---
tags:
  - sema
  - adr
  - arquitectura
---

# 0004. Event Bus tipado para desacoplar módulos

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0008, D-0045

## Contexto

Reaccionar a lo que pasa en la estación (arranque de lluvia, descarga de un rayo,
alarma de umbral, caída de red) no puede resolverse con llamadas directas entre
módulos: cada nuevo interesado obligaría a modificar el módulo que detecta el hecho,
y un servicio externo lento podría bloquear la adquisición.

## Decisión

Un **Event Bus tipado** interno (D-0008, D-0045): los productores publican y los
consumidores se suscriben por tipo, sin conocerse.

```cpp
// include/core/EventBus.hpp
enum class EventType : uint8_t { Sensor, Rain, Lightning, Battery, Network,
                                 Alarm, System, Wake, Sleep };
enum class Severity : uint8_t { Debug, Info, Notice, Warning, Error, Critical };
struct Event { uint32_t id, timestampMs; String source;
               EventType type; Severity severity; int32_t value;
               String correlationId, target; };
```

- El bus (`EventBus::publish` / `subscribe`) es sincrónico y cooperativo: el handler
  corre en el contexto de quien publica.
- `EventLog` se suscribe a **todos** los tipos en su constructor y persiste los eventos
  en LittleFS (`/events.jsonl`).
- El `RuleEngine` publica eventos `Alarm` que el Core registra en el bus.

## Consecuencias

- ✅ Agregar un consumidor (MQTT, webhook, dashboard, wake manager) no toca al
  productor.
- ✅ `GET /api/v1/events` y `GET /api/v1/alarms` leen el `EventLog`
  (buffer de 100 eventos en RAM, rotación en LittleFS al llegar a 200 líneas).
- ⚠️ El payload es un único `int32_t`, insuficiente para datos ricos (una medición
  completa no cabe en un evento); la estructura final del README preveía un payload
  genérico.
- ⚠️ Todos los handlers corren en el mismo hilo que publica (no hay cola ni
  aislamiento): un handler lento frena el ciclo que lo publicó.
- ⚠️ `EventType` no tiene centinela `Count`: `EventLog` itera hasta `Sleep`
  (`src/core/events/EventLog.cpp` §11, con `TODO` en el propio código).

## Ver también

- [Decisiones](../Decisiones.md) · [Enumeraciones y tipos](../Enumeraciones-y-tipos.md) ·
  [Alarmas y reglas](../Alarmas-y-reglas.md) · [Tareas y concurrencia](../Tareas-y-concurrencia.md)
