---
tags:
  - sema
  - roadmap
---

# Evolución

> **Tipo:** Roadmap | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Desarrollo incremental de SEMA en las **9 fases** definidas en `README.md` §104 y
verificadas contra el código y el [CHANGELOG](CHANGELOG.md).

Estado: ✅ hecho · 🔄 parcial · ⬜ pendiente.

## 1. Resumen de fases

| Fase | Tema | Estado | Versiones |
|:----:|------|:------:|-----------|
| 1 | Core (web, config, NVS, diagnóstico) | ✅ | v0.34.0 |
| 2 | Sensores básicos | ✅ | v0.39.0 |
| 3 | Expansión (MCP23017, 74HC595/165, ADS1115) | ✅ | v1.14.0 – v1.17.0 |
| 4 | Meteorología (viento, lluvia, radiación, UV, rayos) | ✅ | v1.13.0 – v1.19.0 |
| 5 | Calidad ambiental (CO₂, PM, CO) | ✅ | v1.10.0 – v1.19.0 |
| 6 | Industrial (RS485/Modbus, CAN) | ✅ | v1.20.0 – v1.21.0 |
| 7 | Comunicaciones remotas (LoRa, Zigbee, Ethernet, MQTT) | 🔄 | v1.22.0 – v1.25.0 |
| 8 | Energía (batería, perfiles, deep sleep) | 🔄 | v0.34.0 en adelante |
| 9 | Plataforma distribuida (multi-estación, mapas, alertas) | ⬜ | — |

> El proyecto pasó de `v0.1.0` (2026-10-03) a `v1.103.0` (2026-10-08): **160 entradas
> en el CHANGELOG** y **159 releases** en GitHub. Cada incremento lógico es una versión
> MINOR/PATCH (ver [Versionado](../inicio/Versionado.md)).

---

## 2. Fase 1 — Core

```text
ESP32 · Web · Configuración · NVS · Diagnóstico
```

✅ **Esencial completa (v0.34.0)**: `ConfigManager` + NVS, Storage API, Capability/Runtime
Manager, `Scheduler`, servidor web y REST `/api/v1/*`, WebSocket, mDNS, autenticación
(`api_key`/`server_key`) y OTA con particiones A/B.

- Completado después: `Watchdog` (v1.x), `HealthMonitor`, hot reload de sensores/reglas/
  publicadores, rate limiting y expiración de sesión (v1.12.0), tema claro/oscuro (v1.34.0),
  verificación de integridad de OTA por SHA-256 (v1.35.0).
- Diagnóstico: enriquecido de forma continua (`/api/v1/diagnostics`, `/api/v1/health`).

Ver [Arquitectura](Arquitectura.md) y [Diagnóstico y salud](Diagnostico-y-salud.md).

## 3. Fase 2 — Sensores básicos

```text
DS18B20 · AHT20/AHT21/AHT30 · SHT31/SHT40 · BME280 · BMP280 · BH1750
```

✅ **(v0.39.0)**: BME280, SHT40, SHT31, DS18B20, BH1750, AHT20 y BMP280, más detección
automática de dispositivos I²C (`I2cScanner`) y 1-Wire multi-dispositivo.

Ver [Sensores](Sensores.md).

## 4. Fase 3 — Expansión

```text
MCP23017 · 74HC595 · 74HC165 · ADS1115
```

✅ **Completa**: MCP23017 (**v1.14.0**), ADS1115 (**v1.15.0**), 74HC595/74HC165
(**v1.17.0**). En v1.98.0–v1.103.0 se sumó la configuración de **expansores SPI de bus**
(SC18IS602B para I²C, MAX14830 con 4 puertos UART).

Ver [Expansores de entrada/salida](Expansores-de-entrada-salida.md) y
[Buses y periféricos](Buses-y-perifericos.md).

## 5. Fase 4 — Meteorología

```text
Viento · Lluvia · Radiación · UV · Rayos
```

✅ **Completa (v1.13.0 – v1.19.0)**:

| Magnitud | Implementación | Versión |
|----------|----------------|---------|
| Rayos | AS3935 (`lightning_distance`) | v1.13.0 |
| Lluvia y velocidad de viento | PCNT (conteo de pulsos) | v0.48.0 |
| UV | VEML6075 (UVA/UVB/índice) | v0.48.0 |
| Radiación solar | driver `SOLAR` por ADC | v1.19.0 |
| Dirección de viento (veleta) | tabla de 8 resistencias (WH-SP-WD) | v1.29.0 |

## 6. Fase 5 — Calidad ambiental

```text
CO₂ · PM · CO
```

✅ **Completa**: SGP30 (eCO₂/TVOC, v1.10.0), SCD30 (CO₂ NDIR), PMS5003
(PM1.0/PM2.5/PM10) y CO (**v1.18.0**).

## 7. Fase 6 — Industrial

```text
RS485 · Modbus · CAN
```

✅ **Completa**: maestro Modbus RTU sobre RS485 (**v1.20.0**, transceptor aislado
TD501D485H) y CAN/TWAI (**v1.21.0**).

Ver [Comunicaciones remotas](Comunicaciones-remotas.md).

## 8. Fase 7 — Comunicaciones remotas

```text
LoRa · Zigbee · Ethernet · MQTT · Servidor central
```

🔄 **Parcial**: MQTT (publicador), LoRa SX1262 (**v1.22.0**), Zigbee ZNP por UART
(**v1.23.0**) y Ethernet LAN8720A/W5500 (**v1.24.0**, W5500 por `esp_eth` en v1.25.0)
están implementados. El **Servidor Central** queda **fuera del alcance** (proyecto
separado): SEMA solo se autentica contra él con `server_key` y `SEMA_PROTOCOL_VERSION`.

## 9. Fase 8 — Energía

```text
Panel solar · Batería · Medición energética · Deep Sleep
```

🔄 **Parcial (desde v0.34.0)**: medición de batería por ADC (divisor configurable),
perfiles energéticos (`PowerManager`), deep sleep con timer RTC y wake-up por lluvia.
**Falta** la gestión de carga/MPPT del panel solar y la medición de corriente/consumo.

Ver [Energía y consumo](Energia-y-consumo.md).

## 10. Fase 9 — Plataforma distribuida

```text
Múltiples SEMA · Nodos remotos · Servidor central · Históricos · Mapas · Alertas
```

⬜ **Pendiente**: depende del Servidor Central (fuera de alcance). SEMA ya expone lo
necesario por estación: histórico local, eventos/alarmas persistentes y publicadores
(MQTT/HTTP) para que un sistema externo agregue varias estaciones.

---

## 11. Hitos recientes (v1.9x – v1.103)

| Versión | Hito |
|---------|------|
| v1.103.0 | MAX14830 con 4 puertos UART seleccionables |
| v1.102.0 | Gridstack embebido en PROGMEM (gzip); sin `data/` ni LittleFS para la web |
| v1.101.0 | Bus I²C/UART por expansores SPI (SC18IS602B / MAX14830) |
| v1.100.0 | Assets web comprimidos en flash (ahorro de RAM/flash) |
| v1.99.0 | Expansores visibles en la página de pines de buses |
| v1.98.0 | Expansores SPI MAX14830 / SC18IS602B (configuración) |
| v1.96.0 | Guardado propio de I²C + reinicio controlado |
| v1.35.0 | Verificación de integridad de OTA por SHA-256 |
| v1.34.0 | Tema claro/oscuro persistente en el dashboard |
| v1.33.0 | Retención del histórico por tiempo + export CSV |
| v1.29.0 | Dashboard Gridstack + veleta por tabla de resistencias |
| v1.27.0 | Firmware multi-chip y particiones por tamaño de flash |

El detalle completo está en el [CHANGELOG](CHANGELOG.md) y, archivo por archivo, en el
[Registro de cambios](Registro-de-cambios.md).

---

## Ver también

- [Mejoras y roadmap](Mejoras-y-roadmap.md) · [Futuro](Futuro.md) · [CHANGELOG](CHANGELOG.md)
- [Arquitectura](Arquitectura.md) · [Estadísticas y métricas](Estadisticas-y-metricas.md) · [Home](Home.md)
