---
tags:
  - sema
  - desarrollo
---

# Guía de desarrollo (extender SEMA)

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Cómo extender SEMA **de verdad**: agregar un sensor, un publicador, una regla, una
magnitud derivada, un endpoint REST o un módulo, con las rutas reales del repositorio,
las convenciones que se verifican en cada cambio y el flujo obligatorio de release.

Repo: <https://github.com/AlessandroKlein/SEMA> · Firmware: **v1.103.0**

## 1. Mapa rápido: dónde vive cada cosa

| Quiero tocar… | Cabecera | Implementación |
|---------------|----------|----------------|
| Arranque y orquestación | `include/core/SemaCore.hpp` | `src/core/SemaCore.cpp` |
| Configuración (claves JSON) | `include/core/ConfigManager.hpp` | `src/core/ConfigManager.cpp` |
| Sensores (interfaz y registro) | `include/core/sensors/Sensor.hpp` · `SensorManager.hpp` | `src/core/sensors/SensorManager.cpp` |
| Fábrica de drivers | `include/core/sensors/SensorFactory.hpp` | `src/core/sensors/SensorFactory.cpp` |
| Un driver concreto | `include/core/sensors/<Modelo>Sensor.hpp` | `src/core/sensors/<Modelo>Sensor.cpp` |
| Detección I²C | `include/core/sensors/I2cScanner.hpp` | `src/core/sensors/I2cScanner.cpp` |
| Modelo de datos | `include/core/Measurement.hpp` | — (solo cabecera) |
| Eventos | `include/core/EventBus.hpp` | `src/core/EventBus.cpp` |
| Reglas y alarmas | `include/core/alarms/Rule.hpp` · `RuleEngine.hpp` | `src/core/alarms/RuleEngine.cpp` |
| Derivadas | `include/core/derived/DerivedEngine.hpp` · `DerivedCalculator.hpp` | `src/core/derived/*.cpp` |
| Publicadores | `include/core/publishers/Publisher.hpp` · `PublisherManager.hpp` | `src/core/publishers/*.cpp` |
| API/web/dashboard | `include/core/web/HttpServer.hpp` | `src/core/web/HttpServer.cpp` |
| Almacenamiento | `include/core/storage/Storage.hpp` · `HistoryStore.hpp` | `src/core/storage/*.cpp` |
| Módulos | `include/core/Module.hpp` · `ModuleRegistry.hpp` | `src/core/ModuleRegistry.cpp` |
| Red | `include/core/network/WiFiManager.hpp` | `src/core/network/WiFiManager.cpp` |
| Energía | `include/core/PowerManager.hpp` | `src/core/PowerManager.cpp` |
| Perfil de hardware | `include/hw/HwProfile.hpp` · `include/core/BoardProfile.hpp` | — (macros de build) |
| Versión | `include/core/Version.hpp` | — |

`src/main.cpp` son **18 líneas**: `setup()` y `loop()` llaman a `SemaCore::instance()`.
Toda la lógica va en módulos.

## 2. Convenciones del repositorio

1. **Una clase por archivo**: `.hpp` en `include/…`, `.cpp` en `src/…`, con la misma
   ruta relativa (`include/core/sensors/MiSensor.hpp` ↔ `src/core/sensors/MiSensor.cpp`).
2. **Comentar el por qué, no el qué.** Cada archivo cita la decisión que lo justifica
   (por ejemplo `// D-0056`, `// README.md §173`). Un `TODO`/`FIXME` lleva motivo.
3. **No inventar APIs, librerías, flags ni comandos**: se verifica contra el código o
   la documentación oficial. Ante la duda, preguntar.
4. **No hardcodear secretos** (claves, tokens): van a la configuración (NVS).
5. **No refactorizar** código que no se está tocando (salvo limpieza local < 10 líneas).
6. **Un commit por cambio lógico**, en inglés y con Conventional Commits
   (`feat(sensors): …`, `fix(web): …`, `docs: …`, `refactor: …`).
7. **Español rioplatense** en documentos y comentarios largos; identificadores de
   código en inglés.
8. **Nada de bibliotecas externas en el Core** (D-0043): cada driver encapsula su
   librería y expone solo la interfaz SEMA.

## 3. Agregar un sensor nuevo

### 3.1 Crear el driver

`include/core/sensors/MiSensor.hpp`:

```cpp
#pragma once

#include "core/sensors/Sensor.hpp"

namespace sema {

class MiSensor : public Sensor {
public:
  MiSensor(const char* id, uint8_t sda, uint8_t scl);

  const char* id() const override;         // id lógico ("EXT", "MI_01", …)
  const char* model() const override;      // "MI_SENSOR" (valor de la config)
  const char* interface() const override;  // "I2C" | "SPI" | "UART" | "ADC" | "GPIO" | "1-Wire"
  bool begin() override;                   // inicializa y detecta el hardware
  uint8_t measure(Measurement out[], uint8_t max) override;  // devuelve la cantidad
  bool healthy() const override;

private:
  const char* id_;
  uint8_t sda_, scl_;
  bool ok_ = false;
  uint32_t sequence_ = 0;
};

}  // namespace sema
```

`src/core/sensors/MiSensor.cpp` (modelo real: `Sht40Sensor.cpp`):

```cpp
#include "core/sensors/MiSensor.hpp"

#include <Wire.h>

namespace sema {

MiSensor::MiSensor(const char* id, uint8_t sda, uint8_t scl)
    : id_(id), sda_(sda), scl_(scl) {}

const char* MiSensor::id() const { return id_; }
const char* MiSensor::model() const { return "MI_SENSOR"; }
const char* MiSensor::interface() const { return "I2C"; }

bool MiSensor::begin() {
  Wire.begin(sda_, scl_);
  ok_ = /* inicializar la librería */ true;
  return ok_;
}

bool MiSensor::healthy() const { return ok_; }

uint8_t MiSensor::measure(Measurement out[], uint8_t max) {
  if (!ok_ || max < 1) {
    return 0;  // nunca escribir fuera del buffer
  }
  const uint32_t ts = nowEpoch();  // "core/Time.hpp": epoch, o uptime si no hay NTP

  out[0].sensorId = id_;
  out[0].channelId = "temperature";
  out[0].measurement = "temperature";
  out[0].value = 0.0f;  // valor medido
  out[0].unit = "degC";
  out[0].quality = Quality::Valid;
  out[0].sequence = ++sequence_;
  out[0].timestamp = ts;
  return 1;
}

}  // namespace sema
```

Reglas del contrato:

- `measure()` devuelve **cuántas** mediciones escribió y **nunca** supera `max`.
  `SensorManager` le pasa un buffer de **4** (`Measurement buffer[4]`), así que un
  driver que aporte más de 4 mediciones necesita ampliar ese buffer.
- `begin()` se llama una vez al arrancar (y en cada `applySensors()`); debe ser
  idempotente.
- El driver **retiene el puntero** de `id` (`const char*`), que apunta al `String` de
  la configuración: toda ruta que cambie `sensors[]` debe volver a llamar a
  `SemaCore::applySensors()` (lo hace `PUT /api/v1/config`).
- Si la librería tiene un objeto global (`static Adafruit_X x;`), **solo se puede
  instanciar un sensor de ese modelo** por estación.

### 3.2 Registrarlo en la fábrica

`src/core/sensors/SensorFactory.cpp`:

```cpp
#include "core/sensors/MiSensor.hpp"      // arriba

  if (spec.model == "MI_SENSOR") {
    return new MiSensor(spec.id.c_str(), spec.sda, spec.scl);
  }
```

Si el driver necesita más campos, están en `SensorSpec`
(`include/core/ConfigManager.hpp`): `id`, `model`, `enabled`, `address`, `rom`, `sda`,
`scl`, `bus`, `uart`, `uart_port`, `pin`, `rx`, `tx`, `channel`, `unit`, `scale`,
`offset`.

### 3.3 Agregar la librería

Si el driver usa una librería nueva, se agrega en `platformio.ini` (sección `[env:base]`
→ `lib_deps`, con versión fijada) y se verifica que compile:

```ini
lib_deps =
    adafruit/Adafruit SHT4x Library@^1.0.0
    mi-fabricante/Mi Libreria@^1.2.3
```

### 3.4 Usarlo desde la configuración

```json
{
  "sensors": [
    { "id": "MI_01", "model": "MI_SENSOR", "enabled": true, "sda": 21, "scl": 22,
      "channel": "temperature", "unit": "degC", "scale": 1.0, "offset": 0.0 }
  ]
}
```

!!! warning "Tres trampas frecuentes"
    - `enabled: false` (default) hace que `applySensors()` **saltee** el sensor.
    - El campo `address` se parsea pero **no llega al driver**: la dirección es la del
      código/librería.
    - Si el modelo no existe en la fábrica, se imprime
      `Sensor desconocido: MI_01 (modelo MI_SENSOR)` y no se registra.

## 4. Agregar un publicador

1. Implementar la interfaz (`include/core/publishers/Publisher.hpp`):

```cpp
class MiPublisher : public Publisher {
public:
  const char* id() const override { return "mi_pub"; }
  bool enabled() const override { return url_.length() > 0; }
  bool publish(const Measurement& m) override;
private:
  String url_;
};
```

2. Instanciarlo como miembro de `SemaCore` (`include/core/SemaCore.hpp`) y registrarlo
   en `SemaCore::setup()`:

```cpp
publishers_.registerPublisher(&miPublisher_);
applyPublishers();   // configura desde config.publishers
```

3. Leer su configuración en `SemaCore::applyPublishers()` y agregar las claves JSON en
   `ConfigManager::serialize()`/`parseInto()` más el `struct PublishersConfig`.
4. **No bloquear**: el webhook usa `http.setTimeout(2000)` justamente para eso
   (D-0010). Un publicador lento se descarta, no frena la adquisición.

## 5. Agregar una regla

### 5.1 Por configuración (lo habitual)

```json
{ "rules": [ { "name": "helada", "sensor_id": "EXT", "channel_id": "temperature",
               "op": "lt", "value": 0.0 } ] }
```

- Operadores válidos: `gt`, `lt`, `ge`, `le` (`parseRuleOp()`; un valor desconocido cae
  a `gt`).
- `sensor_id` vacío (`""`) significa **cualquier sensor**; `channel_id` es obligatorio
  (no hay comodín).
- Cada regla que se cumple publica un `Event{type=Alarm, severity=Warning}` con
  `correlationId = name` y `value = valor × 100`; el `EventLog` lo persiste.

### 5.2 Regla por defecto y evaluación en código

Si `rules[]` está vacío, `SemaCore::applyRules()` agrega
`{high_temp, EXT, temperature, Gt, 40.0}`. La evaluación vive en
`RuleEngine::evaluate()`, que corre **en cada ciclo de 10 s** sobre las mediciones
recientes.

!!! danger "Las reglas no tienen memoria"
    `RuleEngine` no detecta flancos: mientras la condición siga siendo verdadera,
    publica una alarma **en cada ciclo**. Con `temperature > 40` sostenida son 6
    alarmas por minuto. Si necesitás avisar una sola vez, filtrá por
    `correlationId`+`sequence` en el consumidor o implementá histéresis/estado en el
    motor (hoy no existe).

## 6. Agregar una magnitud derivada

Hay dos caminos, y conviene elegir según dónde deba aparecer el valor:

| Camino | Archivo | Cuándo |
|--------|---------|--------|
| `DerivedEngine::compute()` | `src/core/derived/DerivedEngine.cpp` | La derivada debe **viajar con cada medición** (almacenarse, publicarse por MQTT/webhook). |
| `DerivedCalculator::compute()` | `src/core/derived/DerivedCalculator.cpp` | La derivada es de **presentación/API** (se calcula al responder `/api/v1/sensors`). |

Ejemplo en `DerivedCalculator` (patrón real de `vpd`):

```cpp
// 1) fórmula en el namespace anónimo
float miIndice(float tC, float rh) { return tC * rh / 100.0f; }

// 2) dentro de compute(), sólo si están las entradas
if (!isnan(t) && !isnan(rh)) {
  emit(out, "mi_indice", miIndice(t, rh), "index");
}
```

- `emit()` publica `sensorId = "DERIVED"`, `channelId = measurement` y
  `quality = Valid`.
- Si la magnitud tiene unidad imperial, agregá su conversión en
  `DerivedCalculator::convertUnit()`.
- Si la derivada depende de la configuración (altitud, veleta), leela de `sys_`
  (`configure()` la copia desde `SystemConfig`).

## 7. Agregar un endpoint REST

1. Declarar el handler en `include/core/web/HttpServer.hpp`:

```cpp
void onMiEndpoint();
```

2. Registrar la ruta en `HttpServer::begin()` (`src/core/web/HttpServer.cpp` §52-121):

```cpp
server_.on("/api/v1/mi-endpoint", HTTP_GET, [this]() { onMiEndpoint(); });
```

3. Implementar el handler: validar, armar el JSON y responder.

```cpp
void HttpServer::onMiEndpoint() {
  if (!webAuthed()) {                                  // sesión o X-API-Key
    server_.send(401, "application/json", "{\"error\":\"unauthorized\"}");
    return;
  }
  DynamicJsonDocument doc(512);                        // tamaño acorde al payload
  doc["mi_campo"] = core_->sensors().count();
  String out;
  serializeJson(doc, out);
  server_.send(200, "application/json", out);
}
```

Convenciones de la API:

| Situación | Respuesta |
|-----------|-----------|
| Falta el body | `400 {"error":"body required"}` |
| JSON inválido | `400 {"error":"invalid json"}` |
| No autorizado | `401 {"error":"unauthorized"}` |
| Login bloqueado | `429` |
| Error interno | `500 {"error":"…"}` |
| Éxito de escritura | `200 {"ok":true}` |

- Para endpoints de **riesgo** (reinicio, OTA, escritura de pines) usá `webAuthed()`
  como mínimo; el OTA además valida `authorized()` en la subida.
- Si cambiás la configuración, después de `apply()` llamá a los `applyXxx()`
  correspondientes para que el cambio tome efecto **sin reiniciar**.
- Documentá la ruta en [API REST](API-REST.md) y en
  [Referencia de código](Referencia-de-codigo.md) (regla de trabajo §2-bis).

## 8. Agregar un módulo

`Module` (`include/core/Module.hpp`) define el contrato:

```cpp
class MiModulo : public Module {
public:
  const char* id() const override { return "mi.modulo"; }
  const char* version() const override { return "1.0.0"; }
  bool install() override;    // recursos
  bool configure() override;  // lee su configuración
  bool enable() override;     // habilita
  void start() override;      // arranca
  void loop() override;       // trabajo periódico
  void stop() override;
  bool disable() override;
};
```

Se registra en el `ModuleRegistry` del Core:

```cpp
modules_.registerModule(miModulo_);   // id único; el registro rechaza duplicados
```

`SemaCore::setup()` llama a `modules_.enableAll()` (recorre
`install → configure → enable → start` para los módulos en `Available`) y
`loopAll()` los ejecuta en cada vuelta del `loop()` si están `Enabled` o `Running`.

!!! warning "Estado de los módulos"
    No hay instalación/desinstalación en runtime ni permisos: `enableAll()` los habilita
    todos al arrancar. Un módulo con `id` repetido no se registra
    (`registerModule()` devuelve `false`).

## 9. Tocar la configuración

Toda clave nueva se toca en **tres lugares** de `ConfigManager`:

1. `struct` correspondiente en `include/core/ConfigManager.hpp` (default en la
   declaración).
2. `serialize()` → `doc["seccion"]["clave"] = …`.
3. `parseInto()` → `c.seccion.clave = doc["seccion"]["clave"] | <default>;`
4. Si la clave es imprescindible, agregá la validación en `validate()`.

Límites y avisos:

- El documento JSON es `DynamicJsonDocument doc(16384)`: **16 KB** de configuración.
  Una configuración que no parsea suele ser una que se pasó de tamaño.
- La ventana de configuración es **transaccional**: `apply()` respalda, guarda y
  revierte si falla. Si tu clave depende de hardware, validala antes.
- `saveDashboardLayout()` usa una **clave NVS separada** (`layout`) para no exceder el
  tamaño de una entrada.
- Cambiar el significado de una clave existente es un cambio **MAJOR** de versión
  (ver [Versionado](../inicio/Versionado.md)); agregar una clave con default es
  MINOR.

## 10. Flujo obligatorio de trabajo

Tras **cada** cambio de código (`docs/REGLAS-DE-TRABAJO.md` §1), en este orden:

```text
1. pio run -e esp32doit-devkit-v1          → debe dar SUCCESS
2. Bump SemVer en include/core/Version.hpp → SEMA_FW_VERSION sin "v"
   (feat → MINOR · fix → PATCH · BREAKING CHANGE → MAJOR)
3. CHANGELOG.md                            → entrada ## [X.Y.Z] - AAAA-MM-DD
4. firmware_manifest.json                  → version + SHA-256 real del .bin
5. Commit (Conventional Commits)           → un cambio lógico = un commit
6. Tag + push                              → git tag -a vX.Y.Z && git push origin main --tags
7. Release en GitHub                       → gh release create vX.Y.Z … firmware.bin
8. Documentación                           → repo SEMA (docs/*) y repo Docs (docs/es/sema/*)
```

Checklist de release (`REGLAS-DE-TRABAJO.md` §6):

- [ ] Tag `vX.Y.Z` creado.
- [ ] Release de GitHub con notas y artefactos (`.bin`, filesystem).
- [ ] `firmware_manifest.json` con el SHA-256 **real** del binario.
- [ ] `CHANGELOG.md` actualizado (Keep a Changelog: Added/Changed/Fixed/Removed).
- [ ] `README.md` del repo actualizado si cambió el comportamiento.
- [ ] Páginas del repo Docs sincronizadas.
- [ ] `python -m mkdocs build` en SUCCESS antes de publicar la documentación.

Mapa «qué cambió → qué página actualizar» (resumen):

| Cambio | Página |
|--------|--------|
| Endpoint nuevo o modificado | [API REST](API-REST.md) + [Referencia de código](Referencia-de-codigo.md) |
| Clave nueva de configuración | [Referencia de configuración](Referencia-configuracion.md) + [Variables modificables](Variables-modificables.md) |
| Sensor nuevo | [Sensores](Sensores.md) + [Compatibilidad](Compatibilidad.md) + [Referencia de código](Referencia-de-codigo.md) |
| Pin nuevo o cambiado | [Guía de pines](Guia-de-pines.md) + [Hardware y conexiones](Hardware-y-Conexiones.md) |
| Clase o módulo nuevo | [Referencia API interna](Referencia-API-interna.md) + [Referencia de código](Referencia-de-codigo.md) |
| Enum o valor nuevo | [Enumeraciones y tipos](Enumeraciones-y-tipos.md) |
| Decisión de arquitectura | [Decisiones](Decisiones.md) (+ ADR nuevo) |
| Limitación o bloqueo nuevo | [Mejoras y roadmap](Mejoras-y-roadmap.md) + la página afectada |
| Release | [CHANGELOG](CHANGELOG.md) + [Registro de cambios](Registro-de-cambios.md) + [Evolución](Evolucion.md) |

!!! info "Docs-only no genera release"
    Un commit que solo toca documentación **no** bumpea versión ni crea tag.

## 11. Errores comunes

| Error | Qué pasa | Cómo evitarlo |
|-------|----------|---------------|
| Escribir más de `max` mediciones en `measure()` | Corrupción de memoria (el buffer es de 4) | Respetar `max` y devolver la cantidad real |
| Devolver `true` en `begin()` sin verificar el hardware | El sensor figura `healthy` y entrega basura | Guardar el resultado real de la librería en `ok_` |
| Olvidar `enabled: true` en `sensors[]` | El sensor no se registra y no hay error visible | Revisar la consola: los modelos desconocidos se avisan, los apagados no |
| Usar `address` de la config esperando que cambie la dirección | No tiene efecto | La dirección vive en el driver/librería |
| Dos drivers del mismo modelo con `static` interno | El segundo pisa al primero | Un objeto por modelo; encapsular la instancia en la clase si hace falta |
| Dos sensores `PCNT` | El driver usa `PCNT_UNIT_0` fijo: el segundo reconfigura el primero | Un solo sensor PCNT por estación |
| Cambiar `sensors[]` sin llamar a `applySensors()` | Punteros a `String` liberados (dangling) | Usar `PUT /api/v1/config` o llamar a `applySensors()` |
| Regla sin `channel_id` | Nunca se cumple (no hay comodín de canal) | Poner `channel_id` correcto o `sensor_id: ""` para cualquiera |
| Confundir regla con histéresis | Alarma repetida cada 10 s | Ver §5: el motor no tiene estado |
| Configuración de más de 16 KB | `parseInto()` falla y la config no se aplica | Reducir el JSON (el layout va aparte) |
| Publicador bloqueante | Frena el ciclo de 10 s y el watchdog puede reiniciar | Timeouts cortos y sin reintentos internos |
| `Serial.printf` con `String` | Salida rara o crash | Usar `.c_str()` |
| Bumpear versión en un cambio docs-only | Release innecesario | Solo `feat`/`fix`/BREAKING bumpean |
| `struct` nuevo sin serializar en `ConfigManager` | El valor se pierde al reiniciar | Tocar `serialize()` **y** `parseInto()` |

## 12. Depuración

| Herramienta | Para qué |
|-------------|----------|
| `pio device monitor -b 115200` | Banner de arranque, detección I²C, sensores registrados, alarmas |
| `GET /api/v1/health` | Estado, heap libre, sensores online y salud por tarea |
| `GET /api/v1/diagnostics` | Reset reason, histórico, tasks, módulos, eventos y dispositivos I²C |
| `GET /api/v1/system` | Board, flash, pines reservados, `pins_from_file`, `demo`, `native_eth` |
| `GET /api/v1/sensors` | Catálogo con `healthy` por sensor y todas las mediciones |
| `GET /api/v1/config` | Configuración viva (requiere sesión o `X-API-Key`) |
| `GET /api/v1/events` | Qué eventos se generaron y con qué severidad |
| Build `demo` (`pio run -e demo`) | Probar la web y los gráficos sin hardware |

## Ver también

- [Home](Home.md) · [Guía de inicio](Guia-de-inicio.md) · [Compilación y flasheo](Compilacion-y-flasheo.md)
- [Arquitectura](Arquitectura.md) · [Referencia de código](Referencia-de-codigo.md) ·
  [Referencia API interna](Referencia-API-interna.md) · [Módulos y ciclo de vida](Modulos-y-ciclo-de-vida.md)
- [Decisiones](Decisiones.md) · [Pruebas y validación](Pruebas-y-validacion.md) ·
  [Registro de cambios](Registro-de-cambios.md)
