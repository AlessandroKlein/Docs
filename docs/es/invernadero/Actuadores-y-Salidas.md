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

## 7. Pools de expansores y canales (v3.26 → v3.29)

Las salidas no se limitan al 74HC595 ni a un único MCP23017: el firmware
**instancia pools** desde el catálogo de expansores (`PUT /api/v1/hardware`).

| Pool | Driver | Canales |
|------|--------|---------|
| `shift` | 74HC595 (SPI bit-banged) | 0..31 (soft-PWM) |
| `mcpPool[0..3]` | MCP23017 (I²C) | 32..95 (device 0..3, pin 0..15) |
| `spiPool[0..3]` | MCP23S17 (SPI) | — |
| `adcPool[0..3]` | ADC SPI (MCP3208) | entradas analógicas |
| `input` | 74HC165 (entradas) | entradas digitales |

**Mapeo de canales** (`ActuatorManager::writeChannel`):

```text
canal  0..31  → 74HC595 setChannelPercent(canal)            [soft-PWM]
canal 32..95  → mcpPool[(canal-32)/16]->digitalWrite((canal-32)%16)
```

Para usar un expansor, agregarlo por `PUT /api/v1/hardware` con su `kind`
(MCP23017/MCP23S17/ADC/HC165), `address` (o CS) y `enabled: true`.
