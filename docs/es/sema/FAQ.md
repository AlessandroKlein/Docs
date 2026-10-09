---
tags:
  - sema
  - faq
---

# Preguntas frecuentes (FAQ)

> **Tipo:** Soporte | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Respuestas verificadas contra el código de `v1.103.0`. Lo que no está implementado se
dice explícitamente.

## 1. General

### ¿Qué es SEMA?

Un **Sistema de Estación Meteorológica Autónoma**: firmware para **ESP32** que mide,
procesa, almacena y publica variables meteorológicas/ambientales, configurable por
completo desde una interfaz web. Principio rector: *el hardware define las capacidades;
la configuración web define cómo se utilizan*.

### ¿Puedo usarlo para algo que no sea meteorología?

Sí. El mismo firmware sirve para monitoreo ambiental, suelos, calidad de aire,
energía solar o nodos industriales (RS485/Modbus, CAN), porque cada canal se define
por configuración y no por código. Ver [Arquitectura](Arquitectura.md).

### ¿Dónde está el código?

<https://github.com/AlessandroKlein/SEMA> (firmware) y
<https://github.com/AlessandroKlein/Docs> (esta documentación).

### ¿Qué licencia tiene?

A la fecha (`v1.103.0`) el repositorio **no incluye un archivo `LICENSE`**: no hay
licencia declarada. Antes de reutilizarlo, consultá con el autor.

### ¿Cómo contribuyo?

Ver [Guía de desarrollo](Guia-de-desarrollo.md) y las reglas de trabajo del repo
(commits convencionales, un cambio lógico por commit, docs actualizadas en el mismo
ciclo).

## 2. Hardware

### ¿Cuál es el hardware mínimo?

Un módulo ESP32 (por ejemplo **DOIT DevKit v1**) con 4 MB de flash y una fuente de
3,3 V. Sin sensores, el firmware arranca y sirve la web/API con el catálogo de
sensores vacío.

### ¿Qué placas soporta?

Tres entornos listos: `esp32doit-devkit-v1` (ESP32-WROOM, Ethernet nativa LAN8720A),
`esp32-s3-devkitc-1` (ESP32-S3, W5500 por SPI) y `esp32-wroom-32u` (16 MB, PCB futura
con pines fijos). Además el entorno `demo` con datos ficticios. Ver
[Compatibilidad](Compatibilidad.md).

### ¿Cuántos sensores puedo conectar?

El código **no impone un límite fijo**: los topes prácticos son las direcciones del
bus I²C, los GPIO disponibles, el tamaño del JSON de configuración (hasta 16 384 B) y
la RAM. Con expansores (MCP23017, 74HC165, ADS1115) se amplían las entradas.

### ¿Soporta alimentación solar y batería?

Sí para **medir** la batería (por ADC con divisor configurable), aplicar perfiles
energéticos y usar *deep sleep* con wake por timer o por lluvia. **No** incluye gestión
de carga/MPPT: eso depende del controlador solar externo. Ver [Energía y consumo](Energia-y-consumo.md).

### ¿Se puede dejar en exteriores?

El firmware está pensado para estaciones remotas (deep sleep, reconexión, watchdog),
pero eso no protege la electrónica: hacen falta gabinete IP65, prensaestopas,
protecciones y apantallamiento. Ver [Instalación y mantenimiento](Instalacion-y-mantenimiento.md).

## 3. Uso

### ¿Cómo accedo a la web?

`http://<ip>/` o `http://sema-001.local/` (mDNS). Si configuraste
`security.api_key` o `security.password`, primero pide login. Ver
[Conectividad y red](Conectividad-y-red.md).

### ¿Cómo configuro los sensores?

Desde el dashboard (`/config/sensors`) o por API: `PUT /api/v1/config` (JSON completo,
transaccional) o `POST /api/v1/config/sensors` (solo esa sección). Si `sensors[]` está
vacío se usa el catálogo por defecto. Ver [Referencia de configuración](Referencia-configuracion.md).

### ¿Qué sensores soporta?

**17 modelos** en 5 interfaces: I²C (BME280, BMP280, SHT40, SHT31, AHT20, BH1750,
VEML6075, SCD30, SGP30, AS3935, ADS1115), 1-Wire (DS18B20), ADC (ADC, CO, SOLAR),
PCNT (pulsos) y UART (PMS5003). Detalle en [Sensores](Sensores.md).

### ¿Cómo publica los datos?

- **MQTT** al topic `sema/measurement` (config: `publishers.mqtt_*`).
- **Webhook HTTP** a `publishers.webhook_url`.
- **WebSocket** local para el dashboard.
- **API REST** bajo demanda.
Ver [MQTT y WebSocket](MQTT-y-WebSocket.md) y [API REST](API-REST.md).

### ¿Se integra con Home Assistant, ThingSpeak o Windy?

No hay integración dedicada en el firmware. Sí hay las piezas para hacerlo:
- **Home Assistant**: vía MQTT (usá la integración MQTT de HA apuntando al topic
  `sema/measurement`, o el broker que ya uses).
- **ThingSpeak/Windy/u otros**: el webhook HTTP es genérico; esos servicios esperan su
  propio formato, así que normalmente va un intermediario (Node-RED, script) que adapte
  el JSON.

### ¿Guarda histórico?

Sí: registro JSONL en LittleFS (o microSD si `storage.sd_enabled`), con rotación
(10 000 entradas por defecto), retención por tiempo, agregados horarios y export CSV
(`GET /api/v1/history?format=csv`). Ver [Almacenamiento e histórico](Almacenamiento-e-historico.md).

### ¿Tiene alarmas?

Sí: reglas configurables (`rules[]`) con operadores `gt`, `lt`, `ge`, `le` sobre un
canal, que generan eventos de tipo alarma con severidad y quedan en un log persistente.
Ver [Alarmas y reglas](Alarmas-y-reglas.md).

### ¿Tiene control PID?

**No está implementado.** El control actual es por umbrales/reglas. El PID está
documentado como idea futura en [Control PID](Control-PID.md).

### ¿Puedo conectar actuadores (relés, ventiladores)?

Sí: salidas digitales por GPIO (`gpio[]`), por MCP23017 y por 74HC595; se escriben con
`POST /api/v1/gpio` (o `/api/v1/shift`). Las reglas generan el evento, pero **no**
accionan la salida por sí solas: la automatización la hace un módulo externo/el Servidor
Central. Ver [Actuadores y salidas](Actuadores-y-Salidas.md).

### ¿Cómo actualizo el firmware?

Por **OTA** (`POST /api/v1/ota`, con verificación de integridad SHA-256) o por **USB**
(`pio run -t upload`). Las particiones A/B permiten rollback. Ver
[OTA y actualización](OTA-y-Actualizacion.md).

### ¿Cómo hago backup y lo restauro?

`GET /api/v1/backup` descarga un JSON autodescriptivo (config + `backup_format`,
`backup_version`, `firmware`, `timestamp`); `POST /api/v1/backup` lo restaura. Ver
[Backup y restauración](Backup-y-restauracion.md).

### ¿Puedo cambiar de placa y conservar la configuración?

Sí, en general: el backup es un JSON de configuración portable. Las claves de pines
pueden no ser válidas en la placa nueva (GPIO inexistentes o reservados), así que
revisá [Guía de pines](Guia-de-pines.md) después de restaurar.

## 4. Confiabilidad y seguridad

### ¿Funciona sin Internet?

Sí. La medición, el histórico, las alarmas y la web local no dependen de Internet. Sin
conexión se pierden solo NTP (hora), el chequeo de actualizaciones y los publicadores
externos (MQTT/webhook).

### ¿Es seguro?

Tiene `api_key`/`server_key` (y `extra_keys` revocables), login web con cookie,
rate limiting (5 intentos/60 s) y expiración de sesión (1 h). **Limitación importante:
la web local usa HTTP sin TLS** y el OTA no está firmado. No la expongas directamente
a Internet: usá red de confianza, VLAN o VPN. Ver [Seguridad](Seguridad.md).

### ¿Por qué se reinició solo?

Consultá `GET /api/v1/diagnostics` → `reset_reason` (5/6/7 = watchdog; 4 = excepción;
8 = wake de deep sleep). Ver [Solución de problemas](Solucion-de-problemas.md).

### ¿Cuánto consume / cuánta autonomía tiene?

Depende del hardware, de los intervalos y del perfil energético; el firmware ofrece
perfiles y deep sleep, pero la autonomía se calcula para cada instalación. Ver
[Energía y consumo](Energia-y-consumo.md).

### ¿Puedo tener varias estaciones?

Sí, cada SEMA es independiente y publica por MQTT/webhook. El dashboard de SEMA es de
**una** estación; la vista multi-estación corresponde al Servidor Central, que está
fuera del alcance de este proyecto.

---

## Ver también

- [Solución de problemas](Solucion-de-problemas.md) · [Guía de inicio](Guia-de-inicio.md) · [Glosario](Glosario.md)
- [Home](Home.md) · [Seguridad](Seguridad.md) · [Sensores](Sensores.md)
