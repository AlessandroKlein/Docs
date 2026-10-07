---
tags:
  - sema
  - glosario
---

# Glosario

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.52.0

| Término | Definición |
|---------|------------|
| **SEMA** | Sistema de Estación Meteorológica Autónoma |
| **Medición canónica** | `Measurement`: representación única de una lectura (sensor, canal, valor, unidad, calidad) |
| **Canal (channel)** | Magnitud lógica de un sensor (`temperature`, `humidity`, …) |
| **Calidad (Quality)** | Flag de validez de una medición (`VALID`, `STALE`, …) |
| **Regla (Rule)** | Condición de alarma (`gt`/`lt`/`ge`/`le`) sobre un canal |
| **Evento (Event)** | Notificación tipada del Event Bus (sensor, lluvia, alarma, …) |
| **Derivada** | Magnitud calculada (punto de rocío, índice de calor, presión de vapor, humedad absoluta) |
| **Publicador** | Módulo que envía mediciones (MQTT, webhook) |
| **Webhook** | URL HTTP que recibe cada medición como JSON |
| **NVS** | Non-Volatile Storage (donde se guarda la config) |
| **LittleFS** | Sistema de archivos en flash (histórico, eventos) |
| **mDNS** | Resolución de nombres `<hostname>.local` sin DNS |
| **OTA** | Over-The-Air: actualización de firmware por red |
| **Partitions A/B** | Dos particiones de app para OTA con rollback |
| **Deep sleep** | Modo de bajo consumo con wake por timer/GPIO |
| **PCNT** | Contador de pulsos (lluvia/viento) |
| **MCP23017** | Expansor I²C de 16 GPIO |
| **ADS1115** | ADC externo I²C de 16 bits (4 canales) |
| **Board Profile** | Pines fijos para PCB propio (`SEMA_FIXED_HARDWARE`) |
| **Capability** | Función del hardware declarada por el chip (wifi, adc, pcnt, …) |
| **Schema version** | Versión del esquema de configuración (actualmente `1`) |
| **Protocol version** | Versión del protocolo con el Servidor Central (actualmente `1`) |
| **Hot reload** | Re-aplicar config sin reinicio (sensores, reglas, GPIO, publicadores) |
| **Rate limiting** | Bloqueo tras N intentos fallidos de login |
| **SemVer** | Versionado semántico `MAJOR.MINOR.PATCH` |
| **DoD** | Definition of Done (criterio de completitud) |
| **PID** | Control Proporcional-Integral-Derivativo |
| **Histéresis** | Control por dos umbrales (encender/apagar) para evitar oscilación |
