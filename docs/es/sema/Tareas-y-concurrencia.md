---
tags:
  - sema
  - concurrencia
---

# Tareas y concurrencia

> **Tipo:** Referencia | **Estado:** En desarrollo | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Esta página documenta el modelo de ejecución real del firmware v1.103.0. La
conclusión principal, verificada contra el código, es que **SEMA no crea ninguna
tarea FreeRTOS propia**: todo el firmware corre en la tarea `loopTask` que crea el
core de Arduino-ESP32.

## 1. Resumen ejecutivo

| Pregunta | Respuesta verificada |
|----------|----------------------|
| ¿Cuántas tareas FreeRTOS crea SEMA? | **Ninguna** |
| ¿Hay una sola tarea `loop()`? | **Sí.** `setup()` y `loop()` corren en `loopTask` |
| ¿Hay tareas con afinidad a un núcleo? | **No** en código de SEMA |
| ¿Hay prioridades asignadas por SEMA? | **No** |
| ¿Hay semáforos, mutex o secciones críticas? | **No** (cero ocurrencias de `SemaphoreHandle_t`, `xSemaphoreCreate*`, `portMUX_TYPE` en el repo) |
| ¿Hay colas entre servicios? | **No** (cero `xQueueCreate`) |
| ¿Hay watchdog de tarea? | **Sí**, TWDT de 10 s sobre `loopTask`, más un watchdog lógico por tarea en `HealthMonitor` |
| ¿Hay condiciones de carrera? | No entre funciones propias, porque todo es una sola hebra; la pila de red del ESP-IDF sí corre en sus propias tareas internas |

## 2. Evidencia: no hay tareas propias

Búsqueda de `xTaskCreate`, `xTaskCreatePinnedToCore`, `Task(` y `new Task` en todos
los `.hpp`/`.cpp` del proyecto:

- **Única aparición de `xTaskCreate`**: `src/core/runtime/Task.cpp`, línea 20,
  dentro de `sema::Task::start()`.
- **Ninguna instanciación de `sema::Task`**: no existe `new Task`, ni un objeto
  `Task` como miembro, ni una llamada a `.start()` sobre esa clase.
- `include/core/runtime/Task.hpp` está incluido únicamente por
  `src/core/runtime/Task.cpp`. Ningún otro archivo lo usa.

Conclusión: `sema::Task` es una **semilla** de Runtime Manager (así la describe su
propio comentario de cabecera: "semilla del Runtime Manager", D-0012/D-0039/D-0052/
D-0053), compilada pero nunca ejercitada. ❌ No hay Runtime Manager activo.

### Detalle de la clase no usada

| Aspecto | Valor en `Task.hpp`/`Task.cpp` |
|---------|-------------------------------|
| Firma del constructor | `Task(TaskFunction fn, const char* name, uint32_t stackBytes, uint32_t priority, void* arg = nullptr)` |
| Función de tarea | `void (*)(void*)` |
| Creación | `xTaskCreate(...)` → afinidad **AUTO** (`tskNO_AFFINITY`), según el comentario de la línea 19 |
| `start()` | Devuelve `false` si ya hay handle; si no, `xTaskCreate` y compara con `pdPASS` |
| `stop()` | `vTaskDelete(handle)` y deja el handle en `nullptr` |
| Destructor | Llama a `stop()` |
| Copia | Deshabilitada (`= delete`) |
| `running()` | `handle_ != nullptr` |

No hay valores de stack ni de prioridad concretos que documentar, porque no se
instancia.

## 3. La tarea real: `loopTask`

El framework Arduino-ESP32 crea `loopTask` antes de llamar a `setup()`. Todo el
firmware de SEMA corre ahí:

```text
FreeRTOS scheduler (esp32)
 ├── tareas internas del sistema: IDLE0/IDLE1, Tmr Svc, ipc0/ipc1, esp_timer,
 │   wifi/bt (IDs internos del IDF), tiT/lwIP, TWAI (si SEMA_USE_CAN),
 │   esp_eth (si hay Ethernet), etc.  ← NO las crea SEMA
 └── loopTask  ← creada por el core Arduino
       ├── setup()  → SemaCore::instance().setup()      [main.cpp:12-14]
       └── loop()   → SemaCore::instance().loop()       [main.cpp:16-18]
             ├── watchdog_.feed()
             ├── modules_.loopAll()        (lista vacía)
             ├── wifi_.loop()
             ├── http_.loop()  → server_.handleClient() + ws_.loop()
             ├── zigbee_.loop()            (si SEMA_USE_ZIGBEE)
             ├── ethernet_.loop()          (si SEMA_USE_ETHERNET; no-op)
             ├── agregación de histórico cada 3600 s
             └── scheduler_.run()          → tareas periódicas
```

Nota terminológica: en el código, la palabra "tarea" se usa para dos cosas
distintas. Las **tareas del `Scheduler`** (`core.heartbeat`, `sensors.read`) son
callbacks cooperativos que corren en `loopTask`, no tareas FreeRTOS. Las
**tareas del `HealthMonitor`** son entradas de vigilancia con nombre y timeout. Solo
`loopTask` es una tarea FreeRTOS real.

## 4. Tareas del `Scheduler` (callbacks, no hilos)

Registradas en `SemaCore::setup()` (`src/core/SemaCore.cpp`, líneas 183-198) y
ejecutadas por `Scheduler::run()` (`src/core/Scheduler.cpp`, líneas 12-20) al final
de cada vuelta del `loop()`:

| Nombre | Intervalo | Qué hace | Línea |
|--------|-----------|----------|-------|
| `core.heartbeat` | 5 000 ms | `health_.tick()` + `health_.taskHeartbeat("core.heartbeat")` | SemaCore.cpp:183-186 |
| `sensors.read` | 10 000 ms | `sensors_.readAll()` → `setSensorStats()` → `taskHeartbeat()` → `broadcastMeasurements()` → `history_.append()` ×N → `publishers_.publishAll()` ×N → `rules_.evaluate()` | SemaCore.cpp:188-198 |

Características del planificador:

- **Es cooperativo y por sondeo**: `run()` recorre el vector y compara
  `millis() - task.lastRun >= task.intervalMs`. No hay temporizadores por hardware
  ni notificaciones.
- **Precisión dependiente de la carga**: si una vuelta del `loop()` tarda más que
  un intervalo, la tarea se ejecuta igual pero se pierden ciclos. `lastRun` se
  actualiza al valor de `millis()` del momento de ejecución, no se acumula el
  retraso (no hay catch-up).
- **Sin exclusión mutua**: las tareas no se pueden solapar entre sí, porque corren
  secuencialmente en la misma hebra.
- **`count()`** es sólo un contador de tareas registradas = **2** hoy; es lo que
  devuelve el campo `tasks` de `/api/v1/diagnostics`.

## 5. Watchdog

Hay dos niveles, y conviene no confundirlos:

### 5.1 TWDT (Task Watchdog Timer) del ESP32

`src/core/Watchdog.cpp` (24 líneas):

```cpp
bool Watchdog::begin(uint32_t timeoutSeconds) {
  const esp_err_t err = esp_task_wdt_init(timeoutSeconds, true);   // panic = true
  if (err != ESP_OK && err != ESP_ERR_INVALID_STATE) return false;
  started_ = (esp_task_wdt_add(NULL) == ESP_OK);                    // NULL = tarea actual
  return started_;
}
void Watchdog::feed() { if (started_) esp_task_wdt_reset(); }
```

| Parámetro | Valor |
|-----------|-------|
| Timeout | **10 s** (`watchdog_.begin(10)` en `SemaCore.cpp:203`) |
| Acción al vencer | `esp_task_wdt_init(10, true)` → **panic** y reinicio del SoC |
| Tarea vigilada | La tarea actual en el momento de `begin()`, es decir `loopTask` (`esp_task_wdt_add(NULL)`) |
| Alimentación | `watchdog_.feed()` al **principio** de cada `loop()` (`SemaCore.cpp:367`) |
| Tolerancia a doble init | Si el core Arduino ya inicializó el TWDT, acepta `ESP_ERR_INVALID_STATE` y conserva esa configuración |

Consecuencia práctica: **cualquier bloqueo de más de 10 s en la tarea del loop
reinicia el ESP32**. Como todas las operaciones (HTTP, SD, MQTT, OTA) ocurren en esa
tarea, el timeout de 10 s es el presupuesto máximo de cualquier operación
individual.

### 5.2 Watchdog jerárquico lógico (`HealthMonitor`)

`src/core/HealthMonitor.cpp` implementa un watchdog por software, independiente del
TWDT, que **no reinicia nada**: solo degrada el estado de salud reportado.

| Tarea registrada | Timeout | Se late en | Efecto al vencer |
|------------------|---------|------------|------------------|
| `core.heartbeat` | 30 000 ms | Cada 5 s, desde el `Scheduler` | `allTasksHealthy()` → `false` → `status()` = `DEGRADED` |
| `sensors.read` | 60 000 ms | Cada 10 s, desde el `Scheduler` | Ídem |

Detalles verificados:

- `kMaxTasks = 10` (`include/core/HealthMonitor.hpp`): se pueden registrar hasta 10
  tareas vigiladas; más allá, `registerTask()` retorna sin registrar.
- `registerTask()` fija `lastMs = millis()` al registrar, no en 0.
- `taskHealthy(nombre)` devuelve `true` si la tarea **no está registrada** (no se
  evalúa lo desconocido).
- El heartbeat global del monitor se considera sano si latió en los últimos
  **30 000 ms** (`heartbeatHealthy()`, constante fija, no configurable).

### 5.3 Otros watchdogs internos

`CanManager` usa `twai_transmit(&msg, pdMS_TO_TICKS(1000))`: si no logra transmitir
en 1 s, devuelve `false`. **Ese segundo se consume en la tarea del loop** cuando se
invoca la transmisión desde la web (`POST /api/v1/can`). `twai_receive` usa
`pdMS_TO_TICKS(0)`, es decir no bloquea.

## 6. Primitivas de sincronización

**No hay ninguna.** Inventario verificado en todo el árbol:

| Primitiva | Ocurrencias en el código de SEMA |
|-----------|----------------------------------|
| `SemaphoreHandle_t` | 0 |
| `xSemaphoreCreate*` | 0 |
| `xQueueCreate` | 0 |
| `portMUX_TYPE` / `portENTER_CRITICAL` | 0 |
| `std::mutex` / `std::atomic` | 0 |
| `volatile` para comunicación entre tareas | No se usa con ese fin |

Las estructuras compartidas (`std::vector<Measurement> measurements_`,
`std::deque<Event> events_`, `std::vector<Sensor*> sensors_`) se acceden desde
distintos puntos del código, pero **siempre en la misma tarea**. La seguridad es
posicional, no por construcción: si en el futuro se agrega una tarea FreeRTOS que
toque `SensorManager` o `EventLog`, hará falta agregar sincronización, porque estos
contenedores no son reentrantes.

## 7. Qué puede bloquear el loop

Todas las siguientes operaciones corren en `loopTask` y consumen el presupuesto del
TWDT de 10 s:

| Operación | Bloqueo máximo | Dónde | Se dispara desde |
|-----------|----------------|-------|------------------|
| `HttpPublisher::publish()` | **2 000 ms** (`http.setTimeout(2000)`) | `HttpPublisher.cpp:35` | Ciclo `sensors.read` |
| `MqttPublisher::publish()` | Reconexión: `MQTT_SOCKET_TIMEOUT` = **15 s** por defecto de PubSubClient | `MqttPublisher.cpp:34-44` | Ciclo `sensors.read` |
| `HttpServer::onUpdateCheck()` | `http.setTimeout(8000)` + `delay(100)` | `HttpServer.cpp:1833`, `1857` | `GET /api/v1/update/check` |
| `HttpServer::onWifiScan()` | `delay(300)` + escaneo sincrónico de `WiFi.scanNetworks()` | `HttpServer.cpp:1526` | `GET /api/v1/wifi/scan` |
| `HttpServer::onConfigNetwork()` | `delay(300)` + `ESP.restart()` | `HttpServer.cpp:1526-1527` | `POST /api/v1/config/network` |
| `HttpServer::onRestart()` | `delay(200)` + `ESP.restart()` | `HttpServer.cpp:1708` | `POST /api/v1/restart` |
| `HttpServer::onOta()` / `onOtaUpload()` | `delay(100)` ×2 + escritura de flash del firmware | `HttpServer.cpp:1984` | `POST /api/v1/ota` |
| `HistoryStore::readRecent()` | Lee **el archivo entero** línea por línea desde SD | `HistoryStore.cpp:176-189` | `GET /api/v1/history` (hasta `limit=3000`) |
| `HistoryStore::aggregate()` | Lee el archivo entero, reescribe el raw y agrega al `.agg` | `HistoryStore.cpp:194-273` | Cada 3 600 s en `loop()` |
| `HistoryStore::prune()` | Lee y reescribe el archivo entero | `HistoryStore.cpp:131-163` | Definida, ⚠️ sin llamadas verificadas |
| `EventLog::rotate()` | Lee hasta 200 líneas y reescribe el archivo | `EventLog.cpp:44-72` | Al alcanzar `maxEntries_ * 2` líneas |
| `HistoryStore::append()` | 1 open + write + close **por medición** | `HistoryStore.cpp:97-129` | Ciclo `sensors.read` |
| `ModbusManager::read()` | Lectura Modbus RTU con timeout propio de la librería | `ModbusManager.cpp:50-63` | `GET /api/v1/modbus` |

Riesgo concreto y verificable: `MQTT_SOCKET_TIMEOUT` por defecto es **15 s**
(`.pio/libdeps/<env>/PubSubClient/src/PubSubClient.h`, líneas 34-36) y `MqttPublisher`
no lo ajusta con `setSocketTimeout()`. Si el broker no responde, la reconexión
puede superar el timeout de 10 s del TWDT y **reiniciar el ESP32**. Es el único
bloqueo del inventario que puede por sí solo exceder el presupuesto del watchdog.

### Lo que NO puede bloquear

- `ZigbeeManager::loop()`: usa `while (Serial1.available() > 0)` sobre un UART con
  buffer; drena lo disponible y sale.
- `EthernetManager::loop()`: cuerpo vacío por diseño (lwIP gestiona DHCP).
- `CanManager::receive()`: `twai_receive(..., pdMS_TO_TICKS(0))`, no bloqueante.
- `ModbusManager::apply()`, `CanManager::apply()`, `LoraManager::apply()` y
  `EthernetManager::apply()`: hacen `Serial2.begin()` / `twai_driver_install()` /
  `begin()` de la radio / `ETH.begin()`. Son sincrónicos y pueden tardar, pero se
  ejecutan en `setup()` o al aplicar configuración desde la web, no en el ciclo
  periódico.

## 8. Modelo mental correcto

```text
        ┌──────────────────────── loopTask (única hebra de la aplicación) ─────────┐
        │                                                                          │
        │  watchdog.feed()                                                         │
        │  wifi.loop() ─ http.loop() ─ zigbee.loop() ─ ethernet.loop()             │
        │  [cada 3600 s] history.aggregate()                                       │
        │  scheduler.run()                                                         │
        │      ├── cada 5 s : health.tick + heartbeat                              │
        │      └── cada 10 s: readAll → append ×N → publishAll ×N → evaluate       │
        │                                                                          │
        │  Todo el estado del firmware vive acá. No hay locking porque           │
        │  no hay a quién bloquear.                                                │
        └──────────────────────────────────────────────────────────────────────────┘
                    ▲                                        ▲
                    │ HTTP :80 / WebSocket :81               │ SD (SPI), I²C, 1-Wire, UART, TWAI, SPI
                    │ (atendidos en la misma tarea)          │ (periféricos, no hebras)
```

## 9. Lo que no está implementado

| Elemento del diseño | Estado |
|---------------------|--------|
| Tareas FreeRTOS por subsistema (`SensorTask`, `MeasurementTask`, `StorageTask`, …) | ❌ No implementado |
| Afinidad de núcleo (`AUTO` o pinning) | ❌ Sin uso; `Task.hpp` documenta `AUTO` como default futuro |
| Prioridades P1…P7 del diseño | ❌ No implementado: no hay tareas que priorizar |
| Cola entre adquisición y publicación | ❌ No implementado: publicación síncrona |
| Mutex por recurso compartido (SD, bus I²C) | ❌ No implementado ni necesario hoy |
| `Scheduler` con intervalos configurables desde la web | ⚠️ Los intervalos están fijos en `SemaCore.cpp` (5 s y 10 s); no hay clave de configuración |
| `HealthMonitor` con timeouts configurables | ⚠️ Fijos en código (30 s y 60 s) |

---

## Ver también

- [Arquitectura](Arquitectura.md) · [Diagramas](Diagramas.md)
- [Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md) · [Rendimiento-y-memoria](Rendimiento-y-memoria.md)
- [Diagnostico-y-salud](Diagnostico-y-salud.md) · [Referencia-API-interna](Referencia-API-interna.md)
- [Almacenamiento-e-historico](Almacenamiento-e-historico.md) · [MQTT-y-WebSocket](MQTT-y-WebSocket.md)
