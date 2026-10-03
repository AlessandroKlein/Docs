---
tags:
  - sema
  - roadmap
---

# Evolución

> **Tipo:** Roadmap | **Estado:** En desarrollo | **Fecha:** 2026-10-03 | **Firmware:** v0.38.0

Desarrollo incremental de SEMA en 9 fases (`README.md` §104).
Estado: ✅ hecho · 🔄 en curso · ⬜ pendiente.

## Fase 1 — Core

```text
ESP32 · Web · Configuración · NVS · Diagnóstico
```

✅ esencial completa (v0.34.0): ConfigManager+NVS, Storage API, Capability/Runtime
Manager, Scheduler, Web/REST `/api/v1`, WebSocket, mDNS, autenticación y OTA.

## Fase 2 — Sensores básicos

```text
DS18B20 · AHT20/AHT21/AHT30 · SHT31/SHT40 · BME280 · BMP280 · BH1750
```

🔄 (v0.34.0): BME280, SHT40, DS18B20, BH1750 y AHT20 + detección I²C.

## Fase 3 — Expansión

```text
MCP23017 · 74HC595 · 74HC165 · ADS1115
```

⬜ pendiente.

## Fase 4 — Meteorología

```text
Viento · Lluvia · Radiación · UV · Rayos
```

⬜ pendiente.

## Fase 5 — Calidad ambiental

```text
CO₂ · PM · CO
```

⬜ pendiente.

## Fase 6 — Industrial

```text
RS485 · Modbus · CAN
```

⬜ pendiente.

## Fase 7 — Comunicaciones remotas

```text
LoRa · Zigbee · Ethernet · MQTT · Servidor central
```

🔄 MQTT listo (publicador); LoRa/Zigbee/Ethernet/Servidor central pendientes.

## Fase 8 — Energía

```text
Panel solar · Batería · Medición energética · Deep Sleep
```

🔄 (v0.34.0): batería por ADC, perfiles energéticos, deep sleep y wake-up por lluvia.

## Fase 9 — Plataforma distribuida

```text
Múltiples SEMA · Nodos remotos · Servidor central · Históricos · Mapas · Alertas
```

⬜ pendiente.

---

Ver también: [Arquitectura](Arquitectura.md) · [Home](Home.md).
