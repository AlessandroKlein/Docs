---
tags:
  - sema
  - bom
  - materiales
---

# Materiales (BOM)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

BOM completa y por bloques para armar una estación SEMA, con referencias reales del
repositorio. Cada fila indica si el componente es **obligatorio** (mínimo para que un
entorno de PlatformIO arranque y publique mediciones) u **opcional** (habilita una
función concreta). Los modelos y versiones de librería salen de `platformio.ini`; los
pines y transceivers, de `include/hw/HwProfile.hpp` y `include/core/BoardProfile.hpp`;
los instrumentos, del `README.md` (secciones 12 a 37).

## 1. Núcleo (obligatorio)

| Componente | Referencia / valor | Cant. | Entorno / nota | Origen |
|------------|--------------------|-------|----------------|--------|
| Placa ESP32 | `esp32doit-devkit-v1` (ESP32-WROOM, 4 MB) | 1 | Entorno **default** de PlatformIO; MAC Ethernet nativa | REPO (`platformio.ini`) |
| — alternativa S3 | `esp32-s3-devkitc-1` (8 MB) | 1 | Para W5500 por SPI + FSPI libre para LoRa/SD | REPO (`platformio.ini`) |
| — alternativa PCB | `esp32-wroom-32u` (16 MB, `board = esp32dev`) | 1 | PCB futura, `SEMA_PINS_FROM_FILE=1`, pines fijos | REPO (`platformio.ini`) |
| Fuente 5 V | 5 V / 2 A con conector acorde a la placa (micro-USB o USB-C) | 1 | Alimenta la placa y los módulos de 5 V | REC |
| Cable USB de datos | micro-USB o USB-C según placa | 1 | Flasheo y alimentación de banco | REC |
| Protoboard o PCB | 830 puntos o placa a medida | 1 | Montaje del prototipo | REC |
| Regulador 3,3 V | Módulo DC/DC 3,3 V ≥ 1 A | 1 | Necesario solo si alimentás desde batería en vez de USB | REC |
| Fuente 12 V | 12 V ≥ 2 A (o batería 12 V + cargador) | 1 | Para el divisor de batería y cargas de 12 V | REC |

## 2. Sensores (todos opcionales, se habilitan en `sensors[]`)

| Componente | Interfaz | Dirección | Cant. típica | Nota | Origen |
|------------|----------|-----------|--------------|------|--------|
| BME280 | I²C | 0x76 / 0x77 | 1 | Temperatura, humedad y presión (`EXT` en el catálogo por defecto) | REPO |
| SHT40 | I²C | 0x44 | 1 | Temperatura y humedad (`INT` por defecto) | REPO |
| SHT31 | I²C | 0x44 | 0–1 | Alternativa al SHT40 | REPO |
| AHT20 | I²C | 0x38 | 0–1 | Alternativa económica (`AUX` por defecto) | REPO |
| BMP280 | I²C | 0x76 | 0–1 | Solo presión y temperatura | REPO |
| BH1750 | I²C | 0x23 | 1 | Iluminancia en lux (`LUX` por defecto) | REPO |
| VEML6075 | I²C | 0x10 | 0–1 | UVA, UVB e índice UV | REPO |
| SCD30 | I²C | 0x61 | 0–1 | CO₂ NDIR, temperatura y humedad | REPO |
| SGP30 | I²C | 0x58 | 0–1 | eCO₂ y TVOC (calidad de aire) | REPO |
| AS3935 | I²C | 0x03 | 0–1 | Detector de rayos; idealmente con antena propia | REPO |
| ADS1115 | I²C | 0x48 | 0–2 | ADC externo de 16 bits; la config usa `pin` = canal A0–A3 | REPO |
| DS18B20 | 1-Wire | ROM propia | 1–N | Sondas de temperatura; varias por bus (`SOIL` por defecto) | REPO |
| PMS5003 | UART | — | 0–1 | Material particulado PM1/PM2.5/PM10 | REPO |
| Anemómetro **WH-SP-WS01** | PCNT | — | 0–1 | Velocidad de viento (pulsos) | REPO (README §19) |
| Veleta **WH-SP-WD** | ADC | — | 0–1 | Dirección de viento (red de resistencias) | REPO (README §19) |
| Pluviómetro **WH-SP-RG** | PCNT | — | 0–1 | Lluvia (pulsos) | REPO (README §20) |
| Sensor de CO analógico | ADC | — | 0–1 | `model = "CO"` (MQ-7, MICS-5524, ZE07-CO, 3-ULPSM-CO 968-001) | REPO (README §18) |
| Piranómetro / sensor solar | ADC o ADS1115 | — | 0–1 | `model = "SOLAR"` | REPO (README §21) |
| Sensor de suelo capacitivo v1.2 | ADC o ADS1115 | — | 0–1 | Humedad de suelo (README §22) | REPO (README §22) |
| Sensor de lluvia (lluvia/no lluvia) | Digital | — | 0–1 | Entrada en `gpio[]` o en el MCP23017 | REPO (README §26) |

## 3. Pasivos y protecciones

| Componente | Valor | Cant. | Uso | Origen |
|------------|-------|-------|-----|--------|
| Resistencia | 4,7 kΩ | 1 | Pull-up del bus 1-Wire (DS18B20), obligatoria | REPO (README §12) |
| Resistencia | 4,7 kΩ | 2 (+2 por cada dispositivo I²C extra) | Pull-up de SDA y SCL a 3,3 V | REPO/REC |
| Resistencia | 10 kΩ | 2 | Pull-up del anemómetro y de la veleta (una por línea) | REPO (README §19) |
| Resistencia | 100 kΩ | 1 | R1 del divisor de batería | NO VERIF. (el repo fija el factor 11,0, no los valores) |
| Resistencia | 10 kΩ | 1 | R2 del divisor de batería | NO VERIF. (idem) |
| Resistencia | 120 Ω | 2 | Terminación del bus RS485 (ambos extremos) | REC |
| Resistencia | 120 Ω | 2 | Terminación del bus CAN (ambos extremos) | REC |
| Resistencia | 220–470 Ω | 1 por LED | Limitadora del LED indicador | REC |
| Resistencia | 680 Ω | 2 | Polarización (bias) del bus RS485 si ningún nodo la trae | REC |
| Capacitor cerámico | 100 nF | 1 por dispositivo | Desacople VDD–GND de cada chip | REC |
| Capacitor de poliéster | 100 nF | 1 | Filtro de la línea del anemómetro (`DATA`→GND) | REPO (README §19) |
| Capacitor | 1 µF | 1 | Filtro de la línea de la veleta (`DATA`→GND) | REPO (README §19) |
| Capacitor electrolítico | 10 µF | 1 por módulo | Bulk junto a los módulos de radio (LoRa/Zigbee) y Ethernet | REC |
| Diodo | 1N4148 / 1N4007 | 1 por bobina | Flyback de relés manejados con transistor propio | REC |
| Diodo TVS | SMBJ 6,8CA o similar | 2 | Protección de A/B en RS485 y CANH/CANL | REC |
| Fusible | 2 A (vidrio o rearmable) | 1 | Protección de la entrada de batería | REC |
| Bornera / regleta | 2,54 mm o 3,5 mm | según instalación | Conexión de sensores de campo | REC |
| Cable dupont / jumper | macho-hembra y hembra-hembra | 1 juego | Prototipado | REC |
| Cable apantallado | 2–4 conductores | según distancia | Tramos largos de 1-Wire, RS485 y pulsos | REC |
| Precanalizado / caja IP65 | — | 1 | Gabinete exterior con pasacables | REC |

## 4. Actuadores y salidas

| Componente | Referencia / valor | Cant. | Nota | Origen |
|------------|--------------------|-------|------|--------|
| Módulo de relé | 1, 2, 4 u 8 canales, con optoacoplador y transistor | 1 | Salidas digitales; `gpio[].mode = "output"` | REPO |
| **MCP23017** | Expansor I²C de 16 GPIO | 0–1 | `gpio[].expander_addr` = dirección (0x20 = 32 decimal); la librería es `Adafruit MCP23X17@^2.3.0` | REPO |
| **MCP23S17** | Expansor SPI de 16 GPIO | 0–1 | Solo configuración (`mcp23s17_cs` + `mcp23s17_pins[16]`); sin driver de escritura | REPO |
| **74HC595** | Shift register de 8 salidas | 0–N | Requiere `SEMA_USE_SHIFT=1`, que **no** está activado en los entornos por defecto | REPO |
| **74HC165** | Shift register de 8 entradas | 0–N | Idem | REPO |
| **MAX14830** | Expansor UART por SPI (4 puertos) | 0–1 | Solo se configura el CS (`spi_expanders[]`); ningún driver lo usa | REPO |
| **SC18IS602B** | Expansor I²C por SPI | 0–1 | Idem | REPO |
| ULN2803A | Array Darlington de 8 canales | 0–1 | Alternativa al módulo de relé para cargas de baja tensión | REC |
| MOSFET de nivel lógico | IRLZ44N / AO3400 | 0–N | Conmutación de cargas DC (tiras LED, bombas de 12 V) | REC |
| Relé de estado sólido | SSR 3–32 VDC de entrada | 0–N | Conmutación silenciosa y sin desgaste mecánico | REC |
| Fuente de carga | 5 V / 12 V / 220 V según actuador | 1 | Fuente externa separada de la lógica | REC |

## 5. Buses y comunicaciones

| Componente | Referencia exacta | Bus | Cant. | Estado en el firmware | Origen |
|------------|-------------------|-----|-------|-----------------------|--------|
| Transceiver RS485 aislado | **TD501D485H** | RS485/Modbus RTU | 1 | `SEMA_MODBUS_ISOLATED=1` (default); usa `modbus.rx/tx/de_re` | REPO (`HwProfile.hpp`) |
| Transceiver RS485 no aislado | **SN65HVD75DR** | RS485/Modbus RTU | 1 | `SEMA_MODBUS_ISOLATED=0` | REPO (`HwProfile.hpp`) |
| Transceiver CAN | Familia **SN65HVD23X** (3,3 V) | CAN/TWAI | 1 | `SEMA_USE_CAN=1`; `can.tx/rx`, 500 kbps por defecto | REPO (README §25) |
| Módulo LoRa | **SX1262PATR8-GC** (Silicontra, SX1262) | SPI | 1 | `SEMA_USE_LORA=1`; CS/RST/DIO1/BUSY, 915 MHz por defecto | REPO |
| Antena LoRa | 915 MHz (o 868 MHz) con conector del módulo | — | 1 | Nunca energizar el módulo sin antena | REC |
| Módulo Zigbee | **RF-BM-2652P2** (CC2652P2, firmware ZNP) | UART | 1 | `SEMA_USE_ZIGBEE=1`; 115200 baudios por defecto | REPO (README §35) |
| PHY Ethernet nativa | **LAN8720A** | RMII | 1 | `SEMA_NATIVE_ETH=1` (WROOM/WROOM32U); MDC 23 / MDIO 18, PHY addr 1 | REPO |
| Módulo Ethernet SPI | **W5500** | SPI | 1 | `SEMA_NATIVE_ETH=0` (S3); CS 5, IRQ 4 | REPO |
| Jack RJ45 | Con transformadores/magnéticos integrados | — | 1 | Para LAN8720A o W5500 | REC |
| microSD | Módulo lector SPI + tarjeta | SPI | 0–1 | `storage.sd_enabled` + `storage.sd_cs` (CS 4); guarda el histórico | REPO |
| Cable UTP + conectores | Cat 5e/6, RJ45 | — | según tramo | Ethernet | REC |

> `SN65HVD23X` es la familia que el README §25 nombra para **CAN**; `SN65HVD75DR` es un
> transceiver **RS485**. No son intercambiables.

## 6. Energía (opcional)

| Componente | Valor / referencia | Cant. | Nota | Origen |
|------------|--------------------|-------|------|--------|
| Panel solar | 12 V, potencia según consumo | 1 | README §36: panel → controlador → batería → SEMA | REPO |
| Controlador de carga solar | PWM o MPPT para 12 V | 1 | Protege la batería | REC |
| Batería | 12 V (plomo-ácido o LiFePO4) | 1 | La tensión se mide por el divisor 11:1 | REPO |
| DC/DC 5 V | Buck 12 V→5 V ≥ 2 A | 1 | Alimenta la placa y periféricos de 5 V | REC |
| DC/DC 3,3 V | Buck 12 V→3,3 V ≥ 1 A | 1 | Alimentación alternativa sin USB | REC |
| Sensor de corriente | INA219 / shunt + ADS1115 | 0–1 | Para las magnitudes de corriente que menciona el README §36 (no hay driver específico) | REC |

> El README §36 lista tensión/corriente/potencia de batería y panel, pero el firmware
> solo tiene drivers para **tensión** (`ADC`) y para el ADC externo: corriente y
> potencia quedan como medición derivada o pendientes.

## 7. Herramientas

| Herramienta | Uso | Origen |
|-------------|-----|--------|
| Multímetro | Verificar 3,3 V/5 V, continuidad, resistencia de pull-ups, consumo | REC |
| Analizador lógico (o el propio `/api/v1/diagnostics`) | Depurar I²C/SPI/UART | REC |
| Osciloscopio (deseable) | Verificar PWM, flancos del PCNT y calidad del bus RS485/CAN | REC |
| Estación de soldadura + estaño | Armado de PCB y módulos | REC |
| Pinza de corte y pelacables | Cables de campo | REC |
| PC con PlatformIO | Compilar y flashear los entornos de `platformio.ini` | REPO |
| Fuente de banco ajustable | Probar el consumo y el divisor de batería | REC |

## 8. Dependencias de software (fijadas por `platformio.ini`)

| Librería | Versión | Uso | Origen |
|----------|---------|-----|--------|
| `bblanchon/ArduinoJson` | ^6.21.5 | Config, API, eventos | REPO |
| `adafruit/Adafruit BME280 Library` | ^2.2.2 | BME280 | REPO |
| `adafruit/Adafruit Unified Sensor` | ^1.1.14 | Base de sensores Adafruit | REPO |
| `adafruit/Adafruit BusIO` | ^1.14.5 | I²C/SPI común | REPO |
| `adafruit/Adafruit MCP23017 Arduino Library` | ^2.3.0 | MCP23017 | REPO |
| `adafruit/Adafruit SHT4x Library` | ^1.0.0 | SHT40 | REPO |
| `adafruit/Adafruit SHT31 Library` | ^2.2.0 | SHT31 | REPO |
| `adafruit/Adafruit AHTx0` | ^2.0.0 | AHT20 | REPO |
| `adafruit/Adafruit BMP280 Library` | ^2.6.7 | BMP280 | REPO |
| `adafruit/Adafruit VEML6075 Library` | ^2.0.0 | VEML6075 | REPO |
| `adafruit/Adafruit SCD30` | ^1.0.6 | SCD30 | REPO |
| `adafruit/Adafruit SGP30 Sensor` | ^2.0.0 | SGP30 | REPO |
| `adafruit/Adafruit ADS1X15` | ^2.0.0 | ADS1115 | REPO |
| `adafruit/Adafruit PM25 AQI Sensor` | ^1.0.6 | PMS5003 | REPO |
| `sparkfun/SparkFun AS3935 Lightning Detector Arduino Library` | ^1.4.9 | AS3935 | REPO |
| `paulstoffregen/OneWire` | ^2.3.7 | Bus 1-Wire | REPO |
| `milesburton/DallasTemperature` | ^3.9.1 | DS18B20 | REPO |
| `knolleary/PubSubClient` | ^2.8 | MQTT | REPO |
| `links2004/WebSockets` | ^2.4.1 | WebSocket del dashboard | REPO |
| `4-20ma/ModbusMaster` | ^2.0.1 | Modbus RTU | REPO |
| `jgromes/RadioLib` | ^6.0.0 | LoRa SX1262 | REPO |

## 9. Qué es obligatorio y qué no

| Bloque | ¿Obligatorio? | Por qué |
|--------|---------------|---------|
| Placa ESP32 + fuente + cable USB | **Sí** | Sin placa no hay firmware; PlatformIO necesita el cable para flashear |
| Al menos un sensor | **Sí, para que haya datos** | Con `sensors[]` vacío se registra el catálogo por defecto de 6 sensores, pero si el hardware no está conectado ninguno queda `healthy` y no se publica nada |
| Pull-up 4,7 kΩ del 1-Wire | **Sí, si usás DS18B20** | Sin pull-up el bus 1-Wire no funciona (README §12) |
| Divisor de batería | **Solo para medir batería** | `BATT` es parte del catálogo por defecto; sin divisor la lectura no tiene sentido |
| Pull-ups I²C | **Sí, si el módulo no los trae** | El bus no arranca sin pull-ups |
| Estación de viento y pluviómetro | No | Habilitan `wind_*` y `rain` |
| Módulos de radio (LoRa, Zigbee) | No | Se pueden compilar pero quedan sin uso |
| Ethernet | No | La estación funciona por WiFi sin Ethernet |
| MCP23017 / MCP23S17 / 74HC595 / 74HC165 | No | Solo si necesitás más de los GPIO disponibles |
| Módulo de relé y cargas | No | Solo si hay algo que accionar |
| Panel solar, controlador y batería | No | Solo para operación autónoma |
| microSD | **Sí, para tener histórico** | El `HistoryStore` escribe solo en SD (`CS` 4 + bus SPI); sin SD, `append()` devuelve `false` y no hay histórico ni gráficas. `storage.backend` no selecciona otro backend |

---

## Ver también

- [Hardware y conexiones](Hardware-y-Conexiones.md) · [Sensores](Sensores.md) ·
  [Actuadores y salidas](Actuadores-y-Salidas.md) · [Guía de pines](Guia-de-pines.md)
- [Buses y periféricos](Buses-y-perifericos.md) ·
  [Expansores de entrada/salida](Expansores-de-entrada-salida.md) ·
  [Compatibilidad](Compatibilidad.md)
- [Energía y consumo](Energia-y-consumo.md) · [Instalación y mantenimiento](Instalacion-y-mantenimiento.md)
