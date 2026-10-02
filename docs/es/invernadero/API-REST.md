---
tags:
  - invernadero
  - api
---

# API REST

> **Tipo:** API | **Estado:** Estable | **Fecha:** 2026-10-02

Base: `http://<IP>/api/v1/` (puerto 80).

## Estado y lectura

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/status` | Estado general (temperatura, humedad, suelo, tanque, luz, bomba, ventilador) |
| GET | `/sensors` | Lista de sensores (con calidad) |
| GET | `/weather` | Estación meteorológica externa |
| GET | `/actuators` | Lista de actuadores |
| GET | `/zones` | Zonas |
| GET | `/events` | Últimos eventos |
| GET | `/alarms` | Alarmas (severidad ≥ 2) |

## Configuración

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET/PUT | `/config` | Configuración JSON completa (incluye `net_interface`, `eth_*`) |
| GET | `/config/schema` | Esquema descriptivo |
| POST | `/config/rollback` | Restaura la configuración anterior |
| POST | `/reset` | `{"level":"network"\|"automation"\|"factory"}` |

## Control de actuadores

```
POST /api/v1/actuators
{"role":"pump","index":0,"output":100}
```

Roles: `pump`, `valve`, `fan`, `extractor`, `heater`, `humidifier`, `light`,
`roof`, `window`, `shade`, `alarm`.

## Identidad, red y diagnóstico

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/device` | UID, id, perfil, firmware, estado, causa de reinicio |
| GET | `/capabilities` | Capacidades (`WIFI`, `I2C`, `SPI`, `RS485`, `MODBUS`, ...) |
| GET | `/network` | SSID, IP, interfaz (wifi/ethernet), MQTT, NTP, DNS |
| GET | `/rs485` + `/rs485/scan` | Estadísticas y escaneo Modbus |
| GET | `/modbus?slave=1&func=3&reg=0` | Lectura de registros |
| GET | `/modbus/profiles` | Perfiles e instancias Modbus (V9) |
| GET | `/modbus/gateway` | Gateway RS485: valores y estado por esclavo |
| GET | `/diagnostics` | WiFi, RSSI, MQTT, heap, uptime |

## Plataforma configurable (V8/V9)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/modules` | Módulos embebidos |
| GET | `/buses` | Buses registrados |
| GET | `/hardware` | Nodos de hardware |
| GET | `/sensors/catalog` | Catálogo de sensores |
| GET | `/actuators/catalog` | Catálogo de actuadores |
| GET | `/storage` | Almacenamiento (LittleFS/SPIFFS/SD) |
| GET | `/health` | Health monitor por tareas |
| GET | `/boot` | Contadores de reinicio |
| GET | `/logs` | Logs estructurados |

## Token de API

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/token/rotate` | Genera token (se muestra una vez) |
| POST | `/token/revoke` | Revoca token |
| GET | `/token/status` | Estado (sin revelar el token) |

## Autenticación

Los endpoints de escritura requieren `Authorization: Bearer <token>` o
`X-Auth-Token` (sesión de admin o token de API).
