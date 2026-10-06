---
tags:
  - sema
  - referencia
---

# Enumeraciones y tipos

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.23.0

Enumeraciones y tipos centrales de SEMA, tal como se serializan en la API, eventos
y configuración.

---

## `EventType` (tipos de evento)

Fuente: `include/core/EventBus.hpp`.

| Valor | Nombre en API | Descripción |
|-------|---------------|-------------|
| `Sensor` | `sensor` | Lectura de un sensor |
| `Rain` | `rain` | Evento de lluvia |
| `Lightning` | `lightning` | Rayo detectado (AS3935) |
| `Battery` | `battery` | Estado de batería |
| `Network` | `network` | Cambio de red |
| `Alarm` | `alarm` | Alarma disparada por regla |
| `System` | `system` | Evento del sistema (arranque, etc.) |
| `Wake` | `wake` | Despertar de deep sleep |
| `Sleep` | `sleep` | Entrada en deep sleep |

## `Severity` (severidad de eventos)

Fuente: `include/core/EventBus.hpp`.

| Valor | Nombre en API |
|-------|---------------|
| `Debug` | `DEBUG` |
| `Info` | `INFO` |
| `Notice` | `NOTICE` |
| `Warning` | `WARNING` |
| `Error` | `ERROR` |
| `Critical` | `CRITICAL` |

## `Quality` (calidad de medición)

Fuente: `include/core/Measurement.hpp`.

| Valor | Nombre en API | Descripción |
|-------|---------------|-------------|
| `Valid` | `VALID` | Lectura válida |
| `Invalid` | `INVALID` | Lectura inválida |
| `Stale` | `STALE` | Dato viejo |
| `Timeout` | `TIMEOUT` | Timeout de lectura |
| `OutOfRange` | `OUT_OF_RANGE` | Fuera de rango |
| `CalibrationError` | `CALIBRATION_ERROR` | Error de calibración |
| `CommunicationError` | `COMMUNICATION_ERROR` | Error de comunicación |
| `SensorDisconnected` | `SENSOR_DISCONNECTED` | Sensor desconectado |

## `RuleOp` (operadores de reglas)

Fuente: `include/core/alarms/Rule.hpp`. En config se usan en minúscula.

| Valor | Config | Significado |
|-------|--------|-------------|
| `Gt` | `gt` | `>` |
| `Lt` | `lt` | `<` |
| `Ge` | `ge` | `>=` |
| `Le` | `le` | `<=` |

## `Capability` (capacidades de la plataforma)

Fuente: `include/core/Capability.hpp`.

| Valor | Nombre en API |
|-------|---------------|
| `WiFi` | `wifi` |
| `Bluetooth` | `bluetooth` |
| `Ethernet` | `ethernet` |
| `Adc` | `adc` |
| `Dac` | `dac` |
| `Pcnt` | `pcnt` |
| `LedcPwm` | `ledc_pwm` |
| `I2c` | `i2c` |
| `Spi` | `spi` |
| `Uart` | `uart` |
| `Can` | `can` |
| `Psram` | `psram` |
| `RtcGpio` | `rtc_gpio` |
| `DeepSleep` | `deep_sleep` |
| `DualCore` | `dual_core` |
| `Ieee802154` | `ieee802154` |

## Modelos de sensor (`sensors[].model`)

| Modelo | Interfaz | Magnitudes |
|--------|----------|------------|
| `BME280` | I²C | temperature, humidity, pressure |
| `BMP280` | I²C | temperature, pressure |
| `SHT40` | I²C | temperature, humidity |
| `SHT31` | I²C | temperature, humidity |
| `AHT20` | I²C | temperature, humidity |
| `BH1750` | I²C | lux |
| `VEML6075` | I²C | uv_index, uva, uvb |
| `SCD30` | I²C | co2, temperature, humidity |
| `SGP30` | I²C | eco2, tvoc |
| `AS3935` | I²C | lightning_distance |
| `ADS1115` | I²C | (canal analógico configurable) |
| `CO` | ADC | co |
| `SOLAR` | ADC | solar_radiation |
| `DS18B20` | 1-Wire | temperature (multi-dispositivo) |
| `ADC` | ADC | (magnitud configurable) |
| `PCNT` | PCNT | conteo de pulsos (lluvia/viento) |
| `PMS5003` | UART | pm1_0, pm2_5, pm10_0 |

## Modos de red (`network.mode`)

| Valor | Descripción |
|-------|-------------|
| `STA` | Cliente WiFi (se conecta a un AP) |
| `AP` | Punto de acceso (SEMA crea su propia red) |

## Modos de GPIO (`gpio[].mode`)

| Valor | Descripción |
|-------|-------------|
| `output` | Salida digital |
| `input` | Entrada digital |
| `input_pullup` | Entrada con pull-up |
| `input_pulldown` | Entrada con pull-down |

## Backends de almacenamiento (`storage.backend`)

| Valor | Descripción |
|-------|-------------|
| `littlefs` | Sistema de archivos LittleFS (flash) |
| `flash` | Flash directa |
| `sd` | Tarjeta SD |

## Modelo canónico `Measurement`

Fuente: `include/core/Measurement.hpp`.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `stationId` | str | Id de la estación |
| `sensorId` | str | Id del sensor |
| `channelId` | str | Canal lógico |
| `measurement` | str | Magnitud (`"temperature"`, `"humidity"`, …) |
| `value` | float | Valor numérico |
| `unit` | str | Unidad canónica (`"degC"`, `"percent"`, `"hPa"`, …) |
| `quality` | enum | Calidad (`Quality`) |
| `sequence` | uint32 | Secuencia monotónica por sensor |
| `timestamp` | uint32 | Epoch seconds UTC |

## Evento tipado `Event`

Fuente: `include/core/EventBus.hpp`.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | uint32 | Secuencia |
| `timestampMs` | uint32 | `millis()` monotónico |
| `source` | str | Id del sensor/módulo |
| `type` | enum | `EventType` |
| `severity` | enum | `Severity` |
| `value` | int32 | Payload numérico |
| `correlationId` | str | Correlación (regla) |
| `target` | str | Destino opcional |
