---
tags:
  - sema
  - bom
---

# Materiales (BOM)

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.76.0

Lista de materiales sugerida para una estación meteorológica SEMA completa.

## Núcleo

| Componente | Cantidad | Notas |
|------------|----------|-------|
| ESP32 DevKit v1 (DOIT) | 1 | Board objetivo de PlatformIO |
| Fuente 5 V / 2 A (micro-USB) | 1 | Alimentación |
| Protoboard / PCB | 1 | Montaje |

## Sensores (opcionales, según necesidad)

| Componente | Interfaz | Cantidad |
|------------|----------|----------|
| BME280 | I²C | 1 |
| SHT40 (o SHT31/AHT20) | I²C | 1 |
| DS18B20 (sonda) | 1-Wire | 1–4 |
| BH1750 | I²C | 1 |
| VEML6075 | I²C | 1 |
| SCD30 (CO₂) | I²C | 1 |
| SGP30 (eCO₂/TVOC) | I²C | 1 |
| PMS5003 (PM) | UART | 1 |
| AS3935 (rayos) | I²C | 1 |
| ADS1115 (ADC ext.) | I²C | 1 |
| Pluviómetro (reed) | PCNT | 1 |
| Anemómetro (hall) | PCNT | 1 |

## Pasivos y conexiones

| Componente | Cantidad | Uso |
|------------|----------|-----|
| Resistencia 4,7 kΩ | 3 | Pull-up I²C (×2) + 1-Wire |
| Resistencia 100 kΩ | 1 | Divisor batería (R1) |
| Resistencia 10 kΩ | 1 | Divisor batería (R2) |
| Resistencia 220–470 Ω | — | LEDs |
| Cables dupont / jumper | — | Conexiones |

## Actuadores (opcionales)

| Componente | Cantidad | Notas |
|------------|----------|-------|
| Módulo de relé (1–8 canales) | 1 | Salidas digitales |
| MCP23017 (expansor I²C) | 1 | 16 GPIO extra |
| Transistor/MOSFET | — | Cargas que superen el pin |

## Buses y comunicación (opcionales)

| Componente | Bus | Cantidad | Notas |
|------------|-----|----------|-------|
| **TD501D485H** | RS485 | 1 | Transceiver **aislado** (protección contra descargas de la línea RS485) |
| **SN65HVD23X** (familia) | CAN | 1 | Transceiver CAN 3,3 V (SN65HVD230/231/232/233/234) |
| **SX1262PATR8-GC** (Silicontra) | LoRa | 1 | Módulo LoRa 868/915 MHz |
| **RF-BM-2652P2** (CC2652P2) | Zigbee | 1 | Módulo Zigbee 3.0 (ZNP) |
| **LAN8720A** | Ethernet (RMII) | 1 | PHY nativa (ESP32-WROOM y derivados) |
| **W5500** | Ethernet (SPI) | 1 | Módulo Ethernet SPI (ESP32-S3 y derivados) |

## Herramientas

- Cable micro-USB.
- Multímetro.
- Estación de soldadura (si se usa PCB).
