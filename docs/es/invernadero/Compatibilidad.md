---
tags:
  - invernadero
  - hardware
---

# Compatibilidad de hardware

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

Catálogo completo de placas, buses, sensores, actuadores y expansores compatibles
con el firmware.

## 1. Placas ESP32 compatibles

El firmware usa un único código fuente con guardas `CONFIG_IDF_TARGET_*`.
`pio run` compila el entorno por defecto; para otro target: `pio run -e <env>`.

| Placa (env) | MCU | Núcleos | Frecuencia | WiFi | CAN/TWAI | Temp. interna | Flash típica |
|-------------|-----|---------|------------|------|----------|---------------|--------------|
| `esp32doit-devkit-v1` | ESP32 | 2 | 240 MHz | 802.11 b/g/n | ✅ | ✅ | 4–16 MB |
| `esp32-s3-devkitc-1` | ESP32-S3 | 2 | 240 MHz | 802.11 b/g/n + BLE | ✅ | ✅ | 8–16 MB |
| `esp32-s2-saola-1` | ESP32-S2 | 1 | 240 MHz | 802.11 b/g/n | ✅ | ❌ | 4 MB |
| `esp32-c3-devkitm-1` | ESP32-C3 | 1 | 160 MHz (RISC-V) | WiFi + BLE | ❌ | ✅ | 4 MB |
| `esp32-c6-devkitc-1` | ESP32-C6 | 1 | 160 MHz (RISC-V) | WiFi 6 + BLE + 802.15.4 | ❌ | ✅ | 8 MB |
| ESP32-C5 (cuando el core lo exponga) | ESP32-C5 | 2 | 240 MHz (RISC-V) | WiFi 6 + BLE | ❌ | ✅ | — |

### Notas de compatibilidad

- **CAN/TWAI** solo existe en ESP32, ESP32-S2 y ESP32-S3. En C3/C5/C6 el driver
  queda inerte (`GH_HAS_TWAI=0`).
- **Sensor interno de temperatura** no existe en ESP32-S2; en los demás se usa
  solo como diagnóstico del silicio (nunca como medición ambiental).
- **UART/RS485**: los pines por defecto (16/17/14) son de la ESP32 clásica; en
  S2/S3/C3/C6 los pines deben reasignarse según el mapeo de cada placa.
- **Flash**: el firmware completo (~1,25 MB) solo entra en particiones de 8 MB o
  16 MB. Los módulos de 4 MB (S2/C3 devkit) requieren recortar funciones o usar
  N8/N16.

## 2. Buses soportados

| Bus | Uso | Notas |
|-----|-----|-------|
| I²C | SHT31, AHT20, ADS1115, BH1750, SCD4x, MCP23017 | pull-up 4,7 kΩ a 3,3 V |
| SPI (bit-banged) | 74HC595 | DATA=23, CLOCK=18, LATCH=5 |
| SPI (nativo) | MCP23S17, ADC, SD, W5500 | SCK=18, MISO=19, MOSI=23 |
| 1-Wire | DS18B20 | GPIO 4, pull-up 4,7 kΩ |
| UART / RS485 | Modbus RTU | RX=16, TX=17, DE=14 |
| GPIO | pulsos, flotadores, parada | pull-up interno |
| CAN/TWAI | nodos remotos (V9) | transceptor externo SN65HVD23X |

## 3. Sensores compatibles

| Magnitud | Modelo | Interfaz | Dirección / pin | Notas |
|----------|--------|----------|-----------------|-------|
| Temperatura interior | SHT31 | I²C | 0x44 | ±0,2 °C / ±2 %RH |
| Humedad interior | SHT31 | I²C | 0x44 | 0–100 %RH |
| Temperatura exterior | AHT20 | I²C | 0x38 | alternativa económica |
| Humedad exterior | AHT20 | I²C | 0x38 | |
| Temperatura 1-Wire | DS18B20 | 1-Wire | GPIO 4 | ±0,5 °C, -10..85 °C, sumergible |
| Humedad de suelo | Capacitivo | ADC (ADS1115) | A0–A3 | calibración por zona |
| Iluminación | BH1750 | I²C | 0x23 | lux (no asumir lux = PPFD) |
| CO₂ | SCD40 / SCD41 | I²C | 0x62 | NDIR |
| Caudal | Caudalímetro | pulsos | GPIO 34 | ~450 pulsos/L |
| Nivel de tanque | Ultrasónico | GPIO | TRIG 25 / ECHO 26 | + flotadores 32/33 |
| Lluvia | Pluviómetro | pulsos | GPIO 35 | mm por pulso configurable |
| Viento | Anemómetro | pulsos | GPIO 36 | km/h por pulso configurable |
| pH | Analógico / Modbus | ADS1115 o RS485 | — | calibración 4/7/10 |
| EC (conductividad) | Analógico / Modbus | ADS1115 o RS485 | — | µS → mS/cm |

## 4. Actuadores (roles)

`pump` · `valve` (hasta 8) · `fan` · `extractor` · `heater` · `humidifier` ·
`light` · `window_open/close` · `roof_open/close` · `shade_open/close` · `alarm`.

Jerarquía: `EMERGENCIA > SEGURIDAD > MANUAL > AUTOMÁTICO > PROGRAMACIÓN`.

## 5. Expansores y ADC

| Dispositivo | Interfaz | Canales | Uso |
|-------------|----------|---------|-----|
| 74HC595 / 74HCT595 | SPI bit-banged | 8 × N (daisy-chain) | salidas digitales / soft-PWM |
| 74HC165 | SPI bit-banged | 8 × N | entradas digitales |
| MCP23017 | I²C (0x20–0x27) | 16 E/S | expansor I²C |
| MCP23S17 | SPI | 16 E/S | expansor SPI |
| ADS1115 | I²C (0x48–0x4B) | 4 canales 16 bits | ADC para suelo/pH/EC |
| MCP3008 | SPI | 8 canales 10 bits | ADC externo |
| MCP3208 | SPI | 8 canales 12 bits | ADC externo |

> El 74HC595 **no** alimenta cargas: solo da la señal lógica a drivers
> (MOSFET/SSR/relé). Para PWM real usar LEDC (GPIO directo) o un controlador
> dedicado.

## 6. Direcciones I²C (resumen)

| Dispositivo | Dirección |
|-------------|-----------|
| SHT31 | 0x44 |
| AHT20 | 0x38 |
| ADS1115 | 0x48–0x4B |
| BH1750 | 0x23 |
| SCD40/SCD41 | 0x62 |
| MCP23017 | 0x20–0x27 |

## 7. Pines con restricciones (ESP32 clásico)

- **Solo entrada**: GPIO 34, 35, 36, 39 (sin pull-up interno en algunos).
- **Strapping** (definen modo de arranque, evitar para E/S generales): 0, 2, 12, 15.
- **Usados por el firmware**: 4, 5, 14, 16, 17, 18, 21, 22, 23, 25, 26, 27, 32, 33, 34, 35, 36.
