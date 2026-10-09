---
tags:
  - sema
  - alarmas
  - reglas
---

# Alarmas y reglas

> **Tipo:** Referencia
> **Estado:** En desarrollo
> **Fecha:** 2026-10-08
> **Firmware:** v1.103.0

Cómo se definen, evalúan y registran las alarmas de SEMA. Fuentes:
`include/core/alarms/Rule.hpp`, `include/core/alarms/RuleEngine.hpp`,
`src/core/alarms/RuleEngine.cpp`, `include/core/ConfigManager.hpp`,
`src/core/ConfigManager.cpp`, `src/core/SemaCore.cpp`, `include/core/EventBus.hpp`,
`include/core/events/EventLog.hpp`, `src/core/web/HttpServer.cpp`.

---

## 1. Modelo: la regla

`Rule` (`include/core/alarms/Rule.hpp`) tiene exactamente cinco campos:

| Campo C++ | Tipo | Origen en config | Descripción |
|-----------|------|------------------|-------------|
| `id` | `String` | `rules[].name` | Id de la regla; viaja como `correlationId` del evento |
| `sensorId` | `String` | `rules[].sensor_id` | Sensor a vigilar; `""` = cualquier sensor |
| `channelId` | `String` | `rules[].channel_id` | Canal lógico a comparar (`temperature`, `voltage`, …) |
| `op` | `RuleOp` | `rules[].op` | Operador de comparación |
| `threshold` | `float` | `rules[].value` | Umbral |

`RuleOp` es un `enum class : uint8_t` con **cuatro** valores:

| Enum | Símbolo | Nombre en config |
|------|---------|------------------|
| `RuleOp::Gt` | `>` | `gt` |
| `RuleOp::Lt` | `<` | `lt` |
| `RuleOp::Ge` | `>=` | `ge` |
| `RuleOp::Le` | `<=` | `le` |

`parseRuleOp()` (header, `inline`) compara con `strcmp`: cualquier cadena distinta de
`"lt"`, `"ge"` o `"le"` —incluido un `nullptr` y un `"gt"` explícito— cae en
`RuleOp::Gt`. No hay error de configuración por operador desconocido.
`RuleEngine` no tiene `removeRule()` ni edición: solo `addRule()`, `clear()` y `evaluate()`.

---

## 2. Formato exacto de `rules[]` en la configuración

`RuleSpec` (`include/core/ConfigManager.hpp`) se serializa así
(`ConfigManager::serialize()`, leído en `parseInto()`):

```json
"rules": [
  { "name": "high_temp",     "sensor_id": "EXT",   "channel_id": "temperature", "op": "gt", "value": 40.0 },
  { "name": "battery_low",   "sensor_id": "BATT",  "channel_id": "voltage",     "op": "lt", "value": 11.5 },
  { "name": "cualquier_gust","sensor_id": "",      "channel_id": "wind_gust",   "op": "ge", "value": 20.0 }
]
```

| Clave | Tipo | Default si falta | Notas |
|-------|------|------------------|-------|
| `name` | str | `""` | Si queda vacío, el evento sale con `rule: ""` (no se valida) |
| `sensor_id` | str | `""` | `""` = compara contra **todos** los sensores |
| `channel_id` | str | `""` | Comparación **exacta y sensible a mayúsculas** contra `Measurement::channelId` |
| `op` | str | `"gt"` | `gt` · `lt` · `ge` · `le`; otro valor → `gt` |
| `value` | float | `0.0` | Umbral |

Notas de comportamiento:

- La comparación es contra el **`channelId`**, no contra `measurement`. Para la mayoría de
  los drivers coinciden, pero conviene verificarlo por driver (ver
  [Calibración §4](Calibracion.md)).
- Si `rules[]` está vacío, `SemaCore::applyRules()` carga **una** regla de fábrica con los
  valores literales del código: `id = "high_temp"`, `sensorId = "EXT"`,
  `channelId = "temperature"`, `op = RuleOp::Gt`, `threshold = 40.0f`.
- Las reglas no tienen campo `enabled`: para desactivar una hay que borrarla del array.
- Las reglas viven en la config (clave NVS `config`) y se re-aplican sin reiniciar cuando
  se guarda desde `PUT /api/v1/config` o `POST /api/v1/backup`
  (`onConfigPut()` llama a `core_->applyRules()`).

---

## 3. Cómo se evalúan

No hay evaluación periódica propia de alarmas ni suscripción a eventos: la evaluación es
**por lote de mediciones**, dentro del ciclo de sensores de 10 s.

```cpp
// src/core/SemaCore.cpp — scheduler "sensors.read", cada 10 000 ms
sensors_.readAll();                       // drivers + calibración + DerivedEngine
http_.broadcastMeasurements(...);
for (const Measurement& m : sensors_.measurements()) {
  history_.append(m);                     // 1) histórico
  publishers_.publishAll(m);              // 2) MQTT / webhook
}
rules_.evaluate(sensors_.measurements()); // 3) alarmas
```

`RuleEngine::evaluate()` recorre el producto cartesiano **reglas × mediciones**:

```cpp
for (const Rule& r : rules_)
  for (const Measurement& m : measurements)
    if (matches(r, m)) { /* publica EventType::Alarm */ }
```

`matches()`:

1. Si `r.sensorId` no está vacío y `m.sensorId != r.sensorId` → no matchea.
2. Si `m.channelId != r.channelId` → no matchea.
3. Aplica el operador sobre `m.value` (float) contra `r.threshold`.

```text
cada 10 s ─► readAll() ─► Vector<Measurement> ─► RuleEngine::evaluate()
                                                    │ por cada match
                                                    ▼
                                        Event{ type = Alarm } ─► EventBus
                                                                    ├─► EventLog (RAM + /events.jsonl)
                                                                    └─► handler serial [ALARM]
```

Consecuencias verificables de este diseño:

- **Sin detección de flanco**: si la condición se mantiene, la alarma se emite **una vez
  por ciclo** (cada 10 s), no una vez por transición.
- **Sin histéresis, sin cooldown, sin duración mínima y sin combinación lógica.**
- **Sin filtro de calidad**: una medición con `quality = OUT_OF_RANGE`,
  `SENSOR_DISCONNECTED` o `STALE` sigue disparando reglas (solo se compara `value`).
- **Incluye derivadas**: `DerivedEngine` agrega sus mediciones (`sensorId = "DERIVED"`)
  antes de `evaluate()`, así que una regla sobre `sensor_id: ""` y
  `channel_id: "heat_index"` también se evalúa.

---

## 4. El evento que se publica

`RuleEngine::evaluate()` construye el evento a mano:

| Campo `Event` | Valor puesto por el motor | En `/api/v1/events` |
|---------------|---------------------------|---------------------|
| `id` | `0` (siempre) | no se expone |
| `timestampMs` | `millis()` (uptime, **no** época) | `ts` |
| `source` | `m.sensorId` | `source` |
| `type` | `EventType::Alarm` | `type: "alarm"` |
| `severity` | `Severity::Warning` (**fijo**) | `severity: "WARNING"` |
| `value` | `static_cast<int32_t>(m.value * 100.0f)` | `value` |
| `correlationId` | `r.id` | `rule` |
| `target` | `""` | no se expone |

Detalles numéricos:

- `value` es el valor **escalado ×100** y truncado a entero de 32 bits: 23,4 °C → `2340`;
  1013,25 hPa → `101325`. El casteo trunca hacia cero, así que −0,5 → `-50`.
- El rango de `int32` (±2 147 483 647) se desborda si `|valor| > 21 474 836`, algo posible
  en magnitudes con unidades grandes (p. ej. una potencia en W o un contador de pulsos).
- `timestampMs = millis()` significa que `ts` **no es una fecha**. La página `/events` lo
  formatea con `new Date(e.ts).toLocaleString()`, es decir lo interpreta como epoch en
  milisegundos: en un equipo real las horas que se muestran son de enero de 1970.

---

## 5. Acciones posibles (las que existen hoy)

| Acción | ¿Implementada? | Dónde |
|--------|----------------|-------|
| Publicar `EventType::Alarm` en el EventBus | ✅ | `RuleEngine::evaluate()` |
| Registro persistente del evento | ✅ (sin condición) | `EventLog::onEvent()` → `/events.jsonl` (LittleFS) |
| Log por serial | ✅ | Suscriptor `EventType::Alarm` en `SemaCore::setup()`: `[ALARM] <regla> → <sensor> = <value>` |
| Consulta por HTTP | ✅ | `GET /api/v1/events` y `GET /api/v1/alarms` |
| Webhook / MQTT | ❌ | `PublisherManager::publishAll()` solo recibe **mediciones**; los eventos no pasan por los publicadores |
| Acción local (GPIO, relé, sirena) | ❌ | El motor no toca `GpioManager` |
| Notificación (mail, push) | ❌ | — |
| Combinación lógica de reglas | ❌ | — |
| Condición por cambio, ausencia o duración | ❌ | — |

```text
Regla cumple ──► Event bus ──┬──► EventLog (RAM 100 + /events.jsonl, rota a 200)
                             ├──► Serial [ALARM]
                             └──► GET /api/v1/alarms
                                     ▲
                          módulo externo / script
                                     │
                                     └──► POST /api/v1/gpio  (actuación real)
```

!!! warning "La actuación automática NO está en el firmware"
    Para cerrar un relé a partir de una alarma hace falta un actor externo que consulte
    `GET /api/v1/alarms` (o que escuche MQTT) y ejecute `POST /api/v1/gpio`. Patrón
    documentado en [Actuadores y salidas](Actuadores-y-Salidas.md) y
    [Control PID](Control-PID.md).

!!! note "La severidad no es configurable"
    `EventType`/`Severity` (`include/core/EventBus.hpp`) ofrecen `DEBUG`, `INFO`,
    `NOTICE`, `WARNING`, `ERROR`, `CRITICAL`, pero el motor de reglas escribe siempre
    `Warning`. No hay campo de severidad ni de prioridad en `rules[]`.

---

## 6. HTTP: `GET /api/v1/events` y `GET /api/v1/alarms`

Ambos handlers responden desde la **cola en RAM** del `EventLog` (máximo 100 eventos, los
más recientes), no desde el archivo, y **no exigen autenticación**.

```json
{ "events": [ { "ts": 523410, "type": "alarm", "source": "EXT", "rule": "high_temp",
                "severity": "WARNING", "value": 4100 } ] }
```

`/api/v1/alarms` usa el mismo formato con la clave raíz `alarms` y descarta todo lo que no
sea `type == "alarm"`:

```json
{ "alarms": [ { "ts": 523410, "source": "EXT", "rule": "high_temp",
                "severity": "WARNING", "value": 4100 } ] }
```

| Endpoint | Clave raíz | Filtro | Límite/`limit` |
|----------|-----------|--------|----------------|
| `GET /api/v1/events` | `events` | ninguno (incluye `system`) | ❌ no acepta parámetros; tamaño acotado por la cola de 100 |
| `GET /api/v1/alarms` | `alarms` | `type == alarm` | ❌ no acepta parámetros |

En la práctica, y sin módulos extra, la cola contiene solo dos tipos de evento: `system`
(uno por arranque, `rule: "boot"`) y `alarm` (los del motor de reglas). Ningún otro
componente publica en el bus.

---

## 7. Ejemplos reales

Los tres ejemplos son válidos contra el código actual (los `channel_id` existen en los
drivers correspondientes) y se aplican con `PUT /api/v1/config`:

**Temperatura exterior alta (la regla de fábrica)**

```json
{ "name": "high_temp", "sensor_id": "EXT", "channel_id": "temperature", "op": "gt", "value": 40.0 }
```

**Batería baja** (necesita un sensor `ADC` con `id: "BATT"`, `channel: "voltage"`)

```json
{ "name": "battery_low", "sensor_id": "BATT", "channel_id": "voltage", "op": "lt", "value": 11.5 }
```

**Ráfaga fuerte, sin importar qué sensor la mida**

```json
{ "name": "strong_gust", "sensor_id": "", "channel_id": "wind_gust", "op": "ge", "value": 20.0 }
```

**Humedad de suelo baja** (canal analógico definido por configuración; ver
[Calibración §7](Calibracion.md))

```json
{ "name": "soil_dry", "sensor_id": "SOIL", "channel_id": "soil_moisture", "op": "lt", "value": 25.0 }
```

Verificación rápida por HTTP:

```bash
curl -X PUT http://sema.local/api/v1/config -H 'X-API-Key: <api_key>' \
     -H 'Content-Type: application/json' --data @config.json
curl http://sema.local/api/v1/alarms
```

---

## 8. Especificación vs. implementación

`README.md` §63 pide que cada alarma permita *Habilitar, Deshabilitar, Umbral,
Histéresis, Tiempo, Prioridad, Acción y Notificación*; `D-0059` (docs/DUDAS-Y-DECISIONES.md)
amplía a condiciones por *cambio, ausencia, duración y combinación lógica* con acciones de
*webhook, publisher, registro y acción local*.

| Requisito | Estado |
|-----------|--------|
| Umbral con `>`, `<`, `>=`, `<=` | ✅ |
| Configurable por NVS/HTTP | ✅ |
| Registro persistente del evento | ✅ |
| Múltiples sensores y reglas | ✅ |
| Habilitar/deshabilitar por regla | ❌ (borrar del array) |
| Histéresis | ❌ (`README.md` §63) |
| Tiempo / duración mínima / cooldown | ❌ |
| Prioridad y severidad configurables | ❌ (siempre `WARNING`) |
| Acción local (GPIO) automática | ❌ (requiere módulo externo) |
| Notificación (webhook/MQTT de eventos) | ❌ |
| Combinación lógica de condiciones | ❌ |
| Condición por cambio / ausencia | ❌ |
| Reglas sobre la calidad (STALE, OUT_OF_RANGE) | ❌ |

**Vías de solución razonables** (sin romper el esquema `schema_version: 1`):

1. Agregar `enabled`, `severity`, `hysteresis` y `cooldown_s` a `RuleSpec`/`Rule`, con
   estado por regla en RAM (última vez disparada y último valor).
2. Evaluar por flanco: guardar `matched` por regla y publicar solo en la transición.
3. Publicar los eventos por `PublisherManager` (hoy solo acepta `Measurement`).
4. Agregar una acción local opcional (`gpio_pin` + `gpio_value` en la regla) reutilizando
   `GpioManager::write()`, con la salvedad de seguridad de que las salidas son físicas.

---

## Ver también

- [Almacenamiento e histórico](Almacenamiento-e-historico.md) · [Calibración](Calibracion.md)
- [Magnitudes derivadas](Magnitudes-derivadas.md) · [Energía y consumo](Energia-y-consumo.md)
- [API REST](API-REST.md) · [Referencia de configuración](Referencia-configuracion.md)
- [Actuadores y salidas](Actuadores-y-Salidas.md) · [Control PID](Control-PID.md)
- [Enumeraciones y tipos](Enumeraciones-y-tipos.md)
