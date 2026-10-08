---
tags:
  - sema
  - metricas
---

# Estadísticas y métricas

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.85.0

Métricas expuestas por SEMA para monitoreo y diagnóstico.

## `GET /api/v1/diagnostics`

| Métrica | Descripción |
|---------|-------------|
| `firmware` | Versión del firmware |
| `hw` | Revisión del hardware |
| `uptime_s` | Tiempo encendido (segundos) |
| `free_heap` | Heap libre (bytes) |
| `reset_reason` | Código de la última causa de reinicio (`esp_reset_reason()`) |
| `health` | Estado de salud agregado |
| `history.entries` | Entradas actuales en el histórico |
| `history.max` | Límite de entradas del histórico |
| `tasks` | Tareas programadas (`Scheduler`) |
| `modules` | Módulos registrados |
| `events` | Eventos en memoria |
| `i2c_devices[]` | Dispositivos detectados en I²C (dirección + modelo) |

## `GET /api/v1/health`

| Métrica | Descripción |
|---------|-------------|
| `status` | `HEALTHY` · `DEGRADED` · `ERROR` |
| `uptime_s` | Uptime |
| `free_heap` | Heap libre |
| `sensors.total` | Sensores registrados |
| `sensors.online` | Sensores online |
| `sensors.error` | Sensores con error (total − online) |

## Estado de salud (`HealthMonitor`)

| Estado | Criterio |
|--------|----------|
| `HEALTHY` | Heartbeat al día y todos los sensores online |
| `DEGRADED` | Algún sensor offline o heartbeat retrasado |
| `ERROR` | Heartbeat perdido (bloqueo) o fallo grave |

## Causas de reinicio (`reset_reason`, esp-idf)

| Código | Significado |
|--------|-------------|
| 1 | Power-on |
| 2 | Reset externo (pin EN) |
| 3 | Software reset (`ESP.restart()`) |
| 4 | Exception/panic |
| 5 | Interrupt watchdog |
| 6 | Task watchdog |
| 7 | Other watchdog |
| 8 | Deep-sleep wake |

## Perfiles energéticos (`PowerManager`)

| Perfil | Nombre API |
|--------|------------|
| Performance | `performance` |
| Normal (default) | `normal` |
| LowPower | `low_power` |
| UltraLowPower | `ultra_low_power` |

## Uso típico

```bash
curl http://sema-001.local/api/v1/diagnostics
curl http://sema-001.local/api/v1/health
```
