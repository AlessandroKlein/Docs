---
tags:
  - invernadero
  - general
---

# Invernadero

> **Tipo:** Embebidos (ESP32) | **Estado:** En desarrollo | **Fecha:** 2026-10-02 | **Firmware:** v3.29.0

Sistema electrónico modular para supervisión, automatización y control de
invernaderos mediante **ESP32**, configurable por **interfaz web** sin recompilar
el firmware.

## Características

- Firmware único para cualquier instalación (interior/exterior, CO₂, pH, EC, techo…).
- Configuración no volátil (NVS/JSON) editable por API REST y web.
- Autonomía total: el ESP32 controla localmente sin servidor ni Internet.
- API REST (48 endpoints), WebSocket, MQTT y OTA con doble partición y rollback.
- Multiprotocolo: I²C, SPI, 1-Wire, ADC, GPIO, RS485/Modbus RTU, CAN/TWAI.
- **Modularidad completa (v3.29.0)**: pines, sensores y expansores configurables
  desde la web sin recompilar.

## Repositorio

- Código: <https://github.com/AlessandroKlein/Invernadero>
- Especificación: `README.md`.
- Wiki del proyecto: <https://github.com/AlessandroKlein/Invernadero/wiki>

## Índice de la documentación

**Empezar**

- [Inicio rápido](Inicio-rapido.md) · [Arquitectura](Arquitectura.md) ·
  [Diagramas](Diagramas.md) · [Compilación](Compilacion.md)

**Referencia técnica**

- [Referencia de código (v3.29.0)](Referencia-de-codigo.md)
- [Referencia API interna](Referencia-API-interna.md)
- [Enumeraciones y tipos](Enumeraciones-y-tipos.md)
- [Referencia de configuración (JSON)](Referencia-configuracion.md)
- [API REST](API-REST.md) · [MQTT y WebSocket](MQTT-y-WebSocket.md)

**Hardware**

- [Compatibilidad](Compatibilidad.md) · [Guía de pines](Guia-de-pines.md) ·
  [Hardware y conexiones](Hardware-y-Conexiones.md) · [Materiales](Materiales.md)

**Uso**

- [Sensores](Sensores.md) · [Actuadores](Actuadores-y-Salidas.md) ·
  [Control PID](Control-PID.md) · [Configuración](Configuracion.md) ·
  [Variables modificables](Variables-Modificables.md)

**Operación**

- [OTA y actualización](OTA-y-Actualizacion.md) · [Seguridad](Seguridad.md) ·
  [Identidad y estados](Identidad-y-Estados.md) ·
  [Estación meteorológica](Estacion-meteorologica.md) ·
  [Servidor central](Servidor-central.md)

**Soporte**

- [Solución de problemas](Solucion-de-problemas.md) · [FAQ](FAQ.md) ·
  [Glosario](Glosario.md)

**Proyecto**

- [Decisiones](Decisiones.md) · [Evolución](Evolucion.md) ·
  [Mejoras](Mejoras.md) · [CHANGELOG](CHANGELOG.md) ·
  [Registro de cambios](Registro-de-cambios.md) ·
  [Guía de desarrollo](Guia-de-desarrollo.md) ·
  [Frontend](Frontend.md)
