# Diagramas de bloques y flujo de datos

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

Diagramas que explican cómo fluyen los datos y cómo se estructura el sistema.

## 1. Arquitectura del sistema

```mermaid
flowchart TB
    subgraph Field["Campo (invernadero)"]
        SENS["Sensores"] --> ESP["ESP32"]
        ESP --> ACT["Actuadores"]
    end
    subgraph Net["Red"]
        MQTT["MQTT broker"]
        HTTP["API REST"]
    end
    subgraph Server["Servidor central"]
        DB[("PostgreSQL")]
        DASH["Dashboard"]
    end
    ESP -->|"telemetría"| MQTT --> DB
    ESP <-->|"control/config"| HTTP
    DB --> DASH
```

## 2. Flujo de datos (telemetría)

```mermaid
sequenceDiagram
    participant S as Sensor
    participant E as ESP32
    participant M as MQTT
    participant W as Worker
    participant D as DB
    S->>E: lectura (I²C/1-Wire/RS485)
    E->>E: validar + calcular (VPD, ...)
    E->>M: publish greenhouse/{id}/sensors
    M->>W: mensaje
    W->>D: INSERT sensor_readings
    D-->>W: OK
```

## 3. Lazo de control local (sin servidor)

```mermaid
flowchart LR
    S["Sensores"] --> C["ControlTask (reglas/safety/PID)"]
    C --> A["Actuadores"]
    A --> S
```

## 4. Ciclo de vida de un dispositivo

```mermaid
stateDiagram-v2
    [*] --> DISCOVERED
    DISCOVERED --> PENDING
    PENDING --> COMMISSIONED
    COMMISSIONED --> ACTIVE
    ACTIVE --> BLOCKED
    BLOCKED --> ACTIVE
    ACTIVE --> REVOKED
    REVOKED --> [*]
```

## 5. Tareas FreeRTOS

```mermaid
flowchart TB
    subgraph Core0["Núcleo 0"]
        ST["SensorTask (adquisición)"] -->|"semáforo"| CT["ControlTask (control)"]
    end
    subgraph Core1["Núcleo 1"]
        LOOP["loop() — red, MQTT, API, OTA"]
    end
```
