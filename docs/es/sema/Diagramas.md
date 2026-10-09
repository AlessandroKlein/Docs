---
tags:
  - sema
  - diagramas
---

# Diagramas

> **Tipo:** Concepto | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Diagramas de la implementación **real** de SEMA. Todos los nodos usan los nombres
de clase, archivo o clave que existen en el código de v1.103.0; no hay nodos
inventados. Donde el diseño aspiracional difiere de lo implementado, se indica.

## 1. Arquitectura de alto nivel

```mermaid
flowchart TB
    subgraph HW["Hardware (ESP32)"]
        SENS["Sensores: I2C · 1-Wire · ADC · PCNT · UART"]
        ACT["GPIO / MCP23017 / SD (SPI)"]
        SD["microSD (SPI, CS configurable)"]
    end

    subgraph CORE["SemaCore — src/core/SemaCore.cpp"]
        SC["SemaCore::setup() / SemaCore::loop()"]
        SENSORS["SensorManager + SensorFactory + 17 drivers"]
        DERIV["DerivedEngine (estático) + DerivedCalculator"]
        STORE["HistoryStore (SD) · EventLog (LittleFS) · NvsStore"]
        CFG["ConfigManager (schema 1)"]
        BUS["EventBus + RuleEngine"]
        PUB["PublisherManager → HttpPublisher · MqttPublisher"]
        WEB["HttpServer (WebServer :80 + WebSocketsServer :81)"]
        HEALTH["HealthMonitor + Watchdog (TWDT 10 s)"]
        SCHED["Scheduler (core.heartbeat 5 s · sensors.read 10 s)"]
        CAPS["CapabilityManager"]
    end

    subgraph OPT["Módulos opcionales (compile-time SEMA_USE_*)"]
        ETH["EthernetManager (LAN8720A RMII / W5500 SPI)"]
        WIFI["WiFiManager (STA/AP + mDNS + backoff)"]
        MODBUS["ModbusManager (Serial2 + ModbusMaster)"]
        CAN["CanManager (TWAI)"]
        LORA["LoraManager (SX1262 vía RadioLib)"]
        ZIG["ZigbeeManager (ZNP por Serial1)"]
    end

    subgraph EXT["Externo"]
        DASH["Dashboard web (navegador)"]
        MQTTB["Broker MQTT"]
        HOOK["Webhook HTTP"]
    end

    SENS --> SENSORS
    SC --> CAPS
    SC --> CFG
    SC --> SENSORS
    SC --> STORE
    SC --> WEB
    SC --> SCHED
    SC --> HEALTH
    SENSORS --> DERIV
    SCHED --> SENSORS
    SCHED --> STORE
    SCHED --> PUB
    SCHED --> BUS
    SENSORS --> WEB
    DERIV --> STORE
    BUS --> STORE
    BUS --> WEB
    CFG --> SENSORS
    CFG --> OPT
    CFG --> PUB
    WEB --> DASH
    PUB --> MQTTB
    PUB --> HOOK
    ACT --> SC
    SD --> STORE
    SC -.->|"composición directa, NO ModuleRegistry"| OPT
```

Lectura del diagrama: la flecha punteada final recuerda que los managers
opcionales son **miembros de `SemaCore`**, no módulos registrados. El
`ModuleRegistry` queda vacío en runtime (ver §6).

## 2. Pipeline de datos (ciclo de 10 s)

Nombres reales: `SensorManager::readAll()`, `applyCalibration()`,
`DerivedEngine::compute()`, `HistoryStore::append()`,
`HttpServer::broadcastMeasurements()`, `PublisherManager::publishAll()`,
`RuleEngine::evaluate()`.

```mermaid
flowchart LR
    MEDIR["1. MEDIR<br/>Sensor::measure(out,4)<br/>SensorManager::readAll"] -->
    VALIDAR["2. VALIDAR<br/>Measurement::quality<br/>(8 Quality flags)"] -->
    CAL["3. CALIBRAR<br/>applyCalibration()<br/>gain · offset · rango"] -->
    PROC["4. PROCESAR<br/>DerivedEngine::compute<br/>dew_point · heat_index<br/>vapor_pressure · absolute_humidity"] -->
    ALM["5. ALMACENAR<br/>HistoryStore::append<br/>JSONL en microSD"] -->
    PUB["6. PUBLICAR<br/>broadcastMeasurements (WS :81)<br/>HttpPublisher · MqttPublisher"] -->
    RULES["7. EVALUAR<br/>RuleEngine::evaluate<br/>EventBus → EventLog"]
```

Detalles verificados que el diagrama no muestra:

- Los pasos 5, 6 y 7 se ejecutan **por cada medición** en un bucle, salvo el
  broadcast WS, que se hace una vez con el vector completo.
- El paso 2 es informativo: ninguna etapa descarta mediciones por `quality`.
- El paso 3 solo actúa si hay una `Calibration` cuya clave sea exactamente
  `sensorId + ":" + channelId`.
- El paso 7 publica un `Event` con `type = EventType::Alarm`,
  `severity = Severity::Warning`, `value = (int32_t)(valor * 100)` y
  `correlationId = id de la regla`.

## 3. Ciclo de arranque (setup → loop)

```mermaid
sequenceDiagram
    autonumber
    participant BOOT as main.cpp setup()
    participant SC as SemaCore
    participant NVS as NvsStore / ConfigManager
    participant FS as LittleFS / EventLog
    participant I2C as Wire + I2cScanner
    participant SEN as SensorManager
    participant NET as WiFiManager / EthernetManager
    participant WEB as HttpServer
    participant PUB as PublisherManager
    participant HEALTH as HealthMonitor + Watchdog

    BOOT->>SC: SemaCore::instance().setup()
    SC->>SC: Serial.begin(115200) + delay(200)
    SC->>NVS: store_.begin("sema")
    SC->>SC: history_.begin() (no monta nada sin SD)
    SC->>FS: eventLog_.begin() → LittleFS.begin(true)
    SC->>NVS: leer/incrementar "boots" → restartCount_
    SC->>SC: CapabilityManager::set(...) × 13
    SC->>NVS: config_.load()
    SC->>SC: history_.setRetentionSeconds(retentionDays×86400)
    SC->>I2C: Wire.begin(21,22) + I2cScanner::scan()
    SC->>SEN: applySensors() → registerSensor + beginAll()
    SC->>SC: applyCalibrations() · gpio_.apply() · SPI.begin()
    SC->>NET: modbus/can/lora/zigbee/ethernet apply() + wifi_.begin()
    SC->>SC: configTzTime(TZ POSIX, ntpServer, "time.nist.gov")
    SC->>WEB: http_.begin(*this) → server_.begin() + ws_.begin() :81
    SC->>PUB: registerPublisher(webhook, mqtt) + applyPublishers()
    SC->>SC: applyRules() + subscribe(Alarm)
    SC->>HEALTH: registerTask x2 · watchdog_.begin(10) · health_.begin()
    SC->>SC: scheduler_.add x2 · modules_.enableAll() (vacío)
    SC->>SC: events_.publish(boot) [EventType::System]
    SC->>SC: log final por serial

    loop SemaCore::loop() — siempre en la misma tarea
        SC->>HEALTH: watchdog_.feed()
        SC->>SC: modules_.loopAll() (sin módulos)
        SC->>NET: wifi_.loop() · zigbee_.loop() · ethernet_.loop()
        SC->>WEB: http_.loop() → handleClient() + ws_.loop()
        SC->>SC: cada 3600 s: history_.aggregate()
        SC->>HEALTH: scheduler_.run()
    end
```

## 4. Scheduler y latidos

```mermaid
flowchart TB
    LOOP["SemaCore::loop() cada ~ms"] --> RUN["Scheduler::run()<br/>(millis() - lastRun >= intervalMs)"]
    RUN --> HB["task 'core.heartbeat'<br/>cada 5 000 ms"]
    RUN --> SR["task 'sensors.read'<br/>cada 10 000 ms"]
    HB --> H1["HealthMonitor::tick()"]
    HB --> H2["HealthMonitor::taskHeartbeat('core.heartbeat')<br/>timeout 30 000 ms"]
    SR --> S1["SensorManager::readAll()"]
    SR --> S2["HealthMonitor::setSensorStats(online, total)"]
    SR --> S3["taskHeartbeat('sensors.read')<br/>timeout 60 000 ms"]
    SR --> S4["broadcastMeasurements + append + publishAll + evaluate"]
    H1 --> STATUS["HealthMonitor::status()"]
    H2 --> STATUS
    S3 --> STATUS
    S2 --> STATUS
    STATUS --> API["GET /api/v1/health"]
```

El intervalo del `Scheduler` se compara con `millis()` en cada vuelta del `loop()`,
así que la precisión depende de cuánto tarde el resto del `loop()`. No hay
temporizadores por hardware ni tareas FreeRTOS propias.

## 5. Bus de eventos (estado real)

`EventBus::publish()` recorre sus suscripciones e invoca el handler **en forma
síncrona**, en la misma tarea que publica. No hay cola ni buffer intermedio.

```mermaid
flowchart TB
    subgraph EMISORES["Quién publica HOY"]
        B["SemaCore::setup()<br/>EventType::System, correlationId='boot'"]
        R["RuleEngine::evaluate()<br/>EventType::Alarm, Severity::Warning"]
    end
    BUS["EventBus::publish(event)<br/>(síncrono, sin cola)"]
    subgraph SUBS["Quiénes están suscritos"]
        EL["EventLog<br/>suscrito a los 9 EventType<br/>→ deque de 100 + /events.jsonl"]
        AL["Lambda en SemaCore::setup()<br/>suscrita a EventType::Alarm<br/>→ Serial.print '[ALARM]'"]
    end
    B --> BUS
    R --> BUS
    BUS --> EL
    BUS --> AL
    EL --> API1["GET /api/v1/events"]
    EL --> API2["GET /api/v1/alarms"]
```

⚠️ De los 9 tipos de `EventType` (`Sensor, Rain, Lightning, Battery, Network,
Alarm, System, Wake, Sleep`), **solo dos se emiten**: `System` (el boot) y `Alarm`
(reglas). No hay emisores de `Sensor`, `Rain`, `Lightning`, `Battery`, `Network`,
`Wake` ni `Sleep` en el firmware; por eso el diagrama conceptual de versiones
anteriores ("RAIN_START → Storage, Alarm, MQTT, Webhook, Dashboard, Wake Manager")
no se corresponde con el código. El campo `Event::id` se inicializa en 0 y nunca se
incrementa.

## 6. Módulos: lo documentado vs. lo implementado

```mermaid
flowchart LR
    subgraph DIS["Diseño documentado (README §53)"]
        A1["Available"] --> A2["Installed"] --> A3["Configured"] --> A4["Enabled"] --> A5["Running"]
        A5 --> A6["Disabled"] --> A7["Uninstalled"]
        A8["InstallError · ConfigError · RuntimeError · UpdateError"]
    end
    subgraph IMP["Implementado en v1.103.0"]
        B1["ModuleRegistry::enableAll()<br/>recorre modules_ (vector vacío)"]
        B2["cada Module::id() se registra<br/>solo si alguien llama registerModule()"]
        B3["NO HAY LLAMADAS a registerModule()<br/>en todo el árbol de código"]
        B1 --> B2 --> B3
    end
    DIS -.->|"el enum ModuleState existe,<br/>el flujo no se ejecuta"| IMP
```

Los servicios reales no usan ese flujo: se aplican con `apply(cfg)` al arrancar y
con los `SemaCore::applyX()` en caliente. Ver
[Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md).

## 7. Almacenamiento

```mermaid
flowchart TB
    subgraph NVS["NVS — namespace 'sema' (Preferences)"]
        K1["clave 'config'<br/>JSON de configuración (schema 1)<br/>DynamicJsonDocument 16 384 B"]
        K2["clave 'boots'<br/>contador de reinicios (uint32)"]
        K3["clave del layout del dashboard<br/>(separada, para no exceder el límite de entrada NVS)"]
    end
    subgraph LFS["LittleFS (partición spiffs, 0x70000 = 448 KB)"]
        F1["/events.jsonl<br/>máx. 100 eventos en RAM<br/>rotación al llegar a 200 líneas"]
    end
    subgraph SDC["microSD (SPI, opcional)"]
        G1["/history.jsonl<br/>histórico crudo, máx. 10 000 entradas<br/>rotación: conserva la mitad más reciente"]
        G2["/history.jsonl.agg<br/>promedios por bucket de 3 600 s"]
    end
    HIST["HistoryStore"] -->|"append/readRecent/prune"| G1
    HIST -->|"aggregate() cada 3600 s"| G2
    EL["EventLog"] --> F1
    CFG["ConfigManager"] --> K1
    SC["SemaCore (restartCount_)"] --> K2
    CFG --> K3
    HIST -.->|"sin sdEnabled: no guarda NADA"| X["gráficas vacías"]
```

Nota: la configuración declara `storage.backend` con los valores válidos
`"littlefs" | "flash" | "sd"` (`ConfigManager::validate()`), pero el histórico se
escribe **siempre** en SD a través de `HistoryStore`; el campo `backend` no cambia
el destino del histórico.

## 8. Red y servicios expuestos

```mermaid
flowchart LR
    subgraph DEV["ESP32"]
        WIFI["WiFiManager<br/>STA (DHCP o IP fija) o AP"]
        MDNS["mDNS: http://&lt;hostname&gt;.local<br/>servicio http/tcp/80"]
        ETH["EthernetManager<br/>LAN8720A RMII (WROOM/WROOM32U)<br/>W5500 SPI (S3)"]
        HTTP["HttpServer<br/>WebServer en 80"]
        WS["WebSocketsServer en 81"]
        NTP["configTzTime → pool.ntp.org + time.nist.gov"]
    end
    BROWSER["Navegador"] -->|"HTTP :80"| HTTP
    BROWSER -->|"WebSocket :81"| WS
    MDNS --> BROWSER
    WIFI --> HTTP
    ETH --> HTTP
    NTP --> DEV
    HTTP -->|"POST"| HOOK["Webhook externo<br/>HttpPublisher, timeout 2 000 ms"]
    HTTP -->|"MQTT 1883"| MQTTB["Broker MQTT<br/>MqttPublisher"]
```

---

## Ver también

- [Arquitectura](Arquitectura.md) · [Modulos-y-ciclo-de-vida](Modulos-y-ciclo-de-vida.md)
- [Tareas-y-concurrencia](Tareas-y-concurrencia.md) · [Rendimiento-y-memoria](Rendimiento-y-memoria.md)
- [Diagnostico-y-salud](Diagnostico-y-salud.md) · [Almacenamiento-e-historico](Almacenamiento-e-historico.md)
- [MQTT-y-WebSocket](MQTT-y-WebSocket.md) · [API-REST](API-REST.md)
