---
tags:
  - sema
  - hardware
  - buses
---

# Buses y periféricos

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Cómo está implementado cada bus de SEMA (I²C, SPI, UART, 1-Wire, ADC, PCNT, RS485/Modbus,
CAN/TWAI, Ethernet, LoRa y Zigbee), con qué claves JSON se configura, qué recursos del
ESP32 consume, qué límites tiene y qué falta. Verificado contra `include/hw/HwProfile.hpp`,
`include/core/ConfigManager.hpp`, `src/core/ConfigManager.cpp`, `src/core/SemaCore.cpp`,
los `*Manager.cpp` de cada bus y `src/core/web/HttpServer.cpp`.

> Los valores eléctricos marcados **(datasheet)** provienen de la hoja de datos del
> fabricante; el repositorio SEMA no los declara. Todo lo demás sale del código.

## 1. Mapa de buses

```text
ESP32
├── I2C0 "Wire" (compartido) ........ SDA/SCL (default 21/22) → 15 drivers de sensor,
│                                     ADS1115, MCP23017, AS3935
├── SPI (Arduino: VSPI en WROOM / FSPI en S3)
│   ├── CS LoRa .... 10 (SX1262 por RadioLib)
│   ├── CS microSD .. 4  (SD.begin)
│   └── MOSI/SCK .... 74HC595/74HC165 (bit-banged, latch propio)
├── SPI host dedicado (solo S3) ..... W5500: SCK 18, MISO 19, MOSI 21, CS 5
├── UART0 "Serial" .. consola 115200
├── UART1 "Serial1" . Zigbee ZNP (CC2652P2)
├── UART2 "Serial2" . Modbus RTU  **o** PMS5003 (nunca los dos)
├── 1-Wire .......... DS18B20 (un pin, N sondas por ROM)
├── ADC1 ............ veleta, batería, CO, solar (ADC2 no convive con Wi-Fi)
├── PCNT ............ anemómetro, pluviómetro (una sola unidad: PCNT_UNIT_0)
├── TWAI ............ CAN 2.0 (125k/250k/500k/1M)
└── EMAC + RMII ..... LAN8720A (solo ESP32 clásico y WROOM-32U)
```

## 2. I²C

| Aspecto | Detalle |
|---------|---------|
| Pines | `i2c_sda` = 21, `i2c_scl` = 22 (`Config.i2cSda/i2cScl`, `ConfigManager.hpp:235`) |
| Inicialización real | `Wire.begin(SEMA_PIN_I2C_SDA, SEMA_PIN_I2C_SCL)` en `src/core/SemaCore.cpp:112`: usa las macros de `BoardProfile.hpp` (21/22), **no** la config |
| Objeto | Un único `Wire` (I2C0); cada driver I²C vuelve a llamar `Wire.begin(sda_, scl_)` con los pines de su `SensorSpec` |
| Escaneo | `I2cScanner::scan()` recorre 1..126 al arrancar y publica `address` + `model` en `GET /api/v1/diagnostics → i2c_devices` |
| Configuración web | Página `/config/sensors`, bloque «Pines de buses» → `POST /api/v1/config/buses` con `i2c_sda` / `i2c_scl` (la web reinicia el equipo después de guardar) |
| Reloj | No configurable: se usa el default de la librería `Wire` (100 kHz nominal) |
| Pull-ups | No hay pull-ups internos activados por SEMA: hacen falta **externos de 4,7 kΩ a 3,3 V** en SDA y SCL (datasheet) |

### 2.1 Direcciones I²C

| Dispositivo | Dirección usada por el driver | Origen |
|-------------|-------------------------------|--------|
| BME280 | 0x76 u 0x77 | `Bme280Sensor.cpp:19` (`begin(0x76) \|\| begin(0x77)`) |
| BMP280 | 0x76 | `Bmp280Sensor.cpp:19` |
| SHT31 | 0x44 | `Sht31Sensor.cpp:19` |
| SHT40 | default de la librería (0x44) | `Sht40Sensor.cpp:19` |
| AHT20 | default de la librería (0x38) | `Aht20Sensor.cpp:19` |
| BH1750 | 0x23 | `Bh1750Sensor.cpp:7` |
| VEML6075 | default de la librería (0x10) | `Veml6075Sensor.cpp:19` |
| SCD30 | default de la librería (0x61) | `Scd30Sensor.cpp:19` |
| SGP30 | default de la librería (0x58) | `Sgp30Sensor.cpp:19` |
| ADS1115 | 0x48 (`begin()` sin dirección) | `Ads1115Sensor.cpp:22` |
| AS3935 | 0x03 | `As3935Sensor.cpp:8` |
| MCP23017 | `gpio[].expander_addr` (0x20-0x27) | `GpioManager.cpp:26` |

> ⚠️ **`address` del JSON no se usa.** El selector «ID» de la web escribe
> `sensors[].address`, pero `SensorFactory::create()` solo pasa `sda`/`scl` a los drivers
> (`src/core/sensors/SensorFactory.cpp`): la dirección queda fija en el código. Con dos
> sensores iguales en el mismo bus (p. ej. dos SHT31) no hay forma de distinguirlos.

### 2.2 Detección automática

`I2cScanner::modelForAddress()` (`src/core/sensors/I2cScanner.cpp:21-37`) sugiere:
`0x23 BH1750` · `0x38/0x39 AHT20` · `0x40 SHT31/HTU21D` · `0x44/0x45 SHT40/SHT3x` ·
`0x5C AM2320` · `0x61 SCD30` · `0x62 SCD40/SCD41` · `0x68 MPU6050/DS3231` ·
`0x76/0x77 BME280/BMP280` · cualquier otra dirección → cadena vacía («desconocido»).

## 3. SPI

SEMA usa **dos hosts SPI distintos** a propósito (`HwProfile.hpp:93-137`):

| Host | Entorno | SCK | MISO | MOSI | Uso | Driver |
|------|---------|-----|------|------|-----|--------|
| SPI de Arduino (VSPI en WROOM/WROOM-32U, FSPI en S3) | devkit-v1 y wroom-32u | 14 | 12 | 15 | LoRa, microSD, shift registers | RadioLib / `SD` / `shiftOut`-`shiftIn` |
| SPI de Arduino (FSPI) | S3 | 12 | 13 | 11 | LoRa, microSD, shift registers | RadioLib / `SD` |
| SPI3 (host 2) | S3 | 18 | 19 | 21 | W5500 | `esp_eth` (ESP-IDF) |
| SPI2 (host 1) | WROOM/WROOM-32U | 14 | 12 | 13 | W5500 | No se usa (esas placas tienen MAC nativa) |

- Inicialización: `SPI.begin(SEMA_SPI_SCK, SEMA_SPI_MISO, SEMA_SPI_MOSI)` una sola vez en
  `SemaCore.cpp:128-131`, bajo `#if SEMA_USE_LORA || SEMA_USE_ETHERNET`.
- Los **chip-select** son independientes por dispositivo: LoRa `SEMA_CS_LORA` = 10,
  microSD `SEMA_PIN_SD_CS` = 4, W5500 `SEMA_CS_ETHERNET_W5500` = 5.
- El W5500 corre a **20 MHz**, modo 0, `command_bits=16`, `address_bits=8`,
  `queue_size=20` (`EthernetManager.cpp:58-64`).
- **No hay mutex ni arbitraje del bus**: microSD y LoRa comparten MOSI/MISO/SCK y cada
  librería maneja su CS. Una escritura de histórico en curso puede intercalarse con una
  recepción LoRa; el firmware no lo serializa.
- Los 74HC595/165 no usan `SPI`: reutilizan MOSI/SCK como bit-bang con `shiftOut`/`shiftIn`
  (ver [Expansores de entrada/salida](Expansores-de-entrada-salida.md)).

### 3.1 microSD (histórico)

| Aspecto | Detalle |
|---------|---------|
| Clave JSON | `storage.sd_enabled` (bool), `storage.sd_cs` (default 4) |
| Driver | `SD.begin(csPin)` sobre el bus SPI por defecto (`HistoryStore.cpp:38-40`) |
| Condición | Solo se monta si `storage.sd_enabled` es `true` (`SemaCore.cpp:84-89`) |
| Estado | `GET /api/v1/system → sd_enabled`, `history_available` (`history.sdEnabled()`) |
| Sin SD | El histórico no se guarda y las gráficas quedan vacías a propósito (no se usa flash interna) |

## 4. UART

| Puerto | Periférico | Baudios | Pines (claves) | Notas |
|--------|-----------|---------|----------------|-------|
| UART0 | `Serial` (consola) | 115200 (`SemaCore.cpp:47`) | 1 / 3 | El log de arranque y los `Serial.printf` de diagnóstico |
| UART1 | `Serial1` (Zigbee ZNP) | `zigbee.baud` = 115200 | `zigbee.rx` / `zigbee.tx` | `ZigbeeManager.cpp:20` |
| UART2 | `Serial2` | `modbus.baud` = 9600 (Modbus) o 9600 fijo (PMS5003) | `modbus.rx`/`modbus.tx` o `sensors[].rx`/`tx` | **Compartido: Modbus y PMS5003 no pueden coexistir** |

- La implementación nativa de UART se limita a `Serial1` (Zigbee) y `Serial2`
  (Modbus / PMS5003). El resto de sensores UART del catálogo es el PMS5003.
- **MAX14830**: expansor de 4 UART por SPI. El firmware solo guarda su `cs`
  (`spi_expanders[].cs`) y expone la opción en los selectores de `bus`/`uart` de la web;
  no hay driver que lo inicialice (ver [Expansores de entrada/salida](Expansores-de-entrada-salida.md)).

## 5. 1-Wire

| Aspecto | Detalle |
|---------|---------|
| Clave JSON | `sensors[model=DS18B20].pin` (un solo pin para todo el bus) |
| Default | 4 (`SEMA_PIN_ONEWIRE`, `BoardProfile.hpp:34`) |
| Driver | `OneWire` + `DallasTemperature` (`Ds18b20Sensor.cpp:43-46`) |
| Varios sensores | Sí: cada entrada del catálogo lleva su `rom` (16 hex) o vacío para autodetección |
| Resolución/parasitario | El firmware no cambia resolución ni modo parásito: usa los defaults de la librería |
| Pull-up | **4,7 kΩ a 3,3 V** entre DQ y VDD (README §12 lo exige; `SemaCore.cpp:286` lo comenta) |
| Conflicto | El pin 4 también es `sd_cs` por defecto y `can.rx` en el perfil fijo |

## 6. ADC interno

| Aspecto | Detalle |
|---------|---------|
| Claves JSON | `sensors[model=ADC\|CO\|SOLAR].pin`, `system.wind_direction_pin`, `system.wind_rpull` |
| Resolución | 12 bits: `analogReadResolution(12)` en `AdcSensor::begin()` |
| Escala | `valor = raw × scale + offset`, con `scale` y `offset` por sensor (JSON) |
| Batería (catálogo fijo) | GPIO34, divisor **11:1** para 12 V, escala `3,3 × 11 / 4095` (`SemaCore.cpp:290`) |
| Veleta | `system.wind_direction_pin` (0 = deshabilitada), `analogRead` directo en `DerivedCalculator.cpp:218-219` y `HttpServer.cpp:2214` |
| Red de la veleta | 8 resistencias (`system.wind_resistors`, N/NE/E/SE/S/SO/O/NO) + pull-up `system.wind_rpull` = 10 000 Ω |
| Límite real | Con Wi-Fi activo **solo sirve ADC1 (GPIO32-39 en ESP32 clásico, GPIO1-10 en S3)**; ADC2 está tomado por el radio |

## 7. PCNT (contador de pulsos)

| Aspecto | Detalle |
|---------|---------|
| Clave JSON | `sensors[model=PCNT].pin`, más `channel` (`wind_speed`, `rain`) y `scale` |
| Unidad | `PCNT_UNIT_0` / `PCNT_CHANNEL_0` **hardcodeadas** (`PcntSensor.cpp:27-28`) |
| Modo | Cuenta por flanco ascendente, `PCNT_COUNT_INC`, sin pin de control |
| Rango | `counter_l_lim = 0`, `counter_h_lim = 32767`; se lee y se limpia en cada ciclo (10 s) |
| Límite | Dos sensores PCNT simultáneos (anemómetro + pluviómetro) reconfiguran la misma unidad: gana el último `begin()` |

## 8. RS485 / Modbus RTU

| Aspecto | Detalle |
|---------|---------|
| Claves JSON | `modbus{enabled,rx,tx,de_re,uart,uart_port,baud,slave_id,register,count}` |
| Defaults | `rx=16`, `tx=17`, `de_re=0`, `baud=9600`, `slave_id=1`, `register=0`, `count=4` |
| Rol | **Maestro** de un único esclavo: `readHoldingRegisters(register, count)` (`ModbusManager.cpp:55`) |
| Librería | `4-20ma/ModbusMaster@^2.0.1` |
| Transceiver | `SEMA_MODBUS_ISOLATED=1` → **TD501D485H** (aislado); `=0` → **SN65HVD75DR** |
| Control DE/RE | Si `de_re != 0` se marca OUTPUT y se conmuta LOW→HIGH antes de transmitir y HIGH→LOW después (`preTransmission`/`postTransmission`) |
| Lectura | `GET /api/v1/modbus` → `{ready, result, values[]}` (sin autenticación); `result` es el código crudo de ModbusMaster |
| Config web | Página `/config/sensors`, bloque «Pines de buses»; `POST /api/v1/config/buses` acepta `enabled/rx/tx/de_re/uart/uart_port`; `baud/slave_id/register/count` solo por `PUT /api/v1/config` |
| Límites | Un esclavo, solo holding registers, `count` según el buffer de ModbusMaster (máx. 125 registros por trama Modbus); no hay escritura |

## 9. CAN / TWAI

| Aspecto | Detalle |
|---------|---------|
| Claves JSON | `can{enabled,tx,rx,speed}` |
| Defaults | `tx=5`, `rx=4`, `speed=500000` |
| Velocidades soportadas | 125 000 · 250 000 · 500 000 · 1 000 000 bps (`CanManager.cpp:12-19`); cualquier otro valor cae a 500 kbps |
| Modo | `TWAI_MODE_NORMAL`, filtro `ACCEPT_ALL`, sin modo escucha ni filtros por ID |
| Endpoints | `GET /api/v1/can` (recibe una trama, timeout 0), `POST /api/v1/can` (`{id,extd,data[]}`, máx. 8 bytes, timeout 1000 ms) |
| Requiere | Transceiver externo (p. ej. SN65HVD230/TCAN332) a 3,3 V + terminación de 120 Ω en los extremos del bus (datasheet) |
| Config web | `POST /api/v1/config/buses` acepta `can{enabled,tx,rx}`; `speed` solo por `PUT /api/v1/config` |

## 10. Ethernet

Dos caminos excluyentes por compilación (`EthernetManager.cpp:13-22`):

| Board | Chip | Capa física | Driver | `SEMA_NATIVE_ETH` |
|-------|------|-------------|--------|-------------------|
| `esp32doit-devkit-v1`, `esp32-wroom-32u` | MAC EMAC nativa | LAN8720A por RMII | `ETH.h` (Arduino, lwIP) | 1 |
| `esp32-s3-devkitc-1` | sin MAC | W5500 por SPI | `esp_eth` (ESP-IDF, lwIP) | 0 |

### 10.1 LAN8720A (RMII)

| Aspecto | Detalle |
|---------|---------|
| Pines fijos del EMAC | TXD0=19 · TXD1=22 · TX_EN=21 · RXD0=25 · RXD1=26 · CRS_DV=27 · RX_ER=13 |
| Pines configurables | `ethernet.mdc` (23), `ethernet.mdio` (18), `ethernet.phy_addr` (1), `ethernet.power` (-1) |
| Reloj | 50 MHz entrando por **GPIO0** (`ETH_CLOCK_GPIO0_IN`, `EthernetManager.cpp:39-40`) |
| Reserva | Con `ethernet.enabled=true` el firmware reserva los 10 pines RMII (`HttpServer.cpp:1434-1443`) |
| Estado | `GET /api/v1/network → ethernet{enabled,connected,ip}` |

### 10.2 W5500 (SPI)

| Aspecto | Detalle |
|---------|---------|
| Claves JSON | `ethernet{enabled,mdc,mdio,phy_addr,power,cs,rst,irq,sck,miso,mosi}` (por web solo `enabled` y `cs`) |
| Defaults | `cs=5`, `rst=-1`, `irq=4`, `phy_addr=1` |
| Secuencia | `spi_bus_initialize(SPI3_HOST)` → `spi_bus_add_device` (20 MHz, modo 0) → `esp_eth_mac_new_w5500` → `esp_eth_phy_new_w5500` → `esp_eth_driver_install` → `esp_netif_new` + `esp_eth_set_default_handlers` + `esp_netif_attach` → `esp_eth_start` (DHCP) |
| DHCP | Gestionado por lwIP, sin mantenimiento en `loop()` |

## 11. LoRa (SX1262)

| Aspecto | Detalle |
|---------|---------|
| Claves JSON | `lora{enabled,cs,rst,dio1,busy,frequency,bandwidth,spreading,coding_rate,tx_power}` |
| Módulo | Silicontra **SX1262PATR8-GC** (`ConfigManager.hpp:172`) |
| Defaults | `cs=10`, `rst=14` (web) / `32` (parseo), `dio1=26`, `busy=27`, `frequency=915 MHz`, `bandwidth=125 kHz`, `spreading=7`, `codingRate=5` (4/5), `txPower=14 dBm` |
| Driver | RadioLib 6.x: `Module(cs, dio1, rst, busy)` + `SX1262::begin(...)` con sync word privada, y `startReceive()` al terminar |
| Bus | SPI de Arduino (14/12/15 en WROOM; 12/13/11 en S3); CS 10 |
| Endpoints | `GET /api/v1/lora` (intenta recibir hasta 64 bytes), `POST /api/v1/lora` (`{data:[...]}`, máx. 64 bytes) |
| Límites | No hay configuración de CRC, sync word, preamble ni modo; una sola llamada `transmit` bloqueante por envío; sin CAD |
| Reservas | En el perfil fijo `esp32-wroom-32u`, `dio1=26` y `busy=27` chocan con RMII RXD1/CRS_DV |

## 12. Zigbee (CC2652P2 / ZNP)

| Aspecto | Detalle |
|---------|---------|
| Claves JSON | `zigbee{enabled,rx,tx,uart,uart_port,baud}` |
| Módulo | **RF-BM-2652P2** (CC2652P2) con firmware ZNP (`ConfigManager.hpp:186`) |
| Defaults | `rx=16` (header) / `18` (parseo+web), `tx=17` (header) / `19` (parseo+web), `baud=115200` |
| Puerto | `Serial1`, 8N1 |
| Trama | ZNP: `SOF(0xFE) LEN CMD0 CMD1 payload FCS` con FCS = XOR de LEN, CMD0, CMD1 y payload |
| Comandos | `SYS_RESET` (0x4100) al arrancar; `AF_DATA_REQUEST` (0x2401) para enviar; `AF_INCOMING_MSG` (0x4481) para recibir |
| Endpoints | `GET /api/v1/zigbee` → `{ready,received,src,data[]}` (máx. 128 bytes), `POST /api/v1/zigbee` (`{destination,data[]}`, máx. 110 bytes) |
| Límites | Envío a un solo endpoint/cluster fijos (src/dst endpoint 1, cluster 0x0001), sin coordinador propio: el módulo debe estar ya en una red |
| Reservas | 18/19 son MDIO/TXD0 del RMII en WROOM y el host SPI del W5500 en S3: Zigbee y Ethernet son excluyentes con los defaults |

## 13. Recursos compartidos y límites

| Recurso | Lo usan | Consecuencia práctica |
|---------|---------|----------------------|
| `Wire` (I2C0) | 12 drivers I²C + ADS1115 + MCP23017 | Pines efectivos = los del último `begin()`; sin `address` por dispositivo |
| `Serial2` | Modbus y PMS5003 | Excluyentes entre sí |
| `PCNT_UNIT_0` | Anemómetro y pluviómetro | Excluyentes entre sí (o lectura mezclada) |
| Bus SPI de Arduino | LoRa, microSD, shift registers | Sin mutex: el orden depende de las tareas |
| SPI3 (S3) | W5500 | Separado del bus de Arduino a propósito (`HwProfile.hpp:126-137`) |
| GPIO0 | RMII REF_CLK / botón BOOT / `modbus.de_re` fijo en 32U | Triple uso según la placa |
| GPIO4 | 1-Wire por defecto, `sd_cs`, `can.rx`, `ethernet.irq` (S3) | Cuádruple uso: hay que elegir |
| GPIO5 | `can.tx`, `SEMA_CS_ETHERNET_W5500`, `mcp23s17_cs` en demo | Elegir según el bus activo |

## 14. Fichas de conexión

### 14.1 LAN8720A (PHY Ethernet RMII)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | RMII hacia el EMAC del ESP32 (MDC/MDIO para gestión; PHY addr 1) |
| Tensión | 3,3 V (datasheet); niveles RMII 3,3 V |
| Resistencia | 49,9 Ω de terminación en el par TX y en el par RX hacia el RJ45 con transformador (datasheet); strap PHYAD0 para dirección 1 |
| Capacitor | 100 nF por pin de alimentación + 10 µF de bulk (datasheet) |
| Conexión | MDC→GPIO23 · MDIO→GPIO18 · TXD0→GPIO19 · TXD1→GPIO22 · TX_EN→GPIO21 · RXD0→GPIO25 · RXD1→GPIO26 · CRS_DV→GPIO27 · RX_ER→GPIO13 · REF_CLK 50 MHz→GPIO0 · `ethernet.power` = -1 (sin control de alimentación) |
| Notas | El reloj de 50 MHz debe venir de la PHY u oscilador externo; SEMA no controla el reset de la PHY |

### 14.2 W5500 (Ethernet SPI)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | SPI, modo 0, 20 MHz, CS dedicado |
| Tensión | 3,3 V (datasheet) |
| Resistencia | 49,9 Ω en TX±/RX± hacia el RJ45 con magnetics; pull-up de 10 kΩ en `/RST` si se usa reset externo (datasheet) |
| Capacitor | 100 nF por alimentación + 10 µF de bulk; cristal de 25 MHz con 2 × 22 pF (datasheet) |
| Conexión | SCK→GPIO18 · MISO→GPIO19 · MOSI→GPIO21 · CS→GPIO5 · IRQ→GPIO4 · RST = -1 |
| Notas | Solo placas sin MAC nativa (S3). El host SPI3 debe ser distinto del SPI de RadioLib |

### 14.3 SX1262 (LoRa, módulo SX1262PATR8-GC)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | SPI (bus de Arduino) + CS/RST/DIO1/BUSY |
| Tensión | 3,3 V (datasheet); el módulo no tolera 5 V en las líneas de control |
| Resistencia | Ninguna obligatoria en la placa; la antena debe presentar 50 Ω (datasheet) |
| Capacitor | 100 nF + 10 µF junto al pin de alimentación (datasheet); picos de TX de hasta ~120 mA |
| Conexión | SCK→14 · MISO→12 · MOSI→15 · CS→10 · RST→14 (web) / 32 · DIO1→26 · BUSY→27 (WROOM). En S3: SCK→12, MISO→13, MOSI→11 |
| Notas | `dio1=26` y `busy=27` **no son válidos en ESP32-S3** (flash SPI interna) |

### 14.4 RF-BM-2652P2 (Zigbee / ZNP)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | UART 115200 8N1 (`Serial1`) |
| Tensión | 3,3 V (datasheet) |
| Resistencia | Sin resistencias obligatorias; el módulo trae su antena/balun |
| Capacitor | 100 nF + 10 µF de desacople (datasheet); picos de TX considerables |
| Conexión | RX del módulo←TX del ESP32 (GPIO19 por defecto) · TX del módulo→RX del ESP32 (GPIO18 por defecto) · GND común |
| Notas | 802.15.4 **no** lo aporta el ESP32: lo aporta este módulo externo. `Capability::Ieee802154` nunca se declara en el firmware |

### 14.5 TD501D485H (transceiver RS485 aislado) — default

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | UART (`Serial2`) + control DE/RE |
| Tensión | Módulo aislado de 5 V en el lado lógico (verificar la hoja de datos del módulo): la lógica del ESP32 es 3,3 V |
| Resistencia | 120 Ω de terminación en los extremos del bus RS485 (datasheet); polarización de fallo (pull-up/pull-down) si el bus queda sin maestro |
| Capacitor | 100 nF + 10 µF en la alimentación del módulo (datasheet) |
| Conexión | RXD del módulo←GPIO17 (TX del ESP32) · TXD del módulo→GPIO16 (RX del ESP32) · DE/RE→`modbus.de_re` (0 = sin control) |
| Notas | Aislamiento galvánico ≥ 2,5 kV (datasheet); con `de_re = 0` el módulo debe autodirigirse o quedar siempre en TX |

### 14.6 SN65HVD75DR (transceiver RS485 sin aislar)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | UART (`Serial2`) + DE/RE activo-alto |
| Tensión | 3,0-3,6 V; lógica 3,3 V compatible (datasheet) |
| Resistencia | 120 Ω de terminación en los extremos; pull-up/pull-down de polarización (datasheet) |
| Capacitor | 100 nF de desacople (datasheet) |
| Conexión | RO→GPIO16 · DI→GPIO17 · DE y /RE unidos a `modbus.de_re` (o a VCC si no se controla) |
| Notas | Se selecciona con `-D SEMA_MODBUS_ISOLATED=0`; comparte el mismo código de conmutación que el aislado |

### 14.7 microSD (SPI)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | SPI (bus de Arduino) + CS |
| Tensión | 3,3 V; los módulos con regulador y level-shifter aceptan 5 V, el socket directo **no** |
| Resistencia | Pull-up de 10 kΩ en CS y en las líneas del bus si el módulo no los trae (datasheet) |
| Capacitor | 100 nF + 10 µF junto al socket (datasheet) |
| Conexión | CS→GPIO4 · SCK→GPIO14 · MISO→GPIO12 · MOSI→GPIO15 (WROOM/WROOM-32U); en S3 CS→4, SCK→12, MISO→13, MOSI→11 |
| Notas | El CS por defecto (4) comparte pin con 1-Wire por defecto y con `can.rx` |

---

## Ver también

- [Guía de pines](Guia-de-pines.md) · [Expansores de entrada/salida](Expansores-de-entrada-salida.md) · [Compatibilidad](Compatibilidad.md)
- [Hardware y conexiones](Hardware-y-Conexiones.md) · [Sensores](Sensores.md) · [Actuadores y salidas](Actuadores-y-Salidas.md)
- [Conectividad y red](Conectividad-y-red.md) · [Comunicaciones remotas](Comunicaciones-remotas.md) · [Referencia de configuración](Referencia-configuracion.md)
