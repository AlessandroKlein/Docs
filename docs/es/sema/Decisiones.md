---
tags:
  - sema
  - decisiones
---

# Decisiones

> **Tipo:** Referencia | **Estado:** Planificación | **Fecha:** 2026-10-03 | **Firmware:** v0.29.0

Resumen de las decisiones de arquitectura (ADR). El registro completo con motivos
y consecuencias vive en el repo del código:
<https://github.com/AlessandroKlein/SEMA/blob/main/docs/DUDAS-Y-DECISIONES.md>.

## 1. Principio clave

Compatibilidad por **Board/Chip Profiles**, no por `#ifdef`: Capability Manager,
Resource Manager, Runtime Manager y HAL. Ver [Arquitectura](Arquitectura.md).

## 2. Decisiones por área

- **Plataforma** (D-0015…D-0017, D-0038, D-0050, D-0051): perfiles de placa/chip,
  Capability Manager, Resource Manager, Capability Matrix.
- **Core y runtime** (D-0012…D-0014, D-0018…D-0024, D-0039, D-0052, D-0053):
  FreeRTOS, SMP, `AUTO`, Event-driven + scheduler, watchdog, health, configuración
  transaccional, Safe Mode, perfiles de runtime.
- **Datos y sensores** (D-0007, D-0025, D-0026, D-0028, D-0032, D-0033, D-0044,
  D-0045, D-0055…D-0058): modelo canónico, Event Bus tipado, calibración, quality
  flags, retención, discovery.
- **Comunicación** (D-0006, D-0009…D-0011, D-0030, D-0036, D-0037, D-0041, D-0047,
  D-0048): REST `/api/v1` + WebSocket, Store & Forward, publishers, Servidor
  Central, RBAC + API Keys.
- **Operación** (D-0027, D-0029, D-0031, D-0034, D-0035, D-0040, D-0049, D-0054,
  D-0059, D-0060): OTA + rollback, perfiles energéticos, módulos, seguridad, motor
  de reglas.

## 3. Total

60 decisiones cerradas (D-0001…D-0060).

Ver también: [Arquitectura](Arquitectura.md) · [Evolución](Evolucion.md).
