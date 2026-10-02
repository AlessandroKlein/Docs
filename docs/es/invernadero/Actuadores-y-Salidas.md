---
tags:
  - invernadero
  - actuadores
---

# Actuadores y salidas

> **Tipo:** Embebidos | **Estado:** Estable | **Fecha:** 2026-10-02

## 1. Roles soportados

`PUMP` · `VALVE` (1..8) · `FAN` · `EXTRACTOR` · `HEATER` · `HUMIDIFIER` ·
`LIGHT` · `WINDOW_OPEN/CLOSE` · `ROOF_OPEN/CLOSE` · `SHADE_OPEN/CLOSE` · `ALARM`.

## 2. Expansión de salidas

### 74HC595 / 74HCT595 (daisy-chain)

- 3 pines: DATA (23), CLOCK (18), LATCH (5). 4 registros = 32 salidas.
- Digital ON/OFF o **Soft-PWM** (temporizador). Para PWM de alta frecuencia usar
  LEDC o controlador dedicado (TLC5947).

### MCP23017 (I²C) / MCP23S17 (SPI)

- 16 E/S configurables, hasta 8 en el bus.

## 3. Drivers de potencia

```text
ESP32 → 74HC595 → Driver (MOSFET/SSR/Relé) → Actuador
```

- Cargas DC: MOSFET + diodo flyback.
- Cargas de red: relé/SSR/contactor con aislamiento y protecciones.

## 4. Estados seguros

Al arrancar, todas las salidas van a estado seguro (bomba OFF, válvulas OFF, ...).

## 5. Jerarquía

```text
EMERGENCIA > SEGURIDAD > MANUAL > AUTOMÁTICO > PROGRAMACIÓN
```

## 6. Entradas de seguridad

EMERGENCY_STOP (27), TANK_LOW (32), TANK_HIGH (33), PUMP_FAULT,
WINDOW_LIMIT_OPEN/CLOSE, ROOF_LIMIT_OPEN/CLOSE.
