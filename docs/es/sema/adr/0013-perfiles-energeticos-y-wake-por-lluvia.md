---
tags:
  - sema
  - adr
  - energia
---

# 0013. Perfiles energéticos y wake-up por lluvia

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0021, D-0022, D-0054

## Contexto

Una estación a batería no puede mantener Wi-Fi, web y lectura continua sin agotarse.
Pero dormir "a ciegas" pierde precisamente el evento que importa: el comienzo de la
lluvia. Se necesita un ciclo dormir/despertar centralizado que además pueda despertar
por el pluviómetro.

## Decisión

1. **D-0021 — Deep Sleep + Wake-up Manager**: ciclo dormir/despertar centralizado (RTC,
   GPIO INT, lluvia) según el perfil energético.
2. **D-0022 — Lluvia como wake-up**: el GPIO/INT del pluviómetro despierta la estación
   y suma al contador de pulsos.
3. **D-0054 — Cuatro perfiles de consumo**: `Performance`, `Normal`, `LowPower` y
   `UltraLowPower`; cada uno define frecuencia de lectura, conectividad, sleep y
   fuentes de wake-up según capacidades.

Implementación verificable (`include/core/PowerManager.hpp`,
`src/core/PowerManager.cpp`):

| Elemento | Estado |
|----------|--------|
| `EnergyProfile` + `energyProfileName()` → `performance`, `normal`, `low_power`, `ultra_low_power` | ✅ |
| Perfil por defecto | `Normal` |
| `sleep(seconds)` → `esp_sleep_enable_timer_wakeup` + `esp_deep_sleep_start()` | ✅ (primitiva) |
| `enableRainWakeup(pin)` → `esp_sleep_enable_ext0_wakeup(pin, 1)` | ✅ (se arma si `energy.rain_pin != 0`) |
| `wakeReason()` → `esp_sleep_get_wakeup_cause()` | ✅ (se expone en `/api/v1/energy`) |

## Consecuencias

- ✅ El pluviómetro puede despertar la estación por un pulso (nivel HIGH) y el conteo
  de pulsos sigue en PCNT.
- ✅ `GET /api/v1/energy` devuelve `profile` y `wake_reason`; el `wake_reason` también
  se imprime en el arranque por serie.
- ⚠️ **Ningún módulo llama a `sleep()`**: no hay planificador de sueño, ni paso a
  deep sleep por inactividad o por perfil. El "deep sleep" existe como primitiva
  disponible, no como comportamiento automático.
- ⚠️ El perfil energético **no cambia** la frecuencia de lectura ni la conectividad: el
  scheduler siempre lee cada 10 s y el Wi-Fi queda activo. Cambiar `profile()` solo
  cambia el valor que informa la API.
- ⚠️ `energy.rain_pin` usa un GPIO RTC-capable y `ext0` (un solo pin, nivel HIGH);
  requiere que el pin elegido soporte RTC y no esté ocupado por otro periférico.

## Ver también

- [Decisiones](../Decisiones.md) · [Energía y consumo](../Energia-y-consumo.md) ·
  [Sensores](../Sensores.md) · [Identidad y estados](../Identidad-y-estados.md)
