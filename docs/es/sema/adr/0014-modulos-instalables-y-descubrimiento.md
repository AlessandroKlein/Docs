---
tags:
  - sema
  - adr
  - sensores
---

# 0014. Módulos instalables, selección de drivers y descubrimiento

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0029, D-0043, D-0058, D-0060

## Contexto

Un sistema que va a crecer con sensores y buses nuevos necesita reglas claras para
(a) incorporar código de terceros sin ensuciar el núcleo, (b) agregar funcionalidad
opcional sin recompilar el Core cada vez, y (c) saber qué hay conectado sin que el
usuario declare cada dispositivo a mano.

## Decisión

1. **D-0029 / D-0060 — Módulos instalables/habilitables** con ciclo de vida
   `AVAILABLE → INSTALLED → CONFIGURED → ENABLED → RUNNING`, y estados de error
   `INSTALL_ERROR`, `CONFIG_ERROR`, `RUNTIME_ERROR`, `UPDATE_ERROR`. Cada módulo
   comunica sus capacidades y recursos al Core.
2. **D-0043 — Política de selección de drivers**: preferir drivers oficiales de
   ESP-IDF cuando existan; encapsular drivers externos detrás de la interfaz SEMA.
   **Ninguna biblioteca externa contamina el Core.**
3. **D-0058 — Descubrimiento de sensores**: I²C y 1-Wire ofrecen descubrimiento cuando
   el protocolo lo permite; analógicos y GPIO requieren configuración explícita.

Implementación verificable:

- `Module` (`include/core/Module.hpp`) define `id()`, `version()`, `install()`,
  `configure()`, `enable()`, `start()`, `loop()`, `stop()`, `disable()` y `state()`;
  `ModuleRegistry` garantiza IDs únicos y `enableAll()` recorre el ciclo completo para
  los módulos en `Available`.
- `I2cScanner::scan()` recorre las direcciones `0x01…0x7E` al arrancar y sugiere
  modelo por dirección; el resultado se imprime por serie y se expone en
  `GET /api/v1/diagnostics` (`i2c_devices`).
- `SensorFactory::create()` mapea el `model` de la configuración a un driver; un
  modelo desconocido devuelve `nullptr` y se informa por serie.
- `SEMA_DEMO=1` simula catálogo y mediciones sin hardware.

## Consecuencias

- ✅ Los drivers externos viven encapsulados: el Core solo ve `Sensor`, `Publisher`,
  `Module` y `Storage`.
- ✅ Agregar un sensor es una rama más en `SensorFactory` (ver
  [Guía de desarrollo](../Guia-de-desarrollo.md)).
- ✅ El descubrimiento I²C sugiere `BH1750` (0x23), `AHT20` (0x38/0x39),
  `SHT31/HTU21D` (0x40), `SHT40/SHT3x` (0x44/0x45), `AM2320` (0x5C), `SCD30` (0x61),
  `SCD40/SCD41` (0x62), `MPU6050/DS3231` (0x68) y `BME280/BMP280` (0x76/0x77).
- ⚠️ **No hay módulos reales**: el registro existe y se recorre, pero ningún módulo
  implementa `Module` todavía; la instalación/desinstalación y los permisos quedan
  pendientes.
- ⚠️ El descubrimiento es **solo I²C**: no hay descubrimiento por 1-Wire (el DS18B20
  enumera dispositivos del bus, pero no se autoregistran) ni por Modbus.
- ⚠️ El campo `address` de `sensors[]` se parsea pero **no se usa**: `SensorFactory` no
  lo pasa a los drivers, que quedan con la dirección por defecto de su librería.

## Ver también

- [Decisiones](../Decisiones.md) · [Módulos y ciclo de vida](../Modulos-y-ciclo-de-vida.md) ·
  [Sensores](../Sensores.md) · [Guía de desarrollo](../Guia-de-desarrollo.md)
