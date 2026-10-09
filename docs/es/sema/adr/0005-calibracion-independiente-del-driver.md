---
tags:
  - sema
  - adr
  - sensores
---

# 0005. Calibración independiente del driver

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0025, D-0055

## Contexto

Cada sensor real tiene error propio: un offset de ADC, un divisor resistivo distinto,
un rango válido acotado. Si la corrección vive dentro del driver, calibrar obliga a
recompilar y a mantener una variante de código por instalación; además, dos unidades
del mismo modelo necesitan correcciones distintas.

## Decisión

La calibración vive en la **configuración del canal lógico**, no en el driver
(D-0025, D-0055). El framework soporta `offset`, `gain`, calibración multipunto,
curva/polinómica, límites (`min`/`max`) y filtros, y se guarda junto al canal.

```cpp
// include/core/ConfigManager.hpp
struct CalibrationSpec {
  String sensorId, channelId;
  float gain = 1.0f, offset = 0.0f;
  bool hasRange = false;
  float min = 0.0f, max = 0.0f;
};
```

La aplicación es por clave `sensorId:channelId` sobre cada `Measurement` recién
medido (`SensorManager::readAll()`, `src/core/sensors/SensorManager.cpp` §31-35), y se
reconfigura **sin reinicio** con `SemaCore::applyCalibrations()`.

## Consecuencias

- ✅ Calibrar una estación es editar `calibrations[]` y aplicar (o usar la web); no se
  recompila ni se toca el driver.
- ✅ Los drivers quedan simples y genéricos: reportan el valor crudo convertido a
  unidad canónica.
- ✅ Default: si `calibrations[]` está vacío, se aplica ganancia 1, offset 0 y rango
  `−40…85` a `EXT:temperature` e `INT:temperature`
  (`src/core/SemaCore.cpp` §227-251).
- ⚠️ `calibration.multipoint` y `curva/polinómica` están especificados pero **no
  implementados**: `Calibration` (`include/core/Calibration.hpp`) solo tiene
  `gain`/`offset`/`hasRange`/`min`/`max`.
- ⚠️ La calibración del **viento** es un caso aparte: la veleta WH-SP-WD se resuelve
  con la tabla `system.wind_resistors[8]` + `wind_rpull` + `wind_north_offset`
  (`DerivedCalculator`), no con `calibrations[]`.

## Ver también

- [Decisiones](../Decisiones.md) · [Calibración](../Calibracion.md) ·
  [Magnitudes derivadas](../Magnitudes-derivadas.md) · [Sensores](../Sensores.md)
