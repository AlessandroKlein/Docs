# Hardware compatibility

> **Type:** Reference | **Status:** Stable | **Date:** 2026-10-02

## ESP32 boards

| Board (env) | MCU | Cores | WiFi | CAN/TWAI | Int. temp |
|-------------|-----|-------|------|----------|-----------|
| `esp32doit-devkit-v1` | ESP32 | 2 | ✅ | ✅ | ✅ |
| `esp32-s3-devkitc-1` | ESP32-S3 | 2 | ✅ + BLE | ✅ | ✅ |
| `esp32-s2-saola-1` | ESP32-S2 | 1 | ✅ | ✅ | ❌ |
| `esp32-c3-devkitm-1` | ESP32-C3 | 1 | ✅ + BLE | ❌ | ✅ |
| `esp32-c6-devkitc-1` | ESP32-C6 | 1 | WiFi 6 + BLE | ❌ | ✅ |

> Full firmware (~1.25 MB) only fits 8 MB or 16 MB partitions.

## Buses

I²C · SPI (bit-banged + native) · 1-Wire · UART/RS485 · GPIO · CAN/TWAI.

## Sensors

| Magnitude | Model | Interface | Address / pin |
|-----------|-------|-----------|---------------|
| Temperature/Humidity | SHT31 | I²C | 0x44 |
| Temperature/Humidity | AHT20 | I²C | 0x38 |
| Temperature | DS18B20 | 1-Wire | GPIO 4 |
| Soil moisture | Capacitive | ADC | A0–A3 |
| Light | BH1750 | I²C | 0x23 |
| CO₂ | SCD40/SCD41 | I²C | 0x62 |
| Flow | Flow meter | pulse | GPIO 34 |
| Tank level | Ultrasonic | GPIO | 25/26 |
| Rain | Rain gauge | pulse | GPIO 35 |
| Wind | Anemometer | pulse | GPIO 36 |
| pH / EC | Analog/Modbus | ADS1115/RS485 | — |

## Actuators

`pump` · `valve` · `fan` · `extractor` · `heater` · `humidifier` · `light` ·
`window` · `roof` · `shade` · `alarm`.

## Expanders / ADC

74HC595 · 74HC165 · MCP23017 (I²C) · MCP23S17 (SPI) · ADS1115 · MCP3008/MCP3208.

## I²C addresses

SHT31 0x44 · AHT20 0x38 · ADS1115 0x48–0x4B · BH1750 0x23 · SCD4x 0x62 · MCP23017 0x20–0x27.
