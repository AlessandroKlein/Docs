---
tags:
  - sema
  - hardware
  - pines
---

# Guía de pines

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Asignación de pines de SEMA para los **cuatro entornos de PlatformIO**, con la clave JSON
real de configuración, el GPIO por defecto, la función, la dirección eléctrica y las
restricciones que aplican. Todas las claves fueron verificadas contra
`include/core/ConfigManager.hpp`, `src/core/ConfigManager.cpp` y
`src/core/web/HttpServer.cpp`; los pines fijos, contra `include/hw/HwProfile.hpp` y
`include/core/BoardProfile.hpp`.

## 1. Cómo leer esta guía

- **Clave JSON**: la propiedad exacta que se escribe en `/api/v1/config` (o en el
  endpoint indicado en §11). Si dice `sensors[model=DS18B20].pin`, es el campo `pin`
  del objeto sensor de ese modelo.
- **GPIO**: valor por defecto en ese entorno. `0` en una clave significa
  *deshabilitado* para esa clave (no «GPIO 0»), salvo donde se aclara lo contrario.
- **Dirección**: punto de vista del ESP32 (`entrada`, `salida`, `bidireccional`).
- Los valores eléctricos marcados **(datasheet)** provienen de la hoja de datos del
  fabricante; el repositorio SEMA **no** los declara y no fueron verificados en el código.

### 1.1 Dos interruptores distintos que se confunden

| Macro | Archivo | Qué fija | Valor en `platformio.ini` |
|-------|---------|----------|---------------------------|
| `SEMA_PINS_FROM_FILE` | `include/hw/HwProfile.hpp` | Pines de **buses** (CAN, Modbus, Zigbee, LoRa, Ethernet) tomados de `HwProfile.hpp` e ignorando la web | `0` en devkit-v1 y S3; `1` en `esp32-wroom-32u` |
| `SEMA_FIXED_HARDWARE` | `include/core/BoardProfile.hpp` | Catálogo de **sensores** fijo (I²C, 1-Wire, ADC batería) e ignorando `sensors[]` | `0` en todos los entornos (no se define por `-D`) |

Con `SEMA_PINS_FROM_FILE=1` se ejecuta `applyHwProfile()` (`src/core/ConfigManager.cpp:13-41`),
que sobrescribe `can.*`, `modbus.*`, `zigbee.*`, `lora.*` y `ethernet.*` **después** de
parsear el JSON: lo que llega desde la web se descarta en silencio.
`SEMA_FIXED_HARDWARE=1` es el único camino que ignora `sensors[]` del todo
(`src/core/SemaCore.cpp:282`); si `sensors[]` está vacío, también se usa el catálogo fijo
aunque la macro valga `0`.

## 2. Pines del catálogo fijo (`BoardProfile.hpp`)

| Clave JSON | GPIO | Función | Dirección | Notas |
|------------|------|---------|-----------|-------|
| `sensors[].sda` (fallback) | 21 | I²C SDA del catálogo fijo | bidireccional | `SEMA_PIN_I2C_SDA`; se usa en el arranque sin importar la config |
| `sensors[].scl` (fallback) | 22 | I²C SCL del catálogo fijo | bidireccional | `SEMA_PIN_I2C_SCL` |
| `sensors[model=DS18B20].pin` | 4 | 1-Wire (SOIL del catálogo fijo) | bidireccional | `SEMA_PIN_ONEWIRE`; pull-up 4,7 kΩ (README §12) |
| `sensors[model=ADC].pin` | 34 | ADC de batería (BATT del catálogo fijo) | entrada | `SEMA_PIN_BATTERY_ADC`; ADC1_CH6; divisor 11:1, escala `3,3 × 11 / 4095` (`SemaCore.cpp:290`) |

> El catálogo fijo de `SemaCore.cpp:284-297` registra además `BME280` (EXT), `SHT40` (INT),
> `BH1750` (LUX) y `AHT20` (AUX), los cuatro sobre 21/22.

## 3. `esp32doit-devkit-v1` (ESP32-WROOM, 4 MB) — entorno por defecto

`board = esp32doit-devkit-v1` · `BOARD_ESP32_WROOM` · `SEMA_BOARD_ID "esp32-wroom-4mb"` ·
`SEMA_FLASH_MB 4` · `SEMA_NATIVE_ETH 1` · `SEMA_PINS_FROM_FILE=0` (**pines configurables**).

### 3.1 Buses y transceivers (configurables)

| Clave JSON | GPIO | Función | Dirección | Notas |
|------------|------|---------|-----------|-------|
| `i2c_sda` | 21 | I²C SDA compartido | bidireccional | `Config.i2cSda`; el arranque igual usa la macro de `BoardProfile.hpp` |
| `i2c_scl` | 22 | I²C SCL compartido | bidireccional | `Config.i2cScl`; mismo caveat |
| `sensors[].sda` / `sensors[].scl` | 21 / 22 | Pines que cada driver I²C pasa a `Wire.begin()` | bidireccional | Todos comparten el objeto `Wire`: gana el último `begin()` |
| `sensors[model=DS18B20].pin` | 4 | Bus 1-Wire (varios DS18B20) | bidireccional | Pull-up 4,7 kΩ a 3,3 V; ROM de 16 hex para distinguirlos |
| `sensors[model=ADC].pin` | 0 | Entrada analógica (veleta, batería, CO, solar) | entrada | Usar **ADC1** (32-39); `0` = sin asignar |
| `sensors[model=PCNT].pin` | 0 | Pulsos (anemómetro, pluviómetro) | entrada | `PcntSensor` fija `PCNT_UNIT_0` para todos |
| `system.wind_direction_pin` | 0 | ADC de la veleta WH-SP-WD | entrada | `0` = sin veleta; se lee con `analogRead` directo |
| `energy.rain_pin` | 0 | Wake por lluvia (`ext0`) | entrada | Requiere GPIO con capacidad RTC (`PowerManager.cpp:19`) |
| `sensors[model=PMS5003].rx` / `.tx` | 0 / 0 | UART del PMS5003 | entrada / salida | `Serial2` a 9600 8N1; **choca con Modbus** |
| `modbus.rx` / `.tx` / `.de_re` | 16 / 17 / 0 | RS485 / Modbus RTU | entrada / salida / salida | `de_re = 0` → DE/RE sin control por software |
| `can.tx` / `can.rx` | 5 / 4 | CAN/TWAI | salida / entrada | GPIO5 es strapping de SDIO |
| `lora.cs` | 10 | Chip-select SX1262 | salida | `SEMA_CS_LORA` |
| `lora.rst` | 14 (web/header) · 32 (parse) | Reset del SX1262 | salida | **Discrepancia de defaults**, ver §7 |
| `lora.dio1` | 26 | IRQ DIO1 del SX1262 | entrada | GPIO26 = DAC2 |
| `lora.busy` | 27 | BUSY del SX1262 | entrada | GPIO27 = CRS_DV del RMII |
| `zigbee.rx` / `.tx` | 18 / 19 | ZNP del CC2652P2 (`Serial1`) | entrada / salida | 18/19 son MDIO/TXD0 del RMII |
| `ethernet.mdc` / `.mdio` | 23 / 18 | Bus MDIO de la PHY LAN8720A | salida / bidireccional | Configurables en `ETH.begin()` |
| `ethernet.phy_addr` | 1 | Dirección de la PHY | — | Strap PHYAD0 |
| `ethernet.enabled` | false | Habilita RMII | — | Con `true`, el firmware reserva los 10 pines RMII |
| `sd_cs` | 4 | Chip-select de la microSD | salida | `SEMA_PIN_SD_CS`; choca con 1-Wire y con `can.rx` |
| `gpio[].pin` | 0 | GPIO standalone (relés, LEDs) | según `gpio[].mode` | 34-39 solo entrada |
| `gpio[].expander_addr` | 0 | Dirección I²C del MCP23017 | — | `0` = pin nativo; `0x20`+ = pin del expansor |
| `mcp23s17_cs` | 0 | Chip-select del MCP23S17 | salida | Sin driver: solo configuración (§7) |
| `spi_expanders[].cs` | 0 | CS de MAX14830 / SC18IS602B | salida | Sin driver: solo configuración (§7) |
| `shift_registers[].latch_pin` | 0 | LATCH del 74HC595 / 74HC165 | salida | `SEMA_USE_SHIFT=0` en este entorno (§7) |

### 3.2 Pines fijos del bus SPI y de Ethernet RMII (no configurables)

| Recurso | GPIO | Función | Dirección | Notas |
|---------|------|---------|-----------|-------|
| `SEMA_SPI_SCK` | 14 | Reloj del bus SPI de Arduino (LoRa/SD/shift) | salida | Fijo en `HwProfile.hpp:107` |
| `SEMA_SPI_MISO` | 12 | MISO del bus SPI | entrada | GPIO12 = MTDI (strapping) |
| `SEMA_SPI_MOSI` | 15 | MOSI del bus SPI | salida | GPIO15 = MTDO (strapping) |
| `SEMA_PIN_ETH_MDC` | 23 | RMII MDC | salida | También informado en `/api/v1/system` |
| `SEMA_PIN_ETH_MDIO` | 18 | RMII MDIO | bidireccional | |
| `SEMA_PIN_ETH_TXD0` | 19 | RMII TXD0 | salida | Fijo del EMAC |
| `SEMA_PIN_ETH_TXD1` | 22 | RMII TXD1 | salida | Fijo del EMAC |
| `SEMA_PIN_ETH_TX_EN` | 21 | RMII TX_EN | salida | Fijo del EMAC |
| `SEMA_PIN_ETH_RXD0` | 25 | RMII RXD0 | entrada | Fijo del EMAC; GPIO25 = DAC1 |
| `SEMA_PIN_ETH_RXD1` | 26 | RMII RXD1 | entrada | Fijo del EMAC |
| `SEMA_PIN_ETH_CRS_DV` | 27 | RMII CRS_DV | entrada | Fijo del EMAC |
| `SEMA_PIN_ETH_RX_ER` | 13 | RMII RX_ER | entrada | Fijo del EMAC |
| `SEMA_PIN_ETH_REF_CLK` | 0 | Reloj de 50 MHz entrante (`ETH_CLOCK_GPIO0_IN`) | entrada | GPIO0 = botón BOOT / strapping |
| `ethernet.power` | -1 | Alimentación de la PHY | — | `-1` = sin control |
| `SEMA_CS_ETHERNET_W5500` | 5 | CS del W5500 | salida | **No se usa** en esta placa (`SEMA_NATIVE_ETH=1`) |
| `SEMA_PIN_ETH_W5500_RST` | -1 | Reset del W5500 | — | No se usa |
| `SEMA_PIN_ETH_W5500_IRQ` | 4 | IRQ del W5500 | entrada | No se usa; choca con 1-Wire y `sd_cs` |
| `SEMA_ETH_SPI_SCK/MISO/MOSI` | 14 / 12 / 13 | Host SPI2 del W5500 | salida/entrada/salida | Definidos pero **no usados** en esta placa |

## 4. `esp32-s3-devkitc-1` (ESP32-S3, 8 MB)

`board = esp32-s3-devkitc-1` · `BOARD_ESP32_S3` · `SEMA_BOARD_ID "esp32-s3-8mb"` ·
`SEMA_FLASH_MB 8` · `SEMA_NATIVE_ETH 0` (W5500 por SPI) · `SEMA_PINS_FROM_FILE=0`
(**pines configurables**).

| Clave JSON | GPIO | Función | Dirección | Notas |
|------------|------|---------|-----------|-------|
| `i2c_sda` / `i2c_scl` | 21 / 22 | I²C compartido | bidireccional | Igual que en WROOM |
| `sensors[].sda` / `.scl` | 21 / 22 | I²C por sensor | bidireccional | Gana el último `Wire.begin()` |
| `sensors[model=DS18B20].pin` | 4 | 1-Wire | bidireccional | Pull-up 4,7 kΩ |
| `sensors[model=ADC].pin` | 0 | Entrada analógica | entrada | En S3 usar ADC1 (GPIO1-10); ADC2 no convive con Wi-Fi |
| `sensors[model=PCNT].pin` | 0 | Pulsos | entrada | `PCNT_UNIT_0` compartida |
| `system.wind_direction_pin` | 0 | Veleta (ADC) | entrada | `0` = sin veleta |
| `energy.rain_pin` | 0 | Wake por lluvia | entrada | GPIO RTC-capable |
| `modbus.rx` / `.tx` / `.de_re` | 16 / 17 / 0 | RS485 / Modbus | entrada / salida / salida | `Serial2` |
| `can.tx` / `can.rx` | 5 / 4 | CAN/TWAI | salida / entrada | **`can.tx=5` choca con `ethernet.cs`** |
| `lora.cs` | 10 | CS del SX1262 | salida | `SEMA_CS_LORA`; el bus SPI es FSPI 12/13/11 |
| `lora.rst` | 14 (web/header) · 32 (parse) | Reset del SX1262 | salida | Discrepancia de defaults, ver §7 |
| `lora.dio1` / `lora.busy` | 26 / 27 | DIO1 / BUSY del SX1262 | entrada / entrada | ⚠️ **GPIO26-32 están ocupados por la flash SPI** en ESP32-S3: mover estos dos |
| `zigbee.rx` / `.tx` | 18 / 19 | ZNP del CC2652P2 | entrada / salida | Choca con el host SPI del W5500 (18/19) |
| `ethernet.cs` (W5500) | 5 | CS del W5500 | salida | `SEMA_CS_ETHERNET_W5500`; choca con `can.tx` |
| `ethernet.irq` | 4 | IRQ del W5500 | entrada | Choca con 1-Wire y `sd_cs` |
| `ethernet.rst` | -1 | Reset del W5500 | — | `-1` = sin pin |
| `ethernet.phy_addr` | 1 | Dirección de la PHY W5500 | — | |
| `sd_cs` | 4 | CS de la microSD | salida | Comparte FSPI con LoRa |
| `gpio[].pin` | 0 | GPIO standalone (relés, LEDs) | según `mode` | |
| `mcp23s17_cs` | 0 | CS del MCP23S17 | salida | Sin driver |
| `spi_expanders[].cs` | 0 | CS de MAX14830 / SC18IS602B | salida | Sin driver |
| `shift_registers[].latch_pin` | 0 | LATCH del 74HC595 / 74HC165 | salida | `SEMA_USE_SHIFT=0` |

### 4.1 Pines fijos de los dos buses SPI en S3

| Recurso | GPIO | Función | Dirección | Notas |
|---------|------|---------|-----------|-------|
| `SEMA_SPI_SCK` | 12 | Reloj del FSPI (LoRa + SD + shift) | salida | `HwProfile.hpp:102` |
| `SEMA_SPI_MISO` | 13 | MISO del FSPI | entrada | |
| `SEMA_SPI_MOSI` | 11 | MOSI del FSPI | salida | |
| `SEMA_ETH_SPI_SCK` | 18 | Reloj del host SPI3 (W5500) | salida | `SEMA_ETH_SPI_HOST 2` |
| `SEMA_ETH_SPI_MISO` | 19 | MISO del host SPI3 | entrada | ⚠️ GPIO19/20 son USB D-/D+ en el DevKitC-1 |
| `SEMA_ETH_SPI_MOSI` | 21 | MOSI del host SPI3 | salida | |
| `SEMA_ETH_SPI_HOST` | 2 | Periférico SPI3 dedicado | — | Separado del SPI de Arduino para no chocar con RadioLib |

## 5. `esp32-wroom-32u` (ESP32-WROOM-32U, 16 MB) — PCB futura

`board = esp32dev` · `BOARD_ESP32_WROOM32U` · `SEMA_BOARD_ID "esp32-wroom32u-16mb"` ·
`SEMA_FLASH_MB 16` · `SEMA_NATIVE_ETH 1` · **`SEMA_PINS_FROM_FILE=1` → pines fijos**.
La web muestra los selectores, pero `applyHwProfile()` los sobrescribe: **no son
configurables**. El bus SPI (14/12/15) y los pines RMII son los mismos que en devkit-v1.

| Clave JSON | GPIO (fijo) | Función | Dirección | Notas |
|------------|-------------|---------|-----------|-------|
| `can.tx` / `can.rx` | 5 / 4 | CAN/TWAI | salida / entrada | `SEMA_PIN_CAN_TX` / `SEMA_PIN_CAN_RX` |
| `modbus.rx` / `.tx` / `.de_re` | 16 / 17 / 0 | RS485 / Modbus | entrada / salida / salida | ⚠️ `de_re=0` **choca con RMII REF_CLK (GPIO0)** |
| `zigbee.rx` / `.tx` | 18 / 19 | ZNP del CC2652P2 | entrada / salida | ⚠️ Choca con MDIO (18) y TXD0 (19) del RMII |
| `lora.cs` | 10 | CS del SX1262 | salida | `SEMA_CS_LORA` |
| `lora.rst` | 32 | Reset del SX1262 | salida | `SEMA_PIN_LORA_RST` |
| `lora.dio1` | 26 | DIO1 del SX1262 | entrada | ⚠️ Choca con RMII RXD1 |
| `lora.busy` | 27 | BUSY del SX1262 | entrada | ⚠️ Choca con RMII CRS_DV |
| `ethernet.mdc` / `.mdio` | 23 / 18 | MDIO de la LAN8720A | salida / bidireccional | `SEMA_PIN_ETH_MDC` / `SEMA_PIN_ETH_MDIO` |
| `ethernet.phy_addr` / `.power` | 1 / -1 | Dirección y alimentación de la PHY | — | `SEMA_PIN_ETH_PHY_ADDR` / `SEMA_PIN_ETH_POWER` |
| `ethernet.cs` / `.rst` / `.irq` | 5 / -1 / 4 | W5500 | — | `SEMA_NATIVE_ETH=1`: **no se usan** |
| `ethernet.sck` / `.miso` / `.mosi` | 14 / 12 / 15 | Host SPI del W5500 | — | Se cargan desde `SEMA_SPI_*`; **no se usan** |
| `sd_cs` | 4 | CS de la microSD | salida | `SEMA_PIN_SD_CS`; `applyHwProfile()` **no** lo toca, así que la web sí puede cambiarlo |
| `i2c_sda` / `i2c_scl` | 21 / 22 | I²C compartido | bidireccional | `applyHwProfile()` **no** los toca; el arranque usa `SEMA_PIN_I2C_*` |
| `sensors[model=DS18B20].pin` | 4 (catálogo fijo) | 1-Wire | bidireccional | Solo si `sensors[]` está vacío o `SEMA_FIXED_HARDWARE=1` |
| `sensors[model=ADC].pin` | 34 (catálogo fijo) | ADC de batería | entrada | Divisor 11:1 |
| `system.wind_direction_pin` | 0 | Veleta (ADC) | entrada | `0` = sin veleta |
| `energy.rain_pin` | 0 | Wake por lluvia | entrada | GPIO RTC-capable |
| `gpio[].pin` | 0 | GPIO standalone (relés, LEDs) | según `mode` | 34-39 solo entrada |
| `gpio[].expander_addr` | 0 | Dirección I²C del MCP23017 | — | `0` = pin nativo |
| `mcp23s17_cs` | 0 | CS del MCP23S17 | salida | Sin driver |
| `spi_expanders[].cs` | 0 | CS de MAX14830 / SC18IS602B | salida | Sin driver |
| `shift_registers[].latch_pin` | 0 | LATCH del 74HC595 / 74HC165 | salida | `SEMA_USE_SHIFT=0` |

## 6. `demo`

`extends = env:esp32doit-devkit-v1` + `-D SEMA_DEMO=1`. Hereda **todos** los pines, la
partición de 4 MB y `SEMA_NATIVE_ETH=1` de devkit-v1. Particularidad verificada en
`src/core/ConfigManager.cpp:385-391`: en modo demo, si `mcp23s17_cs` es `0`, se fuerza a
**5** y los 16 pines del MCP23S17 quedan como **salidas** para que la página de E/S
muestre el expansor aunque no exista hardware.

## 7. Restricciones reales del MCU

### 7.1 ESP32 clásico (WROOM / WROOM-32U)

| Restricción | Pines | Consecuencia en SEMA |
|-------------|-------|----------------------|
| Solo entrada, sin pull-up/pull-down interno | 34, 35, 36, 39 | Sirven como ADC y como entradas digitales, nunca como salida: no pueden manejar relés ni ser LATCH/CS |
| Flash SPI integrada (SPI0/1) | 6, 7, 8, 9, 10, 11 | No usar como GPIO; el selector web **sí** los ofrece (1..39) |
| UART0 de consola | 1, 3 | El selector web los ofrece; usarlos corta el log por `Serial` |
| ADC2 inutilizable con Wi-Fi activo | 0, 2, 4, 12, 13, 14, 15, 25, 26, 27 | Toda entrada analógica (veleta, batería, CO, solar) debe ir a **ADC1 = 32-39** |
| Strapping de arranque | 0, 2, 5, 12, 15 | `12` (MTDI) no debe estar en HIGH al arrancar (elige tensión de flash); `15` (MTDO) debe estar en HIGH; `0` (BOOT) en HIGH; `5` es strapping de SDIO |
| DAC | 25 (DAC1), 26 (DAC2) | No hay API de DAC en SEMA (`Capability::Dac` se declara, no se usa) |
| Capacidad RTC para wake | 0, 2, 4, 12-15, 25-27, 32-39 | `energy.rain_pin` debe caer en uno de estos |
| GPIO del EMAC | 0, 13, 18, 19, 21, 22, 23, 25, 26, 27 | Reservados por hardware **solo si `ethernet.enabled`** (`HttpServer.cpp:1434-1443`) |

### 7.2 ESP32-S3

| Restricción | Pines | Consecuencia en SEMA |
|-------------|-------|----------------------|
| Flash SPI integrada (SPI0/1) | 26, 27, 28, 29, 30, 31, 32 | ⚠️ Los defaults de LoRa `dio1=26` y `busy=27` caen acá: **hay que reasignarlos** |
| USB nativo / USB-JTAG del DevKitC-1 | 19, 20 | `SEMA_ETH_SPI_MISO=19` compite con el puerto USB nativo; usar el puerto UART (CP2102) para flashear |
| Solo entrada | 46 (strapping/entrada) | — |
| ADC2 no convive con Wi-Fi | 11-20 | Para analógicas, usar **ADC1 (GPIO1-10)** |
| PSRAM (solo módulos N8R8/R8) | 33-37 | El entorno usa `esp32-s3-devkitc-1` **N8** (sin PSRAM): esos pines quedan libres |
| Sin MAC Ethernet | — | Ethernet solo por W5500 SPI (`SEMA_NATIVE_ETH 0`) |

## 8. Conflictos entre valores por defecto

### 8.1 `esp32doit-devkit-v1` (con `ethernet.enabled=true`)

| Señal A | GPIO | Señal B | GPIO | Efecto |
|---------|------|---------|------|--------|
| `i2c_sda` | 21 | RMII `TX_EN` | 21 | I²C inutilizable con Ethernet nativo |
| `i2c_scl` | 22 | RMII `TXD1` | 22 | I²C inutilizable con Ethernet nativo |
| `zigbee.rx` | 18 | RMII `MDIO` | 18 | Zigbee inutilizable con Ethernet nativo |
| `zigbee.tx` | 19 | RMII `TXD0` | 19 | Zigbee inutilizable con Ethernet nativo |
| `lora.dio1` | 26 | RMII `RXD1` | 26 | LoRa inutilizable con Ethernet nativo |
| `lora.busy` | 27 | RMII `CRS_DV` | 27 | LoRa inutilizable con Ethernet nativo |
| `sd_cs` | 4 | 1-Wire (catálogo fijo) | 4 | MicroSD y DS18B20 comparten pin |
| `sd_cs` | 4 | `can.rx` | 4 | MicroSD y CAN comparten pin |
| `can.tx` | 5 | `SEMA_CS_ETHERNET_W5500` | 5 | Irrelevante acá (W5500 no se usa en WROOM) |
| `lora.rst` | 14 | `SEMA_SPI_SCK` | 14 | Con el default de la web/header, el reset del LoRa pisa el reloj SPI |
| `lora.rst` | 32 | — | — | El default de parseo (32) sí es libre |

### 8.2 `esp32-s3-devkitc-1`

| Señal A | GPIO | Señal B | GPIO | Efecto |
|---------|------|---------|------|--------|
| `can.tx` | 5 | `ethernet.cs` (W5500) | 5 | CAN y W5500 comparten chip-select |
| `zigbee.rx` / `.tx` | 18 / 19 | `SEMA_ETH_SPI_SCK` / `_MISO` | 18 / 19 | Zigbee inutilizable con W5500 activo |
| `ethernet.irq` | 4 | `sd_cs` y 1-Wire | 4 | Tres funciones sobre el mismo GPIO |
| `lora.dio1` / `.busy` | 26 / 27 | Flash SPI integrada | 26 / 27 | ⚠️ El LoRa no arranca con los defaults |
| `sd_cs` | 4 | `can.rx` | 4 | MicroSD y CAN comparten pin |

### 8.3 `esp32-wroom-32u` (pines fijos, Ethernet nativo)

| Señal A | GPIO | Señal B | GPIO | Efecto |
|---------|------|---------|------|--------|
| `modbus.de_re` | 0 | RMII `REF_CLK` | 0 | ⚠️ Conflicto **entre dos pines fijos** del mismo perfil: no se puede resolver por web |
| `zigbee.rx` / `.tx` | 18 / 19 | RMII `MDIO` / `TXD0` | 18 / 19 | Zigbee y Ethernet nativo son mutuamente excluyentes |
| `lora.dio1` / `.busy` | 26 / 27 | RMII `RXD1` / `CRS_DV` | 26 / 27 | LoRa y Ethernet nativo son mutuamente excluyentes |
| `ethernet.cs` / `.irq` | 5 / 4 | `can.tx` / `sd_cs` | 5 / 4 | Irrelevante mientras `SEMA_NATIVE_ETH=1` |
| `sd_cs` | 4 | 1-Wire (catálogo fijo) | 4 | MicroSD y DS18B20 comparten pin |

### 8.4 Conflictos de recursos que no se ven en el pinout

| Recurso | Consumido por | Consecuencia |
|---------|---------------|--------------|
| `Serial2` | Modbus **y** PMS5003 | No se pueden usar los dos a la vez: el segundo `begin()` reconfigura el periférico |
| `PCNT_UNIT_0` / `PCNT_CHANNEL_0` | Todos los sensores `PCNT` | Anemómetro y pluviómetro simultáneos: el último `begin()` gana el pin |
| Objeto `Wire` | Todos los drivers I²C | `Wire.begin(sda, scl)` por sensor: los pines efectivos son los del último sensor inicializado |
| Expansor MCP23017 | `GpioManager::apply()` | Solo inicializa **una** dirección (la del primer `expander_addr` no nulo); el resto de las direcciones se escriben contra ese mismo chip |
| Selector de dirección I²C de la web | 15 drivers I²C | `SensorSpec.address` **nunca** se pasa al driver: las direcciones están fijas en el código |

## 9. Pines reservados informados al frontend

`GET /api/v1/system` devuelve `reserved_pins` (`src/core/web/HttpServer.cpp:1428-1443`):

1. Siempre: `SEMA_SPI_SCK`, `SEMA_SPI_MISO`, `SEMA_SPI_MOSI` → 14/12/15 en WROOM y
   WROOM-32U; 12/13/11 en S3.
2. Solo si `SEMA_NATIVE_ETH && SEMA_USE_ETHERNET && ethernet.enabled`: los 10 pines RMII
   (0, 13, 18, 19, 21, 22, 23, 25, 26, 27).

El selector de pines del frontend recorre **GPIO 1..39** y deshabilita los que estén en
`reserved_pins` o ya usados (`usedPins()`, `HttpServer.cpp:1004-1021`). No filtra GPIO6-11
(flash) ni GPIO1/3 (consola), así que la validación de esos casos queda a criterio del
instalador.

## 10. Cómo se cambian los pines

| Endpoint | Método | Qué acepta |
|----------|--------|------------|
| `/api/v1/config/buses` | POST | `i2c_sda`, `i2c_scl`, `sd_cs`, `modbus{enabled,rx,tx,de_re,uart,uart_port}`, `can{enabled,tx,rx}`, `lora{enabled,cs,rst,dio1,busy}`, `zigbee{enabled,rx,tx,uart,uart_port}`, `ethernet{enabled,mdc,mdio,cs}` |
| `/api/v1/config/io` | POST | `mcp23s17_cs`, `mcp23s17_pins[16]`, `shift_registers[{type,latch,pins[8]}]` (solo con `SEMA_USE_SHIFT=1`), `spi_expanders[{type,cs}]`, `gpio[{id,pin,mode,initial,expander}]` |
| `/api/v1/config/sensors` | POST | `sensors[{id,model,enabled,address,rom,sda,scl,pin,rx,tx,channel,unit,scale,offset}]` — **no** acepta `bus`, `uart` ni `uart_port` |
| `/api/v1/config` | PUT | Config completa, incluidos `modbus.baud`, `modbus.slave_id`, `modbus.register`, `modbus.count`, `can.speed`, `lora.frequency`, `ethernet.phy_addr`, `ethernet.rst`, `ethernet.irq`, `storage.sd_cs`, `sensors[].bus/uart/uart_port` |
| `/api/v1/gpio` | POST | `{pin, value}` sobre un `gpio[].pin` ya configurado (requiere autenticación) |
| `/api/v1/shift` | POST | `{value}` (0-255) hacia los 74HC595 (requiere autenticación) |

> Con `SEMA_PINS_FROM_FILE=1` (placa `esp32-wroom-32u`), escribir `can`, `modbus`, `zigbee`,
> `lora` o `ethernet` por la web **no tiene efecto**: `applyHwProfile()` los pisa.

---

## Ver también

- [Hardware y conexiones](Hardware-y-Conexiones.md) · [Buses y periféricos](Buses-y-perifericos.md) · [Expansores de entrada/salida](Expansores-de-entrada-salida.md)
- [Compatibilidad](Compatibilidad.md) · [Sensores](Sensores.md) · [Actuadores y salidas](Actuadores-y-Salidas.md)
- [Referencia de configuración](Referencia-configuracion.md) · [Materiales](Materiales.md) · [API REST](API-REST.md)
