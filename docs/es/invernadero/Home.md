---
tags:
  - invernadero
  - general
---

# Invernadero

> **Tipo:** Embebidos (ESP32) | **Estado:** En desarrollo | **Fecha:** 2026-10-02 | **Firmware:** v3.12.0

Sistema electrónico modular para supervisión, automatización y control de
invernaderos mediante **ESP32**, configurable por **interfaz web** sin recompilar
el firmware.

## Características

- Firmware único para cualquier instalación (interior/exterior, CO₂, pH, EC, techo…).
- Configuración no volátil (NVS/JSON) editable por API REST y web.
- Autonomía total: el ESP32 controla localmente sin servidor ni Internet.
- API REST, WebSocket, MQTT y OTA con doble partición y rollback.
- Multiprotocolo: I²C, SPI, 1-Wire, ADC, GPIO, RS485/Modbus RTU, CAN/TWAI.
- Plataforma configurable (V8/V9): registros, módulos, buses, catálogos.

## Repositorio

- Código: <https://github.com/AlessandroKlein/Invernadero>
- Especificación: `README.md` (286 secciones).

## Índice

- [Arquitectura](Arquitectura.md)
- [Hardware y conexiones](Hardware-y-Conexiones.md)
- [Decisiones](Decisiones.md)
- [Registro de cambios](Registro-de-cambios.md)
