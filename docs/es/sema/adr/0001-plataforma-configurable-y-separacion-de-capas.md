---
tags:
  - sema
  - adr
  - arquitectura
---

# 0001. SEMA es una plataforma configurable, no una estación fija

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0001, D-0002, D-0003

## Contexto

SEMA tiene que servir a instalaciones muy distintas (interior o exterior, con o sin
viento, con o sin calidad de aire) sin mantener un firmware por variante. Había dos
caminos: un firmware específico por estación, o una **plataforma única** cuya conducta
se define por configuración. En paralelo, mezclar drivers de sensor, lógica de medición
y servicios (web, MQTT, almacenamiento) en un mismo bloque hace que cambiar un sensor
arrastre a todo el resto.

## Decisión

Se adopta la plataforma única y se separa el sistema en tres capas desacopladas
(D-0001, D-0002):

```text
Hardware (GPIO, I²C, SPI, UART, ADC, RS485, CAN, 1-Wire, expansores)
        │
        ▼
Sensores (magnitudes: temperatura, humedad, presión, luz, UV, CO₂, PM, viento, lluvia…)
        │
        ▼
Servicios (web, API REST, WebSocket, MQTT/HTTP, storage, OTA, energía)
```

Reglas asociadas:

1. **El hardware define capacidades; la configuración web define cómo se usan.**
2. **El Core nunca depende de módulos opcionales**: una falla opcional no detiene la
   adquisición.
3. El **almacenamiento y los protocolos son abstracciones del Core**, no módulos que
   contaminan el núcleo meteorológico (D-0003 corregida): SEMA funciona sin microSD,
   pero el Core **conoce** el concepto de almacenamiento.

## Consecuencias

- ✅ Un solo firmware: conectar un sensor es cablearlo y declararlo en `sensors[]`.
- ✅ Drivers intercambiables tras interfaces estables: `Sensor`
  (`include/core/sensors/Sensor.hpp`), `Publisher`
  (`include/core/publishers/Publisher.hpp`), `Module` (`include/core/Module.hpp`) y
  `KeyValueStore` (`include/core/storage/Storage.hpp`).
- ✅ `SemaCore::applySensors()` (`src/core/SemaCore.cpp` §270-315) arma el catálogo
  desde `config.sensors[]` o cae al catálogo fijo de `BoardProfile.hpp`.
- ⚠️ Costo: cada medición es un `Measurement` con 5 `String`; el modelo canónico
  cuesta RAM y copias (ver ADR 0003).
- ⚠️ "Modular" hoy significa **compilación + configuración**: no existe carga dinámica
  de módulos en runtime (ver ADR 0014).
- ⚠️ El enum `Capability` declara 16 capacidades, pero el perfil base se carga a mano
  en `SemaCore::setup()` (ver ADR 0002).

## Ver también

- [Decisiones](../Decisiones.md) · [Arquitectura](../Arquitectura.md) ·
  [Módulos y ciclo de vida](../Modulos-y-ciclo-de-vida.md) ·
  [Referencia de código](../Referencia-de-codigo.md)
