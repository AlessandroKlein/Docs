---
tags:
  - sema
  - changelog
---

# Registro de versiones (CHANGELOG)

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.22.0

Historial de versiones de SEMA (resumen). El changelog completo con enlaces está en
[`CHANGELOG.md`](https://github.com/AlessandroKlein/SEMA/blob/main/CHANGELOG.md).

## 1.x (estable)

| Versión | Resumen |
|---------|---------|
| **1.22.0** | LoRa (SX1262) |
| **1.21.0** | CAN 2.0 (TWAI) |
| **1.20.0** | RS485/Modbus RTU |
| **1.19.0** | Sensor SOLAR (radiación solar) |
| **1.18.0** | Sensor CO (monóxido de carbono) |
| **1.17.0** | Shift registers 74HC595/74HC165 |
| **1.16.0** | Cookie de sesión `SameSite=Strict` (mitiga CSRF) |
| **1.15.0** | Sensor ADS1115 (ADC externo 16 bits, I²C) |
| **1.14.0** | Expansor MCP23017 (16 GPIO por I²C) |
| **1.13.0** | Sensor AS3935 (detección de rayos) |
| **1.12.0** | Expiración de sesión del login (1 h) |
| **1.11.0** | Autenticación MQTT (usuario/contraseña) |
| **1.10.0** | Sensor SGP30 (eCO₂/TVOC) |
| **1.9.0** | Hot reload de sensores |
| **1.8.0** | Hot reload de publicadores |
| **1.7.0** | GPIO standalone configurable + API `/api/v1/gpio` |
| **1.6.0** | Rate limiting en el login |
| **1.5.0** | Edición de config desde la UI |
| **1.4.0** | Gráfico de temperatura en el dashboard |
| **1.3.0** | Histórico en el dashboard |
| **1.2.0** | Hot reload de reglas/calibración |
| **1.1.0** | Rotación del EventLog |
| **1.0.0** | Primera versión estable |

## 0.x (desarrollo)

| Versión | Resumen |
|---------|---------|
| 0.56.0 | Login web con sesión |
| 0.55.0 | NTP/RTC |
| 0.54.0 | Board Profile (pines fijos) |
| 0.53.0 | Rotación del histórico |
| 0.52.0 y anteriores | Core, sensores, API, OTA, energía, watchdog, health, eventos/alarmas |
