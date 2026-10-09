---
tags:
  - sema
  - adr
  - datos
---

# 0003. Modelo canónico de mediciones con quality flags

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0007, D-0026, D-0028, D-0044, D-0056

## Contexto

Cada sensor entrega datos con forma propia (un float de temperatura, tres partículas
PM, un contador de pulsos). Si cada consumidor —storage, API, publishers, dashboard—
leyera el formato del driver, agregar un sensor obligaría a tocar todos. Además, el
sistema necesita distinguir "no hay dato" de "el dato es viejo" o "está fuera de rango".

## Decisión

Una **única representación** de la medición, el *Modelo Canónico* (D-0007, D-0044),
compartida por adquisición, calibración, storage, API y publishers:

```cpp
// include/core/Measurement.hpp
struct Measurement {
  String stationId, sensorId, channelId, measurement, unit;
  float value;
  Quality quality;
  uint32_t sequence, timestamp;
};
```

- **D-0028** — identidad estable: estación, dispositivo, sensor y canal.
- **D-0026 / D-0056** — toda medición lleva un **Quality Flag**, con 8 valores fijos:
  `VALID`, `INVALID`, `STALE`, `TIMEOUT`, `OUT_OF_RANGE`, `CALIBRATION_ERROR`,
  `COMMUNICATION_ERROR`, `SENSOR_DISCONNECTED`; `qualityName()`/`parseQuality()`
  serializan y deserializan, y `UNKNOWN` se devuelve para un valor no reconocido.
- JSON equivalente en la API:

```json
{ "sensor_id": "EXT", "channel_id": "temperature", "measurement": "temperature",
  "value": 24.7, "unit": "degC", "quality": "VALID", "sequence": 1234,
  "timestamp": 1790000000 }
```

## Consecuencias

- ✅ Un sensor nuevo solo implementa `measure(Measurement out[], uint8_t max)`; no hay
  que tocar storage, API ni publishers.
- ✅ `quality` viaja hasta el JSON: el dashboard y el central pueden descartar datos.
- ✅ La **derivación** también produce `Measurement` con `sensorId = "DERIVED"`
  (ver ADR 0005).
- ⚠️ Costo en RAM/CPU: 5 `String` por medición y copias por valor
  (`std::vector<Measurement>`), sensible en un ESP32 de 4 MB.
- ⚠️ `timestamp` no es epoch UTC real: `nowEpoch()` (`include/core/Time.hpp`) usa
  `time()` si NTP sincronizó y cae a `millis()/1000` si no; los eventos usan
  `timestampMs` (millis monotónico) y no epoch.
- ⚠️ De los 8 flags, hoy solo se generan **3**: `VALID`, `COMMUNICATION_ERROR`
  (BMP280, SHT31, BH1750 con lectura inválida) y `SENSOR_DISCONNECTED` (BH1750 y
  DS18B20 desconectados). `INVALID`, `STALE`, `TIMEOUT`, `OUT_OF_RANGE` y
  `CALIBRATION_ERROR` están definidos pero ningún driver los emite.

## Ver también

- [Decisiones](../Decisiones.md) · [Enumeraciones y tipos](../Enumeraciones-y-tipos.md) ·
  [Magnitudes derivadas](../Magnitudes-derivadas.md) · [Almacenamiento e histórico](../Almacenamiento-e-historico.md)
