---
tags:
  - sema
  - desarrollo
---

# Compilación y flasheo

> **Tipo:** Guía | **Estado:** Estable | **Firmware:** v1.16.0

## Requisitos

- PlatformIO Core (CLI) o PlatformIO IDE (VS Code).
- Board: `esp32doit-devkit-v1`.

## Compilar

```bash
pio run -e esp32doit-devkit-v1
```

Salida esperada:

```text
RAM:   [==        ]  16.3% (used 53432 bytes from 327680 bytes)
Flash: [======    ]  64.8% (used 1189797 bytes from 1835008 bytes)
========================= [SUCCESS] Took ~25 s =========================
```

El binario se genera en `.pio/build/esp32doit-devkit-v1/firmware.bin`.

## Flashear (USB)

```bash
pio run -e esp32doit-devkit-v1 -t upload
```

(o `-t upload --upload-port COM3`).

## Monitor serial

```bash
pio device monitor -b 115200
```

## Actualizar por OTA

```bash
curl -X POST http://<ip>/api/v1/ota \
  -H "X-API-Key: <api_key>" \
  -F "firmware=@.pio/build/esp32doit-devkit-v1/firmware.bin"
```

## Dependencias (lib_deps)

| Librería | Uso |
|----------|-----|
| ArduinoJson 6 | JSON (config, API, publishers) |
| Adafruit BME280/BMP280/SHT4x/SHT31/AHTx0/VEML6075/SCD30/SGP30/PM25 AQI/ADS1X15 | sensores I²C |
| Adafruit Unified Sensor + BusIO | base de Adafruit |
| OneWire + DallasTemperature | DS18B20 |
| PubSubClient | MQTT |
| WebSockets | WebSocket |
| SparkFun AS3935 | rayos |
| Adafruit MCP23017 | expansor GPIO |

## Particiones

`partitions.csv` define app0/app1 (OTA) + spiffs (LittleFS).
