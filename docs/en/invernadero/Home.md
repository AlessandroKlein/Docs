# Greenhouse (Invernadero)

> **Type:** Embedded (ESP32) | **Status:** In development | **Date:** 2026-10-02 | **Firmware:** v3.14.0

Modular electronic system for supervision, automation and control of greenhouses
using an **ESP32**, configured entirely through a **web interface** without
recompiling the firmware.

## Features

- Single firmware for any installation (indoor/outdoor, CO₂, pH, EC, roof...).
- Non-volatile configuration (NVS/JSON), editable via REST API and web.
- Full autonomy: the ESP32 controls locally without server or Internet.
- REST API, WebSocket, MQTT and OTA with dual partition and rollback.
- Multiprotocol: I²C, SPI, 1-Wire, ADC, GPIO, RS485/Modbus RTU, CAN/TWAI.
- Configurable platform (V8/V9): registries, modules, buses, catalogs.

## Repository

- Code: <https://github.com/AlessandroKlein/Invernadero>
