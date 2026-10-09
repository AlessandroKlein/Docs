---
tags:
  - sema
  - compatibilidad
  - hardware
---

# Compatibilidad

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Matriz real de compatibilidad: entornos de compilación soportados, capacidades de cada
placa, periféricos disponibles, sensores por interfaz y qué necesita el navegador para el
dashboard. Verificado contra `platformio.ini`, `include/hw/HwProfile.hpp`,
`include/core/BoardProfile.hpp`, `include/core/Capability.hpp`, `src/core/CapabilityManager.cpp`,
`src/core/SemaCore.cpp`, `src/core/sensors/SensorFactory.cpp`, los `partitions_*.csv` y
`src/core/web/HttpServer.cpp`.

## 1. Cómo se decide la compatibilidad

1. **Compilación**: `HwProfile.hpp:28-30` aborta con `#error` si no se define
   `BOARD_ESP32_WROOM`, `BOARD_ESP32_WROOM32U` o `BOARD_ESP32_S3`. No hay rama para S2, C3,
   C5, C6 ni H2.
2. **Identidad**: de la macro `BOARD_*` salen `SEMA_BOARD_ID`, `SEMA_FLASH_MB` y
   `SEMA_NATIVE_ETH`, que `GET /api/v1/system` publica como `board`, `flash_mb` y `native_eth`.
3. **Features**: `SEMA_USE_ETHERNET`, `SEMA_USE_LORA`, `SEMA_USE_MODBUS`, `SEMA_USE_CAN`,
   `SEMA_USE_ZIGBEE` valen `1` por defecto (`HwProfile.hpp:50-65`) y los cuatro entornos los
   activan explícitamente. `SEMA_USE_SHIFT` vale `0` por defecto y **ningún entorno lo activa**.
4. **Capacidades en runtime**: `SemaCore.cpp:66-78` declara `WiFi`, `Bluetooth`, `Adc`,
   `Dac`, `Pcnt`, `LedcPwm`, `I2c`, `Spi`, `Uart`, `Can`, `RtcGpio`, `DeepSleep` y `DualCore`
   **siempre**, con independencia de la placa. `GET /api/v1/capabilities` devuelve esa lista.
   `Ethernet`, `Psram` e `Ieee802154` existen en el enum (`Capability.hpp:13-32`) pero
   **nunca se declaran**.

## 2. Matriz placa × capacidad

| Capacidad | esp32doit-devkit-v1 (WROOM 4 MB) | esp32-s3-devkitc-1 (S3 8 MB) | esp32-wroom-32u (16 MB, PCB futura) | demo |
|-----------|:--------------------------------:|:----------------------------:|:-----------------------------------:|:----:|
| Wi-Fi 2,4 GHz | ✅ `WiFi.h`, STA + AP, escaneo y mDNS | ✅ igual | ✅ igual | ✅ (valores ficticios) |
| Bluetooth/BLE | ⚠️ `Capability::Bluetooth` declarada, **sin código BLE ni BT Classic** en `src/` | ⚠️ igual (el S3 solo tiene BLE 5.0, sin Classic, datasheet) | ⚠️ igual | ⚠️ igual |
| Ethernet nativa (LAN8720A RMII) | ✅ `SEMA_NATIVE_ETH 1` | ❌ no tiene EMAC (`SEMA_NATIVE_ETH 0`) | ✅ `SEMA_NATIVE_ETH 1` | ✅ |
| Ethernet SPI (W5500) | ❌ no se usa | ✅ host SPI3 (18/19/21) + CS 5 | ❌ no se usa | ❌ |
| 802.15.4 / Zigbee | ⚠️ solo con el módulo externo CC2652P2 por UART | ⚠️ igual | ⚠️ igual | ⚠️ igual |
| CAN / TWAI | ✅ 1 controlador TWAI | ✅ 1 controlador TWAI | ✅ | ✅ |
| LoRa (SX1262 por SPI) | ✅ CS 10, bus 14/12/15 | ✅ CS 10, bus 12/13/11 | ✅ CS 10, bus 14/12/15 | ✅ |
| ADC interno | ✅ ADC1 (32-39) usable con Wi-Fi; ADC2 no | ✅ ADC1 (1-10) usable con Wi-Fi; ADC2 no | ✅ ADC1 | ✅ |
| DAC | ✅ DAC1 (25), DAC2 (26); sin API en el firmware | ❌ el S3 no tiene DAC, pero `Capability::Dac` se declara igual | ✅ | ✅ |
| PCNT | ✅ 1 unidad usada (`PCNT_UNIT_0`) | ✅ igual | ✅ igual | ✅ |
| PSRAM | ❌ no se habilita ni se declara (`Capability::Psram` nunca se marca) | ❌ el entorno es la variante **N8** (sin PSRAM) | ❌ no se habilita | ❌ |
| Doble núcleo | ✅ `Capability::DualCore` declarada | ✅ declarada | ✅ declarada | ✅ |
| RTC GPIO / deep sleep | ✅ `ext0` y timer (`PowerManager.cpp`) | ✅ | ✅ | ✅ |
| microSD (SPI) | ✅ CS 4 (default) | ✅ CS 4 | ✅ CS 4 | ✅ |
| Registros de desplazamiento | ❌ `SEMA_USE_SHIFT=0` compila fuera el driver | ❌ igual | ❌ igual | ❌ igual |
| Pines de bus | configurables por web (`SEMA_PINS_FROM_FILE=0`) | configurables por web | **fijos** (`SEMA_PINS_FROM_FILE=1`) | configurables por web |

## 3. Matriz placa × periférico y bus

| Periférico / bus | WROOM 4 MB | S3 8 MB | WROOM-32U 16 MB | Claves JSON | Requiere |
|------------------|:----------:|:-------:|:---------------:|-------------|----------|
| I²C (compartido) | ✅ 21/22 | ✅ 21/22 | ✅ 21/22 (no fijados por `applyHwProfile`) | `i2c_sda`, `i2c_scl`, `sensors[].sda/scl` | Pull-ups de 4,7 kΩ |
| SPI (Arduino) | ✅ 14/12/15 | ✅ 12/13/11 | ✅ 14/12/15 | fijo en `HwProfile.hpp` | — |
| UART consola | ✅ 115200 en GPIO1/3 | ✅ | ✅ | fijo | — |
| UART Zigbee (`Serial1`) | ✅ | ✅ | ✅ | `zigbee.rx/tx/baud` | Módulo CC2652P2 con firmware ZNP |
| UART Modbus/PMS5003 (`Serial2`) | ✅ (uno por vez) | ✅ | ✅ | `modbus.rx/tx/baud` · `sensors[].rx/tx` | Transceiver RS485 |
| 1-Wire | ✅ | ✅ | ✅ | `sensors[model=DS18B20].pin` | Pull-up de 4,7 kΩ |
| RS485 / Modbus RTU | ✅ maestro de 1 esclavo | ✅ | ✅ | `modbus{...}` | TD501D485H (default) o SN65HVD75DR |
| CAN / TWAI | ✅ 125k-1M | ✅ | ✅ | `can{enabled,tx,rx,speed}` | Transceiver CAN 3,3 V + 120 Ω |
| Ethernet LAN8720A | ✅ RMII, reloj en GPIO0 | ❌ | ✅ RMII | `ethernet{...}` | PHY + magnetics |
| Ethernet W5500 | ❌ | ✅ SPI3 | ❌ | `ethernet{enabled,cs}` | Módulo W5500 |
| LoRa SX1262 | ✅ | ✅ (⚠️ mover `dio1`/`busy` fuera de 26-32) | ✅ (⚠️ 26/27 chocan con RMII) | `lora{...}` | Módulo SX1262 + antena 50 Ω |
| Zigbee CC2652P2 | ✅ | ⚠️ `rx/tx` 18/19 chocan con el SPI del W5500 | ⚠️ 18/19 chocan con RMII | `zigbee{...}` | Módulo ZNP + red existente |
| microSD | ✅ | ✅ | ✅ | `storage.sd_enabled`, `storage.sd_cs` | Socket 3,3 V |
| MCP23017 (I²C) | ✅ 16 E/S | ✅ | ✅ | `gpio[].expander_addr` | Pull-ups I²C |
| MCP23S17 (SPI) | ❌ sin driver | ❌ | ❌ | `mcp23s17_cs`, `mcp23s17_pins` | — |
| ADS1115 (I²C) | ✅ 4 canales | ✅ | ✅ | `sensors[model=ADS1115]` | Dirección fija 0x48 |
| 74HC595 / 74HC165 | ❌ compilado fuera | ❌ | ❌ | `shift_registers[]` | `-D SEMA_USE_SHIFT=1` |
| MAX14830 / SC18IS602B | ❌ sin driver | ❌ | ❌ | `spi_expanders[]` | — |

## 4. Flash y particiones

`board_build.filesystem = littlefs` en `[env:base]`, así que la partición `spiffs` de las
tablas se monta como **LittleFS**.

| Entorno | Flash | Tabla | `app0` / `app1` | Partición LittleFS | `nvs` | `otadata` |
|---------|-------|-------|-----------------|--------------------|-------|-----------|
| esp32doit-devkit-v1 | 4 MB | `partitions_4mb.csv` | 0x1C0000 cada una (1792 KB) | 0x70000 (448 KB) desde 0x390000 | 0x5000 (20 KB) | 0x2000 (8 KB) |
| esp32-s3-devkitc-1 | 8 MB | `partitions_8mb.csv` | 0x300000 cada una (3072 KB) | 0x1F0000 (1984 KB) desde 0x610000 | 0x5000 | 0x2000 |
| esp32-wroom-32u | 16 MB | `partitions_16mb.csv` | 0x300000 cada una (3072 KB) | 0x9F0000 (10 176 KB) desde 0x610000 | 0x5000 | 0x2000 |
| demo | 4 MB | `partitions_4mb.csv` | 0x1C0000 cada una | 0x70000 (448 KB) | 0x5000 | 0x2000 |

- OTA con doble ranura (`app0`/`app1`) en las tres placas: alcanza para actualizar sin
  quedar sin firmware.
- El histórico de gráficas **no** usa la partición interna: vive en la microSD
  (`HistoryStore::enableSd`). Sin SD, las gráficas quedan vacías a propósito.
- El nombre del binario lo arma `GET /api/v1/system`: `sema_<versión>_<board_id>.bin`
  (p. ej. `sema_1.103.0_esp32-s3-8mb.bin`).

## 5. Sensores por interfaz

Los 17 modelos de `SensorFactory::create()` (`src/core/sensors/SensorFactory.cpp`), con la
cadena que devuelve `Sensor::interface()`:

| Modelo (`sensors[].model`) | Interfaz declarada | Claves de conexión | Librería | Estado |
|----------------------------|--------------------|--------------------|----------|--------|
| `BME280` | I2C | `sda`, `scl` (0x76 u 0x77) | Adafruit BME280 | ✅ |
| `BMP280` | I2C | `sda`, `scl` (0x76) | Adafruit BMP280 | ✅ |
| `SHT40` | I2C | `sda`, `scl` (default de la librería) | Adafruit SHT4x | ✅ |
| `SHT31` | I2C | `sda`, `scl` (0x44) | Adafruit SHT31 | ✅ |
| `AHT20` | I2C | `sda`, `scl` (default de la librería) | Adafruit AHTx0 | ✅ |
| `BH1750` | I2C | `sda`, `scl` (0x23) | driver propio sobre `Wire` | ✅ |
| `VEML6075` | I2C | `sda`, `scl` (default de la librería) | Adafruit VEML6075 | ✅ |
| `SCD30` | I2C | `sda`, `scl` (default de la librería) | Adafruit SCD30 | ✅ |
| `SGP30` | I2C | `sda`, `scl` (default de la librería) | Adafruit SGP30 | ✅ |
| `AS3935` | I2C | `sda`, `scl` (0x03) | SparkFun AS3935 | ✅ |
| `ADS1115` | I2C | `sda`, `scl` (0x48) + `pin` = canal 0-3 | Adafruit ADS1X15 | ✅ |
| `DS18B20` | 1-Wire | `pin` (bus completo) + `rom` | OneWire + DallasTemperature | ✅ |
| `ADC` | ADC | `pin`, `scale`, `offset` | `analogRead` (12 bits) | ✅ |
| `CO` | ADC | `pin`, `scale`, `offset` | `analogRead` | ✅ |
| `SOLAR` | ADC | `pin`, `scale`, `offset` | `analogRead` | ✅ |
| `PCNT` | GPIO | `pin`, `channel`, `scale` | driver `pcnt` (legacy) | ✅ |
| `PMS5003` | UART | `rx`, `tx` (Serial2, 9600 8N1) | Adafruit PM25 AQI | ✅ |

> ⚠️ `sensors[].address` (selector «ID» de la web) **no** llega a los drivers: la dirección
> queda fija en el código (ver [Buses y periféricos](Buses-y-perifericos.md) §2.1).

## 6. Expansores

| Expansor | Estado | Detalle |
|----------|--------|---------|
| MCP23017 (I²C, 16 E/S) | ✅ implementado | Solo se inicializa **una** dirección; no tiene editor en la web |
| ADS1115 (I²C, 4 ADC) | ✅ implementado como sensor | Dirección fija 0x48 |
| 74HC595 / 74HC165 | ❌ compilado fuera | `SEMA_USE_SHIFT=0` en los cuatro entornos; sin cascada |
| MCP23S17 (SPI, 16 E/S) | ❌ solo configuración | No hay driver |
| MAX14830 (4 UART por SPI) | ❌ solo configuración | No hay driver ni campo de IRQ |
| SC18IS602B (I²C por SPI) | ❌ solo configuración | `sensors[].bus` no lo lee ningún driver |

Detalle completo en [Expansores de entrada/salida](Expansores-de-entrada-salida.md).

## 7. Otros chips ESP32

| Chip | Estado | Qué haría falta |
|------|--------|-----------------|
| ESP32 clásico (WROOM-32, WROVER, D0WD, S0WD, P4) | ✅ soportado por `BOARD_ESP32_WROOM` | Nada: mismo EMAC y mismo VSPI |
| ESP32-WROOM-32U | ✅ soportado por `BOARD_ESP32_WROOM32U` | Perfil de pines de la PCB |
| ESP32-S3 | ✅ soportado por `BOARD_ESP32_S3` | Reasignar `lora.dio1`/`busy` (26/27 son de la flash) |
| ESP32-S2 | ❌ | No tiene Bluetooth; `HwProfile.hpp` no tiene rama: habría que agregar `BOARD_*`, el `#elif` de SPI/ETH y revisar particiones |
| ESP32-C3 / C5 / C6 / H2 | ❌ | RISC-V de un núcleo (o sin Wi-Fi en H2): `Capability::DualCore` se declara igual y el `#error` de `HwProfile.hpp` bloquea la compilación |

## 8. Navegador y dashboard

| Requisito | Estado | Detalle |
|-----------|--------|---------|
| Servidor | ✅ embebido | `WebServer` de Arduino en el puerto 80, sobre Wi-Fi o Ethernet (lwIP) |
| HTML/JS/CSS | ✅ embebido | Todo servido por el firmware; sin CDN externo |
| GridStack | ✅ | `/gridstack-all.min.js` y `/gridstack.min.css` embebidos gzip; grilla de **12 columnas**, `cellHeight: 72`, `margin: 6` |
| Canvas 2D | ✅ | Gráficos propios con `getContext('2d')`, escala por `devicePixelRatio`, series por sensor |
| WebSocket | ⚠️ a medias | Hay `WebSocketsServer ws_{81}` y `broadcastMeasurements()` emite las mediciones por `broadcastTXT`, pero **la página no abre ningún WebSocket**: `HttpServer.hpp:15` lo marca como pendiente |
| Datos en vivo | ✅ por polling | `/api/v1/sensors` cada **5 s** (`setInterval(refresh,5000)`) |
| Gráficos | ✅ por polling | `/api/v1/history?limit=3000`, redibujo cada **60 s** |
| Layout persistente | ✅ | `POST/GET /api/v1/dashboard/layout`, guardado en NVS en una clave aparte |
| Login | ✅ | `/login` (formulario) + `X-API-Key`; sin claves configuradas todo queda abierto |
| Traducción | ✅ | Atributos `data-i18n` con `system.lang` (`es`/`en`) |
| Modales | ✅ propios | `uiModal()` / `uiAlert()` en JS: no dependen de librerías |
| Navegadores | ✅ modernos | Requiere `fetch`, `Promise`, `Canvas 2D`, `Set`/spread y CSS moderno (flex + grid) |

## 9. Firmware y plataforma

| Componente | Versión |
|------------|---------|
| Firmware SEMA | 1.103.0 · HW `rev0` |
| Schema de configuración | 1 (`SEMA_CONFIG_SCHEMA_VERSION`) |
| Protocolo | 1 (`SEMA_PROTOCOL_VERSION`) |
| Plataforma (PlatformIO) | `espressif32` con `framework = arduino` |
| Sistema de archivos | LittleFS (`board_build.filesystem = littlefs`) |
| ArduinoJson | ^6.21.5 (documentos de 16 KB en el parseo de config, 4096 en los POST parciales) |
| RadioLib | ^6.0.0 |
| ModbusMaster | ^2.0.1 |

---

## Ver también

- [Guía de pines](Guia-de-pines.md) · [Buses y periféricos](Buses-y-perifericos.md) · [Expansores de entrada/salida](Expansores-de-entrada-salida.md)
- [Hardware y conexiones](Hardware-y-Conexiones.md) · [Sensores](Sensores.md) · [Materiales](Materiales.md)
- [Compatibilidad de versiones](Compatibilidad-de-versiones.md) · [Guía de inicio](Guia-de-inicio.md) · [Guía de desarrollo](Guia-de-desarrollo.md)
