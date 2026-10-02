---
tags:
  - invernadero
  - referencia
---

# Enumeraciones y tipos

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

Todos los enums y estructuras del firmware, con **todos** sus valores y su código
numérico. Fuente: `include/core/Types.hpp` y `include/core/PlatformTypes.hpp`.

## 1. Estado de lectura — `SensorStatus`

Nunca se asume que una lectura es válida.

| Valor | Código | Significado |
|-------|:------:|-------------|
| `UNKNOWN` | 0 | Aún no leído |
| `OK` | 1 | Lectura válida (calidad GOOD) |
| `WARNING` | 2 | Lectura dudosa |
| `ERROR` | 3 | Hardware presente pero falla |
| `DISCONNECTED` | 4 | Hardware no detectado |
| `OUT_OF_RANGE` | 5 | Fuera de los límites configurados |
| `INVALID` | 6 | Valor imposible |
| `TIMEOUT` | 7 | Sin respuesta a tiempo |
| `CALIBRATION` | 8 | Calibración pendiente |

## 2. Tipo de sensor — `SensorType`

| Valor | Código | Hardware típico |
|-------|:------:|-----------------|
| `NONE` | 0 | — |
| `TEMP_SHT31` | 1 | SHT31 |
| `HUM_SHT31` | 2 | SHT31 |
| `TEMP_AHT20` | 3 | AHT20 |
| `HUM_AHT20` | 4 | AHT20 |
| `TEMP_DS18B20` | 5 | DS18B20 (1-Wire) |
| `SOIL_MOISTURE` | 6 | Capacitivo + ADS1115 |
| `LIGHT_LUX` | 7 | BH1750 |
| `CO2` | 8 | SCD40/SCD41 |
| `FLOW_RATE` | 9 | Caudalímetro |
| `TANK_LEVEL` | 10 | Ultrasónico |
| `RAIN_ACCUM` | 11 | Pluviómetro |
| `WIND_SPEED` | 12 | Anemómetro |
| `PH` | 13 | Analógico/Modbus |
| `EC` | 14 | Analógico/Modbus |
| `TEMP_EXTERIOR` | 15 | AHT20 |
| `HUM_EXTERIOR` | 16 | AHT20 |

## 3. Rol de actuador — `ActuatorRole`

| Valor | Código | Valor | Código |
|-------|:------:|-------|:------:|
| `GENERIC` | 0 | `WINDOW_OPEN` | 8 |
| `PUMP` | 1 | `WINDOW_CLOSE` | 9 |
| `VALVE` | 2 | `ROOF_OPEN` | 10 |
| `FAN` | 3 | `ROOF_CLOSE` | 11 |
| `EXTRACTOR` | 4 | `SHADE_OPEN` | 12 |
| `HEATER` | 5 | `SHADE_CLOSE` | 13 |
| `HUMIDIFIER` | 6 | `ALARM` | 14 |
| `LIGHT` | 7 | | |

## 4. Modos y estados

| Enum | Valores | Uso |
|------|---------|-----|
| `OutputKind` | `DIGITAL`=0, `PWM`=1 | Tipo eléctrico de salida |
| `OpMode` | `AUTO`=0, `MANUAL`=1, `PROGRAMMED`=2, `SAFETY`=3 | Modo de operación |
| `GreenhouseType` | `INDOOR`=0, `OUTDOOR`=1, `MIXED`=2 | Tipo de instalación |
| `UserRole` | `ADMIN`=0, `OPERATOR`=1, `VIEWER`=2 | Rol de usuario (servidor) |
| `ConfigSource` | `LOCAL`=0, `CENTRAL`=1 | Dueño de la config |
| `UpdateChannel` | `STABLE`=0, `BETA`=1, `DEVELOPMENT`=2 | Canal OTA |
| `NetInterface` | `WIFI`=0, `ETHERNET`=1 | Interfaz de red activa |

## 5. Estado del dispositivo — `DeviceState`

| Valor | Código | Significado |
|-------|:------:|-------------|
| `BOOTING` | 0 | Arrancando |
| `INITIALIZING` | 1 | Inicializando periféricos |
| `SELF_TEST` | 2 | Autodiagnóstico |
| `NETWORK` | 3 | Conectando red |
| `SYNC` | 4 | Sincronizando config/hora |
| `RUN` | 5 | Operación normal |
| `DEGRADED` | 6 | Funciona con fallos parciales |
| `ERROR` | 7 | Error |
| `MAINTENANCE` | 8 | Mantenimiento |
| `UPDATING` | 9 | Actualizando firmware |
| `RECOVERY` | 10 | Recuperación |

## 6. Causa de reinicio — `ResetCause`

`POWER_ON`=0 · `SOFTWARE_RESET`=1 · `WATCHDOG`=2 · `BROWNOUT`=3 · `PANIC`=4 ·
`OTA`=5 · `FACTORY_RESET`=6 · `UNKNOWN`=7

## 7. Buses — `BusType` y `BusState`

| `BusType` | Código | | `BusState` | Código |
|-----------|:------:|-|------------|:------:|
| `NONE` | 0 | | `UNREGISTERED` | 0 |
| `I2C` | 1 | | `READY` | 1 |
| `SPI` | 2 | | `BUSY` | 2 |
| `UART` | 3 | | `ERROR` | 3 |
| `RS485` | 4 | | | |
| `ONEWIRE` | 5 | | | |
| `GPIO` | 6 | | | |
| `CAN` | 7 | | | |

## 8. Módulos y hardware

| `ModuleState` | Código | | `HardwareKind` | Código |
|---------------|:------:|-|----------------|:------:|
| `NOT_INSTALLED` | 0 | | `NONE` | 0 |
| `INSTALLED` | 1 | | `HC595` | 1 |
| `MODULE_ENABLED` | 2 | | `HC165` | 2 |
| `MODULE_DISABLED` | 3 | | `MCP23017` | 3 |
| `MODULE_ERROR` | 4 | | `MCP23S17` | 4 |
| | | | `ADC` | 5 |
| | | | `SD` | 6 |
| | | | `W5500` | 7 |
| | | | `SENSOR` | 8 |
| | | | `ACTUATOR` | 9 |

> Los enumeradores de `ModuleState` van prefijados con `MODULE_` porque
> `ENABLED`/`DISABLED` colisionan con macros del core ESP32 (`esp32-hal-gpio.h`).

## 9. Almacenamiento y ADC

| `StorageBackend` | Código | | `AdcKind` | Código |
|------------------|:------:|-|-----------|:------:|
| `NONE` | 0 | | `NONE` | 0 |
| `NVS` | 1 | | `ADS1115` | 1 |
| `SPIFFS` | 2 | | `MCP3008` | 2 |
| `LITTLEFS` | 3 | | `MCP3208` | 3 |
| `SD` | 4 | | `ADS8688` | 4 |

## 10. Modbus y provisioning

| `ModbusDataType` | Código | | `ProvisioningState` | Código |
|------------------|:------:|-|---------------------|:------:|
| `UINT16` | 0 | | `DISCOVERED` | 0 |
| `INT16` | 1 | | `PENDING` | 1 |
| `UINT32` | 2 | | `COMMISSIONED` | 2 |
| `INT32` | 3 | | `ACTIVE` | 3 |
| `FLOAT32` | 4 | | `BLOCKED` | 4 |
| | | | `REVOKED` | 5 |

## 11. Reglas — `RuleVariable` y `RuleOp`

`RuleVariable`: `TEMPERATURE`=0 · `HUMIDITY`=1 · `SOIL`=2 · `LIGHT`=3 · `CO2`=4 ·
`TANK`=5 · `FLOW`=6 · `PH`=7 · `EC`=8 · `VPD`=9 · `DEWPOINT`=10 ·
`EXTERIOR_TEMP`=11 · `EXTERIOR_HUM`=12 · `WIND_SPEED`=13 · `RAIN`=14

`RuleOp`: `GT`=0 · `LT`=1 · `GTE`=2 · `LTE`=3 · `EQ`=4

## 12. Capas de configuración — `ConfigLayer`

| Valor | Código | Ámbito |
|-------|:------:|--------|
| `FACTORY` | 0 | Valores de fábrica (no editables) |
| `HARDWARE` | 1 | Buses, pines, direcciones (solo local) |
| `DRIVERS` | 2 | Configuración de drivers |
| `INSTALLATION` | 3 | Qué hardware hay en esta instalación |
| `AUTOMATION` | 4 | Reglas, umbrales, horarios, zonas |
| `USER` | 5 | Preferencias del operador |

## 13. Estructuras clave

| Estructura | Archivo | Campos |
|-----------|---------|--------|
| `SystemConfig` | `Types.hpp` | ~120 campos (ver [Referencia de configuración](Referencia-configuracion.md)) |
| `DeviceInfo` | `Types.hpp` | UID, perfil, versiones, chip, MAC, capacidades |
| `SensorValue` | `Types.hpp` | type, name, zone, enabled, value, raw, status, unit |
| `ActuatorState` | `Types.hpp` | role, name, index, zone, enabled, kind, mode, output, fault |
| `SensorEntry` | `PlatformTypes.hpp` | id, name, driver, type, bus, address, zone, enabled |
| `ActuatorEntry` | `PlatformTypes.hpp` | id, name, role, kind, bus, channel, zone, enabled |
| `HardwareNode` | `PlatformTypes.hpp` | id, kind, bus, busIndex, address, enabled, owner |
| `ModbusProfile` | `PlatformTypes.hpp` | id, vendor, product, slaveId, register, tipo, escala |
| `SensorInstance` | `PlatformTypes.hpp` | id, name, profileId, slaveId, zone, state |
| `AutomationRule` | `Types.hpp` | name, enabled, variable, op, threshold, action |
| `ModuleDescriptor` | `PlatformTypes.hpp` | id, version, dependencies, capabilities, state |
| `ModbusStats` | `Types.hpp` | tx, rx, crcErrors, timeouts, retries, lastError |

Ver también: [Referencia de configuración](Referencia-configuracion.md) ·
[Referencia de código](Referencia-de-codigo.md).
