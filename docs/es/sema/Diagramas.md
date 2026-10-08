---
tags:
  - sema
  - diagramas
---

# Diagramas de bloques y flujo de datos

> **Tipo:** Concepto | **Estado:** Estable | **Firmware:** v1.78.0

## Arquitectura de alto nivel

```mermaid
flowchart TB
    subgraph HW["Hardware"]
        S["Sensores (I²C, 1-Wire, ADC, PCNT, UART)"]
        A["Actuadores (GPIO / MCP23017)"]
    end

    subgraph FW["Firmware SEMA"]
        core["SemaCore"]
        sensors["SensorManager + SensorFactory"]
        config["ConfigManager (NVS)"]
        events["EventBus"]
        rules["RuleEngine"]
        derived["DerivedEngine"]
        history["HistoryStore (LittleFS)"]
        eventlog["EventLog (LittleFS)"]
        publishers["PublisherManager"]
        web["HttpServer + WebSocket"]
        power["PowerManager"]
        health["HealthMonitor + Watchdog"]
    end

    subgraph EXT["Externo"]
        dash["Dashboard web"]
        mqtt["Broker MQTT"]
        webhook["Webhook HTTP"]
        central["Servidor Central"]
    end

    S --> sensors --> derived --> history
    sensors --> events
    events --> rules --> events
    events --> eventlog
    history --> web --> dash
    history --> publishers --> mqtt
    publishers --> webhook
    core --> config
    core --> power
    core --> health
    web --> central
    A <--> web
```

## Flujo de datos (pipeline)

```mermaid
flowchart LR
    MEDIR["1. MEDIR<br/>(sensor.read)"] -->
    VALIDAR["2. VALIDAR<br/>(quality flags)"] -->
    PROCESAR["3. PROCESAR<br/>(derivadas, calibración)"] -->
    ALMACENAR["4. ALMACENAR<br/>(HistoryStore)"] -->
    PUBLICAR["5. PUBLICAR<br/>(MQTT/webhook/WS)"]
```

## Ciclo de vida (setup → loop)

```mermaid
sequenceDiagram
    participant B as Boot
    participant C as SemaCore
    participant S as Sensors
    participant W as Web
    participant P as Publishers

    B->>C: setup()
    C->>C: config.load() (NVS)
    C->>S: applySensors() + beginAll()
    C->>C: applyRules/Calibrations/Gpio/Publishers
    C->>W: http.begin() (WebServer + WS)
    C->>P: registerPublisher(webhook, mqtt)
    loop loop()
        C->>C: watchdog.feed() + scheduler.run()
        S->>S: readAll() cada 10 s
        S->>W: broadcastMeasurements (WS)
        S->>P: publishAll()
    end
```

## Eventos

```mermaid
flowchart LR
    PUB["EventBus.publish(event)"] --> SUB1["RuleEngine (alarmas)"]
    PUB --> SUB2["EventLog (JSONL)"]
    PUB --> SUB3["Serial (log)"]
```

## Almacenamiento

```mermaid
flowchart TB
    NVS["NVS<br/>config (clave=config)"]
    FS["LittleFS<br/>/history.jsonl (histórico)<br/>/events.jsonl (eventos)"]
```
