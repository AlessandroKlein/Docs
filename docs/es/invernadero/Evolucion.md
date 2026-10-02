# Evolución (V8 / V9 / V10)

> **Tipo:** Roadmap | **Estado:** En desarrollo | **Fecha:** 2026-10-02

Objetivo: que el firmware pase de representar **una instalación** a representar
**una plataforma**.

```text
FIRMWARE       → "qué puede hacer"     (capacidades)
CONFIGURACIÓN  → "qué hardware hay"    (instalación)
AUTOMATIZACIÓN → "qué debe hacer"      (comportamiento)
```

## Estado

| Área | Estado |
|------|--------|
| Firmware de campo | ✅ v3.13.0 |
| Servidor central | ✅ entregado |
| V8 base + registros REST | ✅ |
| V8.1 (SpiManager, 74HC165, MCP23S17, ADC) | ✅ |
| V8.4 StorageManager (LittleFS/SPIFFS/SD) | ✅ |
| V9 Modbus profiles + capa CAN | ✅ |
| Interfaz WiFi/Ethernet intercambiable (W5500) | ✅ v3.13.0 |
| Multi-board + página `/pins` | ✅ |
| W5500 (interfaz completa con HTTP/OTA sobre Ethernet) | ⚠️ parcial (MQTT sí; HTTP/OTA pendiente) |
| Configuración por capas: merge/migraciones | ❌ |
| Store & Forward | ❌ |
