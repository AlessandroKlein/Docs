---
tags:
  - invernadero
  - configuracion
---

# Variables modificables (configuración JSON)

> **Tipo:** Configuración | **Estado:** Estable | **Fecha:** 2026-10-02

Estructura del JSON de configuración, editable por `PUT /api/v1/config` o la web.

## 1. Dispositivo e identidad

| Campo | Tipo | Default |
|-------|------|---------|
| `device.id` | string | `GH001` |
| `device.name` | string | `Invernadero` |
| `device.type` | int (0 interior, 1 exterior, 2 mixto) | `1` |
| `device.greenhouse_id` | string | `GREENHOUSE-001` |
| `config_version` | int | autoincrementado |
| `schema_version` | int | `2` (migraciones) |
| `configuration_source` | `LOCAL`/`CENTRAL` | `LOCAL` |
| `simulation` | bool | `false` |
| `update_channel` | `stable`/`beta`/`development` | `stable` |

## 2. Funciones (`features`)

`climate`, `irrigation`, `lighting`, `co2`, `heating`, `humidification`, `roof`,
`windows`, `shade` (bool). Default: `climate` y `irrigation` en `true`.

## 3. Clima (`climate`)

| Campo | Default |
|-------|---------|
| `temp_min` / `temp_target` / `temp_max` | 18 / 24 / 28 |
| `temp_emergency` | 35 |
| `temp_hysteresis` | 2 |
| `hum_min` / `hum_target` / `hum_max` | 55 / 70 / 85 |
| `hum_hysteresis` | 5 |

## 4. Riego (`irrigation`)

| Campo | Default |
|-------|---------|
| `soil_min` / `soil_target` | 35 / 55 |
| `max_time_ms` | 900000 (15 min) |
| `flow_min` | 1.0 (L/min) |
| `flow_check_delay_ms` | 5000 |

## 5. Ventilación / techo / iluminación

| Bloque | Campos |
|--------|--------|
| `ventilation` | `on_temp` 28, `off_temp` 25, `hum_max` 85 |
| `roof` | `open_temp` 28, `close_temp` 24, `close_on_rain` true, `close_on_wind` true, `wind_max` 30 |
| `lighting` | `min_lux` 5000, `start_hour` 6, `end_hour` 22, `intensity` 80 |

## 6. Calibración (`calibration`)

| Campo | Descripción |
|-------|-------------|
| `soil_dry[]` / `soil_wet[]` | ADC seco/húmedo por zona |
| `ph4` / `ph7` / `ph10` | tensión de patrones pH |
| `flow_lpp` | litros por pulso |
| `rain_mmp` | mm por pulso |
| `wind_khpp` | km/h por pulso |
| `tank_depth` | profundidad del tanque (cm) |

## 7. Red (`network`)

`ssid`, `pass`, `hostname`, `ap_ssid`, `ap_pass`, `mqtt_host`, `mqtt_port` (1883),
`mqtt_user`, `mqtt_pass`, `ntp`, `tz` (-3), `timezone`, `dns1` (8.8.8.8),
`dns2` (1.1.1.1), `central_managed`, `central_url`, `central_port`, `central_token`,
`rs485_baud` (9600), `rs485_parity` (0), `rs485_stop` (1).

### Interfaz de red (WiFi o Ethernet)

| Campo | Default | Descripción |
|-------|---------|-------------|
| `net_interface` | `wifi` | `wifi` \| `ethernet` (W5500) |
| `eth_cs` | 5 | pin CS del W5500 |
| `eth_dhcp` | true | DHCP o IP estática |
| `eth_ip` / `eth_gateway` / `eth_mask` / `eth_dns` | "" | IP estática |

## 8. Sensores habilitados (`sensors`)

`sht31`, `ds18b20`, `soil`, `light`, `co2`, `rain`, `wind`, `tank`, `flow`, `ph`,
`ec`, `exterior` (bool). Default: `sht31/ds18b20/soil/light/tank/flow=true`.

## 9. Actuadores (`actuators`)

`pump` (bool), `valves` (4), `fans` (2), `extractors` (1), `lights` (1),
`heater`/`humidifier`/`roof`/`window`/`shade` (bool, `false`).

## 10. Zonas (`zones`)

Array de nombres: `["Zona 1","Zona 2","Zona 3","Zona 4", ...]` (máx. 8).
