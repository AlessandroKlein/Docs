---
tags:
  - invernadero
  - configuracion
---

# Referencia de configuración (JSON completa)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

Referencia **exhaustiva** de cada clave del JSON de configuración
(`SystemConfig`, `include/core/Types.hpp`). Editable por `PUT /api/v1/config` o la
web. Esquema actual: **`schema_version: 2`**.

!!! info "Cómo leer esta página"
    - **Clave**: nombre exacto en el JSON.
    - **Tipo**: `bool`, `int`, `float`, `str`, `enum`.
    - **Default**: valor de fábrica si no se especifica.
    - Todos los campos son opcionales: si faltan, se usa el default.

---

## Raíz

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `config_version` | int | `1` | Versión de la configuración (autoincrementa en cada guardado) |
| `schema_version` | int | `1`→`2` | Esquema; dispara las migraciones de `ConfigManager::migrate()` |
| `simulation` | bool | `false` | Genera datos sintéticos sin hardware |
| `update_channel` | enum | `stable` | `stable` · `beta` · `development` |
| `configuration_source` | enum | `local` | `local` · `central` (quién es dueño de la config) |
| `device` | objeto | — | Identidad del dispositivo |
| `features` | objeto | — | Funciones habilitadas |
| `climate` | objeto | — | Control climático |
| `irrigation` | objeto | — | Riego |
| `ventilation` | objeto | — | Ventilación |
| `roof` | objeto | — | Techo/ventanas |
| `lighting` | objeto | — | Iluminación |
| `calibration` | objeto | — | Calibración de sensores |
| `network` | objeto | — | Red, MQTT, Ethernet, servidor |
| `weather` | objeto | — | Estación meteorológica externa |
| `sensors` | objeto | — | Sensores habilitados |
| `actuators` | objeto | — | Actuadores habilitados |
| `zones` | objeto | — | Zonas |
| `auth` | objeto | — | Autenticación local |

## `device`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `id` | str | `"GH001"` | Identificador lógico del dispositivo |
| `name` | str | `"Invernadero"` | Nombre visible |
| `type` | enum | `outdoor` | `indoor` · `outdoor` · `mixed` |
| `greenhouse_id` | str | `"GREENHOUSE-001"` | Identificación del invernadero |

## `features`

Todos `bool`. Habilitan módulos funcionales completos.

| Clave | Default | Descripción |
|-------|---------|-------------|
| `climate` | `true` | Control de temperatura/humedad |
| `irrigation` | `true` | Control de riego |
| `lighting` | `false` | Iluminación |
| `co2` | `false` | Control de CO₂ |
| `heating` | `false` | Calefacción |
| `humidification` | `false` | Humidificación |
| `roof` | `false` | Techo automático |
| `windows` | `false` | Ventanas automáticas |
| `shade` | `false` | Sombreado automático |

## `climate`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `temp_min` | float | `18.0` | Temperatura mínima aceptable |
| `temp_target` | float | `24.0` | Objetivo |
| `temp_max` | float | `28.0` | Máxima aceptable (dispara ventilación) |
| `temp_emergency` | float | `35.0` | Emergencia (fuerza seguridad) |
| `temp_hysteresis` | float | `2.0` | Banda para evitar oscilación |
| `hum_min` | float | `55.0` | Humedad mínima |
| `hum_target` | float | `70.0` | Objetivo |
| `hum_max` | float | `85.0` | Máxima |
| `hum_hysteresis` | float | `5.0` | Banda de histéresis |

## `irrigation`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `soil_min` | float | `35.0` | Humedad de suelo mínima (%) |
| `soil_target` | float | `55.0` | Objetivo (%) |
| `max_time_ms` | int | `900000` | Corte de seguridad del riego (15 min) |
| `flow_min` | float | `1.0` | Caudal mínimo (L/min) para detectar fallo de bomba |
| `flow_check_delay_ms` | int | `5000` | Espera antes de verificar el caudal |

## `ventilation`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `on_temp` | float | `28.0` | Enciende ventilación |
| `off_temp` | float | `25.0` | Apaga ventilación |
| `hum_max` | float | `85.0` | Humedad que fuerza ventilación |

## `roof`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `open_temp` | float | `28.0` | Abre techo |
| `close_temp` | float | `24.0` | Cierra techo |
| `close_on_rain` | bool | `true` | Cierra si llueve |
| `close_on_wind` | bool | `true` | Cierra si hay viento fuerte |
| `wind_max` | float | `30.0` | Viento máximo (km/h) |

## `lighting`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `min_lux` | float | `5000.0` | Lux mínimo para encender |
| `start_hour` | int | `6` | Hora de inicio permitida |
| `end_hour` | int | `22` | Hora de fin permitida |
| `intensity` | float | `80.0` | Intensidad PWM (%) |

## `calibration`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `soil_dry` | float[4] | `2850` | Lectura ADC en seco por zona |
| `soil_wet` | float[4] | `1450` | Lectura ADC en húmedo por zona |
| `ph4` | float | `3.02` | Tensión del patrón pH 4.00 |
| `ph7` | float | `2.51` | Tensión del patrón pH 7.00 |
| `ph10` | float | `2.01` | Tensión del patrón pH 10.00 |
| `flow_lpp` | float | `0.00222` | Litros por pulso (~450 pulsos/L) |
| `rain_mmp` | float | `0.2794` | mm por pulso del pluviómetro |
| `wind_khpp` | float | `2.4` | km/h por pulso del anemómetro |
| `tank_depth` | float | `100.0` | Profundidad del tanque (cm) |

## `network`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `ssid` | str | `""` | SSID WiFi |
| `pass` | str | `""` | Contraseña WiFi |
| `hostname` | str | `"invernadero"` | Hostname (mDNS `invernadero.local`) |
| `ap_ssid` | str | `"Invernadero-AP"` | SSID del AP de configuración |
| `ap_pass` | str | `"invernadero"` | Clave del AP |
| `mqtt_host` | str | `""` | Broker MQTT (vacío = MQTT apagado) |
| `mqtt_port` | int | `1883` | Puerto MQTT |
| `mqtt_user` / `mqtt_pass` | str | `""` | Credenciales MQTT |
| `ntp` | str | `"pool.ntp.org"` | Servidor NTP |
| `tz` | int | `-3` | Offset de zona horaria (horas) |
| `timezone` | str | `"America/Argentina/Buenos_Aires"` | Zona IANA |
| `dns1` / `dns2` | str | `8.8.8.8` / `1.1.1.1` | DNS |
| `net_interface` | enum | `wifi` | `wifi` · `ethernet` |
| `eth_cs` | int | `5` | Pin CS del W5500 |
| `eth_dhcp` | bool | `true` | DHCP o IP estática |
| `eth_ip` `eth_gateway` `eth_mask` `eth_dns` | str | `""` | IP estática Ethernet |
| `central_managed` | bool | `false` | Administrado por el servidor central |
| `central_url` | str | `""` | URL del servidor |
| `central_port` | int | `443` | Puerto |
| `central_token` | str | `""` | Token de API |
| `rs485_baud` | int | `9600` | Baudios RS485/Modbus |
| `rs485_parity` | int | `0` | `0`=NONE · `1`=EVEN · `2`=ODD |
| `rs485_stop` | int | `1` | Bits de stop |

## `weather`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `enabled` | bool | `false` | Activa la consulta externa |
| `url` | str | `""` | URL que publica JSON |
| `interval_ms` | int | `60000` | Frecuencia de consulta |
| `root` | str | `""` | Subobjeto raíz (opcional) |
| `key_temp` | str | `"temp"` | Nombre del campo de temperatura |
| `key_hum` | str | `"hum"` | Campo de humedad |
| `key_wind` | str | `"ane"` | Campo de viento |
| `key_rain` | str | `"pluv"` | Campo de lluvia |
| `key_pressure` | str | `"pres"` | Campo de presión |
| `key_light` | str | `"lux"` | Campo de luz |

## `sensors` (todos `bool`)

| Clave | Default | Sensor |
|-------|---------|--------|
| `sht31` | `true` | Temperatura/humedad interior (I²C) |
| `ds18b20` | `true` | 1-Wire |
| `soil` | `true` | Humedad de suelo (ADS1115) |
| `light` | `true` | BH1750 |
| `co2` | `false` | SCD40/SCD41 |
| `rain` | `false` | Pluviómetro |
| `wind` | `false` | Anemómetro |
| `tank` | `true` | Ultrasónico de tanque |
| `flow` | `true` | Caudalímetro |
| `ph` | `false` | pH |
| `ec` | `false` | Conductividad |
| `exterior` | `false` | AHT20 exterior |

## `actuators`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `pump` | bool | `true` | Bomba |
| `valves` | int | `4` | Nº de electroválvulas (0..8) |
| `fans` | int | `2` | Nº de ventiladores |
| `extractors` | int | `1` | Nº de extractores |
| `lights` | int | `1` | Nº de luces |
| `heater` | bool | `false` | Calefacción |
| `humidifier` | bool | `false` | Humidificador |
| `roof` | bool | `false` | Techo |
| `window` | bool | `false` | Ventana |
| `shade` | bool | `false` | Sombra |

## `zones` y `auth`

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `zones[].name` | str | `"Zona 1".."Zona 4"` | Nombres (hasta 8) |
| `auth.user` | str | `"admin"` | Usuario administrador |
| `auth.pass` | str | `""` | Si vacío, se deriva del UID en el primer arranque |

---

## Ejemplo mínimo

```json
{
  "device": { "id": "GH002", "name": "Invernadero Norte" },
  "climate": { "temp_min": 16, "temp_target": 23, "temp_max": 29 },
  "sensors": { "sht31": true, "soil": true, "co2": true },
  "actuators": { "pump": true, "valves": 6 },
  "network": { "ssid": "MiWiFi", "pass": "secreto", "mqtt_host": "192.168.1.10" }
}
```

Ver también: [Variables modificables](Variables-Modificables.md) ·
[Configuración](Configuracion.md) · [API REST](API-REST.md).
