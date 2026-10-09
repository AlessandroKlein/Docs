---
tags:
  - sema
  - calibracion
  - sensores
---

# Calibración

> **Tipo:** Guía
> **Estado:** Estable
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Qué tipos de calibración soporta SEMA, dónde se aplican en la cadena de medición y cómo
calibrar cada sensor en la práctica. Fuentes: `include/core/Calibration.hpp`,
`src/core/Calibration.cpp`, `include/core/sensors/SensorManager.hpp`,
`src/core/sensors/SensorManager.cpp`, `include/core/ConfigManager.hpp`,
`src/core/ConfigManager.cpp`, `src/core/SemaCore.cpp`,
`include/core/derived/DerivedCalculator.cpp`, `src/core/web/HttpServer.cpp`.

---

## 1. Modelo

```cpp
// include/core/Calibration.hpp
struct Calibration {
  float offset = 0.0f;
  float gain   = 1.0f;
  float min    = 0.0f;
  float max    = 0.0f;
  bool  hasRange = false;  // false → no se valida rango
  bool  enabled  = false;
};

void applyCalibration(Measurement& m, const Calibration& c);
```

```cpp
// src/core/Calibration.cpp — implementación completa
if (!c.enabled) return;
m.value = (m.value * c.gain) + c.offset;
if (c.hasRange && (m.value < c.min || m.value > c.max)) {
  m.quality = Quality::OutOfRange;
}
```

En configuración, el equivalente es `CalibrationSpec` (`include/core/ConfigManager.hpp`),
sin el campo `enabled`: SemaCore fuerza `enabled = true` para **todo** lo que venga de
`calibrations[]`.

---

## 2. Tipos soportados

| Tipo de calibración | ¿Soportado? | Cómo |
|---------------------|-------------|------|
| Offset (suma) | ✅ | `offset` |
| Ganancia / escala (producto) | ✅ | `gain` |
| Lineal de 2 puntos (`y = m·x + b`) | ✅ | Es la combinación de `gain` + `offset` |
| Validación de rango | ✅ | `has_range` + `min` + `max` → `Quality::OutOfRange` |
| Multipunto / tabla de interpolación | ❌ | D-0055 lo prevé; no hay código |
| Curva polinómica | ❌ | ídem |
| Filtros (media móvil, mediana, IIR) | ❌ | el comentario de `Calibration.hpp` dice «y (más adelante) filtro» |
| Calibración por sensor completo (afecta a todos sus canales) | ❌ | la clave es por **canal** |
| `Quality::CalibrationError` | ❌ | el enum existe (`Measurement.hpp`), ningún camino lo asigna |
| Página web de calibración | ❌ | solo por `PUT /api/v1/config` (o restaurando un backup) |

---

## 3. Dónde se aplica en la cadena de medición

El único punto de aplicación es `SensorManager::readAll()`, es decir **una vez por ciclo de
10 s**, inmediatamente después de leer cada driver y antes de las derivadas:

```cpp
// src/core/sensors/SensorManager.cpp
Measurement buffer[4];
for (Sensor* sensor : sensors_) {
  if (!sensor->healthy()) continue;
  const uint8_t n = sensor->measure(buffer, 4);
  for (uint8_t i = 0; i < n; ++i) {
    const String key = buffer[i].sensorId + ":" + buffer[i].channelId;
    auto it = calibrations_.find(key);
    if (it != calibrations_.end()) applyCalibration(buffer[i], it->second);
    measurements_.push_back(buffer[i]);
  }
}
DerivedEngine::compute(measurements_);   // las derivadas NO se calibran
```

```text
driver ──► scale/offset del SensorSpec ──► applyCalibration(gain,offset) ──► rango/calidad
(solo ADC, PCNT, SOLAR, ADS1115, CO)              │
                                                  ▼
                        Vector<Measurement> ──► DerivedEngine ──► histórico ──► MQTT/webhook
                                                  └──────────────► reglas de alarma
```

Puntos finos, todos verificables en el código:

1. **Doble escalado en sensores analógicos.** `AdcSensor::measure()` ya aplica
   `value = raw * scale + offset` con los campos `scale`/`offset` del `SensorSpec`; después
   `applyCalibration()` aplica `gain`/`offset` de `calibrations[]`. El resultado neto es
   `(raw · scale + offsetSensor) · gain + offsetCal`.
2. **La clave es `sensorId + ":" + channelId`.** No usa `measurement` ni el modelo. Un
   `sensor_id`/`channel_id` que no coincidan exactamente simplemente **no se aplican**, sin
   aviso ni error.
3. **Las derivadas no se calibran** (nacen después del bucle de calibración).
4. **La calidad no se propaga al cálculo derivado**: `DerivedEngine` ignora `quality`.
5. El histórico **sí** guarda la calidad resultante (`OUT_OF_RANGE`, …), así que un rango
   mal configurado es visible en `GET /api/v1/history` y en las gráficas.
6. La reinicialización es explícita: `SemaCore::applyCalibrations()` hace
   `sensors_.clearCalibrations()` y vuelve a cargar todo el mapa. Se invoca en `setup()` y
   en cada `PUT /api/v1/config` / `POST /api/v1/backup`.

---

## 4. Claves reales de calibración por driver

| Modelo | `channelId` que hay que usar en la clave | Ejemplo de `sensor_id`+`channel_id` |
|--------|------------------------------------------|-------------------------------------|
| `BME280` | `temperature`, `humidity`, `pressure` | `EXT` + `temperature` |
| `SHT40` / `SHT31` / `AHT20` | `temperature`, `humidity` | `INT` + `humidity` |
| `DS18B20` | `temperature` (`SOIL`, `SOIL_1`, … si hay varios) | `SOIL` + `temperature` |
| `BH1750` | `light` | `LUX` + `light` |
| `SCD30` | `co2`, `temperature`, `humidity` | `CO2` + `co2` |
| `SGP30` | `eco2`, `tvoc` | `VOC` + `tvoc` |
| `VEML6075` | `uv_index`, `uva`, `uvb` | `UV` + `uv_index` |
| `AS3935` | `lightning_distance` | `RAYO` + `lightning_distance` |
| `PMS5003` | `pm1_0`, `pm2_5`, `pm10_0` | `PM` + `pm2_5` |
| `ADS1115` | el valor de `spec.channel` | `ADC1` + `voltage` |
| `ADC` | el valor de `spec.channel` (p. ej. `voltage`, `soil_moisture`) | `BATT` + `voltage` |
| `PCNT` | el valor de `spec.channel` (p. ej. `rain`, `wind_speed`) | `RAIN` + `rain` |
| `SOLAR` | `solar_radiation` (fijo en el driver) | `PYRA` + `solar_radiation` |
| `CO` | `co` | `CO` + `co` |

En el catálogo fijo (`SEMA_FIXED_HARDWARE=1`) los ids son `EXT`, `INT`, `SOIL`, `LUX`,
`AUX` y `BATT`.

---

## 5. Formato exacto de `calibrations[]`

```json
"calibrations": [
  { "sensor_id": "EXT",  "channel_id": "temperature", "gain": 1.0,   "offset": -0.5, "has_range": true, "min": -40.0, "max": 85.0 },
  { "sensor_id": "BATT", "channel_id": "voltage",     "gain": 1.021, "offset": -0.35, "has_range": true, "min": 9.0,  "max": 15.0 },
  { "sensor_id": "SOIL", "channel_id": "soil_moisture","gain": 1.0,  "offset": 0.0,  "has_range": true, "min": 0.0,  "max": 100.0 }
]
```

| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `sensor_id` | str | `""` | Id lógico del sensor; se concatena con `channel_id` para formar la clave |
| `channel_id` | str | `""` | Canal lógico a calibrar |
| `gain` | float | `1.0` | Factor multiplicativo |
| `offset` | float | `0.0` | Término aditivo (en la unidad del canal) |
| `has_range` | bool | `false` | Si es `false`, `min`/`max` se ignoran |
| `min` | float | `0.0` | Límite inferior |
| `max` | float | `0.0` | Límite superior |

**Calibración por defecto** (cuando `calibrations[]` está vacío) — el código crea una sola
estructura y la registra para dos canales:

```text
enabled = true · gain = 1.0 · offset = 0.0 · has_range = true · min = -40.0 · max = 85.0
→ "EXT:temperature"
→ "INT:temperature"
```

Es decir: de fábrica **solo** temperatura exterior e interior tienen validación de rango.
Todos los demás canales quedan sin calibrar.

!!! danger "`has_range: true` con `min`/`max` en 0"
    Como los defaults de `min` y `max` son `0.0`, activar `has_range` sin poner límites
    marca **todo** valor distinto de 0 como `OUT_OF_RANGE`. No hay validación que lo impida.

---

## 6. Órdenes de magnitud y ejemplos numéricos

### Batería por ADC (divisor 11:1)

En el catálogo fijo (`src/core/SemaCore.cpp`):

```cpp
static AdcSensor battery("BATT", SEMA_PIN_BATTERY_ADC, "voltage", "V",
                         3.3f * 11.0f / 4095.0f, 0.0f);
```

- Pin: `SEMA_PIN_BATTERY_ADC` = **GPIO 34** (`include/core/BoardProfile.hpp`); ADC de 12
  bits (`analogReadResolution(12)`), lectura 0–4095.
- Escala del sensor: `3.3 × 11 / 4095` = **0,0088645 V/count** → fondo de escala 36,3 V.
- Canal/magnitud: `voltage`, unidad `V`.

Ejemplo numérico (aritmética propia sobre esos datos):

| `raw` | `value = raw × 0,0088645` | Con `gain 1,021`, `offset −0,35` |
|-------|---------------------------|-----------------------------------|
| 1000 | 8,86 V | 8,70 V |
| 1240 | 10,99 V | 10,87 V |
| 1350 | 11,97 V | 11,87 V |
| 1500 | 13,30 V | 13,23 V |
| 1860 | 16,49 V | 16,49 V |

El par `gain`/`offset` se obtiene comparando contra un multímetro: `gain` corrige el error
de escala del divisor y `offset` el error de cero del ADC. Dos puntos de referencia bien
separados (por ejemplo 11,0 V y 14,0 V) alcanzan para resolver ambos.

### Humedad de suelo

No hay driver dedicado de humedad de suelo entre los 17 modelos
(`src/core/sensors/SensorFactory.cpp`): se usa un sensor analógico `ADC` con
`channel: "soil_moisture"` y `unit: "%"`, cuyo `scale`/`offset` mapean la lectura cruda a
porcentaje. Ejemplo de configuración coherente con el driver:

```json
{ "id": "SOIL", "model": "ADC", "pin": 35, "channel": "soil_moisture", "unit": "%",
  "scale": -0.02442, "offset": 100.0, "enabled": true }
```

Con esa escala, `raw 4095` → ≈ 0 % y `raw 0` → 100 % (sensor capacitivo: más voltaje =
más seco). Ajuste fino con `calibrations[]`: si el suelo saturado (100 %) se lee 96 %,
`gain = 100/96 = 1,0417`; si el suelo seco (0 %) se lee 2 %, `offset = -2,0`.

### Presión (BME280)

Contra una estación de referencia: `gain = 1.0` y `offset` igual a la diferencia medida
(p. ej. el sensor lee 1011,2 hPa y la referencia 1013,3 hPa → `offset: +2.1`). La presión
del BME280 ya viene en hPa (`readPressure() / 100.0f`).

### Veleta WH-SP-WD (tabla de resistencias)

La veleta **no** usa `calibrations[]`. Se calibra con dos claves de `system`:

| Clave | Default | Rol |
|-------|---------|-----|
| `wind_resistors[8]` | `33000, 8200, 1000, 2200, 3900, 16000, 120000, 64900` Ω | Una resistencia por posición, en orden del datasheet: N, NE, E, SE, S, SO, O, NO |
| `wind_rpull` | `10000.0` Ω | Resistencia de pull-up del divisor |
| `wind_direction_pin` | `0` | Pin ADC de la veleta; `0` = sin veleta (no se emite `wind_direction`) |
| `wind_north_offset` | `0.0`° | Corrimiento de norte, en grados |

El cálculo de cada posición (`DerivedCalculator::windVanePositions()`) es un divisor
resistivo de 12 bits:

```text
posición directa:    ADC_i   = round(4095 · R_i / (R_i + Rpull))         ángulo = i · 45°
posición intermedia: R_par   = R_i · R_(i+1) / (R_i + R_(i+1))           ángulo = i · 45° + 22,5°
                     ADC_par = round(4095 · R_par / (R_par + Rpull))
```

`windVaneRawAngle()` devuelve el ángulo de la posición cuyo `ADC_i` esté más cerca del
valor leído (distancia en cuentas, búsqueda lineal sobre 16 posiciones).
`windDirection()` resta `wind_north_offset` y normaliza a 0–360.

Tabla de las 16 posiciones con los **defaults del repo** *(cuentas calculadas con la
fórmula de arriba; sirven de referencia, no están en el código)*:

| # | Dirección | Ángulo | Origen | R (Ω) | ADC esperado |
|---|-----------|--------|--------|-------|--------------|
| 0 | N | 0° | directa | 33 000 | 3143 |
| 1 | NNE | 22,5° | 33 k ∥ 8,2 k | 6568 | 1623 |
| 2 | NE | 45° | directa | 8 200 | 1845 |
| 3 | ENE | 67,5° | 8,2 k ∥ 1 k | 891 | 335 |
| 4 | E | 90° | directa | 1 000 | 372 |
| 5 | ESE | 112,5° | 1 k ∥ 2,2 k | 688 | 263 |
| 6 | SE | 135° | directa | 2 200 | 738 |
| 7 | SSE | 157,5° | 2,2 k ∥ 3,9 k | 1407 | 505 |
| 8 | S | 180° | directa | 3 900 | 1149 |
| 9 | SSO | 202,5° | 3,9 k ∥ 16 k | 3136 | 978 |
| 10 | SO | 225° | directa | 16 000 | 2520 |
| 11 | OSO | 247,5° | 16 k ∥ 120 k | 14 118 | 2397 |
| 12 | O | 270° | directa | 120 000 | 3780 |
| 13 | ONO | 292,5° | 120 k ∥ 64,9 k | 42 120 | 3309 |
| 14 | NO | 315° | directa | 64 900 | 3548 |
| 15 | NNO | 337,5° | 64,9 k ∥ 33 k | 21 876 | 2810 |

Procedimiento:

1. Medir con un multímetro las 8 resistencias reales de la veleta y el pull-up, y cargarlos
   en la página «Veleta» (o `POST /api/v1/wind/resistors`) — el POST acepta `rpull` y el
   array `resistors` de 8 elementos.
2. Orientar la veleta al norte real y usar «🧭 Calibrar norte»
   (`POST /api/v1/wind/north`): guarda el ángulo bruto actual como `wind_north_offset`.
3. Verificar con `GET /api/v1/sensors` el valor de `wind_direction` (unidad `deg`).
4. Comprobar que cada cuarto de vuelta cae en el ángulo esperado; si una posición salta a
   su vecina, revisar el resistor correspondiente o el valor de `wind_rpull`.

!!! note "La dirección de viento se calcula por pedido, no en el ciclo"
    `DerivedCalculator::compute()` —donde vive la lectura del ADC de la veleta— se ejecuta
    desde `GET /api/v1/sensors`, no desde el ciclo de 10 s. Por eso `wind_direction`
    aparece en la API pero **no** se guarda en `/history.jsonl`.

### Piranómetro / radiación solar

No hay driver específico de piranómetro: se usa `model: "SOLAR"` (`SolarSensor`), que es un
ADC genérico con canal y unidad fijos `solar_radiation` / `W/m2`. El `scale` convierte
mV del sensor a W/m² y el `offset` corrige el cero nocturno:

```json
{ "id": "PYRA", "model": "SOLAR", "pin": 36, "scale": 0.2, "offset": 0.0, "enabled": true }
```

Ejemplo: `raw 1500` con `scale 0.2` → 300 W/m²; si de noche el sensor marca 12 W/m²,
`offset: -12.0`. Un techo físico razonable se fuerza con `calibrations[]`
(`has_range: true`, `min: 0`, `max: 1500`) para que una lectura fuera de rango quede
marcada como `OUT_OF_RANGE` en lugar de ensuciar el histórico.

---

## 7. Procedimiento general de calibración

1. **Estabilizar**: dejar el sensor al menos 2 ciclos de lectura (20 s) en la condición de
   referencia (baño térmico, aire saturado, patrón de presión).
2. **Leer el crudo**: `GET /api/v1/sensors` y anotar `value` para el `sensor_id`/`channel_id`
   objetivo (no confundir con los valores `DERIVED`).
3. **Calcular**: `gain = (ref_alto − ref_bajo) / (med_alto − med_bajo)`;
   `offset = ref_bajo − gain · med_bajo`.
4. **Cargar** con `PUT /api/v1/config` incluyendo el array `calibrations[]` completo (el
   `PUT` reemplaza la config entera; el `parseInto()` reconstruye el vector desde cero).
5. **Verificar** sin reiniciar: `onConfigPut()` re-aplica calibraciones al vuelo
   (`applyCalibrations()`), así que el siguiente ciclo ya sale calibrado.
6. **Persistir**: la config queda en NVS (`config`); un backup (`GET /api/v1/backup`) la
   incluye y `POST /api/v1/backup` la restaura.

```bash
# 1) ver el valor actual
curl -s http://sema.local/api/v1/sensors | jq '.measurements[] | select(.sensor_id=="BATT")'
# 2) subir la calibración (config completa)
curl -X PUT http://sema.local/api/v1/config -H 'X-API-Key: <api_key>' \
     -H 'Content-Type: application/json' --data @config.json
```

---

## Ver también

- [Sensores](Sensores.md) · [Magnitudes derivadas](Magnitudes-derivadas.md)
- [Alarmas y reglas](Alarmas-y-reglas.md) · [Energía y consumo](Energia-y-consumo.md)
- [Almacenamiento e histórico](Almacenamiento-e-historico.md)
- [Referencia de configuración](Referencia-configuracion.md) · [API REST](API-REST.md)
- [Guía de pines](Guia-de-pines.md) · [Enumeraciones y tipos](Enumeraciones-y-tipos.md)
