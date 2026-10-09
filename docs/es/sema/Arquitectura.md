---
tags:
  - sema
  - arquitectura
---

# Arquitectura

> **Tipo:** Concepto | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Este documento describe la arquitectura **tal como está implementada** en el firmware
v1.103.0 (rama `main`). Todo lo que figura acá se verificó contra el código en
`src/` e `include/`; lo que es diseño aspiracional y todavía no existe se marca
explícitamente con ⚠️ o ❌.

## 1. Qué es SEMA (y qué no)

SEMA es un firmware para ESP32 escrito en C++ con el framework **Arduino**
(`platformio.ini`, `framework = arduino`). Usa APIs del ESP-IDF solo donde la capa
Arduino no alcanza: TWAI (`driver/twai.h`), `esp_eth` (W5500), `esp_task_wdt`
(watchdog), `Preferences` (NVS) y `esp_reset_reason()`.

La arquitectura real es **un único binario monolítico, de una sola tarea de
aplicación**: los servicios son miembros de un objeto central (`sema::SemaCore`) y
los orquesta un planificador cooperativo por sondeo. No hay hilos de aplicación, ni
colas entre servicios, ni comunicación inter-proceso. Ver
[Tareas-y-concurrencia](Tareas-y-concurrencia.md).

## 2. Principios de diseño

Los cinco principios que ordenan el diseño y que el código respeta:

```text
SENSOR  ≠ FUNCIÓN    Un modelo mide varias magnitudes: el driver devuelve N mediciones.
GPIO    ≠ SENSOR     Un pin es un recurso asignable desde la configuración.
BUS     ≠ SENSOR     I²C/1-Wire/SPI son transporte, no identidad.
MODELO  ≠ MAGNITUD   BME280/SHT40/SHT31/AHT20/BMP280 son intercambiables porque
                     todos producen Measurement canónicas.
HARDWARE ≠ CONFIG    El hardware aporta capacidades (Capability); la config decide el uso.
```

Consecuencias que se ven en el código:

- **Magnitud, no modelo.** `Sensor::measure()` (`include/core/sensors/Sensor.hpp`)
  devuelve `Measurement` (`include/core/Measurement.hpp`) con `sensorId`,
  `channelId`, `measurement`, `value`, `unit` y `quality`; nadie aguas abajo
  pregunta por el modelo físico.
- **El núcleo no depende de los módulos opcionales.** Ethernet, LoRa, Modbus, CAN y
  Zigbee están envueltos en `#if SEMA_USE_*` (`include/hw/HwProfile.hpp`). Un fallo
  de un módulo opcional no detiene la adquisición: `HealthMonitor::status()` degrada
  el estado pero el `Scheduler` sigue corriendo.
- **Las capacidades se preguntan, no se deducen del modelo de placa.**
  `CapabilityManager` (`include/core/Capability.hpp`) expone
  `WiFi, Bluetooth, Ethernet, Adc, Dac, Pcnt, LedcPwm, I2c, Spi, Uart, Can, Psram,
  RtcGpio, DeepSleep, DualCore, Ieee802154` y el código consulta
  `caps.has(Capability::X)`.

## 3. Capas reales

La separación efectiva en el código es esta (de abajo hacia arriba):

```text
┌─ 4. Aplicación ─── src/main.cpp (18 líneas) → SemaCore::setup() / loop()
├─ 3. Servicios del núcleo (miembros de SemaCore) ─────────────────────────
│   ConfigManager · NvsStore · HistoryStore · EventLog · EventBus · Scheduler
│   Watchdog · HealthMonitor · CapabilityManager · SensorManager
│   DerivedEngine/DerivedCalculator · RuleEngine · GpioManager
│   ShiftRegisterManager · PublisherManager(HttpPublisher, MqttPublisher)
│   PowerManager · HttpServer
├─ 2. Módulos opcionales (compilados condicionalmente) ───────────────────
│   EthernetManager · LoraManager · ModbusManager · CanManager
│   ZigbeeManager · WiFiManager
└─ 1. HAL / perfil de hardware (compile-time) ────────────────────────────
    include/hw/HwProfile.hpp (SEMA_USE_*, pines de bus, board) ·
    include/core/BoardProfile.hpp (SEMA_PIN_* del catálogo fijo) + APIs del core
    Arduino-ESP32 (Wire, SPI, Serial1/2, Preferences, LittleFS, SD, WiFi,
    ETH/esp_eth, TWAI, esp_task_wdt)
```

Puntos clave de esta separación:

- **No hay capa de abstracción propia sobre el hardware.** SEMA no envuelve
  `Wire`, `SPI` ni `Serial` en interfaces propias: los drivers de sensor llaman
  directamente a `Wire.begin(...)` y a las librerías de Adafruit (ver
  `src/core/sensors/Sht40Sensor.cpp` o `src/core/sensors/Bme280Sensor.cpp`). La
  única abstracción propia de plataforma es `CapabilityManager`.
- **La capa 2 no está registrada como módulos.** `EthernetManager`, `LoraManager`,
  `ModbusManager`, `CanManager` y `ZigbeeManager` son **miembros por composición**
  de `SemaCore` (`include/core/SemaCore.hpp`, líneas 136-150), no implementaciones
  de `sema::Module`. Ver [Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md).
- **`sema::Task` (Runtime Manager) existe pero no se usa.** La clase está en
  `include/core/runtime/Task.hpp` / `src/core/runtime/Task.cpp` y envuelve
  `xTaskCreate`, pero no hay ninguna instanciación en el firmware (verificado por
  búsqueda de `new Task`/`Task(` en todo el árbol). ❌ No hay Runtime Manager activo.

## 4. Estructura de carpetas real

Verificada con `Get-ChildItem -Recurse -Directory` sobre el repo. La estructura
**no coincide** con la que describían versiones anteriores de esta página.

```text
SEMA/
├── src/
│   ├── main.cpp                  ← 18 líneas: setup() y loop() delegan en SemaCore
│   └── core/
│       ├── SemaCore.cpp          ← orquestador (392 líneas)
│       ├── (raíz)                ← CapabilityManager · ConfigManager · EventBus ·
│       │                           GpioManager · HealthMonitor · PowerManager ·
│       │                           Scheduler · ShiftRegisterManager · Watchdog ·
│       │                           ModbusManager · CanManager · LoraManager ·
│       │                           ZigbeeManager · EthernetManager · Calibration
│       ├── alarms/               ← RuleEngine.cpp
│       ├── derived/              ← DerivedCalculator.cpp · DerivedEngine.cpp
│       ├── events/               ← EventLog.cpp
│       ├── network/              ← WiFiManager.cpp
│       ├── publishers/           ← HttpPublisher.cpp · MqttPublisher.cpp · PublisherManager.cpp
│       ├── runtime/              ← Task.cpp (abstracción sin uso)
│       ├── sensors/              ← SensorFactory.cpp · SensorManager.cpp · I2cScanner.cpp
│       │                           + 17 drivers *Sensor.cpp
│       ├── storage/              ← HistoryStore.cpp · NvsStore.cpp
│       └── web/                  ← HttpServer.cpp (2637 líneas) + assets gzip
├── include/
│   ├── hw/HwProfile.hpp          ← perfil de hardware (compile-time)
│   └── core/                     ← cabeceras espejo de src/core/ + Module.hpp ·
│                                    ModuleRegistry.hpp · Capability.hpp · Version.hpp ·
│                                    BoardProfile.hpp · Measurement.hpp · Time.hpp
├── lib/  · test/                 ← solo un README placeholder cada uno
├── docs/                         ← GUIA.md · IMPLEMENTACION.md · PINES-POR-BOARD.md · …
├── platformio.ini · partitions_4mb.csv · partitions_8mb.csv · partitions_16mb.csv
└── CHANGELOG.md · README.md · DESIGN-SYSTEM.md · SECURITY.md · firmware_manifest.json
```

**No existen** (pese a figurar en versiones previas de esta página) las carpetas
`src/config/`, `src/hardware/`, `src/buses/`, `src/actuators/`,
`src/communications/`, `src/energy/`, `src/diagnostics/`, `src/api/`, `src/ota/`
ni `src/modules/`. Las cabeceras viven en `include/core/` (no todo en
`include/core/` plano: hay subcarpetas `alarms/`, `derived/`, `events/`,
`network/`, `publishers/`, `runtime/`, `sensors/`, `storage/`, `web/`).

## 5. Secuencia de arranque real

`void setup()` en `src/main.cpp` llama a `sema::SemaCore::instance().setup()`.
`SemaCore::setup()` (`src/core/SemaCore.cpp`, líneas 46-225) ejecuta, **en este
orden exacto**:

| # | Paso | Código |
|---|------|--------|
| 1 | `Serial.begin(115200)` + `delay(200)` | SemaCore.cpp:47 |
| 2 | Almacenamiento base: `store_.begin("sema")` (NVS) → `history_.begin()` → `eventLog_.begin()` (monta LittleFS) | SemaCore.cpp:50-52 |
| 3 | Contador de reinicios persistido (`boots`), leído e incrementado | SemaCore.cpp:55-61 |
| 4 | Declarar 13 capacidades del ESP32 clásico | SemaCore.cpp:65-78 |
| 5 | `config_.load()` → `history_.setRetentionSeconds(retentionDays × 86400)` → `history_.enableSd(sdCsPin)` si `storage.sdEnabled` | SemaCore.cpp:80-89 |
| 6 | Banner por serial (versión, HW, schema, protocolo, estación, wake reason, capacidades) | SemaCore.cpp:91-109 |
| 7 | `Wire.begin(SDA, SCL)` + `I2cScanner::scan()` → `detectedDevices_` | SemaCore.cpp:112-117 |
| 8 | `applySensors()` → `derived_.configure()` → `applyCalibrations()` → `gpio_.apply()` y `shift_.apply()` (si `SEMA_USE_SHIFT`) | SemaCore.cpp:119-126 |
| 9 | `SPI.begin(SCK, MISO, MOSI)` una sola vez; luego `modbus_.apply()` → `can_.apply()` → `lora_.apply()` → `zigbee_.apply()` → `ethernet_.apply()` | SemaCore.cpp:129-147 |
| 10 | `wifi_.begin(...)` (STA o AP + mDNS) → `configTzTime(...)` con la TZ POSIX mapeada desde IANA | SemaCore.cpp:149-162 |
| 11 | `http_.begin(*this)`: registra rutas y hace `server_.begin()` + `ws_.begin()` (puerto 81) | SemaCore.cpp:164 |
| 12 | Registrar publicadores (webhook, MQTT) → `applyPublishers()` → `applyRules()` | SemaCore.cpp:167-171 |
| 13 | Suscribir el log de alarma a `EventType::Alarm` (solo imprime por serial) | SemaCore.cpp:174-176 |
| 14 | Registrar 2 tareas en `HealthMonitor` (30 s y 60 s) y 2 en el `Scheduler` (5 s y 10 s) | SemaCore.cpp:180-198 |
| 15 | `modules_.enableAll()` — **no hay módulos registrados** | SemaCore.cpp:200 |
| 16 | `watchdog_.begin(10)` → `health_.begin()` → publicar el evento `boot` (`EventType::System`) → reporte final por serial | SemaCore.cpp:203-224 |

Después, `loop()` (`src/core/SemaCore.cpp`, líneas 366-390) repite:

```text
watchdog_.feed()          → alimenta el TWDT
modules_.loopAll()        → itera (lista vacía hoy)
wifi_.loop()              → backoff de reconexión 2 s → 4 s → … → 60 s
http_.loop()              → server_.handleClient() + ws_.loop()
zigbee_.loop()            → parseo de tramas ZNP (si SEMA_USE_ZIGBEE)
ethernet_.loop()          → vacío (lwIP gestiona DHCP solo)
cada 3600 s: history_.aggregate(3600, now - retentionDays*86400)
scheduler_.run()          → heartbeats y lectura de sensores
```

Detalle importante: el `Scheduler` corre **al final** de `loop()`, y el web server
**antes**. Como todo ocurre en la misma tarea, una petición HTTP lenta retrasa la
lectura de sensores, no al revés.

## 6. Frontera HAL / BoardProfile

Hay **dos** archivos que definen el hardware, y cumplen roles distintos:

| Archivo | Cuándo actúa | Qué define |
|---------|--------------|------------|
| `include/hw/HwProfile.hpp` | Compile-time, por `build_flags` de `platformio.ini` | Board (`BOARD_ESP32_WROOM`, `BOARD_ESP32_WROOM32U`, `BOARD_ESP32_S3`), features (`SEMA_USE_ETHERNET/LORA/MODBUS/CAN/ZIGBEE/SHIFT`), variante de transceiver RS485, pines del bus SPI y de los chip-select, `SEMA_PINS_FROM_FILE`, `SEMA_DEMO` |
| `include/core/BoardProfile.hpp` | Runtime | Pines del **catálogo fijo** de sensores (`SEMA_PIN_I2C_SDA=21`, `SEMA_PIN_I2C_SCL=22`, `SEMA_PIN_ONEWIRE=4`, `SEMA_PIN_BATTERY_ADC=34`) y el interruptor `SEMA_FIXED_HARDWARE` (0 por defecto) |

`HwProfile.hpp` aborta la compilación con `#error` si no se definió ninguna board
(líneas 28-30) y deriva de ella tres datos de identidad que se exponen por API:
`SEMA_BOARD_ID` (`"esp32-wroom-4mb"`, `"esp32-wroom32u-16mb"`, `"esp32-s3-8mb"`),
`SEMA_FLASH_MB` (4/16/8) y `SEMA_NATIVE_ETH` (1/1/0). La convivencia entre los dos
perfiles se resuelve en `SemaCore::applySensors()` (líneas 280-313):

```text
if (SEMA_FIXED_HARDWARE == 1 || config.sensors está vacío):
      catálogo fijo de BoardProfile.hpp (BME280 "EXT", SHT40 "INT", DS18B20 "SOIL",
      BH1750 "LUX", AHT20 "AUX", AdcSensor "BATT" con escala 3.3*11/4095 y offset 0)
else:
      config.sensors[] → SensorFactory::create(spec) por cada entrada habilitada
```

Con `SEMA_FIXED_HARDWARE = 1` la configuración `sensors[]` de la web **se ignora
por completo**; con el valor por defecto (0) los `SEMA_PIN_*` de `BoardProfile.hpp`
quedan solo como fallback. Los dos interruptores son **independientes y
complementarios**:

| Interruptor | Archivo | Qué fija | Qué pisa |
|-------------|---------|----------|----------|
| `SEMA_PINS_FROM_FILE=1` | `HwProfile.hpp` | Pines de los **buses**: CAN (TX/RX), Modbus (RX/TX/DE-RE), Zigbee (RX/TX), LoRa (CS/RST/DIO1/BUSY) y Ethernet (MDC/MDIO/PHY/power/CS/RST/IRQ/SCK/MISO/MOSI) | Al final de `ConfigManager::parseInto()` se llama `applyHwProfile(c)` (`ConfigManager.cpp:459`), que sobrescribe esos pines con los de `HwProfile.hpp` e **ignora lo que venga de la web** |
| `SEMA_FIXED_HARDWARE=1` | `BoardProfile.hpp` | Pines del **catálogo de sensores** (SDA, SCL, 1-Wire, ADC de batería) y qué catálogo se usa | `applySensors()` ignora `config.sensors[]` por completo |

Con ambos en 0 (default de `esp32doit-devkit-v1` y `esp32-s3-devkitc-1`) todos los
pines son configurables desde la web; el entorno `esp32-wroom-32u` usa
`SEMA_PINS_FROM_FILE=1` para congelar los pines de bus de la PCB futura.
⚠️ `SEMA_PINS_FROM_FILE` **no** fija los pines I²C de los sensores (`i2c_sda` /
`i2c_scl` siguen tomándose del JSON), y `SEMA_FIXED_HARDWARE` no está definido en
`platformio.ini`, así que su valor por defecto (0) rige en todos los entornos.

## 7. Grafo de dependencias entre servicios

Dependencias verificadas por construcción (`include/core/SemaCore.hpp`, líneas
38-44 y 122-158):

```text
SemaCore
 ├── NvsStore ──────────► ConfigManager (inyectado por constructor)
 ├── EventBus ◄────────── RuleEngine (recibe el bus), EventLog (se suscribe a
 │                       los 9 tipos de EventType)
 ├── ConfigManager ─────► casi todos (HistoryStore, sensores, buses, publicadores,
 │                       reglas, calibraciones, GPIO, scheduler…)
 ├── SensorManager ─────► drivers Sensor* (registrados por SensorFactory)
 │                       y DerivedEngine (estático, se llama desde readAll())
 ├── HistoryStore ──────► SD (SPI) — NUNCA LittleFS para el histórico
 ├── EventLog ──────────► LittleFS (JSONL en /events.jsonl)
 ├── HttpServer ────────► SemaCore* (referencia al core; usa todos los servicios)
 ├── PublisherManager ──► HttpPublisher (HTTPClient) + MqttPublisher (PubSubClient)
 ├── HealthMonitor ◄──── Scheduler (heartbeat) + HistoryStore/SensorManager (stats)
 └── Watchdog ──────────► esp_task_wdt
```

Reglas de dependencia que el código respeta:

- `HttpServer` es el único servicio que tiene un puntero al `SemaCore` completo;
  el resto recibe solo lo que necesita.
- `SensorManager` **no** conoce `HistoryStore`, `HttpServer` ni los publicadores:
  es `SemaCore` quien, en la tarea del scheduler, encadena
  `sensors_.readAll()` → `history_.append()` → `publishers_.publishAll()` →
  `rules_.evaluate()`.
- `RuleEngine` solo habla por `EventBus`; nunca llama a un publicador.

## 8. Flujo de datos: MEDIR → VALIDAR → PROCESAR → ALMACENAR → PUBLICAR

La cadena real está en el callback `sensors.read` del `Scheduler`
(`src/core/SemaCore.cpp`, líneas 188-198) más `SensorManager::readAll()`
(`src/core/sensors/SensorManager.cpp`, líneas 21-42). Cada 10 000 ms:

```text
1. MEDIR      sensors_.readAll()
              └── por cada Sensor::healthy(): sensor->measure(buffer[4], 4)
                  → hasta 4 Measurement por sensor en un buffer de pila

2. VALIDAR    por cada medición: clave = sensorId + ":" + channelId
   CALIBRAR     ├── si hay Calibration para esa clave: applyCalibration(m, cal)
                └── el driver ya pobló Measurement::quality con uno de los 8 flags
                    (Valid, Invalid, Stale, Timeout, OutOfRange, CalibrationError,
                     CommunicationError, SensorDisconnected)
              ⚠️ Nadie aguas abajo descarta por quality: las mediciones no válidas se
                 almacenan y se publican igual. El filtro es informativo.

3. PROCESAR   DerivedEngine::compute(measurements_)  [estático, dentro de readAll]
              └── agrega 4 mediciones con sensorId "DERIVED": dew_point (degC) ·
                  heat_index (degC) · vapor_pressure (hPa) · absolute_humidity (g/m3)
              └── requiere un sensor que aporte "temperature" y "humidity" con el mismo id
              Aparte, DerivedCalculator (no estático) calcula QNH, VPD, AQI, altitud, tasa
              de lluvia, sensación térmica y dirección de viento, y convierte unidades
              métrico↔imperial. Se invoca desde HttpServer al servir /api/v1/sensors y
              /api/v1/dashboard; NO forma parte de la cadena de almacenamiento.

4. ALMACENAR  por cada medición: history_.append(m) → SD, JSONL, máx. 10 000 entradas
              ⚠️ Un open/write/close por medición; sin SD habilitada devuelve false y no
                 guarda nada (por diseño, para no gastar flash interna)

5. PUBLICAR   http_.broadcastMeasurements(measurements) → WebSocket puerto 81 (si hay clientes)
              por cada medición: publishers_.publishAll(m)
                ├── HttpPublisher: POST JSON con timeout de 2000 ms (solo si hay URL)
                └── MqttPublisher: publish MQTT, reconecta si hace falta (solo si hay host)

6. EVALUAR    rules_.evaluate(measurements) → EventBus.publish(EventType::Alarm)
   ALARMAS    └── EventLog lo persiste en LittleFS y el suscriptor de SemaCore lo imprime
```

No hay etapas de SINCRONIZAR ni DORMIR dentro del ciclo: el `PowerManager` existe
(`sleep()`, `enableRainWakeup()`, `wakeReason()`), pero solo se usa para leer el
wake reason al arrancar y habilitar el despertar por lluvia. ❌ El ciclo de
adquisición no duerme el SoC entre muestras.

## 9. Qué no está implementado

| Elemento | Estado | Evidencia |
|----------|--------|-----------|
| Runtime Manager con tareas propias y afinidad | ❌ No implementado | `include/core/runtime/Task.hpp` no se instancia en ningún lado |
| Resource Manager (asignación y validación de recursos) | ❌ No existe | No hay archivo ni clase con ese rol en `src/` ni `include/` |
| Registro efectivo de módulos en `ModuleRegistry` | ❌ No implementado | `registerModule()` nunca se llama; `modules_.count() == 0` |
| Ciclo de vida `Available → … → Running` aplicado a servicios reales | ❌ No implementado | Los managers se aplican con `apply(cfg)`, no con `install/configure/enable/start` |
| Histórico en LittleFS | ❌ Descartado | `HistoryStore` escribe solo en SD; sin SD no hay histórico |
| Tareas FreeRTOS de aplicación (SensorTask, StorageTask, …) | ❌ No existen | Ver [Tareas-y-concurrencia](Tareas-y-concurrencia.md) |
| WebSocket en `/ws` | ⚠️ Parcial | El servidor WS escucha en el puerto **81**, no en la ruta `/ws`. `IMPLEMENTACION.md` §13.5 lo documenta como `/ws` |
| Deep sleep con wake por timer/lluvia | ⚠️ Parcial | API presente en `PowerManager`; no hay llamada a `sleep()` en el ciclo normal |
| Sensores máximos / límites de catálogo | ⚠️ Sin límite | `ConfigManager::validate()` no limita la cantidad de `sensors[]` |

## 10. Niveles de despliegue

```text
Nivel 1 — Estación autónoma: web local en el propio ESP32 (implementado)
Nivel 2 — Estación + servicios externos: MQTT y webhook HTTP (implementado)
Nivel 3 — Servidor Central: SEMA_PROTOCOL_VERSION=1 y `serverKey` en la config,
          pero no hay cliente del servidor central en el código ❌
```

---

## Ver también

- [Diagramas](Diagramas.md) · [Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md)
- [Tareas-y-concurrencia](Tareas-y-concurrencia.md) · [Rendimiento-y-memoria](Rendimiento-y-memoria.md)
- [Diagnostico-y-salud](Diagnostico-y-salud.md) · [Referencia-de-codigo](Referencia-de-codigo.md)
- [Hardware-y-Conexiones](Hardware-y-Conexiones.md) · [Guia-de-pines](Guia-de-pines.md)
