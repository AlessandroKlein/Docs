---
tags:
  - sema
  - decisiones
  - adr
---

# Decisiones de arquitectura (ADR)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

SEMA registra sus decisiones en dos niveles: el **registro de trabajo**
`docs/DUDAS-Y-DECISIONES.md` del repo de código (decisiones `D-0001`…`D-0060`, con
motivo y consecuencias) y los **ADR** de esta wiki, que formalizan las decisiones más
estructurales con contexto / decisión / consecuencias.

- Registro completo: <https://github.com/AlessandroKlein/SEMA/blob/main/docs/DUDAS-Y-DECISIONES.md>
- Formato de los ADR: `docs/ESTANDAR-DOCUMENTACION.md` §6.

## 1. Índice de ADR

| ADR | Título | Decisiones origen | Estado |
|-----|--------|-------------------|:------:|
| [0001](adr/0001-plataforma-configurable-y-separacion-de-capas.md) | SEMA es una plataforma configurable, no una estación fija | D-0001 · D-0002 · D-0003 | Aceptada |
| [0002](adr/0002-perfiles-de-placa-capacidades-y-hal.md) | Compatibilidad por perfiles de placa/chip y HAL, no por `#ifdef` | D-0015 · D-0016 · D-0017 · D-0038 · D-0050 · D-0051 | Aceptada |
| [0003](adr/0003-modelo-canonico-de-mediciones.md) | Modelo canónico de mediciones con quality flags | D-0007 · D-0026 · D-0028 · D-0044 · D-0056 | Aceptada |
| [0004](adr/0004-event-bus-tipado.md) | Event Bus tipado para desacoplar módulos | D-0008 · D-0045 | Aceptada |
| [0005](adr/0005-calibracion-independiente-del-driver.md) | Calibración independiente del driver | D-0025 · D-0055 | Aceptada |
| [0006](adr/0006-almacenamiento-por-capas-y-retencion.md) | Almacenamiento por capas y retención por niveles | D-0032 · D-0046 · D-0057 | Aceptada |
| [0007](adr/0007-offline-first-y-publishers-desacoplados.md) | Offline-first con publicadores desacoplados | D-0009 · D-0010 · D-0011 · D-0030 · D-0036 · D-0037 · D-0047 | Aceptada |
| [0008](adr/0008-runtime-freertos-y-afinidad-auto.md) | Runtime sobre FreeRTOS con SMP adaptativo y afinidad `AUTO` | D-0012 · D-0013 · D-0014 · D-0039 · D-0052 · D-0053 | Aceptada |
| [0009](adr/0009-configuracion-transaccional-y-safe-mode.md) | Configuración transaccional con rollback y Safe Mode | D-0023 · D-0024 · D-0042 | Aceptada |
| [0010](adr/0010-api-rest-v1-y-seguridad.md) | API REST `/api/v1` en el Core y seguridad por claves | D-0006 · D-0027 · D-0035 · D-0041 · D-0048 | Aceptada |
| [0011](adr/0011-ota-con-rollback.md) | OTA con particiones redundantes y rollback | D-0031 · D-0049 | Aceptada |
| [0012](adr/0012-watchdog-health-y-diagnostico.md) | Watchdog jerárquico, Health Monitor y diagnóstico | D-0019 · D-0020 · D-0034 | Aceptada |
| [0013](adr/0013-perfiles-energeticos-y-wake-por-lluvia.md) | Perfiles energéticos y wake-up por lluvia | D-0021 · D-0022 · D-0054 | Aceptada |
| [0014](adr/0014-modulos-instalables-y-descubrimiento.md) | Módulos instalables, selección de drivers y descubrimiento | D-0029 · D-0043 · D-0058 · D-0060 | Aceptada |

## 2. Registro completo `D-0001`…`D-0060`

Todas las decisiones están **cerradas** en el registro de trabajo. La columna «ADR»
indica el archivo de esta wiki que la desarrolla; «—» significa que la decisión se
documenta en la página temática indicada en «Área».

| ID | Decisión | Área | ADR |
|----|----------|------|-----|
| D-0001 | Plataforma configurable, no estación fija | Fundamentos | 0001 |
| D-0002 | Separación Hardware / Sensor / Servicio | Fundamentos | 0001 |
| D-0003 | Almacenamiento y protocolos como abstracciones del Core | Fundamentos | 0001 · 0006 |
| D-0004 | Web local autónoma; Servidor Central opcional | Fundamentos | 0007 |
| D-0005 | Versionado SemVer con prefijo `v` | Fundamentos | — (ver [Versionado](../inicio/Versionado.md)) |
| D-0006 | REST API + WebSocket forman parte del Core | Comunicación | 0010 |
| D-0007 | Modelo Canónico de Mediciones | Datos | 0003 |
| D-0008 | Event Bus interno | Fundamentos | 0004 |
| D-0009 | Store & Forward para datos | Comunicación | 0007 |
| D-0010 | Publishers externos desacoplados | Comunicación | 0007 |
| D-0011 | Servidor Central opcional | Comunicación | 0007 |
| D-0012 | FreeRTOS como runtime | Runtime | 0008 |
| D-0013 | SMP adaptativo | Runtime | 0008 |
| D-0014 | Task Affinity `AUTO` por defecto | Runtime | 0008 |
| D-0015 | Compatibilidad basada en Board/Chip Profiles | Plataforma | 0002 |
| D-0016 | Capability Manager | Plataforma | 0002 |
| D-0017 | Resource Manager y detección de conflictos | Plataforma | 0002 |
| D-0018 | Event-driven + scheduler híbrido | Runtime | 0004 · 0008 |
| D-0019 | Watchdog jerárquico | Runtime | 0012 |
| D-0020 | Health Monitor | Runtime | 0012 |
| D-0021 | Deep Sleep + Wake-up Manager | Operación | 0013 |
| D-0022 | GPIO/INT de lluvia como fuente de wake-up | Operación | 0013 |
| D-0023 | Configuración transaccional con rollback | Configuración | 0009 |
| D-0024 | Safe Mode / recuperación | Configuración | 0009 |
| D-0025 | Calibración independiente del driver | Datos | 0005 |
| D-0026 | Quality Flags para mediciones | Datos | 0003 |
| D-0027 | OpenAPI/JSON Schema para API | API | 0010 |
| D-0028 | Identidad única de estación/dispositivo/sensor | Datos | 0003 |
| D-0029 | Sistema de módulos instalables/habilitables | Módulos | 0014 |
| D-0030 | Arquitectura Offline-First | Comunicación | 0007 |
| D-0031 | OTA con rollback | Operación | 0011 |
| D-0032 | Almacenamiento por capas | Datos | 0006 |
| D-0033 | Sistema de eventos y alarmas | Datos | 0004 · [Alarmas y reglas](Alarmas-y-reglas.md) |
| D-0034 | Diagnóstico y observabilidad | Operación | 0012 |
| D-0035 | Seguridad por roles/capacidades | Seguridad | 0010 |
| D-0036 | Servidor Central multiestación | Comunicación | 0007 |
| D-0037 | API de sincronización estación ↔ central | Comunicación | 0007 |
| D-0038 | Compatibilidad ESP32 single-core y multicore | Plataforma | 0002 · 0008 |
| D-0039 | Abstracción RT (no acoplar SEMA a FreeRTOS) | Runtime | 0008 |
| D-0040 | Configuración avanzada de Tasks solo en Expert Mode | Runtime | 0008 |
| D-0041 | REST API `/api/v1` y WebSocket | API | 0010 |
| D-0042 | JSON Schema de configuración v1 | Configuración | 0009 |
| D-0043 | Política de selección de drivers | Sensores | 0014 |
| D-0044 | Canonical Data Model definitivo | Datos | 0003 |
| D-0045 | Event Bus tipado | Fundamentos | 0004 |
| D-0046 | Storage API y backends | Almacenamiento | 0006 |
| D-0047 | Protocolo estación ↔ Servidor Central | Comunicación | 0007 |
| D-0048 | Autenticación RBAC + API Keys | Seguridad | 0010 |
| D-0049 | Política OTA y rollback | Operación | 0011 |
| D-0050 | Board/Chip Profiles oficiales | Plataforma | 0002 |
| D-0051 | Capability Matrix | Plataforma | 0002 |
| D-0052 | Perfiles de FreeRTOS | Runtime | 0008 |
| D-0053 | Task Affinity `AUTO` | Runtime | 0008 |
| D-0054 | Perfiles de consumo energético | Operación | 0013 |
| D-0055 | Framework de calibración | Datos | 0005 |
| D-0056 | Quality Flags | Datos | 0003 |
| D-0057 | Política de retención histórica | Datos | 0006 |
| D-0058 | Descubrimiento de sensores | Sensores | 0014 |
| D-0059 | Motor de reglas y alarmas | Alarmas | [Alarmas y reglas](Alarmas-y-reglas.md) |
| D-0060 | Interfaz de módulos externos | Módulos | 0014 |

## 3. Resumen por área (estado verificado v1.103.0)

| Área | Decisiones | Estado de implementación |
|------|------------|--------------------------|
| Fundamentos | D-0001, D-0002, D-0003, D-0004, D-0005, D-0008, D-0045 | ✅ Plataforma configurable, capas separadas y Event Bus operativos. Web local autónoma. SemVer aplicado (`SEMA_FW_VERSION`). |
| Plataforma / HAL | D-0015, D-0016, D-0017, D-0038, D-0050, D-0051 | 🔄 `Capability` (16 capacidades) + `CapabilityManager` + perfiles de compilación por board. Perfil base cargado a mano y **sin** `ResourceManager` (ver ADR 0002). |
| Runtime | D-0012, D-0013, D-0014, D-0018, D-0019, D-0020, D-0039, D-0052, D-0053 | 🔄 `Scheduler` cooperativo y abstracción `Task` presentes; el firmware es **mono-hilo** (sin tareas FreeRTOS en uso). Watchdog TWDT + Health Monitor activos (ver ADR 0008 y 0012). |
| Configuración | D-0023, D-0024, D-0042 | 🔄 Transaccional con rollback y `schema_version = 1` operativos. `migrate()` y Safe Mode **pendientes** (ver ADR 0009). |
| Datos y sensores | D-0007, D-0025, D-0026, D-0028, D-0044, D-0055, D-0056, D-0057, D-0058 | ✅ Modelo canónico, calibración por canal y 8 quality flags definidos. 🔄 Solo se emiten `VALID`, `COMMUNICATION_ERROR` y `SENSOR_DISCONNECTED`. ⚠️ Retención por niveles solo sobre microSD (ver ADR 0003, 0005 y 0006). |
| Almacenamiento | D-0032, D-0046, D-0057 | 🔄 NVS para configuración, LittleFS para eventos, microSD para histórico. `storage.backend` no selecciona backend (ver ADR 0006). |
| Comunicación | D-0006, D-0009, D-0010, D-0011, D-0030, D-0036, D-0037, D-0047 | 🔄 API/web local, WebSocket, webhook y MQTT implementados. Store & Forward y Servidor Central **pendientes/fuera de alcance** (ver ADR 0007). |
| API y seguridad | D-0027, D-0035, D-0048 | 🔄 `api_key` + `server_key` + `extra_keys` + login web con rate limiting. Sin TLS, sin RBAC por roles y sin OpenAPI generado (ver ADR 0010). |
| OTA | D-0031, D-0049 | ✅ Particiones A/B + verificación SHA-256 opcional + `update/check` contra el manifiesto. ⚠️ Sin firma de binario (ver ADR 0011). |
| Energía | D-0021, D-0022, D-0054 | 🔄 Perfiles y primitivas de deep sleep/wake por lluvia disponibles; **nadie invoca** `sleep()` y el perfil no cambia la operación (ver ADR 0013). |
| Módulos | D-0029, D-0060 | 🔄 Interfaz `Module` + `ModuleRegistry` con ciclo completo; sin módulos reales (ver ADR 0014). |
| Alarmas | D-0033, D-0059 | 🔄 Reglas `gt/lt/ge/le` configurables que publican eventos `Alarm` persistentes. Sin duración, combinación lógica ni acciones (ver [Alarmas y reglas](Alarmas-y-reglas.md)). |
| Sensores / drivers | D-0043, D-0058 | ✅ 17 modelos vía `SensorFactory`, drivers encapsulados, descubrimiento I²C al arrancar. ⚠️ `address` de config ignorado (ver ADR 0014). |

## 4. Principio clave: perfiles, no `#ifdef`

La compatibilidad entre variantes de ESP32 **no se resuelve con `#ifdef` repartidos**
(ver [`DUDAS-Y-DECISIONES.md`](https://github.com/AlessandroKlein/SEMA/blob/main/docs/DUDAS-Y-DECISIONES.md) §2):

```text
SEMA Application → SEMA Core
                        ├── Capability Manager
                        ├── Resource Manager
                        ├── Runtime Manager
                        └── Hardware Abstraction Layer (HAL)
                                        │
                                        ▼
                                 Board/Chip Profile
```

El código pregunta capacidades (`¿ADC? ¿PCNT? ¿PSRAM? ¿2 cores? ¿CAN/TWAI?`) en lugar
de preguntar por modelo. Detalle en [ADR 0002](adr/0002-perfiles-de-placa-capacidades-y-hal.md)
y [Arquitectura](Arquitectura.md).

## 5. Regla de actualización

Cada decisión nueva se registra con motivo y consecuencias en
`docs/DUDAS-Y-DECISIONES.md`; cuando es estructural, se formaliza como ADR acá con
numeración secuencial y se enlaza desde §1. Las dudas `Q-xxxx` se cierran con una
decisión `D-xxxx` **antes** de implementar el área afectada.

## Ver también

- [Arquitectura](Arquitectura.md) · [Evolución](Evolucion.md) · [Mejoras y roadmap](Mejoras-y-roadmap.md)
- [Referencia de código](Referencia-de-codigo.md) · [Compatibilidad de versiones](Compatibilidad-de-versiones.md)
