---
tags:
  - sema
  - referencia
  - codigo
---

# Referencia de código

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Mapa del código fuente real de SEMA (PlatformIO + framework Arduino, ESP32/ESP32-S3).
Todos los conteos de líneas de esta página se midieron sobre el repo local con
`Get-ChildItem -Recurse` + conteo de líneas totales (incluyendo líneas vacías).

Cifras globales verificadas: **112 archivos** entre `include/` y `src/`,
**10 989 líneas** en total (sin contar `docs/`, `lib/` ni `.pio/`).

## 1. Raíz del repo

| Ruta | Tipo | Qué es |
|------|------|--------|
| `platformio.ini` | 102 líneas | Entornos, `lib_deps` y `build_flags` de cada board |
| `partitions_4mb.csv` | 7 líneas | Tabla de particiones 4 MB (OTA A/B + spiffs) |
| `partitions_8mb.csv` | 7 líneas | Tabla de particiones 8 MB (ESP32-S3) |
| `partitions_16mb.csv` | 7 líneas | Tabla de particiones 16 MB (PCB WROOM-32U) |
| `firmware_manifest.json` | 27 líneas | Versión, `hw_version`, schemas y SHA-256 de cada binario |
| `CHANGELOG.md` | 843 líneas | Historial Keep a Changelog (última entrada `[1.103.0] - 2026-10-08`) |
| `README.md` | especificación | Spec completa del proyecto (fuente original de la arquitectura) |
| `DESIGN-SYSTEM.md` | documento | Referencia de diseño (§102, §105, §224 citados en el código) |
| `SECURITY.md` | documento | Política de seguridad |
| `gen_gridstack.py` | script | Genera `src/core/web/GridstackAssets.h` (assets gzip en PROGMEM) |
| `include/` | 2 979 líneas · 69 archivos | Cabeceras (interfaces) del Core y del perfil de hardware |
| `src/` | 8 010 líneas · 43 archivos | Implementaciones |
| `lib/` | solo `README` | Sin librerías privadas: las dependencias son de `lib_deps` |
| `test/` | solo `README` | Sin tests (ver [Pruebas y validación](Pruebas-y-validacion.md)) |
| `docs/` | 12 documentos | Docs de trabajo del repo de código |
| `.release/` | binarios | Binarios por release (p. ej. `sema_1.103.0_esp32-wroom-4mb.bin`) |
| `.pio/`, `.vscode/` | tooling | Build y configuración de IDE (no versionar) |

## 2. Árbol real de `include/` y `src/`

```text
include/                                69 archivos · 2 979 líneas
├── core/                               23 archivos · 1 292 líneas
│   ├── SemaCore.hpp                    161   orquestador (singleton)
│   ├── ConfigManager.hpp               269   config schema=1 + structs de spec
│   ├── BoardProfile.hpp                 39   pines del catálogo fijo (PCB)
│   ├── Capability.hpp / CapabilityManager.hpp   57 / 28
│   ├── EventBus.hpp                    124   pub/sub tipado + Event
│   ├── Measurement.hpp                  68   modelo canónico + Quality
│   ├── Module.hpp / ModuleRegistry.hpp  60 / 30
│   ├── Scheduler.hpp                    33   tareas periódicas
│   ├── Watchdog.hpp                     22   TWDT
│   ├── HealthMonitor.hpp                51   heartbeat + watchdog por tarea
│   ├── PowerManager.hpp                 46   perfiles + deep sleep
│   ├── GpioManager.hpp                  33   GPIO nativo + MCP23017
│   ├── ShiftRegisterManager.hpp         28   74HC595 / 74HC165
│   ├── Calibration.hpp                  26   gain/offset + rango
│   ├── Time.hpp                         37   nowEpoch() / offset de zona
│   ├── Version.hpp                      18   FW / HW / schema / protocol
│   ├── EthernetManager.hpp              30
│   ├── ModbusManager.hpp                30
│   ├── CanManager.hpp                   28
│   ├── LoraManager.hpp                  28
│   └── ZigbeeManager.hpp                46
├── core/alarms/                         2 archivos · 70 líneas
├── core/derived/                        2 archivos · 68 líneas
├── core/events/                         1 archivo · 33 líneas
├── core/network/                        1 archivo · 36 líneas
├── core/publishers/                     4 archivos · 114 líneas
├── core/runtime/                        1 archivo · 37 líneas
├── core/sensors/                       21 archivos · 735 líneas
│   ├── Sensor.hpp                       29   interfaz de driver
│   ├── SensorManager.hpp                49   registro + lectura + calibración
│   ├── SensorFactory.hpp                18   model → driver
│   ├── I2cScanner.hpp                   26   escaneo I²C + DetectedDevice
│   └── <17 drivers>Sensor.hpp           25–42 c/u
├── core/storage/                        3 archivos · 113 líneas
├── core/web/HttpServer.hpp             105   handlers + sesión + OTA
├── hw/HwProfile.hpp                    176   board, features, SPI, CS, pines
└── README                               37   texto de plantilla de PlatformIO

src/                                    43 archivos · 8 010 líneas
├── main.cpp                             18   setup()/loop() → SemaCore
└── core/                              17 archivos · 1 750 líneas
    ├── SemaCore.cpp                    392
    ├── ConfigManager.cpp               483
    ├── HealthMonitor.cpp                92
    ├── GpioManager.cpp / ShiftRegisterManager.cpp   72 / 59
    ├── EthernetManager.cpp             147
    ├── CanManager.cpp / LoraManager.cpp  72 / 61
    ├── ModbusManager.cpp / ZigbeeManager.cpp  66 / 128
    ├── Watchdog.cpp / Scheduler.cpp / EventBus.cpp / CapabilityManager.cpp
    ├── ModuleRegistry.cpp / PowerManager.cpp / Calibration.cpp
    ├── core/alarms/RuleEngine.cpp       48
    ├── core/derived/DerivedCalculator.cpp 224 · DerivedEngine.cpp 108
    ├── core/events/EventLog.cpp        117
    ├── core/network/WiFiManager.cpp     88
    ├── core/publishers/                HttpPublisher 41 · MqttPublisher 62 · PublisherManager 17
    ├── core/runtime/Task.cpp            31
    ├── core/sensors/                  20 archivos · 1 220 líneas
    └── core/web/                      HttpServer.cpp 2 637 · GridstackAssets.h 1 463
```

## 3. Módulos: archivos, responsabilidad y tamaño

Todos los conteos son líneas totales del archivo (incluyen comentarios y blancos).

| Módulo | Archivos | Responsabilidad | Líneas |
|--------|----------|-----------------|-------:|
| Arranque | `src/main.cpp` | Enlaza `setup()`/`loop()` con `SemaCore::instance()` | 18 |
| Núcleo | `SemaCore.{hpp,cpp}` | Inicializa y conduce todos los servicios; accesores; `apply*()` | 553 |
| Configuración | `ConfigManager.{hpp,cpp}` | Structs de spec + config schema=1; validación, persistencia NVS, rollback, JSON | 752 |
| Capacidades | `Capability.{hpp}`, `CapabilityManager.{hpp,cpp}` | Máscara de 16 capacidades consultables | 109 |
| Módulos | `Module.hpp`, `ModuleRegistry.{hpp,cpp}` | Contrato y ciclo de vida de módulos | 134 |
| Eventos | `EventBus.{hpp,cpp}` | Pub/sub tipado por `EventType` | 143 |
| Eventos (persistencia) | `events/EventLog.{hpp,cpp}` | Suscribe a todos los tipos, buffer acotado + JSONL en LittleFS | 150 |
| Alarmas | `alarms/Rule.hpp`, `alarms/RuleEngine.{hpp,cpp}` | `RuleOp`, evaluación de reglas y publicación de `Alarm` | 118 |
| Derivadas (motor viejo) | `derived/DerivedEngine.{hpp,cpp}` | Punto de rocío, índice de calor, presión de vapor, humedad absoluta | 135 |
| Derivadas (motor activo) | `derived/DerivedCalculator.{hpp,cpp}` | VPD, QNH, AQI, altitud, lluvia, veleta + conversión de unidades | 265 |
| Calibración | `Calibration.{hpp,cpp}` | `applyCalibration()`: gain/offset + `OUT_OF_RANGE` | 42 |
| Sensores | `sensors/*.hpp` (21), `sensors/*.cpp` (20) | Interfaz `Sensor`, 17 drivers, manager, factoría, escáner I²C | 1 955 |
| Sensores (tiempo) | `Time.hpp` | `nowEpoch()`, `timezoneOffsetSeconds()` (header-only) | 37 |
| Almacenamiento | `storage/Storage.hpp`, `NvsStore.{hpp,cpp}` | `KeyValueStore` (interfaz) + backend NVS/Preferences | 107 |
| Histórico | `storage/HistoryStore.{hpp,cpp}` | JSONL en microSD, rotación, poda, agregación por niveles | 355 |
| Red WiFi | `network/WiFiManager.{hpp,cpp}` | STA/AP, IP estática, mDNS, backoff de reconexión | 124 |
| Ethernet | `EthernetManager.{hpp,cpp}` | LAN8720A (RMII) o W5500 (SPI, ESP-IDF) según board | 177 |
| Modbus | `ModbusManager.{hpp,cpp}` | Maestro Modbus RTU sobre RS485 (holding registers) | 96 |
| CAN | `CanManager.{hpp,cpp}` | TWAI: instalación, `send`/`receive` | 100 |
| LoRa | `LoraManager.{hpp,cpp}` | SX1262 (RadioLib): `begin`, `transmit`, `receive` | 89 |
| Zigbee | `ZigbeeManager.{hpp,cpp}` | ZNP por UART: tramas MT, `AF_DATA_REQUEST` / `AF_INCOMING_MSG` | 174 |
| Publicadores | `publishers/` (4 hpp + 3 cpp) | Interfaz `Publisher`, manager, webhook HTTP, MQTT | 234 |
| GPIO | `GpioManager.{hpp,cpp}` | Pines standalone nativos + MCP23017 por I²C | 105 |
| Shift registers | `ShiftRegisterManager.{hpp,cpp}` | 74HC595 (salida) / 74HC165 (entrada) bit-banged | 87 |
| Runtime | `runtime/Task.{hpp,cpp}` | Envoltura de `xTaskCreate`/`vTaskDelete` | 68 |
| Scheduler | `Scheduler.{hpp,cpp}` | Tareas periódicas por `millis()` | 55 |
| Watchdog | `Watchdog.{hpp,cpp}` | `esp_task_wdt` sobre el loop principal | 46 |
| Salud | `HealthMonitor.{hpp,cpp}` | Heartbeat, estado `HEALTHY/DEGRADED/ERROR`, watchdog por tarea | 143 |
| Energía | `PowerManager.{hpp,cpp}` | `EnergyProfile`, deep sleep, wake por GPIO de lluvia | 75 |
| Web/API | `web/HttpServer.{hpp,cpp}` | 4 páginas de config, dashboard, API `/api/v1`, `/ws`, OTA, login | 2 742 |
| Assets web | `web/GridstackAssets.h` | CSS/JS de Gridstack en PROGMEM (gzip), generado por `gen_gridstack.py` | 1 463 |
| Hardware | `hw/HwProfile.hpp` | Board, features, transceiver RS485, SPI, CS y pines fijos | 176 |
| Board (PCB) | `core/BoardProfile.hpp` | `SEMA_FIXED_HARDWARE` + pines del catálogo fijo | 39 |

## 4. Flujo de arranque (`SemaCore::setup()`, `src/core/SemaCore.cpp`)

| # | Paso | Línea | Detalle verificado |
|--:|------|------:|--------------------|
| 1 | Serie a 115200 | 47 | `Serial.begin(115200)` + `delay(200)` |
| 2 | NVS | 50 | `store_.begin("sema")` (namespace NVS `sema`) |
| 3 | Histórico y eventos | 51–52 | `history_.begin()` (`/history.jsonl`), `eventLog_.begin()` (`/events.jsonl`) |
| 4 | Contador de reinicios | 55–61 | Lee/incrementa/persiste `boots` en NVS → `restartCount_` |
| 5 | Capacidades | 65–78 | Declara 13 capacidades: WiFi, Bluetooth, ADC, DAC, PCNT, LEDC, I²C, SPI, UART, CAN, RTC GPIO, DeepSleep, DualCore (**no** declara `Ethernet`, `Psram` ni `Ieee802154`) |
| 6 | Configuración | 80 | `config_.load()` (NVS o defaults) |
| 7 | Retención del histórico | 81 | `retentionDays * 86400` segundos |
| 8 | microSD | 84–89 | Si `storage.sdEnabled`: `history_.enableSd(sdCsPin)` |
| 9 | Banner | 92–100 | Versión, HW, schema, protocol, estación, validez de config |
| 10 | Wake reason y wake por lluvia | 101–104 | Imprime `wakeReason()`; si `energy.rainPin != 0` habilita EXTO |
| 11 | Detección I²C | 112–117 | `Wire.begin(SDA, SCL)` + `I2cScanner::scan()` a `detectedDevices_` |
| 12 | Sensores | 119 | `applySensors()` (catálogo fijo o desde `config.sensors[]`) |
| 13 | Derivadas | 120 | `derived_.configure(config_.get().system)` |
| 14 | Calibración y GPIO | 122–123 | `applyCalibrations()`, `gpio_.apply(config.gpio)` |
| 15 | Shift registers | 124–126 | Solo bajo `#if SEMA_USE_SHIFT` (**apagado por defecto**, ver §7) |
| 16 | Bus SPI | 129–131 | `SPI.begin(SCK, MISO, MOSI)` bajo LoRa o Ethernet |
| 17 | Buses y módulos | 133–147 | Modbus, CAN, LoRa, Zigbee, Ethernet según `SEMA_USE_*` |
| 18 | WiFi | 149–157 | `wifi_.begin(...)` con modo/SSID/IP/mDNS |
| 19 | NTP | 161–162 | `configTzTime(posixTz(tz), ntpServer, "time.nist.gov")` |
| 20 | Servidor web | 164 | `http_.begin(*this)` (HTTP :80 + WebSocket :81) |
| 21 | Publicadores | 167–169 | Registra `webhook_` y `mqtt_` + `applyPublishers()` |
| 22 | Reglas | 171 | `applyRules()` |
| 23 | Suscriptor de alarmas | 174–176 | Lambda que imprime `[ALARM]` por serie |
| 24 | Watchdog por tarea | 180–181 | Registra `core.heartbeat` (30 s) y `sensors.read` (60 s) |
| 25 | Scheduler | 183–198 | `core.heartbeat` cada 5 000 ms; `sensors.read` cada 10 000 ms |
| 26 | Módulos | 200 | `modules_.enableAll()` (**no hay módulos registrados**, ver §7) |
| 27 | Watchdog TWDT | 203 | `watchdog_.begin(10)` (10 s) |
| 28 | Health monitor | 206 | `health_.begin()` |
| 29 | Evento de arranque | 209–215 | `Event{System, Info, source="core", correlationId="boot"}` |

Orden resumido:

```text
Serial → NVS → History/EventLog → boots → Capabilities → Config → SD
      → I2C scan → Sensores → Derivadas → Calibración → GPIO → SHIFT
      → SPI → Modbus/CAN/LoRa/Zigbee/Ethernet → WiFi → NTP → HTTP/WS
      → Publishers → Reglas → Alarmas → Watchdog por tarea → Scheduler
      → Módulos → TWDT → Health → Evento de arranque
```

### 4.1 Bucle principal (`SemaCore::loop()`, líneas 366–390)

```text
watchdog_.feed()            # 367  alimenta el TWDT
modules_.loopAll()          # 368  recorre módulos Enabled/Running (hoy vacío)
wifi_.loop()                # 369  reconexión con backoff
http_.loop()                # 370  handleClient() + ws_.loop()
zigbee_.loop()              # 372  solo si SEMA_USE_ZIGBEE
ethernet_.loop()            # 375  solo si SEMA_USE_ETHERNET (hoy no-op)
cada 3600 s: history_.aggregate(3600, now - retentionDays*86400)   # 378–388
scheduler_.run()            # 389  dispara las tareas vencidas
```

El ciclo funcional de datos (medir → derivar → almacenar → publicar → evaluar) **no
vive en `loop()`** sino en la tarea del scheduler `sensors.read`:

```text
sensors_.readAll()                       # SensorManager.cpp:21
health_.setSensorStats(online, total)
health_.taskHeartbeat("sensors.read")
http_.broadcastMeasurements(measurements)   # WebSocket :81
for m in measurements: history_.append(m); publishers_.publishAll(m)
rules_.evaluate(measurements)
```

## 5. Puntos de extensión

| Qué agregar | Cómo | Archivos a tocar | Estado |
|-------------|------|------------------|--------|
| Driver de sensor | Implementar `Sensor` (`id`/`model`/`interface`/`begin`/`measure`/`healthy`) y añadir el `if (spec.model == "…")` en la factoría | `include/core/sensors/Sensor.hpp`, `src/core/sensors/SensorFactory.cpp`, nuevo `src/core/sensors/XxxSensor.cpp` | Contrato estable, 17 implementaciones |
| Publicador externo | Implementar `Publisher` (`id`/`enabled`/`publish`) y registrarlo con `publishers_.registerPublisher()` | `include/core/publishers/Publisher.hpp`, `src/core/SemaCore.cpp` | Contrato estable, 2 implementaciones |
| Módulo opcional | Derivar de `Module` e invocar `modules().registerModule()` | `include/core/Module.hpp`, `src/core/SemaCore.cpp` | ⚠️ **Sin implementaciones**: el registro queda vacío y `enableAll()` no hace nada |
| Reacción a un evento | `SemaCore::events().subscribe(EventType::X, handler)` | `include/core/EventBus.hpp` | Estable; usado por `EventLog` (todos los tipos) y la lambda de alarmas |
| Capacidad de plataforma | `CapabilityManager::instance().set(Capability::X, true)` | `include/core/Capability.hpp`, `src/core/SemaCore.cpp:65-78` | Estable; `/api/v1/capabilities` lo expone |
| Backend de almacenamiento | Implementar `KeyValueStore` (`begin`/`get|putString`/`get|putUInt`/`clear`) | `include/core/storage/Storage.hpp` | Solo existe `NvsStore`; `storage.backend` es informativo (ver §7) |
| Cálculo derivado | Agregar la fórmula y el `emit(...)` | `src/core/derived/DerivedCalculator.cpp` | También existe `DerivedEngine` (motor previo, ver §7) |
| Expansor | Agregar el tipo a la config y el driver correspondiente | `ConfigManager.hpp` (`spiExpanders`), `GpioManager.cpp`, `HttpServer.cpp` | MCP23017, MCP23S17, MAX14830 y SC18IS602B son opciones de configuración |
| Tabla de particiones / board | `partitions_*.csv` + `build_flags` | `platformio.ini`, `include/hw/HwProfile.hpp` | 3 boards + demo |
| Endpoint REST | Handler privado + `server_.on(...)` | `include/core/web/HttpServer.hpp`, `src/core/web/HttpServer.cpp:52-121` | 53 llamadas a `server_.on(...)` registradas |

### 5.1 Flags de compilación que cambian el binario

| Flag | Default | Efecto real en el código |
|------|---------|--------------------------|
| `BOARD_ESP32_WROOM` / `_S3` / `_WROOM32U` | obligatorio | `#error` si falta; fija `SEMA_BOARD_ID`, `SEMA_FLASH_MB`, `SEMA_NATIVE_ETH` |
| `SEMA_USE_ETHERNET` | `1` (`SEMA_ON`) | Compila `EthernetManager` |
| `SEMA_USE_LORA` | `1` | Compila `LoraManager` + `SPI.begin()` |
| `SEMA_USE_MODBUS` | `1` | Compila `ModbusManager` |
| `SEMA_USE_CAN` | `1` | Compila `CanManager` |
| `SEMA_USE_ZIGBEE` | `1` | Compila `ZigbeeManager` + `zigbee_.loop()` |
| `SEMA_USE_SHIFT` | `0` (**SEMA_OFF**) | Sin él, `shift_.apply()` no se llama y `/api/v1/config/io` ignora `shift_registers` |
| `SEMA_MODBUS_ISOLATED` | `1` | Elige `SEMA_MODBUS_TRANSCEIVER`: `TD501D485H` (1) o `SN65HVD75DR` (0) |
| `SEMA_PINS_FROM_FILE` | `0` | Con `1`, `ConfigManager::applyHwProfile()` sobrescribe pines de CAN/Modbus/Zigbee/LoRa/Ethernet |
| `SEMA_FIXED_HARDWARE` | `0` | Con `1`, se ignora `config.sensors[]` y se usa el catálogo fijo de `SemaCore.cpp:284-297` |
| `SEMA_DEMO` | `0` | Valores ficticios en `/api/v1/sensors` y `/api/v1/history`, y MCP23S17 forzado (CS=5, 16 salidas) |

## 6. Todas las clases

| Clase | Header | Implementación | Responsabilidad |
|-------|--------|----------------|-----------------|
| `SemaCore` | `core/SemaCore.hpp` | `core/SemaCore.cpp` | Singleton: inicializa y conduce el sistema; expone accesores |
| `ConfigManager` | `core/ConfigManager.hpp` | `core/ConfigManager.cpp` | Config schema=1: `load`/`save`/`apply`/`applyJson`, validación y rollback |
| `CapabilityManager` | `core/CapabilityManager.hpp` | `core/CapabilityManager.cpp` | Singleton: máscara de capacidades (`set`/`has`) |
| `ModuleRegistry` | `core/ModuleRegistry.hpp` | `core/ModuleRegistry.cpp` | Registra módulos y conduce `enableAll`/`loopAll` |
| `Module` | `core/Module.hpp` | — (abstracta) | Contrato de módulo: `id`/`version`/install/configure/enable/start/loop/stop/disable |
| `EventBus` | `core/EventBus.hpp` | `core/EventBus.cpp` | Pub/sub síncrono filtrado por `EventType` |
| `Scheduler` | `core/Scheduler.hpp` | `core/Scheduler.cpp` | Tareas periódicas por intervalo en ms |
| `Watchdog` | `core/Watchdog.hpp` | `core/Watchdog.cpp` | TWDT del ESP32 con `begin(timeoutSeconds)`/`feed()` |
| `HealthMonitor` | `core/HealthMonitor.hpp` | `core/HealthMonitor.cpp` | Heartbeat, estado agregado y watchdog jerárquico por tarea (máx. 10) |
| `PowerManager` | `core/PowerManager.hpp` | `core/PowerManager.cpp` | Singleton: perfil energético, deep sleep, wake por GPIO, `wakeReason()` |
| `GpioManager` | `core/GpioManager.hpp` | `core/GpioManager.cpp` | Aplica `GpioSpec[]` (nativo o MCP23017) y lee/escribe pines |
| `ShiftRegisterManager` | `core/ShiftRegisterManager.hpp` | `core/ShiftRegisterManager.cpp` | `writeByte` a 74HC595 / `readByte` de 74HC165 por `shiftOut`/`shiftIn` |
| `Task` | `core/runtime/Task.hpp` | `core/runtime/Task.cpp` | Envoltura RAII de `xTaskCreate` (afinidad AUTO) / `vTaskDelete` |
| `WiFiManager` | `core/network/WiFiManager.hpp` | `core/network/WiFiManager.cpp` | STA/AP, IP estática, mDNS, reconexión con backoff 2 s→60 s |
| `EthernetManager` | `core/EthernetManager.hpp` | `core/EthernetManager.cpp` | LAN8720A (RMII) o W5500 (SPI + ESP-IDF/lwIP) |
| `ModbusManager` | `core/ModbusManager.hpp` | `core/ModbusManager.cpp` | `ModbusMaster` sobre `Serial2`, control DE/RE opcional |
| `CanManager` | `core/CanManager.hpp` | `core/CanManager.cpp` | TWAI: instalación por velocidad y `send`/`receive` |
| `LoraManager` | `core/LoraManager.hpp` | `core/LoraManager.cpp` | SX1262 por RadioLib; `send`/`receive` |
| `ZigbeeManager` | `core/ZigbeeManager.hpp` | `core/ZigbeeManager.cpp` | Tramas MT del ZNP por `Serial1`; parser de `AF_INCOMING_MSG` |
| `Sensor` | `core/sensors/Sensor.hpp` | — (abstracta) | Contrato de driver: `id`/`model`/`interface`/`begin`/`measure`/`healthy` |
| `SensorManager` | `core/sensors/SensorManager.hpp` | `core/sensors/SensorManager.cpp` | Registra drivers, lee, aplica calibración y llama a `DerivedEngine` |
| `SensorFactory` | `core/sensors/SensorFactory.hpp` | `core/sensors/SensorFactory.cpp` | `create(SensorSpec)` → driver de 17 modelos (o `nullptr`) |
| `I2cScanner` | `core/sensors/I2cScanner.hpp` | `core/sensors/I2cScanner.cpp` | Escaneo 1..126 y `modelForAddress()` (10 direcciones mapeadas) |
| `AdcSensor` | `core/sensors/AdcSensor.hpp` | `…/AdcSensor.cpp` | ADC interno con `scale`/`offset`; canal/unidad configurables |
| `Ads1115Sensor` | `core/sensors/Ads1115Sensor.hpp` | `…/Ads1115Sensor.cpp` | ADC externo I²C de 16 bits, 4 canales (`pin` = canal) |
| `Aht20Sensor` | `core/sensors/Aht20Sensor.hpp` | `…/Aht20Sensor.cpp` | I²C: temperatura + humedad |
| `As3935Sensor` | `core/sensors/As3935Sensor.hpp` | `…/As3935Sensor.cpp` | I²C: distancia al frente de tormenta (km) |
| `Bh1750Sensor` | `core/sensors/Bh1750Sensor.hpp` | `…/Bh1750Sensor.cpp` | I²C: luminosidad (lux) |
| `Bme280Sensor` | `core/sensors/Bme280Sensor.hpp` | `…/Bme280Sensor.cpp` | I²C: temperatura, humedad, presión |
| `Bmp280Sensor` | `core/sensors/Bmp280Sensor.hpp` | `…/Bmp280Sensor.cpp` | I²C: temperatura, presión |
| `CoSensor` | `core/sensors/CoSensor.hpp` | `…/CoSensor.cpp` | ADC: monóxido de carbono (ppm) |
| `Ds18b20Sensor` | `core/sensors/Ds18b20Sensor.hpp` | `…/Ds18b20Sensor.cpp` | 1-Wire: temperatura, multi-dispositivo por ROM |
| `PcntSensor` | `core/sensors/PcntSensor.hpp` | `…/PcntSensor.cpp` | PCNT: conteo de pulsos × `scale` |
| `Pms5003Sensor` | `core/sensors/Pms5003Sensor.hpp` | `…/Pms5003Sensor.cpp` | UART: PM1 / PM2.5 / PM10 |
| `Scd30Sensor` | `core/sensors/Scd30Sensor.hpp` | `…/Scd30Sensor.cpp` | I²C: CO₂, temperatura, humedad |
| `Sgp30Sensor` | `core/sensors/Sgp30Sensor.hpp` | `…/Sgp30Sensor.cpp` | I²C: eCO₂ (ppm) y TVOC (ppb) |
| `Sht31Sensor` | `core/sensors/Sht31Sensor.hpp` | `…/Sht31Sensor.cpp` | I²C: temperatura + humedad |
| `Sht40Sensor` | `core/sensors/Sht40Sensor.hpp` | `…/Sht40Sensor.cpp` | I²C: temperatura + humedad |
| `SolarSensor` | `core/sensors/SolarSensor.hpp` | `…/SolarSensor.cpp` | ADC: radiación solar (W/m²) |
| `Veml6075Sensor` | `core/sensors/Veml6075Sensor.hpp` | `…/Veml6075Sensor.cpp` | I²C: UVA, UVB e índice UV |
| `KeyValueStore` | `core/storage/Storage.hpp` | — (abstracta) | Frontera clave-valor del Core |
| `NvsStore` | `core/storage/NvsStore.hpp` | `core/storage/NvsStore.cpp` | `KeyValueStore` sobre `Preferences` (NVS) |
| `HistoryStore` | `core/storage/HistoryStore.hpp` | `core/storage/HistoryStore.cpp` | JSONL en microSD: `append`, `readRecent`, `prune`, `aggregate`, `readAggregated` |
| `EventLog` | `core/events/EventLog.hpp` | `core/events/EventLog.cpp` | Buffer acotado + JSONL en LittleFS, rotación al doble del límite |
| `RuleEngine` | `core/alarms/RuleEngine.hpp` | `core/alarms/RuleEngine.cpp` | Evalúa `Rule[]` sobre las mediciones y publica `Alarm` (severidad `Warning`) |
| `DerivedEngine` | `core/derived/DerivedEngine.hpp` | `core/derived/DerivedEngine.cpp` | Motor de derivadas que sí se invoca desde `SensorManager::readAll()` |
| `DerivedCalculator` | `core/derived/DerivedCalculator.hpp` | `core/derived/DerivedCalculator.cpp` | Motor de derivadas + unidades + veleta; lo usa `HttpServer` (no `SensorManager`) |
| `Publisher` | `core/publishers/Publisher.hpp` | — (abstracta) | Contrato de publicador externo |
| `PublisherManager` | `core/publishers/PublisherManager.hpp` | `core/publishers/PublisherManager.cpp` | Recorre publicadores habilitados por medición |
| `MqttPublisher` | `core/publishers/MqttPublisher.hpp` | `core/publishers/MqttPublisher.cpp` | Publica el JSON canónico en un topic (PubSubClient) |
| `HttpPublisher` | `core/publishers/HttpPublisher.hpp` | `core/publishers/HttpPublisher.cpp` | Webhook HTTP POST con timeout de 2 000 ms |
| `HttpServer` | `core/web/HttpServer.hpp` | `core/web/HttpServer.cpp` | 53 rutas HTTP, WebSocket :81, login/sesión, backup, OTA con SHA-256 |

## 7. Cosas que la documentación previa afirmaba y el código contradice

| Afirmación previa | Estado real en el código |
|-------------------|--------------------------|
| "15 drivers de sensor" (`Referencia-de-codigo.md` anterior) | Son **17** drivers: `SensorFactory::create()` cubre ADC, ADS1115, AHT20, AS3935, BH1750, BME280, BMP280, CO, DS18B20, PCNT, PMS5003, SCD30, SGP30, SHT31, SHT40, SOLAR, VEML6075 |
| `include/core/Capability.hpp/.hpp` (dos archivos) | Solo existe `Capability.hpp`; el manager está en `CapabilityManager.hpp` |
| `partitions.csv` | Hay tres: `partitions_4mb.csv`, `partitions_8mb.csv`, `partitions_16mb.csv` |
| "WebSocket `/ws` … se añadirá" (`HttpServer.hpp`) | Ya está implementado: `WebSocketsServer ws_{81}`, `ws_.begin()` y `broadcastMeasurements()` |
| `PowerManager::sleep(uint64_t us)` | El parámetro es **segundos** (`seconds * 1000000ULL` → timer de µs) |
| `DerivedEngine::compute(in, out)` | La firma real es `static void compute(std::vector<Measurement>&)` — muta el vector |
| "Watchdog jerárquico" | El TWDT (`Watchdog`) es plano; lo jerárquico es el watchdog **por tarea** de `HealthMonitor` |
| `HistoryStore` "JSONL sobre LittleFS" | El histórico vive en **microSD** (`enableSd`); sin SD `append()` devuelve `false`. LittleFS se usa solo para `EventLog` |
| `storage.backend` ("littlefs"/"flash"/"sd") | Solo se valida como cadena; **ningún** código elige backend con ese valor |
| `modules()` / `ModuleRegistry` | No hay ninguna clase derivada de `Module` en el repo: `modules_.count()` siempre es 0 |
| Sistema de módulos "✅" en `docs/MEJORAS.md` | Es 🔄 en el propio `MEJORAS.md` §2 y en `PENDIENTES.md` §2.1: interfaz + registro, sin módulos reales |

---

## Ver también

- [Referencia de API interna](Referencia-API-interna.md) · [Enumeraciones y tipos](Enumeraciones-y-tipos.md) · [Arquitectura](Arquitectura.md) · [Diagramas](Diagramas.md) · [Módulos y ciclo de vida](Modulos-y-ciclo-de-vida.md) · [Tareas y concurrencia](Tareas-y-concurrencia.md) · [Pruebas y validación](Pruebas-y-validacion.md) · [Compatibilidad de versiones](Compatibilidad-de-versiones.md) · [Guía de desarrollo](Guia-de-desarrollo.md)
