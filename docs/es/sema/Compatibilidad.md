---
tags:
  - sema
  - compatibilidad
---

# Compatibilidad (placas, sensores, buses)

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.23.0

## Placas

| Placa | Estado | Notas |
|-------|--------|-------|
| ESP32 DOIT DevKit v1 | ✅ Objetivo | Board por defecto en `platformio.ini` |
| ESP32 (genérico, 4 MB) | ✅ Compatible | Mismo chip; revisar pines ADC/GPIO |
| ESP32 con PSRAM | ✅ | Se declara por `Capability::Psram` |
| ESP32-S2/S3/C3 | ⚠️ Parcial | Mismas APIs, pero pines distintos; ajustar `BoardProfile.hpp` |

> El código consulta **capacidades** (`CapabilityManager`) en lugar de preguntar el
> modelo de placa, por lo que portar a otro ESP32 es principalmente configurar pines.

## Sensores

| Modelo | Interfaz | Soportado |
|--------|----------|-----------|
| BME280 / BMP280 | I²C | ✅ |
| SHT40 / SHT31 / AHT20 | I²C | ✅ |
| BH1750 | I²C | ✅ |
| VEML6075 | I²C | ✅ |
| SCD30 | I²C | ✅ |
| SGP30 | I²C | ✅ |
| AS3935 | I²C | ✅ |
| ADS1115 | I²C | ✅ |
| DS18B20 | 1-Wire | ✅ |
| ADC genérico (analógico) | ADC | ✅ |
| PCNT (pulsos) | PCNT | ✅ |
| PMS5003 | UART | ✅ |

## Buses

| Bus | Estado |
|-----|--------|
| I²C (SDA/SCL) | ✅ |
| 1-Wire | ✅ |
| UART (Serial2) | ✅ |
| SPI | ⚠️ Declarado en capacidades; sin driver dedicado aún |
| CAN (TWAI) | ✅ |
| RS485/Modbus | ✅ |
| LoRa (SX1262) | ✅ |
| Zigbee (CC2652P2) | ✅ |

## Expansores

| Expansor | Estado |
|----------|--------|
| MCP23017 (16 GPIO) | ✅ |
| ADS1115 (4 ADC) | ✅ |
| 74HC595 / 74HC165 | ✅ |

## Navegador / dashboard

- Cualquier navegador moderno (HTML + JS + WebSocket + Canvas).
- Acceso por `http://<ip>/` o `http://<hostname>.local/` (mDNS).
