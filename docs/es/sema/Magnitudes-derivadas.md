---
tags:
  - sema
  - derivadas
  - calculos
---

# Magnitudes derivadas

> **Tipo:** Referencia
> **Estado:** En desarrollo
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Todas las magnitudes que SEMA calcula sin sensor propio, con la fórmula y las constantes
exactas del código. Fuentes: `include/core/derived/DerivedEngine.hpp`,
`src/core/derived/DerivedEngine.cpp`, `include/core/derived/DerivedCalculator.hpp`,
`src/core/derived/DerivedCalculator.cpp`, `src/core/sensors/SensorManager.cpp`,
`src/core/derived/DerivedEngine.cpp`, `src/core/web/HttpServer.cpp` (líneas 2054–2200).

---

## 1. Dos motores, no uno

Hay **dos** implementaciones distintas de magnitudes derivadas y conviene no confundirlas:

| | `DerivedEngine` | `DerivedCalculator` |
|---|---|---|
| Tipo | Clase con métodos `static` puros | Instancia miembro de `SemaCore` (`derived_`) |
| Entrada | `std::vector<Measurement>&` (in-place) | `(raw, out, units)` |
| Quién lo llama | `SensorManager::readAll()` — **siempre**, en cada ciclo de 10 s | `HttpServer::onSensors()` — **por pedido HTTP** |
| Magnitudes | `dew_point`, `heat_index`, `vapor_pressure`, `absolute_humidity` | `dew_point`, `heat_index`, `vpd`, `wind_chill`, `qnh`, `barometric_altitude`, `aqi`, `rain_rate`, `rain_accumulated`, `wind_direction` |
| Requisito | Temperatura **y** humedad del **mismo** `sensorId` | Primera `temperature` y primera `humidity` válidas, **aunque sean de sensores distintos** |
| Filtra por calidad | ❌ no mira `quality` | ✅ ignora todo lo que no sea `Quality::Valid` |
| Entra al histórico | ✅ (viaja dentro de `sensors_.measurements()`) | ❌ es efímero: solo se serializa en la respuesta |
| Unidades | Métricas fijas (`degC`, `hPa`, `g/m3`) | Convierte a imperial si `system.units == "imperial"` |

!!! warning "Las dos versiones del punto de rocío usan constantes distintas"
    `DerivedEngine` usa Magnus con `a = 17.62`, `b = 243.12`; `DerivedCalculator` usa
    `a = 17.27`, `b = 237.7`. En `/api/v1/sensors` el campo `dew_point` aparece **dos
    veces** (una de cada motor) con valores levemente distintos. Ver §5.

---

## 2. `DerivedEngine` (camino en vivo)

Se ejecuta al final de cada `readAll()`, después de la calibración. Busca un sensor que
aporte `channelId == "temperature"` y `channelId == "humidity"` con el mismo `sensorId`
(gana el primero que encuentre) y, si no lo encuentra, **no calcula nada**.

### 2.1 Punto de rocío — `dew_point` (°C)

```text
a = 17.62        b = 243.12
gamma = ln(RH / 100) + (a · T) / (b + T)
Td    = (b · gamma) / (a − gamma)
```

### 2.2 Índice de calor — `heat_index` (°C)

Regresión de Rothfusz (NOAA) evaluada en **°F** y reconvertida a °C:

```text
Tf = T · 9/5 + 32
HI = −42.379 + 2.04901523·Tf + 10.14333127·RH
     − 0.22475541·Tf·RH − 0.00683783·Tf² − 0.05481717·RH²
     + 0.00122874·Tf²·RH + 0.00085282·Tf·RH² − 0.00000199·Tf²·RH²
heat_index = (HI − 32) · 5/9
```

El comentario del código acota la validez: **T ≳ 27 °C y RH ≳ 40 %**. Fuera de ese rango
la regresión pierde sentido físico, pero **el firmware la calcula igual**.

### 2.3 Presión de vapor de saturación — uso interno (°C → hPa)

```text
es = 6.112 · exp( (17.67 · T) / (T + 243.5) )
```

### 2.4 Presión de vapor real — `vapor_pressure` (hPa)

```text
e = es(T) · RH / 100
```

### 2.5 Humedad absoluta — `absolute_humidity` (g/m³)

```text
AH = 216.7 · e / (T + 273.15)
```

### 2.6 Metadatos de lo que emite

| Campo | Valor |
|-------|-------|
| `sensorId` | `"DERIVED"` |
| `channelId` / `measurement` | Iguales al nombre de la magnitud (`dew_point`, `heat_index`, `vapor_pressure`, `absolute_humidity`) |
| `unit` | `degC`, `degC`, `hPa`, `g/m3` |
| `quality` | `Quality::Valid` (siempre, sin mirar la calidad de las entradas) |
| `sequence` | Copiada de la medición de temperatura |
| `timestamp` | Copiada de la medición de temperatura |

Como estas mediciones quedan dentro de `sensors_.measurements()`, **se guardan en el
histórico** (`sensor: "DERIVED"`) y **se publican** por MQTT/webhook junto al resto.

---

## 3. `DerivedCalculator` (camino por pedido)

Se invoca desde `GET /api/v1/sensors` —y en modo demo desde el handler de histórico y el
de sensores— con el vector de mediciones crudas. Primero extrae las entradas válidas
disponibles (`NAN` si no hay):

| Entrada | `measurement` buscado |
|---------|-----------------------|
| `t` | `temperature` |
| `rh` | `humidity` |
| `p` | `pressure` |
| `wind` | `wind_speed` |
| `rain` | `rain` |
| `pm25` | `pm25` |
| `pm10` | `pm10` (leído pero **nunca usado**: no hay derivada de PM10) |

### 3.1 Punto de rocío — `dew_point` (°C) · requiere `t` + `rh`

```text
a = 17.27        b = 237.7
gamma = a · T / (b + T) + ln(RH / 100)
Td    = b · gamma / (a − gamma)
```

### 3.2 Índice de calor — `heat_index` (°C) · requiere `t` + `rh`

Idéntico al de `DerivedEngine` (§2.2), con los mismos nueve coeficientes.

### 3.3 Déficit de presión de vapor — `vpd` (kPa) · requiere `t` + `rh`

```text
es  = 0.6108 · exp( 17.27 · T / (T + 237.3) )      // kPa
VPD = es · (1 − RH/100)
```

### 3.4 Sensación térmica por viento — `wind_chill` (°C) · requiere `t` + `wind_speed`

```text
Tf   = T · 9/5 + 32
vMph = v · 2.23694
si Tf > 50 °F  o  vMph < 3 mph  →  wind_chill = T   (devuelve la temperatura, sin aviso)
si no:
WC = 35.74 + 0.6215·Tf − 35.75·vMph^0.16 + 0.4275·Tf·vMph^0.16
wind_chill = (WC − 32) · 5/9
```

Ojo: fuera del rango de validez la salida **no** se omite ni se marca inválida, sale el
mismo valor que la temperatura del aire.

### 3.5 Presión reducida al nivel del mar — `qnh` (hPa) · requiere `p` + `t`

```text
QNH = p · ( 1 + 0.0065 · altitud / (T + 273.15) )^5.257
```

`altitud` es `system.altitude` en metros (default `0.0`). **Con la altitud en 0, `qnh` es
numéricamente igual a la presión medida.**

### 3.6 Altitud barométrica — `barometric_altitude` (m) · requiere `p`

```text
h = 44330 · ( 1 − (p / 1013.25)^(1/5.255) )
```

Usa la referencia ISA fija 1013,25 hPa, no el QNH calculado: es altitud respecto de la
atmósfera estándar, no respecto del nivel del mar local.

### 3.7 Índice de calidad del aire — `aqi` (adimensional) · requiere `pm25`

Interpolación lineal por tramos sobre la tabla PM2.5 ↔ AQI (µg/m³):

| PM2.5 desde | PM2.5 hasta | AQI desde | AQI hasta |
|-------------|-------------|-----------|-----------|
| 0,0 | 12,0 | 0 | 50 |
| 12,1 | 35,4 | 51 | 100 |
| 35,5 | 55,4 | 101 | 150 |
| 55,5 | 150,4 | 151 | 200 |
| 150,5 | 250,4 | 201 | 300 |
| 250,5 | 350,4 | 301 | 400 |
| 350,5 | 500,4 | 401 | 500 |

```text
AQI = AQI_lo + (AQI_hi − AQI_lo) · (pm25 − PM_lo) / (PM_hi − PM_lo)
```

Si `pm25 > 500,4` devuelve `500.0` (tope). Un `pm25` entre 12,0 y 12,1 (o cualquier hueco
entre tramos) cae en el tramo siguiente porque la comparación es `pm25 <= PM_hi`.

### 3.8 Lluvia — `rain_rate` (mm/h) y `rain_accumulated` (mm) · requiere `rain` válido

El sensor `PCNT` con `channel: "rain"` entrega **mm por ciclo** (pulsos × `scale`).

```text
rainTotal_ += rain                                  // acumulador en RAM
rate = rain / ((now − lastRainTs) / 3600)           // mm/h; 0 si es la primera muestra
                                                     // o si now == lastRainTs
```

`rainTotal_` y `lastRainTs_` viven en la instancia `derived_` de `SemaCore`:
**se pierden al reiniciar** y no hay derivada de intensidad máxima ni duración.

!!! caution "El acumulado depende de cuántas veces se pida `/api/v1/sensors`"
    `rainTotal_ += rain` ocurre en **cada** llamada a `compute()`. Si el mismo valor de
    `rain` (mm del último ciclo) sigue presente, cada request lo vuelve a sumar. La tasa
    tampoco es una tasa de integración: es `mm del ciclo / tiempo entre requests`, así que
    dos pedidos seguidos pueden producir un `rain_rate` muy alto o `0`.

### 3.9 Dirección de viento — `wind_direction` (°) · requiere `system.wind_direction_pin != 0`

```text
adc   = analogRead(wind_direction_pin)
ángulo = posición más cercana de 16 (tabla de resistencias de la veleta)
dir    = (ángulo − wind_north_offset) normalizado a [0, 360)
```

Se emite con `sensorId = "DERIVED"`, `channelId = "wind_direction"`, unidad `deg`. La
calibración de la veleta se detalla en [Calibración §6](Calibracion.md).

### 3.10 Metadatos de lo que emite

`sensorId = "DERIVED"`, `quality = Quality::Valid`, `timestamp = nowEpoch()`,
`sequence = 0` (default del struct, nunca se asigna).

---

## 4. Cuándo **no** tiene sentido calcularlas

| Situación | Efecto real |
|-----------|-------------|
| Falta una entrada | `DerivedEngine` no emite nada si no hay par T/RH del mismo sensor; `DerivedCalculator` omite solo el bloque que depende de esa entrada |
| Temperatura y humedad de **sensores distintos** | `DerivedCalculator` los mezcla igual (p. ej. T de `EXT` con RH de `INT`) → punto de rocío/índice de calor sin sentido físico |
| `T < 27 °C` o `RH < 40 %` | El índice de calor se calcula igual, aunque Rothfusz no aplique |
| `Tf > 50 °F` o viento `< 3 mph` | `wind_chill` devuelve la temperatura del aire: un valor inútil presentado como válido |
| `system.altitude == 0` | `qnh` duplica la presión medida y `barometric_altitude` es altitud ISA |
| `RH` fuera de 0–100 | El `log` de `ln(RH/100)` da `NaN`/`−inf` → valor propagado sin marca de error |
| Medición con calidad mala | `DerivedEngine` la usa igual (no mira `quality`); `DerivedCalculator` la descarta |
| Veleta sin calibrar | `wind_north_offset = 0` → el ángulo es geométrico, no magnético/geográfico |
| `rain` repetido entre requests | Acumulado inflado y tasa irreal (§3.8) |
| Altitud barométrica con clima cambiante | Deriva con la presión del día (±1 hPa ≈ ±8 m): no sirve como altímetro fino |

---

## 5. Duplicados y diferencias visibles en la API

En un equipo real, `GET /api/v1/sensors` devuelve:

1. Los 4 derivados de `DerivedEngine`, que ya vienen dentro de `measurements` —
   `dew_point`, `heat_index` (además de `vapor_pressure` y `absolute_humidity`).
2. Los derivados de `DerivedCalculator`, agregados aparte — `dew_point`, `heat_index`,
   `vpd`, `wind_chill`, `qnh`, `barometric_altitude`, `aqi`, `rain_rate`,
   `rain_accumulated`, `wind_direction`.

Por lo tanto los campos `dew_point` y `heat_index` aparecen **dos veces** con el mismo
`sensor_id: "DERIVED"`, el mismo `channel_id` y valores algo distintos.

Ejemplo concreto a 25,0 °C y 60 % HR *(cálculo propio aplicando las dos fórmulas de §2.1 y
§3.1)*:

| Motor | Constantes | `dew_point` |
|-------|-----------|-------------|
| `DerivedEngine` | `a = 17.62`, `b = 243.12` | 16,69 °C |
| `DerivedCalculator` | `a = 17.27`, `b = 237.7` | 16,68 °C |

La diferencia es de ≈0,01 °C: irrelevante en la práctica, pero hay que saber que el gráfico
y el consumidor de la API pueden estar mirando series distintas.

---

## 6. Conversión de unidades (solo en la respuesta HTTP)

`DerivedCalculator::convertUnit()` se aplica a cada medición serializada en
`/api/v1/sensors` cuando `system.units == "imperial"`. **No** se aplica al histórico, ni a
MQTT/webhook (esos publican siempre las unidades canónicas métricas).

| `measurement` | Unidad imperial | Factor exacto |
|---------------|-----------------|---------------|
| `temperature`, `dew_point`, `heat_index`, `wind_chill` | `degF` | `v · 9/5 + 32` |
| `pressure`, `qnh` | `inHg` | `v / 33.8639` |
| `wind_speed`, `wind_gust` | `mph` | `v · 2.23694` |
| `rain`, `precipitation`, `rain_accumulated` | `in` | `v / 25.4` |
| `rain_rate` | `in/h` | `v / 25.4` |
| `barometric_altitude` | `ft` | `v · 3.28084` |
| Cualquier otra | sin cambio | `v` |

El nombre de la salida se devuelve en el campo `unit` de la respuesta.

---

## 7. Pendientes

| Falta | Vía de solución |
|-------|-----------------|
| Una sola implementación de derivadas | Eliminar `DerivedCalculator` como motor paralelo y exponer `DerivedEngine` con parámetros (unidades, altitud) |
| Constantes unificadas de Magnus | Fijar `a`/`b` en un único `constexpr` compartido |
| Banderas de validez por fórmula | Marcar `Quality::Invalid` (o una magnitud aparte) cuando T/RH/viento están fuera del rango de la regresión |
| Derivadas en el ciclo (VPD, QNH, AQI) | Moverlas a `DerivedEngine` para que se guarden en el histórico |
| Persistencia del acumulado de lluvia | Guardarlo en NVS o derivarlo del histórico |
| Derivadas de PM10, UV, CO₂ | Hoy solo hay AQI sobre PM2.5 |
| `pm10` extraído y no usado | Eliminarlo o usarlo (`aqi_pm10`) |
| Índice de calor «de sombra»/medio | `heat_index` es el de Rothfusz completo; no hay variante con radiación |

---

## Ver también

- [Calibración](Calibracion.md) · [Sensores](Sensores.md)
- [Almacenamiento e histórico](Almacenamiento-e-historico.md) · [Alarmas y reglas](Alarmas-y-reglas.md)
- [Energía y consumo](Energia-y-consumo.md) · [API REST](API-REST.md)
- [Enumeraciones y tipos](Enumeraciones-y-tipos.md) · [Referencia de configuración](Referencia-configuracion.md)
