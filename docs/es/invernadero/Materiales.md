---
tags:
  - hardware
  - bom
---

# Materiales (BOM)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

Lista de materiales por categoría. Cantidades indicativas; ajustar según la
instalación.

## 1. Controlador

| Componente | Cant. | Notas |
|-----------|-------|-------|
| ESP32 DevKit (WROOM N8/N16) | 1 | o S2/S3/C3/C6 |
| Fuente 5 V (2 A) | 1 | alimentación lógica |

## 2. Sensores

| Componente | Interfaz | Notas |
|-----------|----------|-------|
| SHT31 (temp + humedad) | I²C 0x44 | ±0,2 °C / ±2 %RH |
| AHT20 (temp + humedad) | I²C 0x38 | opcional |
| DS18B20 (temp) | 1-Wire | + resistencia 4,7 kΩ |
| Humedad de suelo capacitiva | ADC | por zona |
| BH1750 (lux) | I²C 0x23 | |
| SCD40/SCD41 (CO₂) | I²C 0x62 | opcional |
| Caudalímetro | pulsos | + optoacoplador |
| Pluviómetro | pulsos | |
| Anemómetro | pulsos | |
| Ultrasónico (tanque) | GPIO 25/26 | + flotadores |
| pH (analógico o Modbus) | ADS1115/RS485 | |
| EC (analógico o Modbus) | ADS1115/RS485 | |

## 3. Actuadores

- Bomba (DC) + MOSFET + diodo flyback.
- Electroválvulas (1..8) + drivers.
- Ventiladores/extractores.
- Calefacción / humidificador (opcional).
- Relé/SSR para cargas de red.

## 4. Expansores y ADC

| Componente | Notas |
|-----------|-------|
| 74HC595 (×4) | 32 salidas |
| 74HC165 | entradas |
| MCP23017 | expansor I²C |
| MCP23S17 | expansor SPI |
| ADS1115 | ADC I²C 16 bits |

## 5. Pasivos

- Resistencias de pull-up 4,7 kΩ (I²C, 1-Wire).
- Capacitores de desacople 100 nF.
- Divisores resistivos para sensores analógicos.

## 6. Protecciones

- Diodos flyback (cargas inductivas).
- Optoacopladores (pulsos de 24 V).
- Fusibles.
- Separación física lógica / potencia.
