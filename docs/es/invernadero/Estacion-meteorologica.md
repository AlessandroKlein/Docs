---
tags:
  - invernadero
  - red
---

# Estación meteorológica externa

> **Tipo:** Firmware (ESP32) | **Estado:** Estable | **Fecha:** 2026-10-02

El ESP32 consulta una **estación meteorológica externa** que publique JSON en un
URL (HTTP/HTTPS). Los nombres de campo son configurables.

## Configuración

```json
{
  "weather": {
    "enabled": true,
    "url": "http://192.168.1.50/weather.json",
    "interval_ms": 60000,
    "root": "",
    "key_temp": "temp",
    "key_hum": "hum",
    "key_wind": "ane",
    "key_rain": "pluv",
    "key_pressure": "pres",
    "key_light": "lux"
  }
}
```

| Campo | Descripción |
|-------|-------------|
| `enabled` | Activa/desactiva |
| `url` | URL que publica el JSON |
| `interval_ms` | Frecuencia (default 60000) |
| `root` | Subobjeto raíz opcional |
| `key_*` | Nombres de campo (libres, según esquema) |

## Lectura

```bash
curl http://invernadero.local/api/v1/weather
```

```json
{ "enabled": true, "available": true, "temperature": 23.5, "humidity": 71,
  "wind_speed": 12.2, "rain": 0, "pressure": 1012.4, "light": 642 }
```

También se incluye en `/api/v1/status` (`weather`, `ext_temp`, `ext_hum`, `wind`,
`rain`, `pressure`).

## Integración MQTT

Publica en `greenhouse/{device_id}/weather`; el servidor lo guarda en
`device_shadow.reported.weather`.

## Usos

Ventilación, techo (cierre por lluvia/viento), sombreado, riego, alarmas.
