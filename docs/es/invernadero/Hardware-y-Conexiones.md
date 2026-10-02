---
tags:
  - invernadero
  - hardware
  - pines
---

# Hardware y conexiones (fichas detalladas)

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

Mapa de pines y **fichas de conexión** de cada componente: interfaz, pines,
resistencias, capacitores, tensión, protecciones y estado seguro.

## 1. Pines por defecto

| Función | GPIO | Tipo | Notas |
|---------|------|------|-------|
| I²C SDA / SCL | 21 / 22 | bidi | pull-up 4,7 kΩ a 3,3 V |
| 74HC595 DATA/CLOCK/LATCH | 23 / 18 / 5 | salida | SPI bit-banged |
| 74HC165 DATA/CLOCK/LATCH | 12 / 13 / 14 | E/S | entradas en cascada |
| SPI SCK/MISO/MOSI | 18 / 19 / 23 | SPI nativo | MCP23S17/ADC/SD/W5500 |
| 1-Wire | 4 | datos | pull-up 4,7 kΩ a 3,3 V |
| Caudalímetro / Pluviómetro / Anemómetro | 34 / 35 / 36 | entrada | solo entrada |
| Tanque TRIG / ECHO | 25 / 26 | salida/entrada | ultrasónico |
| Flotador bajo / alto | 32 / 33 | entrada | pull-up interno |
| Parada de emergencia | 27 | entrada | pull-up interno |
| RS485 RX / TX / DE | 16 / 17 / 14 | UART | RO / DI / control dirección |

> Detalle editable de cada pin en [Guía de pines](Guia-de-pines.md).

## 2. Direcciones I²C

| Dispositivo | Dirección | Selección |
|-------------|-----------|-----------|
| SHT31 | 0x44 | fija |
| AHT20 | 0x38 | fija |
| ADS1115 | 0x48–0x4B | ADDR→GND/VDD/SDA/SCL |
| BH1750 | 0x23 | ADDR→GND (0x5C si VDD) |
| SCD40/SCD41 | 0x62 | fija |
| MCP23017 | 0x20–0x27 | A0/A1/A2 |

---

## Fichas de conexión

### SHT31 — temperatura y humedad interior

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x44 |
| Tensión | 2,4–5,5 V (alimentar a 3,3 V) |
| Resistencia | **pull-up 4,7 kΩ** en SDA y SCL (una vez por bus) |
| Capacitor | **100 nF** de desacople entre VDD y GND |
| Conexión | VDD→3,3 V · GND→GND · SDA→GPIO21 · SCL→GPIO22 |
| Estado seguro | No aplica (solo lectura) |

### AHT20 — temperatura y humedad exterior

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x38 |
| Tensión | 2,2–5,5 V (a 3,3 V) |
| Resistencia | pull-up 4,7 kΩ (compartido con el bus) |
| Capacitor | 100 nF de desacople |
| Protección | Si va a la intemperie: **filtro de membrana** y cápsula ventilada |

### DS18B20 — temperatura (1-Wire, sumergible)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | 1-Wire, GPIO4 |
| Tensión | 3,0–5,5 V (modo parásito o alimentado) |
| Resistencia | **pull-up 4,7 kΩ** entre DATA y 3,3 V (obligatoria) |
| Capacitor | 100 nF de desacople |
| Conexión | VDD→3,3 V · GND→GND · DQ→GPIO4 |
| Protección | Cable apantallado en tramos largos; sonda estanca en agua |

### ADS1115 — ADC de suelo / pH / EC

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x48 (ADDR→GND) |
| Tensión | 2,0–5,5 V (a 3,3 V, misma referencia que las entradas) |
| Resistencia | pull-up 4,7 kΩ; divisor si la señal supera 3,3 V |
| Capacitor | 100 nF de desacople; **100 nF** de filtro en cada entrada |
| Canales | A0..A3 → zonas 1..4 (o pH/EC) |
| Protección | Diodos de sujeción a 3,3 V para entradas externas |

#### Sensor capacitivo de suelo

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | Analógica → ADS1115 (A0..A3) |
| Alimentación | 3,3 V (¡no 5 V si la salida llega a 5 V!) |
| Calibración | `soil_dry` (aire) y `soil_wet` (agua) por zona |
| Protección | Sellar la electrónica del sensor contra humedad |

### BH1750 — iluminación

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x23 (ADDR→GND) |
| Tensión | 2,4–3,6 V (3,3 V) |
| Resistencia | pull-up 4,7 kΩ |
| Capacitor | 100 nF |
| Montaje | Sin obstrucciones ni sombras; horizontal |

### SCD40 / SCD41 — CO₂ NDIR

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x62 |
| Tensión | 2,4–5,5 V (3,3 V) |
| Resistencia | pull-up 4,7 kΩ |
| Capacitor | 100 nF |
| Notas | Necesita **ventilación** y ~30 s de precalentamiento |

### Caudalímetro — caudal de riego

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | Pulsos → GPIO34 (solo entrada) |
| Alimentación | 5 V típico (sensor), salida **optoacoplada a 3,3 V** |
| Resistencia | **divisor o optoacoplador** si la salida es 5–24 V |
| Capacitor | 100 nF de filtrado de ruido en la entrada |
| Protección | **OPTOACOPLADOR** obligatorio si viene de 24 V |
| Estado seguro | Si hay caudal sin orden → corta la bomba |

### Pluviómetro — lluvia

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | Pulsos (reed switch) → GPIO35 |
| Resistencia | pull-up 10 kΩ + 100 nF (antirrebote) |
| Calibración | `rain_mmp` (mm por pulso, típico 0,2794) |

### Anemómetro — viento

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | Pulsos → GPIO36 |
| Resistencia | pull-up 10 kΩ + 100 nF |
| Calibración | `wind_khpp` (km/h por pulso) |
| Uso | Cierre de techo por viento |

### Ultrasónico HC-SR04 — nivel de tanque

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | TRIG→GPIO25 (salida) · ECHO→GPIO26 (entrada) |
| Tensión | 5 V (ECHO entrega 5 V → **divisor 1k/2k** o level shifter) |
| Capacitor | 100 nF de desacople |
| Protección | **Divisor resistivo en ECHO** (obligatorio) |
| Estado seguro | Junto con flotadores corta la bomba si el nivel es bajo |

### Flotadores de nivel

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | GPIO32 (bajo) / GPIO33 (alto), contacto seco |
| Resistencia | **pull-up interno** activado por firmware |
| Protección | Contacto NC/NA según lógica; usar optoacoplador si es 24 V |
| Estado seguro | Nivel bajo → **bloquea la bomba** |

### Parada de emergencia

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | GPIO27, pulsador **NC** (normalmente cerrado) |
| Resistencia | pull-up interno |
| Protección | Pulsador tipo hongo con enclave |
| Estado seguro | Abrir → **todo a 0** (jerarquía máxima) |

### RS485 — Modbus RTU

| Elemento | Valor / Detalle |
|----------|-----------------|
| Transceptor | SN65HVD23X / SP3485 / MAX485 a 3,3 V |
| Conexión | RO→GPIO16 · DI→GPIO17 · DE/RE→GPIO14 |
| Terminación | **120 Ω** en ambos extremos del bus |
| Protección | TVS + resistencias de polarización (680 Ω) |
| Alimentación | 3,3 V; GND común |
| Estado seguro | Si no responde → sensor `TIMEOUT`/`CRC` |

### 74HC595 — expansión de salidas

| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | GPIO23 (DS) · GPIO18 (SHCP) · GPIO5 (STCP) |
| Cascada | `hc595_count` × 8 salidas (por defecto 4 → 32) |
| Resistencia | **330 Ω** en serie con cada salida (si maneja LED) |
| Capacitor | 100 nF de desacople por integrado |
| Protección | **Los 595 NO manejan cargas**: van a drivers (MOSFET/SSR/relé) |
| Estado seguro | `allOff()` al arrancar |

### Drivers de potencia

| Carga | Driver | Protección |
|-------|--------|-----------|
| Bomba DC | MOSFET + diodo flyback | flyback 1N4007/1N5819 |
| Electroválvulas | MOSFET/relé | diodo flyback |
| Cargas 220 V | SSR/relé/contactor | aislamiento + varistor |
| Ventiladores | MOSFET PWM | flyback |

### MCP23017 / MCP23S17 — expansores de E/S

| Elemento | Valor / Detalle |
|----------|-----------------|
| MCP23017 | I²C, 0x20–0x27 (A0/A1/A2) |
| MCP23S17 | SPI, CS desde el catálogo |
| Resistencia | pull-up 4,7 kΩ (I²C) |
| Capacitor | 100 nF de desacople |
| Uso | Canales 32..95 en `ActuatorManager` (device 0..3, pin 0..15) |

### SD y W5500 (SPI)

| Elemento | Valor / Detalle |
|----------|-----------------|
| Bus | SCK=18 · MISO=19 · MOSI=23 (compartido) |
| SD | CS propio; 100 nF de desacople |
| W5500 | CS = `eth_cs` (default 5); 100 nF + cristal según módulo |
| Notas | Comparten bus SPI: **un solo CS activo a la vez** |

---

> ⚠️ **Evitar GPIO de strapping** (0, 2, 12, 15 en ESP32 clásico) y recordar que
> GPIO 34/35/36 son **solo entrada**.
> Ver [Guía de pines](Guia-de-pines.md) y [Materiales](Materiales.md).
