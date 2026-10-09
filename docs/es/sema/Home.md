---
tags:
  - sema
  - general
---

# SEMA — Sistema de Estación Meteorológica Autónoma

> **Tipo:** Guía (portada) | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0
> **Hardware:** `rev0` · **Esquema de configuración:** 1 · **Protocolo:** 1

## 1. Resumen ejecutivo

**SEMA** es una plataforma modular para construir **estaciones meteorológicas y
ambientales** sobre el microcontrolador **ESP32**. Se configura por completo desde una
**interfaz web local**, sin recompilar el firmware: conectás los sensores, los declarás
en la configuración y la estación mide, guarda, publica y se actualiza sola.

> **El hardware define las capacidades. La configuración web define cómo se utilizan.**

- **Un solo firmware** para instalaciones muy distintas (interior/exterior, con o sin
  viento, con o sin calidad de aire).
- **17 modelos de sensor** ya soportados por I²C, 1-Wire, ADC, PCNT y UART.
- **Autonomía real (offline-first)**: la estación mide y sirve su web sin Internet ni
  servidor; la nube es complementaria.
- **Operación por red**: API REST `/api/v1`, dashboard, WebSocket, OTA con rollback y
  backup/restauración.

## 2. ¿Qué es?

SEMA (Sistema de Estación Meteorológica Autónoma) es el **Core** de una estación: un
conjunto de módulos independientes (sensores, alarmas, publicadores, almacenamiento,
red, web) que el núcleo orquesta. Por fuera se ve como una estación que se configura
desde el navegador; por dentro, cada pieza habla con las demás a través de interfaces
estables (`Sensor`, `Publisher`, `Module`, `KeyValueStore`) y de un Event Bus tipado.

Esa separación es la que permite que **una falla opcional no detenga la adquisición**:
si un sensor, el MQTT o la microSD fallan, el resto de la estación sigue midiendo.

```text
Sensores (I²C · SPI · UART · 1-Wire · ADC · PCNT)
        │  Measurement (modelo canónico + quality flag)
        ▼
SEMA Core — ConfigManager · EventBus · Scheduler · Storage API · HAL
        │
        ├── Storage   → NVS (config) · LittleFS (eventos) · microSD (histórico)
        ├── Servicios → API REST /api/v1 · dashboard · WebSocket :81 · OTA
        └── Salidas   → webhook HTTP · MQTT · reglas y alarmas
```

## 3. ¿Qué puede hacer hoy (v1.103.0)?

- **Medir** con 17 modelos: `ADC`, `ADS1115`, `AHT20`, `AS3935`, `BH1750`, `BME280`,
  `BMP280`, `CO`, `DS18B20`, `PCNT`, `PMS5003`, `SCD30`, `SGP30`, `SHT31`, `SHT40`,
  `SOLAR` y `VEML6075`.
- **Detectar** automáticamente los dispositivos conectados al bus I²C (y los DS18B20
  presentes en el bus 1-Wire).
- **Configurar todo desde la web**: catálogo de sensores, pines, buses (I²C/SPI/UART),
  GPIO, registros de desplazamiento, expansores, Modbus/CAN/LoRa/Zigbee/Ethernet,
  reglas, calibración, publicadores, red, seguridad y almacenamiento.
- **Calcular derivadas**: punto de rocío, índice de calor, sensación térmica, presión
  de vapor, humedad absoluta, VPD, QNH, altitud barométrica, AQI, tasa y acumulado de
  lluvia y dirección de viento.
- **Guardar** el histórico de mediciones en microSD (JSONL, con retención y
  agregación horaria) y los **eventos/alarmas** en la flash interna (LittleFS), de modo
  que sobreviven a un reinicio.
- **Publicar** cada medición por **webhook HTTP** y **MQTT**.
- **Exponer** una **API REST** (`/api/v1`, 53 rutas registradas en `HttpServer::begin()`)
  y un **dashboard web** en tiempo real (WebSocket en el puerto 81).
- **Actualizarse por OTA** con particiones A/B, verificación SHA-256 opcional y
  `GET /api/v1/update/check` contra el manifiesto de firmware.
- **Protegerse** con `api_key`, `server_key`, claves adicionales revocables, login web
  con sesión de 1 h y rate limiting (5 intentos / 60 s).
- **Diagnosticarse**: `/api/v1/health`, `/api/v1/diagnostics`, watchdog de 10 s,
  Health Monitor y contador de reinicios.

!!! warning "Lo que **no** está implementado (v1.103.0)"
    - **Store & Forward**: no hay cola de reenvío; una publicación fallida se pierde.
    - **Deep sleep automático**: existen `PowerManager::sleep()` y el wake por lluvia,
      pero ningún componente invoca el sueño, y el perfil energético no cambia la
      operación.
    - **Servidor Central**: es un proyecto separado, fuera del alcance de SEMA.
    - **TLS/HTTPS, RBAC por roles y OTA firmado**: la API es HTTP y la autorización es
      "autenticado o no".
    - **Módulos instalables en runtime**: la interfaz y el registro existen, pero no
      hay módulos reales ni carga dinámica.
    - **Histórico en flash interna**: sin microSD habilitada (`storage.sd_enabled`), el
      histórico queda vacío.

## 4. Características

- **Plataforma, no producto cerrado**: el catálogo de sensores sale de la
  configuración (`sensors[]`), no del código.
- **Modular por dentro**: Core, HAL, Event Bus, Scheduler, Storage API, Web/API y
  diagnóstico como piezas separadas.
- **Compatible por perfiles**, no por `#ifdef`: 4 entornos de PlatformIO
  (`esp32doit-devkit-v1` 4 MB, `esp32-s3-devkitc-1` 8 MB, `esp32-wroom-32u` 16 MB y
  `demo`).
- **Multiprotocolo**: I²C, SPI, UART, 1-Wire, ADC, PCNT, RS485/Modbus RTU, CAN/TWAI,
  Ethernet (LAN8720A por RMII o W5500 por SPI), LoRa SX1262 y Zigbee ZNP.
- **Configuración transaccional** con rollback y JSON versionado (`schema_version: 1`).
- **Calibración por canal** (`gain`, `offset`, rango) independiente del driver.
- **Codebase chico y verificable**: `src/main.cpp` son 18 líneas; la lógica vive en
  módulos bajo `src/core/`.

## 5. Accesos rápidos

| Recurso | Dirección |
|---------|-----------|
| Código fuente | <https://github.com/AlessandroKlein/SEMA> |
| Repositorio de documentación | <https://github.com/AlessandroKlein/Docs> |
| Documentación publicada | <https://alessandroklein.github.io/Docs/sema/> |
| Releases y firmware | <https://github.com/AlessandroKlein/SEMA/releases> |
| Manifiesto de firmware (versión + SHA-256) | <https://raw.githubusercontent.com/AlessandroKlein/SEMA/refs/heads/main/firmware_manifest.json> |
| Especificación completa (`README.md`) | <https://github.com/AlessandroKlein/SEMA/blob/main/README.md> |
| Registro de decisiones (`DUDAS-Y-DECISIONES.md`) | <https://github.com/AlessandroKlein/SEMA/blob/main/docs/DUDAS-Y-DECISIONES.md> |
| Dashboard local | `http://sema-001.local/` |
| Base de la API REST | `http://sema-001.local/api/v1` |
| WebSocket de mediciones | `ws://sema-001.local:81` |
| Verificación de actualización | `GET http://sema-001.local/api/v1/update/check` |
| Actualización OTA | `POST http://sema-001.local/api/v1/ota` |
| Respaldo / restauración | `GET` / `POST http://sema-001.local/api/v1/backup` |
| Páginas de configuración | `/config/network` · `/config/security` · `/config/system` · `/config/wind` · `/config/sensors` |
| Vistas | `/` (dashboard) · `/sensors` · `/events` · `/login` · `/logout` |

!!! tip "Si `sema-001.local` no resuelve"
    Usá la IP que muestra el puerto serie al arrancar, o consultá
    `GET http://<ip>/api/v1/network`. El nombre `.local` depende de mDNS
    (`network.mdns = true`). En modo AP el SSID es el hostname y la IP la informa el
    mismo endpoint.

## 6. Índice completo de la documentación

**🚀 Empezar**

- [SEMA — portada](Home.md) *(esta página)*
- [Guía de inicio](Guia-de-inicio.md) — qué es SEMA, hardware, primer arranque, API.
- [Instalación y mantenimiento](Instalacion-y-mantenimiento.md) — puesta en marcha en
  campo, protecciones, ubicación de sensores, checklists.
- [Compilación y flasheo](Compilacion-y-flasheo.md) — PlatformIO, particiones y OTA.
- [Guía de desarrollo](Guia-de-desarrollo.md) — agregar sensores, publicadores, reglas,
  endpoints y módulos.
- [Arquitectura](Arquitectura.md) — capas, principios, módulos del Core.
- [Diagramas](Diagramas.md) — flujo de datos, arranque y máquinas de estado.

**🔧 Referencia (software)**

- [Referencia de código](Referencia-de-codigo.md) — módulo → archivos → responsabilidad.
- [Referencia API interna](Referencia-API-interna.md) — clases y firmas reales.
- [Enumeraciones y tipos](Enumeraciones-y-tipos.md) — valores numéricos y significados.
- [Módulos y ciclo de vida](Modulos-y-ciclo-de-vida.md) — `AVAILABLE` → `RUNNING`.
- [Tareas y concurrencia](Tareas-y-concurrencia.md) — scheduler, watchdog, hilos.
- [Rendimiento y memoria](Rendimiento-y-memoria.md) — flash, RAM y límites.
- [Diagnóstico y salud](Diagnostico-y-salud.md) — health, logs y métricas.
- [Pruebas y validación](Pruebas-y-validacion.md) — cómo se valida cada cambio.
- [Compatibilidad de versiones](Compatibilidad-de-versiones.md) — schema, protocolo y SemVer.
- [Estadísticas y métricas](Estadisticas-y-metricas.md) — LOC, endpoints, uso de flash y RAM.

**🔌 Hardware**

- [Hardware y conexiones](Hardware-y-Conexiones.md) — fichas de conexión por componente.
- [Guía de pines](Guia-de-pines.md) — pines por placa, reservados y conflictos.
- [Compatibilidad](Compatibilidad.md) — placas, sensores y buses soportados.
- [Buses y periféricos](Buses-y-perifericos.md) — I²C, SPI, UART, 1-Wire, ADC, RS485, CAN.
- [Expansores de entrada/salida](Expansores-de-entrada-salida.md) — MCP23017, MCP23S17, 74HC595/165, ADS1115.
- [Sensores](Sensores.md) — los 17 modelos con su interfaz y magnitudes.
- [Actuadores y salidas](Actuadores-y-Salidas.md) — GPIO, relés y salidas.
- [Materiales](Materiales.md) — BOM y componentes recomendados.

**⚙️ Uso y configuración**

- [Configuración](Configuracion.md) — cómo se edita y se aplica la configuración.
- [Referencia de configuración](Referencia-configuracion.md) — cada clave del JSON.
- [Variables modificables](Variables-modificables.md) — qué se ajusta sin recompilar.
- [Calibración](Calibracion.md) — `gain`, `offset` y rangos por canal.
- [Magnitudes derivadas](Magnitudes-derivadas.md) — fórmulas y cuándo aplican.
- [Alarmas y reglas](Alarmas-y-reglas.md) — umbrales, operadores y eventos.
- [Control PID](Control-PID.md) — concepto y estado (no implementado).
- [Identidad y estados](Identidad-y-estados.md) — IDs, arranque y modos.

**🌐 Operación**

- [API REST](API-REST.md) — rutas, payloads y códigos de respuesta.
- [MQTT y WebSocket](MQTT-y-WebSocket.md) — publicación y tiempo real.
- [Conectividad y red](Conectividad-y-red.md) — Wi-Fi, AP, IP estática y mDNS.
- [Comunicaciones remotas](Comunicaciones-remotas.md) — LoRa, Zigbee, Ethernet y Modbus.
- [Almacenamiento e histórico](Almacenamiento-e-historico.md) — NVS, LittleFS y microSD.
- [Energía y consumo](Energia-y-consumo.md) — perfiles, batería y wake-up.
- [Backup y restauración](Backup-y-restauracion.md) — exportar e importar configuración.
- [OTA y actualización](OTA-y-Actualizacion.md) — particiones A/B, SHA-256 y rollback.
- [Seguridad](Seguridad.md) — claves, sesión web y rate limiting.

**🆘 Soporte**

- [Solución de problemas](Solucion-de-problemas.md) — síntoma → causa → verificación.
- [Preguntas frecuentes (FAQ)](FAQ.md) — dudas habituales.
- [Glosario](Glosario.md) — todos los términos en lenguaje simple.

**📚 Proyecto**

- [Decisiones (ADR)](Decisiones.md) — índice de los 14 ADR y las 60 decisiones `D-xxxx`.
- [Evolución](Evolucion.md) — las 9 fases y su estado.
- [Mejoras y roadmap](Mejoras-y-roadmap.md) — pendientes y Definition of Done.
- [Futuro](Futuro.md) — ideas opcionales a largo plazo.
- [CHANGELOG](CHANGELOG.md) — historial de versiones.
- [Registro de cambios](Registro-de-cambios.md) — qué archivo cambió y por qué.

**📄 Decisiones de arquitectura (ADR)**

- [0001 — Plataforma configurable y separación de capas](adr/0001-plataforma-configurable-y-separacion-de-capas.md)
- [0002 — Perfiles de placa/chip, capacidades y HAL](adr/0002-perfiles-de-placa-capacidades-y-hal.md)
- [0003 — Modelo canónico de mediciones y quality flags](adr/0003-modelo-canonico-de-mediciones.md)
- [0004 — Event Bus tipado](adr/0004-event-bus-tipado.md)
- [0005 — Calibración independiente del driver](adr/0005-calibracion-independiente-del-driver.md)
- [0006 — Almacenamiento por capas y retención](adr/0006-almacenamiento-por-capas-y-retencion.md)
- [0007 — Offline-first y publicadores desacoplados](adr/0007-offline-first-y-publishers-desacoplados.md)
- [0008 — Runtime, SMP y afinidad `AUTO`](adr/0008-runtime-freertos-y-afinidad-auto.md)
- [0009 — Configuración transaccional y Safe Mode](adr/0009-configuracion-transaccional-y-safe-mode.md)
- [0010 — API REST `/api/v1` y seguridad](adr/0010-api-rest-v1-y-seguridad.md)
- [0011 — OTA con rollback](adr/0011-ota-con-rollback.md)
- [0012 — Watchdog, Health Monitor y diagnóstico](adr/0012-watchdog-health-y-diagnostico.md)
- [0013 — Perfiles energéticos y wake por lluvia](adr/0013-perfiles-energeticos-y-wake-por-lluvia.md)
- [0014 — Módulos, drivers y descubrimiento](adr/0014-modulos-instalables-y-descubrimiento.md)

## 7. Por dónde empezar

| Si querés… | Empezá por |
|------------|------------|
| Entender qué es y verlo funcionando | [Guía de inicio](Guia-de-inicio.md) |
| Instalar la estación en el campo | [Instalación y mantenimiento](Instalacion-y-mantenimiento.md) |
| Compilar y flashear el firmware | [Compilación y flasheo](Compilacion-y-flasheo.md) |
| Conectar sensores y ver pines | [Hardware y conexiones](Hardware-y-Conexiones.md) · [Guía de pines](Guia-de-pines.md) |
| Configurar la estación | [Configuración](Configuracion.md) · [Referencia de configuración](Referencia-configuracion.md) |
| Integrar la estación con un sistema | [API REST](API-REST.md) · [MQTT y WebSocket](MQTT-y-WebSocket.md) |
| Programar sobre SEMA | [Guía de desarrollo](Guia-de-desarrollo.md) · [Referencia de código](Referencia-de-codigo.md) |
| Entender por qué se decidió algo | [Decisiones](Decisiones.md) |
| Resolver un problema | [Solución de problemas](Solucion-de-problemas.md) · [FAQ](FAQ.md) |

## Ver también

- [Guía de inicio](Guia-de-inicio.md) · [Arquitectura](Arquitectura.md) · [Evolución](Evolucion.md)
- [Decisiones](Decisiones.md) · [CHANGELOG](CHANGELOG.md) · [Glosario](Glosario.md)
