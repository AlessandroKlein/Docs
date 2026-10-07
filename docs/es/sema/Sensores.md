---
tags:
  - sema
  - sensores
---

# Sensores

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.68.0

Catálogo de **15 tipos de sensores** soportados por SEMA, en **5 interfaces**.

## Resumen

| Interfaz | Modelos | Pines por defecto |
|----------|---------|-------------------|
| I²C | BME280, BMP280, SHT40, SHT31, AHT20, BH1750, VEML6075, SCD30, SGP30, AS3935, ADS1115 | SDA 21, SCL 22 |
| 1-Wire | DS18B20 | GPIO 4 |
| ADC | ADC (genérico) | GPIO 34 |
| PCNT | PCNT (pulsos) | configurable |
| UART | PMS5003 | RX/TX (Serial2) |

---

## I²C

| Modelo | Dirección | Magnitudes | Unidades |
|--------|-----------|------------|----------|
| `BME280` | 0x76 / 0x77 | temperature, humidity, pressure | degC, percent, hPa |
| `BMP280` | 0x76 / 0x77 | temperature, pressure | degC, hPa |
| `SHT40` | 0x44 | temperature, humidity | degC, percent |
| `SHT31` | 0x44 / 0x45 | temperature, humidity | degC, percent |
| `AHT20` | 0x38 | temperature, humidity | degC, percent |
| `BH1750` | 0x23 / 0x5C | lux | lx |
| `VEML6075` | 0x10 | uv_index, uva, uvb | — |
| `SCD30` | 0x61 | co2, temperature, humidity | ppm, degC, percent |
| `SGP30` | 0x58 | eco2, tvoc | ppm, ppb |
| `AS3935` | 0x03 | lightning_distance | km |
| `ADS1115` | 0x48 | (canal analógico) | configurable |
| `CO` | — (ADC) | co | ppm |
| `SOLAR` | — (ADC) | solar_radiation | W/m2 |

### Configuración I²C

```json
{ "id": "EXT", "model": "BME280", "sda": 21, "scl": 22 }
```

> Los sensores I²C comparten el bus (SDA/SCL) y se identifican por su dirección.
> La detección I²C (`I2cScanner`) sugiere modelos por dirección en `/api/v1/diagnostics`.

## 1-Wire (DS18B20)

- Bus 1-Wire en GPIO 4 (pull-up 4,7 kΩ a 3,3 V).
- **Multi-dispositivo**: se leen todos los DS18B20 del bus y se nombran
  `SOIL`, `SOIL_1`, `SOIL_2`, ….

```json
{ "id": "SOIL", "model": "DS18B20", "pin": 4 }
```

## ADC (genérico)

- `pin` = GPIO analógico (por defecto 34).
- `channel`/`unit`/`scale`/`offset` definen la magnitud.
- Ejemplo batería (divisor 11:1 para 12 V): `scale = 3.3 * 11.0 / 4095.0`.

```json
{ "id": "BATT", "model": "ADC", "pin": 34, "channel": "voltage", "unit": "V",
  "scale": 0.00887, "offset": 0.0 }
```

## PCNT (pulsos)

- `pin` = GPIO con contador de pulsos (lluvia, viento).
- `channel`/`unit`/`scale` definen la magnitud.

```json
{ "id": "RAIN", "model": "PCNT", "pin": 27, "channel": "rain", "unit": "mm", "scale": 0.2794 }
```

## UART (PMS5003)

- Partículas PM1.0 / PM2.5 / PM10 por UART (Serial2).
- `rx`/`tx` = pines RX/TX.

```json
{ "id": "PM", "model": "PMS5003", "rx": 16, "tx": 17 }
```

## ADS1115 (ADC externo)

- `pin` = canal (0–3); `channel`/`unit`/`scale`/`offset` definen la magnitud.

```json
{ "id": "SOIL_MOIST", "model": "ADS1115", "sda": 21, "scl": 22, "pin": 0,
  "channel": "soil_moisture", "unit": "percent", "scale": 0.003, "offset": 0.0 }
```

## AS3935 (rayos)

- Reporta `lightning_distance` (km) cuando detecta un rayo.
- Configurado en modo `OUTDOOR` por defecto.

```json
{ "id": "STORM", "model": "AS3935", "sda": 21, "scl": 22 }
```

---

## Catálogo por defecto

Si `sensors[]` está vacío (y `SEMA_FIXED_HARDWARE == 0`), se registra este
catálogo fijo:

| Id | Modelo | Magnitud |
|----|--------|----------|
| `EXT` | BME280 | temperatura/humedad/presión exterior |
| `INT` | SHT40 | temperatura/humedad interior |
| `SOIL` | DS18B20 | temperatura de suelo |
| `LUX` | BH1750 | iluminancia |
| `AUX` | AHT20 | temperatura/humedad auxiliar |
| `BATT` | ADC | voltaje de batería |
