---
tags:
  - invernadero
  - referencia
---

# Estadísticas y métricas del proyecto (v3.29.0)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

Todos los números medidos del proyecto en la versión actual.

## 1. Código fuente

| Métrica | Valor |
|---------|------:|
| Archivos `.cpp` (implementación) | 48 |
| Archivos `.hpp` (interfaces) | 58 |
| Líneas `.cpp` | ~5.238 |
| Líneas `.hpp` | ~2.793 |
| **Total** | **~8.031 líneas** |
| Módulos (carpetas) | 11 |

### Distribución por módulo (líneas `.cpp`)

| Módulo | Líneas | % |
|--------|-------:|--:|
| `sensors` | 1.083 | 20,7 % |
| `hardware` | 968 | 18,5 % |
| `system` | 713 | 13,6 % |
| `api` | 681 | 13,0 % |
| `config` | 599 | 11,4 % |
| `core` | 405 | 7,7 % |
| `main.cpp` | 350 | 6,7 % |
| `actuators` | 332 | 6,3 % |
| `network` | 314 | 6,0 % |
| `control` | 305 | 5,8 % |
| `storage` | 165 | 3,1 % |

## 2. Uso de recursos (ESP32, partición 8 MB)

| Recurso | Uso | Detalle |
|---------|-----|---------|
| Flash | **38,6 %** | 1.291.129 / 3.342.336 bytes |
| RAM | **27,2 %** | 89.064 / 327.680 bytes |
| Tiempo de build | ~27 s | entorno único `esp32doit-devkit-v1` |

## 3. Interfaces

| Interfaz | Cantidad |
|----------|---------:|
| Endpoints REST | 48 |
| Páginas web HTML | 2 (`/` y `/pins`) |
| Tópicos MQTT | 4 (`state`, `sensors`, `actuators`, `weather`) + `cmd` |
| Namespaces NVS propios | 3 (`ghpins`, `ghsensors`, `ghhw`) + config |
| Tareas FreeRTOS | 2 (+ `loop()` en núcleo 1) |

## 4. Límites del sistema

| Límite | Constante | Valor |
|--------|-----------|------:|
| Sensores | `MAX_SENSORS` | 20 |
| Zonas de suelo | `MAX_SOIL_ZONES` | 4 |
| Actuadores | `MAX_ACTUATORS` | 32 |
| Válvulas | `actValves` | 0..8 |
| Zonas | `zoneCount` | 0..8 |
| Entradas del catálogo | `SensorRegistry::MAX_ENTRIES` | 24 |
| Nodos de hardware | `HardwareManager::MAX_NODES` | 24 |
| Buses | `BusManager::MAX_BUSES` | 12 |
| Pool MCP23017 | `mcpPool[4]` | 4 |
| Pool MCP23S17 | `spiPool[4]` | 4 |
| Pool ADC | `adcPool[4]` | 4 |
| Esclavos del gateway | `ModbusGateway::MAX_SLAVES` | 24 |
| Registros históricos | `History` | circular |

## 5. Versiones

| Elemento | Valor |
|----------|-------|
| Firmware | `3.29.0` |
| Hardware | `rev0` |
| Perfil | `ESP32-GH-V1` |
| Esquema de config | `2` |
| Protocolo | `1` |

## 6. Hitos del release actual

```text
Modularidad completa:  v3.18.0 → v3.29.0 (12 releases)
Últimos release:       v3.29.0 (asignación de canales, pool MCP23017)
Endpoints nuevos:      /pins, /hardware, /sensors/catalog, /modbus/gateway, /detect
```

Ver [Referencia de código](Referencia-de-codigo.md) y
[Mejoras](Mejoras.md).
