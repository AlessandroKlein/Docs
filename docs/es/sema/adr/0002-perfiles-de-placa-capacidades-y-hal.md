---
tags:
  - sema
  - adr
  - hardware
---

# 0002. Compatibilidad por perfiles de placa/chip y HAL, no por `#ifdef`

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0015, D-0016, D-0017, D-0038, D-0050, D-0051

## Contexto

La familia ESP32 tiene variantes con capacidades distintas (Ethernet nativa o no,
PSRAM o no, PCNT o no, 802.15.4 en C6, uno o dos núcleos). Resolverlo con `#ifdef`
repartidos por todo el código produce un firmware ilegible y multiplica los caminos de
prueba. La especificación pide explícitamente **no** hacerlo
(`docs/DUDAS-Y-DECISIONES.md` §2).

## Decisión

El código de SEMA **pregunta capacidades** en lugar de preguntar por modelo, detrás de
una HAL y de perfiles de placa/chip:

```text
¿Tengo ADC? ¿PCNT? ¿PSRAM? ¿2 cores? ¿GPIO RTC? ¿Wi-Fi? ¿802.15.4? ¿CAN/TWAI?
```

- **D-0015 / D-0050** — cada placa/chip tiene un perfil que declara sus capacidades;
  la primera matriz oficial cubre ESP32-WROOM-32E/32UE, ESP32-S3, ESP32-C6, ESP32-C5 y
  ESP32-P4.
- **D-0016** — **Capability Manager**: punto único de consulta
  (`device.sensors.read`, `device.can`, `device.psram`, …).
- **D-0017** — **Resource Manager**: asigna y valida recursos (GPIO, periféricos,
  canales ADC, buses) detectando conflictos **antes** de aplicarlos.
- **D-0038 / D-0051** — el mismo código corre en single-core y multicore; PCNT es una
  capacidad explícita (útil para pluviómetro y anemómetro).

## Consecuencias

- ✅ `Capability` (`include/core/Capability.hpp`) declara 16 capacidades + `Count`:
  `wifi`, `bluetooth`, `ethernet`, `adc`, `dac`, `pcnt`, `ledc_pwm`, `i2c`, `spi`,
  `uart`, `can`, `psram`, `rtc_gpio`, `deep_sleep`, `dual_core`, `ieee802154`.
- ✅ `GET /api/v1/capabilities` devuelve las capacidades activas
  (`src/core/web/HttpServer.cpp` §1988-2001).
- ✅ La identidad de placa es de compilación: `SEMA_BOARD_ID`, `SEMA_FLASH_MB`,
  `SEMA_NATIVE_ETH` en `include/hw/HwProfile.hpp`, más las features
  `SEMA_USE_ETHERNET/LORA/MODBUS/CAN/ZIGBEE` y `SEMA_PINS_FROM_FILE`
  (`platformio.ini`).
- ⚠️ **Pendiente**: `CapabilityManager` es un singleton con `set()` manual; el perfil
  base (ESP32 clásico) se escribe a mano en `SemaCore::setup()` §65-78
  (`src/core/SemaCore.cpp`) y **no** se carga desde un Board/Chip Profile. `Ethernet`,
  `Psram` e `Ieee802154` existen en el enum pero el perfil base no las declara.
- ⚠️ **Pendiente**: no existe aún una clase `ResourceManager`; los conflictos se
  manejan en la web de pines y se publican como `reserved_pins` en
  `GET /api/v1/system` (SPI SCK/MISO/MOSI y, con Ethernet RMII habilitado, los 10
  pines RMII).

## Ver también

- [Decisiones](../Decisiones.md) · [Compatibilidad de versiones](../Compatibilidad-de-versiones.md) ·
  [Guía de pines](../Guia-de-pines.md) · [Hardware y conexiones](../Hardware-y-Conexiones.md)
