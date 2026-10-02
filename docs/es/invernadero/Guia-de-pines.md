---
tags:
  - invernadero
  - hardware
  - pines
---

# Guía de pines (referencia completa)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

Todos los pines del firmware, con su clave JSON (editable en `/pins`), valor por
defecto, función, dirección y restricciones. Fuente: `include/core/PinConfig.hpp`.

!!! warning "Pines editables solo si el mapa no está bloqueado"
    Con `GH_PINS_LOCKED = 1` (PCB fabricada) la página `/pins` queda deshabilitada
    y `PUT /api/v1/pins` responde `403`. Con `0` (público) todo es editable.

## 1. Pines de E/S

| Clave JSON | Default | Función | Dirección | Notas |
|-----------|:-------:|---------|-----------|-------|
| `i2c_sda` | 21 | I²C datos (SDA) | bidi | pull-up 4,7 kΩ a 3,3 V |
| `i2c_scl` | 22 | I²C reloj (SCL) | bidi | pull-up 4,7 kΩ a 3,3 V |
| `hc595_mosi` | 23 | 74HC595 DATA (DS) | salida | SPI bit-banged |
| `hc595_sclk` | 18 | 74HC595 CLOCK (SHCP) | salida | SPI bit-banged |
| `hc595_latch` | 5 | 74HC595 LATCH (STCP) | salida | |
| `hc165_data` | 12 | 74HC165 Q7 | entrada | ⚠️ GPIO 12 es *strapping* |
| `hc165_clock` | 13 | 74HC165 SHCP | salida | |
| `hc165_latch` | 14 | 74HC165 PL | salida | ⚠️ comparte con `rs485_de` |
| `onewire` | 4 | DS18B20 (1-Wire) | bidi | pull-up 4,7 kΩ a 3,3 V |
| `flow_pin` | 34 | Caudalímetro | entrada | solo entrada, sin pull-up |
| `rain_pin` | 35 | Pluviómetro | entrada | solo entrada, sin pull-up |
| `wind_pin` | 36 | Anemómetro | entrada | solo entrada, sin pull-up |
| `tank_trig` | 25 | Ultrasónico TRIG | salida | |
| `tank_echo` | 26 | Ultrasónico ECHO | entrada | |
| `float_low` | 32 | Flotador nivel bajo | entrada | pull-up interno |
| `float_high` | 33 | Flotador nivel alto | entrada | pull-up interno |
| `emergency_stop` | 27 | Parada de emergencia | entrada | pull-up interno, contacto NC |
| `rs485_rx` | 16 | RS485 RO → UART RX | entrada | |
| `rs485_tx` | 17 | RS485 DI ← UART TX | salida | |
| `rs485_de` | 14 | RS485 DE/RE (dirección) | salida | ⚠️ comparte con `hc165_latch` |
| `spi_sck` | 18 | SPI nativo SCK | salida | ⚠️ comparte con `hc595_sclk` |
| `spi_miso` | 19 | SPI nativo MISO | entrada | |
| `spi_mosi` | 23 | SPI nativo MOSI | salida | ⚠️ comparte con `hc595_mosi` |

### Valores no-pin (mismo formulario)

| Clave | Default | Descripción |
|-------|:-------:|-------------|
| `i2c_freq` | 100000 | Frecuencia I²C (Hz) |
| `hc595_count` | 4 | Registros 74HC595 en cascada (× 8 salidas) |
| `hc165_count` | 1 | Registros 74HC165 en cascada (× 8 entradas) |

## 2. Direcciones I²C

| Clave JSON | Default | Dispositivo |
|-----------|:-------:|-------------|
| `i2c_addr_sht31` | `0x44` (68) | SHT31 (temp/hum interior) |
| `i2c_addr_aht20` | `0x38` (56) | AHT20 (temp/hum exterior) |
| `i2c_addr_ads1115` | `0x48` (72) | ADC suelo (ADS1115) |
| `i2c_addr_bh1750` | `0x23` (35) | BH1750 (luz) |
| `i2c_addr_scd41` | `0x62` (98) | SCD40/SCD41 (CO₂) |
| `i2c_addr_mcp23017` | `0x20` (32) | Expansor MCP23017 |

> En el formulario web las direcciones se cargan como **decimal**; `0x44` = 68.

## 3. Restricciones del ESP32 clásico

### Solo entrada (no usar como salida)

GPIO **34, 35, 36, 39** — no tienen salida ni pull-up interno. Por eso
caudalímetro, pluviómetro y anemómetro viven ahí.

### Pines *strapping* (evitar para E/S generales)

GPIO **0, 2, 12, 15** definen el modo de arranque. Un nivel incorrecto al
encender puede impedir el boot o forzar modos indeseados.

> ⚠️ El default de `hc165_data` es GPIO **12**, que es *strapping*. Si vas a usar
> 74HC165, conviene reasignarlo (por ejemplo 13/14 libres, o 39).

### Conflicto de ADC2 con WiFi

GPIO **0, 2, 4, 12–15, 25–27** usan ADC2, que **no se puede leer mientras WiFi
está activo** en el ESP32 clásico. Los sensores analógicos del proyecto van por
ADS1115 (I²C), evitando este problema.

## 4. Conflictos entre valores por defecto

Los defaults están pensados para el **PCB de referencia** donde el 74HC595 y el
SPI nativo **comparten** los pines 18/23 (uso exclusivo, no simultáneo):

| GPIO | Usos por defecto | Resolución |
|:----:|------------------|------------|
| 14 | `rs485_de` **y** `hc165_latch` | No usar 74HC165 junto con RS485 (o reasignar uno) |
| 18 | `hc595_sclk` **y** `spi_sck` | Compartido a propósito (mismo bus) |
| 23 | `hc595_mosi` **y** `spi_mosi` | Compartido a propósito (mismo bus) |

## 5. Cómo cambiar los pines

1. Abrir `http://invernadero.local/pins` (o `http://<IP>/pins`).
2. Editar los valores y pulsar **Guardar y reiniciar**.
3. El firmware persiste el mapa en NVS (`ghpins`) y reinicia.

Por API:

```bash
# Leer
curl http://invernadero.local/api/v1/pins
# → {"locked":0,"pins":{"i2c_sda":21,...}}

# Escribir (requiere sesión o token)
curl -X PUT http://invernadero.local/api/v1/pins \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"i2c_sda":21,"i2c_scl":22,"onewire":4}'
```

Ver también: [Hardware y conexiones](Hardware-y-Conexiones.md) ·
[Compatibilidad](Compatibilidad.md) · [PinMap de referencia](Hardware-y-Conexiones.md).
