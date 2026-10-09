---
tags:
  - sema
  - almacenamiento
  - historico
---

# Almacenamiento e histórico

> **Tipo:** Referencia
> **Estado:** Estable
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Dónde guarda SEMA cada cosa, con qué formato, con qué límites y qué se pierde al
reiniciar. Fuentes: `include/core/storage/Storage.hpp`, `include/core/storage/NvsStore.hpp`,
`src/core/storage/NvsStore.cpp`, `include/core/storage/HistoryStore.hpp`,
`src/core/storage/HistoryStore.cpp`, `include/core/events/EventLog.hpp`,
`src/core/events/EventLog.cpp`, `src/core/SemaCore.cpp`, `src/core/web/HttpServer.cpp`.

---

## 1. Resumen: tres medios, tres usos

| Medio | Qué guarda | Archivo / namespace | Sobrevive al reinicio |
|-------|-----------|---------------------|------------------------|
| **NVS** (`Preferences`) | Configuración, contador de reinicios, layout del dashboard | namespace `sema`, claves `config`, `boots`, `layout` | Sí |
| **LittleFS** (flash, partición `spiffs`) | Registro de eventos/alarmas | `/events.jsonl` | Sí |
| **microSD** (SPI, opcional) | Histórico de mediciones | `/history.jsonl` y `/history.jsonl.agg` | Sí (si la SD está presente) |
| RAM | Mediciones del ciclo, cola de eventos, acumulado de lluvia | — | No |

```text
                 ┌──────────────────────── NVS (Preferences, namespace "sema") ──────────┐
   config  ──────┤ config · boots · layout                                                │
                 └───────────────────────────────────────────────────────────────────────┘
                 ┌──────────────────────── LittleFS (partición datos) ───────────────────┐
   eventos ──────┤ /events.jsonl                        (EventLog, máx. 100 en RAM)       │
                 └───────────────────────────────────────────────────────────────────────┘
                 ┌──────────────────────── microSD (SPI, CS configurable) ───────────────┐
   mediciones ───┤ /history.jsonl      (alta resolución, tope 10 000 líneas)             │
                 │ /history.jsonl.agg  (promedios horarios de lo viejo)                  │
                 └───────────────────────────────────────────────────────────────────────┘
```

!!! warning "El histórico NO vive en la flash interna"
    Aunque `include/core/storage/HistoryStore.hpp` dice en su comentario de cabecera
    «backend LittleFS», la implementación real (`src/core/storage/HistoryStore.cpp`)
    incluye `<SD.h>` y **solo** escribe en la microSD. Sin SD habilitada no se guarda
    histórico: no hay *fallback* a LittleFS. Ver §10.

---

## 2. NVS (Preferences)

`NvsStore` (`src/core/storage/NvsStore.cpp`) envuelve la clase `Preferences` del core
Arduino-ESP32 y es el único backend que implementa hoy la interfaz `KeyValueStore`
(`include/core/storage/Storage.hpp`). SemaCore lo abre en `setup()`:

```cpp
store_.begin("sema");   // src/core/SemaCore.cpp
```

| Clave | Tipo | Quién la escribe | Contenido |
|-------|------|------------------|-----------|
| `config` | string (JSON) | `ConfigManager::save()` | Config completa, `schema_version: 1` |
| `boots` | uint32 | `SemaCore::setup()` | Contador de reinicios (se lee, se incrementa y se reescribe en cada arranque) |
| `layout` | string (JSON) | `ConfigManager::saveDashboardLayout()` | Layout Gridstack, en clave aparte para no exceder el tamaño de una entrada NVS |

Partición NVS (`partitions_4mb.csv`, `partitions_8mb.csv`, `partitions_16mb.csv`):
offset `0x9000`, tamaño `0x5000` = **20 480 bytes (20 KB)** en las tres placas.

Fallo de escritura: si `putString("config", …)` falla (NVS lleno o fragmentado),
`ConfigManager::save()` hace `store_.clear()` y reintenta una vez. Consecuencia
documentada en el propio código: **se pierden la config anterior y el layout**, pero se
recupera el guardado sin reflashear.

---

## 3. LittleFS (solo para el registro de eventos)

`EventLog::begin()` monta LittleFS con formato automático si la partición está corrupta:

```cpp
if (!LittleFS.begin(true)) { return; }   // src/core/events/EventLog.cpp
```

- `platformio.ini` fija `board_build.filesystem = littlefs` para todos los entornos.
- La partición se llama `spiffs` en el CSV, pero el sistema de archivos montado es
  LittleFS:

| Entorno | Archivo de particiones | Partición de datos |
|---------|------------------------|--------------------|
| `esp32doit-devkit-v1` (4 MB) | `partitions_4mb.csv` | `0x70000` = 448 KB |
| `esp32-s3-devkitc-1` (8 MB) | `partitions_8mb.csv` | `0x1F0000` = 1,94 MB |
| `esp32-wroom-32u` (16 MB) | `partitions_16mb.csv` | `0x9F0000` = 9,94 MB |

En todo `src/`, LittleFS se usa **únicamente** en `EventLog.cpp`. El servidor web ya no
depende de LittleFS (los assets Gridstack van embebidos en PROGMEM).

---

## 4. microSD: activación y CS

El histórico se habilita desde configuración:

| Clave en `storage` | Default | Efecto |
|--------------------|---------|--------|
| `sd_enabled` | `false` | Si es `true`, SemaCore llama a `history_.enableSd(sdCsPin)` en `setup()` |
| `sd_cs` | `4` | Chip-select SPI de la microSD (`SEMA_PIN_SD_CS` = 4 en `include/hw/HwProfile.hpp`) |

Secuencia real en `SemaCore::setup()`:

```cpp
config_.load();
history_.setRetentionSeconds(config_.get().storage.retentionDays * 86400);
if (config_.get().storage.sdEnabled) {
  const bool sd = history_.enableSd(config_.get().storage.sdCsPin);
  Serial.printf("MicroSD: %s (CS=%u)\n", sd ? "conectada" : "no detectada", ...);
}
```

- `HistoryStore::begin()` **no toca la SD**: solo fija `path_` y `aggPath_`. Quien monta
  la tarjeta es `enableSd(csPin)`, que llama a `SD.begin(csPin)` y luego cuenta las
  líneas ya existentes del archivo para inicializar `count_`.
- Se llama **una sola vez, en el arranque**. Cambiar `sd_enabled` desde la web obliga a
  reiniciar: la propia UI avisa «Guardado (reiniciá para aplicar)».
- Sin SD activa: `append()` devuelve `false` de inmediato y `readRecent()` devuelve
  `false` → las gráficas quedan vacías. No se escribe nada en flash.
- `GET /api/v1/system` expone `sd_enabled` (config) y `history_available`
  (`history_.sdEnabled()`, el estado real tras el arranque).

---

## 5. Formato del histórico (JSONL)

Un objeto JSON por línea, sin array contenedor ni cabecera (JSONL). Lo escribe
`HistoryStore::append()` con un `DynamicJsonDocument` de 256 bytes:

```json
{"ts":1720000000,"sensor":"EXT","channel":"temperature","measurement":"temperature","value":23.4,"unit":"degC","quality":"VALID","seq":123}
```

| Campo | Origen | Descripción |
|-------|--------|-------------|
| `ts` | `Measurement::timestamp` | Época local en **segundos** (`nowEpoch()`); cae a `millis()/1000` hasta que sincronice NTP |
| `sensor` | `sensorId` | Id lógico (`EXT`, `INT`, `SOIL`, `BATT`, `DERIVED`, …) |
| `channel` | `channelId` | Canal lógico (`temperature`, `humidity`, `voltage`, …) |
| `measurement` | `measurement` | Magnitud canónica (`temperature`, `dew_point`, …) |
| `value` | `value` | Valor float ya calibrado |
| `unit` | `unit` | Unidad canónica (`degC`, `percent`, `hPa`, `V`, …) |
| `quality` | `qualityName(quality)` | `VALID`, `OUT_OF_RANGE`, `SENSOR_DISCONNECTED`, … |
| `seq` | `sequence` | Secuencia monotónica por driver |

La misma estructura se devuelve en `GET /api/v1/history` (§8) y es reversible: el parser
`parseMeasurement()` lee exactamente esos ocho campos.

---

## 6. Límites, ring y rotación

### HistoryStore (microSD)

| Parámetro | Valor | Dónde |
|-----------|-------|-------|
| `maxEntries_` | **10 000** | `HistoryStore.hpp` (default de fábrica) |
| `retentionSeconds_` | `retention_days × 86400` (default 30 días) | `SemaCore::setup()` |
| Rotación | conserva la **mitad más reciente** (5 000 líneas) y reescribe el archivo | `HistoryStore::rotate()` |
| Disparador | `count_ >= maxEntries_` al intentar un `append()` | `HistoryStore::append()` |

`setMaxEntries()` existe en la API pública pero **nunca se llama**: el tope efectivo es
siempre 10 000. `count_` no es un contador en RAM: al habilitar la SD se recorre el
archivo completo contando líneas.

### EventLog (LittleFS)

| Parámetro | Valor | Dónde |
|-----------|-------|-------|
| Cola en RAM | 100 eventos (`EventLog(bus, maxEntries = 100)`) | `EventLog.hpp` |
| Rotación en disco | cuando `fileCount_ >= maxEntries_ × 2` = **200 líneas**, conserva las últimas 100 | `EventLog::onEvent()` |
| Reescritura | lee todo el archivo, se queda con las últimas N y lo reescribe | `EventLog::rotate()` |

Al arrancar, `begin()` relee `/events.jsonl`, llena la cola y descarta lo que exceda 100.
Los eventos son, en la práctica, de dos tipos: `system` (arranque, `rule: "boot"`) y
`alarm` (ver [Alarmas y reglas](Alarmas-y-reglas.md)).

---

## 7. Retención por niveles y agregación

`SemaCore::loop()` ejecuta una pasada de compactación como máximo una vez por hora:

```cpp
const uint32_t now = nowEpoch();
if (now - lastAgg >= 3600) {
  lastAgg = now;
  const uint32_t cutoff = now - config_.get().storage.retentionDays * 86400;
  history_.aggregate(3600, cutoff);
}
```

`aggregate(bucketSeconds = 3600, cutoffEpoch)`:

1. Recorre `/history.jsonl`.
2. Las mediciones con `ts >= cutoff` se **conservan tal cual** (alta resolución).
3. Las más viejas se agrupan por clave `sensorId|measurement|bucket` (bucket = hora) y se
   promedian (`suma / cuenta`).
4. Reescribe `/history.jsonl` solo con las recientes → `count_` baja.
5. Anexa un registro por bucket a `/history.jsonl.agg` con `channel: ""`, `unit: ""`,
   `quality: "VALID"` y `seq: 0`.

La serie de agregados se consulta con `GET /api/v1/history?series=aggregated`.

`HistoryStore::prune(nowEpoch)` (descarte puro por antigüedad) está implementado pero
**no se llama desde ningún punto del firmware**: la política activa es agregar, no borrar.

!!! note "El archivo de agregados no tiene tope ni retención"
    Nadie poda `/history.jsonl.agg`: ni `prune()`, ni `aggregate()`, ni un límite de
    entradas. Es un archivo que solo crece (una línea por sensor|magnitud|hora).

---

## 8. `GET /api/v1/history`

```http
GET /api/v1/history?limit=50&series=recent&format=json
```

| Parámetro | Valores | Default | Comportamiento real |
|-----------|---------|---------|---------------------|
| `limit` | entero | `50` | Solo se acepta si `1 ≤ limit ≤ 3000`; en cualquier otro caso (0, negativo, > 3000) se usa **50** en silencio |
| `series` | `recent` · `aggregated` | `recent` | `recent` → `/history.jsonl`; `aggregated` → `/history.jsonl.agg` |
| `format` | `json` · `csv` | `json` | `csv` responde `text/csv` con `Content-Disposition: attachment; filename=sema_history.csv` |

Respuesta JSON (máx. 16 KB de documento):

```json
{ "history": [ { "ts": 1720000000, "sensor": "EXT", "channel": "temperature",
                 "measurement": "temperature", "value": 23.4, "unit": "degC",
                 "quality": "VALID", "seq": 123 } ] }
```

CSV (cabecera literal): `ts,sensor,channel,measurement,value,unit,quality,seq`, con el
valor formateado a 4 decimales.

Detalles de implementación a tener en cuenta:

- `readRecent()`/`readAggregated()` **leen el archivo entero** y van descartando el
  frente de un `std::deque` cuando supera `limit`: el costo es proporcional al tamaño
  del archivo, no al `limit` pedido.
- El dashboard pide `limit=3000` y filtra en el navegador por `measurement` y
  `ts >= ahora - rango_horas` (`Date.now()/1000`). Como `ts` es época en segundos, antes
  de la sincronización NTP (cuando `ts` es uptime) el gráfico no muestra puntos.
- Sin SD habilitada, la respuesta es `{"history": []}`.
- Con `SEMA_DEMO=1` el handler devuelve 40 muestras ficticias por serie y **no** toca la SD.
- El endpoint no exige autenticación (`onHistory()` no llama a `webAuthed()`).

---

## 9. Qué se pierde al reiniciar

| Dato | Persistencia | Al reiniciar |
|------|--------------|--------------|
| Config, contador de reinicios, layout | NVS | Se conserva |
| Eventos y alarmas (últimos 100) | LittleFS + RAM | Se recargan del archivo |
| Histórico de mediciones | microSD | Se conserva (si la SD sigue puesta) |
| Mediciones del último ciclo | RAM | Se pierden |
| Acumulado de lluvia (`DerivedCalculator::rainTotal_`) y `lastRainTs_` | RAM | Se pierden (vuelven a 0) |
| Estado del `HealthMonitor`, watchdog, cola de eventos no escrita | RAM | Se pierden |
| `history_.count_` | RAM (se recalcula) | Se recuenta desde el archivo |

---

## 10. Uso de flash, desgaste y lo que falta

**Escrituras y desgaste**

- NVS tiene *wear leveling* propio del IDF, pero la config se **reescribe completa** en
  cada `apply()` (cada guardado desde la web o cada `PUT /api/v1/config`). En el peor
  caso, `save()` limpia el namespace antes de reintentar.
- `EventLog::rotate()` reescribe `/events.jsonl` entero (hasta ~200 líneas): una
  reescritura completa cada 200 eventos, más un `append` por evento.
- `HistoryStore::rotate()` reescribe el archivo completo (hasta ~10 000 líneas ≈ 1 MB)
  cada 10 000 altas. Ese es el punto de desgaste/latencia más alto del sistema.

**Cuánto tarda en llenarse el histórico** *(cálculo propio sobre datos del repo)*

- El scheduler dispara `sensors.read` cada **10 000 ms** (`SemaCore::setup()`).
- El catálogo fijo produce 10 mediciones (BME280: temperature, humidity, pressure;
  SHT40: 2; DS18B20: 1; BH1750: light; AHT20: 2; ADC batería: voltage) y
  `DerivedEngine` agrega 4 más → **14 registros por ciclo**.
- 14 × 6 ciclos/min = 84 registros/min → 10 000 / 84 ≈ **119 min (~2 h)** hasta la
  primera rotación, que deja ~5 000 registros (~1 h de alta resolución).
- Tamaño estimado de línea: la del ejemplo de §5 tiene 137 bytes; 10 000 × ~137 B ≈
  **1,3 MB** en la SD. Con la retención de 30 días por default, el `aggregate()` horario
  es el que evita que el archivo crezca sin control en el tramo de alta resolución.

**❌ No implementado / ⚠️ Pendiente**

| Falta | Estado real | Vía de solución |
|-------|-------------|-----------------|
| Histórico en flash sin SD | ❌ El comentario de `HistoryStore.hpp` promete LittleFS; el `.cpp` solo usa `SD.h` | Implementar un backend `LittleFS` detrás de la misma interfaz, o corregir el comentario |
| Retención de `/history.jsonl.agg` | ❌ Crece sin tope | Llamar a `prune()` sobre el archivo de agregados o aplicar `maxEntries` propio |
| Llamada a `HistoryStore::prune()` | ⚠️ Existe pero sin invocantes | Invocarla en el bloque horario de `loop()` cuando se prefiera descartar en vez de agregar |
| Hot-plug de la SD | ❌ `enableSd()` solo corre en `setup()` | Reintentar `SD.begin()` en el bloque periódico de `loop()` |
| Rotación por tamaño real | ⚠️ Solo por cantidad de líneas | Usar `File.size()` como disparador adicional |
| Exportación CSV | ✅ Sí está: `?format=csv` + botón «CSV histórico» | — |
| Presión de escritura / `fsync` | ❌ Sin control: se escribe con `println()` y se cierra el archivo | Abrir/cerrar por lote para reducir ciclos de la SD |
| Escritura de eventos en la SD | ❌ Los eventos viven en LittleFS, nunca en la SD | Unificar en la SD cuando esté disponible |

---

## Ver también

- [Alarmas y reglas](Alarmas-y-reglas.md) · [Magnitudes derivadas](Magnitudes-derivadas.md)
- [Energía y consumo](Energia-y-consumo.md) · [Calibración](Calibracion.md)
- [API REST](API-REST.md) · [Referencia de configuración](Referencia-configuracion.md)
- [Backup y restauración](Backup-y-restauracion.md) · [Diagramas](Diagramas.md)
