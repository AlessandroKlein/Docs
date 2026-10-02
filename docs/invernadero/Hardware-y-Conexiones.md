# Hardware y conexiones

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

Mapa de pines por defecto (`include/core/PinMap.hpp`). Los pines se asignan por
instalación y varían según sensores/expansores.

## Pines

| Función | GPIO | Tipo | Notas |
|---------|------|------|-------|
| I²C SDA / SCL | 21 / 22 | bidireccional | pull-up 4,7 kΩ a 3,3 V |
| 74HC595 DATA/CLOCK/LATCH | 23 / 18 / 5 | salida | SPI bit-banged |
| SPI SCK/MISO/MOSI | 18 / 19 / 23 | SPI nativo | MCP23S17/ADC/SD/W5500 |
| 1-Wire | 4 | datos | pull-up 4,7 kΩ a 3,3 V |
| Caudalímetro / Pluviómetro / Anemómetro | 34 / 35 / 36 | entrada | solo entrada |
| Tanque TRIG / ECHO | 25 / 26 | salida/entrada | ultrasónico |
| Flotador bajo / alto | 32 / 33 | entrada | pull-up interno |
| Parada de emergencia | 27 | entrada | pull-up interno |
| RS485 RX / TX / DE | 16 / 17 / 14 | UART | RO / DI / control dirección |

## Direcciones I²C

| Dispositivo | Dirección |
|-------------|-----------|
| SHT31 | 0x44 |
| AHT20 | 0x38 |
| ADS1115 | 0x48–0x4B |
| BH1750 | 0x23 |
| SCD40/SCD41 | 0x62 |
| MCP23017 #1 / #2 | 0x20 / 0x21 |

## Ficha de conexión (ejemplo)

### SHT31 (temperatura y humedad)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x44 |
| Tensión | 2,4–5,5 V (alimentado a 3,3 V) |
| Resistencia | pull-up 4,7 kΩ en SDA y SCL |
| Capacitor | 100 nF de desacople entre VDD y GND |
| Conexión | VDD→3,3 V · GND→GND · SDA→21 · SCL→22 |

> Evitar GPIO de strapping (0, 2, 12, 15 en ESP32 clásico).
