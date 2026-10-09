---
tags:
  - sema
  - sensores
---

# Sensores

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

SEMA soporta **17 modelos de sensor** en **5 interfaces** (I²C, 1-Wire, ADC, PCNT y
UART). El catálogo es *config-driven*: cada sensor se declara en el array `sensors[]`
de la configuración y `SensorFactory` construye el driver correspondiente.

Fuentes: `src/core/sensors/SensorFactory.cpp` (catálogo), cada
`src/core/sensors/*Sensor.cpp` (canales y unidades reales),
`include/core/Measurement.hpp` (modelo canónico) e
`include/core/ConfigManager.hpp` (`struct SensorSpec`).

## 1. Las 5 interfaces

| Interfaz | Valor de `interface()` | Modelos | Driver base |
|----------|------------------------|---------|-------------|
| I²C | `"I2C"` | `BME280`, `BMP280`, `SHT40`, `SHT31`, `AHT20`, `BH1750`, `VEML6075`, `SCD30`, `SGP30`, `AS3935`, `ADS1115` | `Wire` |
| 1-Wire | `"1-Wire"` | `DS18B20` | `OneWire` + `DallasTemperature` |
| ADC interno | `"ADC"` | `ADC`, `CO`, `SOLAR` | `analogRead()` 12 bits |
| Contador de pulsos | `"GPIO"` | `PCNT` | periférico PCNT (ESP-IDF) |
| UART | `"UART"` | `PMS5003` | `Serial2` 9600 8N1 |

> ⚠️ El campo `interface()` del driver **no siempre coincide** con el valor que muestra
> la UI de la web: para los sensores analógicos la web usa `ana`/`pulse` y para el
> DS18B20 usa `1w`. Ver §3.

## 2. Catálogo completo (17 modelos)

| `model` | Interfaz | Dirección / pines | Claves de config que consume el driver | Archivo del driver |
|---------|----------|-------------------|----------------------------------------|--------------------|
| `BME280` | I²C | 0x76, si falla 0x77 | `sda`, `scl` | `Bme280Sensor.cpp` |
| `BMP280` | I²C | 0x76 (fijo) | `sda`, `scl` | `Bmp280Sensor.cpp` |
| `SHT40` | I²C | 0x44 (fijo, librería `Adafruit_SHT4x`) | `sda`, `scl` | `Sht40Sensor.cpp` |
| `SHT31` | I²C | 0x44 (fijo) | `sda`, `scl` | `Sht31Sensor.cpp` |
| `AHT20` | I²C | 0x38 (default de librería) | `sda`, `scl` | `Aht20Sensor.cpp` |
| `BH1750` | I²C | 0x23 (fijo) | `sda`, `scl` | `Bh1750Sensor.cpp` |
| `VEML6075` | I²C | 0x10 (fijo en la librería) | `sda`, `scl` | `Veml6075Sensor.cpp` |
| `SCD30` | I²C | 0x61 (default de librería) | `sda`, `scl` | `Scd30Sensor.cpp` |
| `SGP30` | I²C | 0x58 (dirección única) | `sda`, `scl` | `Sgp30Sensor.cpp` |
| `AS3935` | I²C | 0x03 | `sda`, `scl` | `As3935Sensor.cpp` |
| `ADS1115` | I²C | 0x48 (default de librería) | `sda`, `scl`, `pin` (= canal 0–3), `channel`, `unit`, `scale`, `offset` | `Ads1115Sensor.cpp` |
| `DS18B20` | 1-Wire | `pin` + `rom` (16 hex, opcional) | `pin`, `rom` | `Ds18b20Sensor.cpp` |
| `ADC` | ADC | `pin` (canal del ADC interno) | `pin`, `channel`, `unit`, `scale`, `offset` | `AdcSensor.cpp` |
| `CO` | ADC | `pin` | `pin`, `scale`, `offset` | `CoSensor.cpp` |
| `SOLAR` | ADC | `pin` | `pin`, `scale`, `offset` | `SolarSensor.cpp` |
| `PCNT` | PCNT | `pin` | `pin`, `channel`, `unit`, `scale` | `PcntSensor.cpp` |
| `PMS5003` | UART | `rx`, `tx` (Serial2, 9600 8N1) | `rx`, `tx` | `Pms5003Sensor.cpp` |

### 2.1 Estado de cada clave de `SensorSpec`

| Clave | Tipo | Default | ¿La usa algún driver? | Observación |
|-------|------|---------|-----------------------|-------------|
| `id` | string | `""` | Sí | `sensorId` de cada `Measurement` |
| `model` | string | `""` | Sí | selección en `SensorFactory::create()` |
| `enabled` | bool | `false` | Sí (en `SemaCore::applySensors()`) | si es `false` el sensor no se registra |
| `address` | uint8 | `0` | **No** | se guarda y la web lo ofrece, pero ningún driver lo lee |
| `rom` | string | `""` | Sí (solo `DS18B20`) | vacío = autodetección de todos los DS18B20 del bus |
| `sda` / `scl` | uint8 | `21` / `22` | Sí (todos los I²C) | cada driver llama `Wire.begin(sda, scl)` |
| `bus` | uint8 | `0` | **No** | pensado para el SC18IS602B; sin driver operativo |
| `uart` / `uart_port` | uint8 | `0` / `0` | **No** | pensado para el MAX14830; sin driver operativo |
| `pin` | uint8 | `0` | Sí | GPIO analógico/pulsos, pin 1-Wire, **o canal A0–A3 del ADS1115** |
| `rx` / `tx` | uint8 | `0` / `0` | Sí (solo `PMS5003`) | pines del `Serial2` |
| `channel` | string | `""` | Sí (`ADC`, `ADS1115`, `PCNT`) | pasa a `channelId` y `measurement` |
| `unit` | string | `""` | Sí (`ADC`, `ADS1115`, `PCNT`) | pasa a `unit` |
| `scale` | float | `1.0` | Sí | `value = raw · scale + offset` |
| `offset` | float | `0.0` | Sí (`ADC`, `ADS1115`, `CO`, `SOLAR`) | idem |

> **Discrepancia importante:** la clave `address` **no tiene efecto**. Las direcciones
> I²C están fijas en cada driver (§2). Si cambiás `address` en la web, el driver sigue
> hablando a su dirección hardcodeada, salvo en la medida en que el propio driver
> pruebe varias (BME280 prueba 0x76 y 0x77).

## 3. Instrumentos del catálogo web (19 entradas)

La página `/config/sensors` (`src/core/web/HttpServer.cpp`, array `TYPES`) ofrece estas
entradas; las repetidas son variantes del mismo `model` con distinto `channel`:

| Nombre en la web | `model` | `channel` por defecto | Interfaz de la UI |
|------------------|---------|-----------------------|-------------------|
| BME280 | `BME280` | — | i2c |
| BMP280 | `BMP280` | — | i2c |
| SHT40 | `SHT40` | — | i2c |
| SHT31 | `SHT31` | — | i2c |
| AHT20 | `AHT20` | — | i2c |
| BH1750 | `BH1750` | — | i2c |
| VEML6075 | `VEML6075` | — | i2c |
| SCD30 | `SCD30` | — | i2c |
| SGP30 | `SGP30` | — | i2c |
| ADS1115 | `ADS1115` | — | i2c |
| AS3935 | `AS3935` | — | i2c |
| DS18B20 | `DS18B20` | — | 1w (sección propia, multi) |
| Veleta | `ADC` | `wind_direction` | ana |
| Batería | `ADC` | `voltage` | ana |
| Anemómetro | `PCNT` | `wind_speed` | pulse |
| Pluviómetro | `PCNT` | `rain` | pulse |
| PMS5003 | `PMS5003` | — | uart |
| CO | `CO` | — | ana |
| SOLAR | `SOLAR` | — | ana |

El selector de dirección I²C de la web lista 0x48, 0x49, 0x4A, 0x4B, 0x76, 0x77,
0x44, 0x45, 0x23, 0x5C, 0x38, 0x61 y 0x58. Ese valor se guarda en `address` y, como se
explicó en §2.1, **no cambia la dirección real** que usa el driver.

## 4. Canales que publica cada modelo

Cada `Measurement` lleva `sensor_id`, `channel_id`, `measurement`, `value`, `unit`,
`quality`, `sequence` y `timestamp` (`include/core/Measurement.hpp`). En todos los
drivers `channelId == measurement`.

| `model` | `channelId` / `measurement` | `unit` | Origen del valor | `Quality` posible |
|---------|------------------------------|--------|------------------|-------------------|
| `BME280` | `temperature` | `degC` | `readTemperature()` | siempre `VALID` |
| `BME280` | `humidity` | `percent` | `readHumidity()` | siempre `VALID` |
| `BME280` | `pressure` | `hPa` | `readPressure() / 100.0` (Pa→hPa) | siempre `VALID` |
| `BMP280` | `temperature` | `degC` | `readTemperature()` | `VALID` / `COMMUNICATION_ERROR` si NaN |
| `BMP280` | `pressure` | `hPa` | `readPressure() / 100.0` | `VALID` / `COMMUNICATION_ERROR` si NaN |
| `SHT40` | `temperature`, `humidity` | `degC`, `percent` | `Adafruit_SHT4x::getEvent()` | siempre `VALID` |
| `SHT31` | `temperature`, `humidity` | `degC`, `percent` | `readTemperature()` / `readHumidity()` | `VALID` / `COMMUNICATION_ERROR` si NaN |
| `AHT20` | `temperature`, `humidity` | `degC`, `percent` | `Adafruit_AHTX0::getEvent()` | siempre `VALID` |
| `BH1750` | `light` | `lux` | `raw / 1.2` (modo continuo alta resolución 0x10) | `VALID`, `SENSOR_DISCONNECTED`, `COMMUNICATION_ERROR` |
| `VEML6075` | `uva`, `uvb` | `W/m2` | `readUVA()` / `readUVB()` | siempre `VALID` |
| `VEML6075` | `uvi` | `index` | `readUVI()` | siempre `VALID` |
| `SCD30` | `co2` | `ppm` | `scd30.CO2` | siempre `VALID` |
| `SCD30` | `temperature`, `humidity` | `degC`, `percent` | `scd30.temperature` / `relative_humidity` | siempre `VALID` |
| `SGP30` | `eco2` | `ppm` | `sgp30.eCO2` | siempre `VALID` |
| `SGP30` | `tvoc` | `ppb` | `sgp30.TVOC` | siempre `VALID` |
| `AS3935` | `distance` (measurement `lightning_distance`) | `km` | `distanceToStorm()` | siempre `VALID` |
| `ADS1115` | `channel` (lo que configures) | `unit` (lo que configures) | `readADC_SingleEnded(pin) · scale + offset` | siempre `VALID` |
| `DS18B20` | `temperature` | `degC` | `getTempCByIndex()` / `getTempC(rom)` | `VALID` / `SENSOR_DISCONNECTED` |
| `ADC` | `channel` | `unit` | `analogRead(pin) · scale + offset` | siempre `VALID` |
| `CO` | `co` | `ppm` | `analogRead(pin) · scale + offset` | siempre `VALID` |
| `SOLAR` | `solar_radiation` | `W/m2` | `analogRead(pin) · scale + offset` | siempre `VALID` |
| `PCNT` | `channel` | `unit` | `pulsos · scale` (contador se limpia en cada lectura) | siempre `VALID` |
| `PMS5003` | `pm1`, `pm25`, `pm10` | `ug/m3` | `pm10_env`, `pm25_env`, `pm100_env` | siempre `VALID` |

### 4.1 Particularidades verificadas en el código

- **`PMS5003`**: el canal se llama `pm1` pero su valor es `pm10_env` (PM1.0 en entorno
  estándar), y `pm10` toma `pm100_env` (PM10). Es una inconsistencia de nombres, no de
  escala. Si no hay trama válida (`aqi.read()` falla) devuelve 0 mediciones, sin flag
  de error.
- **`SGP30`**: si `IAQmeasure()` falla devuelve 0 mediciones (sin flag de error).
- **`AS3935`**: solo publica cuando el bit `0x08` (`LIGHTNING`) del registro de
  interrupción está seteado. Se configura `setIndoorOutdoor(OUTDOOR)`.
- **`SCD30`**: solo refresca la lectura si `dataReady()`; el sensor mide cada 2 s.
- **`DS18B20`**: sin `rom` publica **un canal por dispositivo** del bus; el primero
  conserva el `id` y los siguientes son `id_1`, `id_2`, …
- **`PCNT`**: todos los `PCNT` comparten `PCNT_UNIT_0` / `PCNT_CHANNEL_0` y limpian el
  contador en cada lectura, así que el `value` es *pulsos por intervalo de lectura*.
- **`ADS1115`**: el `pin` de configuración **es el canal** (0 = A0 … 3 = A3). La
  ganancia queda en el default de la librería (±6,144 V, 2/3×).
- **`ADC`, `CO`, `SOLAR`**: `analogReadResolution(12)`; `begin()` siempre devuelve
  verdadero (no hay detección de hardware).

## 5. Ejemplos JSON de `sensors[]`

### 5.1 I²C

```json
{
  "sensors": [
    { "id": "EXT", "model": "BME280", "enabled": true, "sda": 21, "scl": 22, "bus": 0 },
    { "id": "INT", "model": "SHT40",  "enabled": true, "sda": 21, "scl": 22 }
  ]
}
```

### 5.2 1-Wire (varios DS18B20 con ROM fija)

```json
{
  "sensors": [
    { "id": "SOIL",  "model": "DS18B20", "enabled": true, "pin": 4, "rom": "28FF64A1B2C3D4E5" },
    { "id": "AIRE",  "model": "DS18B20", "enabled": true, "pin": 4, "rom": "28FF0011223344AA" }
  ]
}
```

### 5.3 ADC (batería con divisor 11:1)

```json
{
  "sensors": [
    { "id": "BATT", "model": "ADC", "enabled": true, "pin": 34,
      "channel": "voltage", "unit": "V",
      "scale": 0.008864, "offset": 0.0 }
  ]
}
```

`scale = 3.3 × 11.0 / 4095 ≈ 0.008864` (SemaCore registra el default con
`3.3f * 11.0f / 4095.0f`).

### 5.4 PCNT (pluviómetro y anemómetro)

```json
{
  "sensors": [
    { "id": "RAIN", "model": "PCNT", "enabled": true, "pin": 27,
      "channel": "rain", "unit": "mm", "scale": 0.2794 },
    { "id": "WIND", "model": "PCNT", "enabled": true, "pin": 26,
      "channel": "wind_speed", "unit": "m/s", "scale": 1.0 }
  ]
}
```

### 5.5 UART (PMS5003)

```json
{
  "sensors": [
    { "id": "PM", "model": "PMS5003", "enabled": true, "rx": 16, "tx": 17 }
  ]
}
```

### 5.6 ADS1115 (canal externo)

```json
{
  "sensors": [
    { "id": "PIRAN", "model": "ADS1115", "enabled": true, "sda": 21, "scl": 22,
      "pin": 0, "channel": "solar_radiation", "unit": "W/m2",
      "scale": 4.0, "offset": 0.0 }
  ]
}
```

**Cómo se calcula `scale` para el ADS1115.** El driver usa la ganancia por defecto de
la librería (2/3×, rango ±6,144 V) y `readADC_SingleEnded()` devuelve el valor crudo de
16 bits con signo (`raw = V_entrada / 6.144 × 32767`). Por lo tanto
`V_entrada = raw × 0.0001875` y, si el sensor entrega 1 V por cada 1000 W/m², queda
`scale = 0.0001875 × 1000 = 0.1875`. El ejemplo de arriba usa `4.0` suponiendo otro
sensor; el valor real depende de la calibración de cada instalación (ver
[Calibración](Calibracion.md)).

## 6. Catálogo por defecto

`SemaCore::applySensors()` registra este catálogo fijo cuando
`SEMA_FIXED_HARDWARE == 1` **o** cuando `sensors[]` está vacío:

| `id` | Modelo | Pines | Canales publicados |
|------|--------|-------|--------------------|
| `EXT` | `BME280` | `SEMA_PIN_I2C_SDA/SCL` (21/22) | `temperature`, `humidity`, `pressure` |
| `INT` | `SHT40` | 21/22 | `temperature`, `humidity` |
| `SOIL` | `DS18B20` | `SEMA_PIN_ONEWIRE` (4), sin ROM | `temperature` (uno por dispositivo) |
| `LUX` | `BH1750` | 21/22 | `light` |
| `AUX` | `AHT20` | 21/22 | `temperature`, `humidity` |
| `BATT` | `ADC` | `SEMA_PIN_BATTERY_ADC` (34) | `voltage` (escala 3,3 × 11 / 4095) |

Calibración por defecto asociada: `EXT:temperature` e `INT:temperature` con
`gain = 1.0`, `offset = 0.0`, rango −40…85 °C.
Regla por defecto: `high_temp` sobre `EXT`/`temperature` con umbral 40 °C.
Pines en `include/core/BoardProfile.hpp`.

### 6.1 Cómo lo reemplaza `sensors[]`

| Condición | Comportamiento |
|-----------|----------------|
| `SEMA_FIXED_HARDWARE == 1` | Se ignora `sensors[]` por completo; siempre el catálogo por defecto. |
| `SEMA_FIXED_HARDWARE == 0` y `sensors[]` vacío | Catálogo por defecto. |
| `SEMA_FIXED_HARDWARE == 0` y `sensors[]` no vacío | **Reemplaza** el catálogo: solo se registran las entradas con `enabled: true`. |
| `model` desconocido | `SensorFactory::create()` devuelve `nullptr`; se ignora y se loguea por serie: `Sensor desconocido: <id> (modelo <model>)`. |

`sensors[]` se reemplaza **completo** en cada POST: `HttpServer::onConfigSensors()`
hace `next.sensors.clear()` antes de parsear el array, y los valores ausentes toman el
default de `SensorSpec` (`sda=21`, `scl=22`, `scale=1.0`, `offset=0.0`).
El endpoint `POST /api/v1/config/sensors` **no parsea** `bus`, `uart` ni `uart_port`
(quedan en 0 aunque el cliente los mande); `PUT /api/v1/config` sí los parsea.

## 7. Ciclo de lectura y diagnóstico

- `SensorManager::readAll()` corre cada **10 s** (`scheduler_.add("sensors.read", 10000, …)`
  en `SemaCore::setup()`, con heartbeat de watchdog a 60 s).
- Buffer por sensor: `Measurement buffer[4]` → ningún driver puede publicar más de
  4 canales por ciclo.
- Se saltean los sensores con `healthy() == false`; los que devuelven 0 mediciones no
  generan flags.
- Calibración por canal, clave `"<sensorId>:<channelId>"` (ganancia/offset/rango).
- `I2cScanner::scan()` corre una vez en el arranque sobre `SEMA_PIN_I2C_SDA/SCL`; el
  resultado se loguea por serie y se expone como `i2c_devices` en `/api/v1/diagnostics`.
- El catálogo vivo (id, model, interface, healthy) se expone en `/api/v1/sensors` junto
  con las mediciones (`sensor_id`, `channel_id`, `measurement`, `value`, `unit`,
  `quality`, `sequence`).

Direcciones conocidas por el escáner I²C (`I2cScanner::modelForAddress()`):

| Dirección | Modelo sugerido |
|-----------|-----------------|
| 0x23 | BH1750 |
| 0x38, 0x39 | AHT20 |
| 0x40 | SHT31/HTU21D |
| 0x44, 0x45 | SHT40/SHT3x |
| 0x5C | AM2320 |
| 0x61 | SCD30 |
| 0x62 | SCD40/SCD41 |
| 0x68 | MPU6050/DS3231 |
| 0x76, 0x77 | BME280/BMP280 |

> ❌ El escáner **no** incluye AS3935 (0x03), BH1750 alterna (0x5C se atribuye a
> AM2320), ADS1115 (0x48–0x4B), SGP30 (0x58) ni VEML6075 (0x10).

## 8. Modelos no implementados

| Modelo | Estado | Nota |
|--------|--------|------|
| `AHT10`, `AHT21`, `AHT30` | ❌ No implementado | Solo `AHT20`; el resto figura como objetivo en README §9/§10. |
| `DHT22`, `SHT3x`, `SHT4x` genéricos | ❌ No implementado | Solo los modelos exactos `SHT31` / `SHT40`. |
| `OPT3001`, `VEML7700`, `TSL2591`, `LTR390` | ❌ No implementado | README §13/§14. |
| `CCS811`, `SCD40`, `SCD41` | ❌ No implementado | README §15. |
| `PMS7003`, `PMSA003`, `SPS30` | ❌ No implementado | README §16. |
| `MQ-7`, `MICS-5524`, `ZE07-CO`, 3-ULPSM-CO | ❌ No implementado como driver propio | Solo el genérico `CO` por ADC. |
| Sensores Modbus / RS485 | ⚠️ Parcial | Vía `ModbusManager` (no como `model` del catálogo). |
| MCP23017 / 74HC165 / MAX14830 / SC18IS602B | ⚠️ Configuración, no driver de sensor | Ver [Expansores de entrada/salida](Expansores-de-entrada-salida.md). |

---

## Ver también

- [Hardware y conexiones](Hardware-y-Conexiones.md) · [Guía de pines](Guia-de-pines.md) ·
  [Actuadores y salidas](Actuadores-y-Salidas.md) · [Materiales](Materiales.md)
- [Buses y periféricos](Buses-y-perifericos.md) · [Expansores de entrada/salida](Expansores-de-entrada-salida.md)
- [Calibración](Calibracion.md) · [Magnitudes derivadas](Magnitudes-derivadas.md) ·
  [Referencia de configuración](Referencia-configuracion.md)
