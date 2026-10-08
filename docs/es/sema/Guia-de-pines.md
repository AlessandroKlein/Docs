---
tags:
  - sema
  - hardware
  - pines
---

# Guía de pines

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.94.0

Asignación de pines del ESP32 (`esp32doit-devkit-v1`).

## Pines por defecto (BoardProfile.hpp)

| Función | Pin | Macro |
|---------|-----|-------|
| I²C SDA | **21** | `SEMA_PIN_I2C_SDA` |
| I²C SCL | **22** | `SEMA_PIN_I2C_SCL` |
| 1-Wire | **4** | `SEMA_PIN_ONEWIRE` |
| ADC batería | **34** | `SEMA_PIN_BATTERY_ADC` |

## Pines fijos vs. configurables

- `SEMA_FIXED_HARDWARE == 0` (default): los pines se configuran por `sensors[]`/`gpio[]`.
- `SEMA_FIXED_HARDWARE == 1` (PCB propio): se usan los pines de `BoardProfile.hpp`
  y se **ignora** `sensors[]`.

## Pines útiles del ESP32 (DOIT DevKit v1)

| GPIO | Notas |
|------|-------|
| 0 | Boot (pull-up); evitable como IO general |
| 2 | LED onboard; usable |
| 4 | 1-Wire (default) |
| 5 | OK |
| 12 | OK (fallo de boot si está HIGH al reiniciar) |
| 13 | OK |
| 14 | OK |
| 15 | OK |
| 16 | RX2 (UART) |
| 17 | TX2 (UART) |
| 18 | OK |
| 19 | OK |
| 21 | SDA (default) |
| 22 | SCL (default) |
| 23 | OK |
| 25 | DAC1 |
| 26 | DAC2 / salida GPIO |
| 27 | OK |
| 32 | OK |
| 33 | OK |
| 34 | Solo **entrada** (ADC) |
| 35 | Solo **entrada** (ADC) |
| 36/VP | Solo **entrada** (ADC) |
| 39/VN | Solo **entrada** (ADC) |

> Los GPIO **34, 35, 36, 39 son solo de entrada** (sin pull-up/pull-down interno).
> Los GPIO **1, 3** son el UART0 de consola (Serial).

## Pluviómetro (wake por lluvia)

- `energy.rain_pin` habilita el wake por lluvia (`ext0`).
- Conectar el reed switch del pluviómetro al pin configurado (default 4, compartido
  con 1-Wire si no se usa DS18B20).
