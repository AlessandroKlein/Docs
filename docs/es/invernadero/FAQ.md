---
tags:
  - invernadero
  - soporte
---

# Preguntas frecuentes (FAQ)

> **Tipo:** Guía | **Estado:** Estable | **Fecha:** 2026-10-02

Desde lo más básico hasta detalles técnicos. Cada respuesta indica dónde
profundizar.

## Lo básico

**¿Qué es este proyecto?**
Un firmware modular para **automatizar un invernadero** con ESP32: lee sensores,
controla actuadores (riego, ventilación, techo, luz) y se conecta opcionalmente a
un servidor central. Ver [Arquitectura](Arquitectura.md).

**¿Necesito Internet o un servidor para que funcione?**
**No.** El ESP32 es autónomo: mide, decide y controla localmente. El servidor es
opcional (administración y telemetría). Ver [Seguridad → invariantes](Seguridad.md).

**¿Qué necesito para empezar?**
Un ESP32 (módulo de **8 MB** de flash), fuente 5 V, y los sensores/actuadores.
Ver [Inicio rápido](Inicio-rapido.md) y [Materiales](Materiales.md).

**¿Se controla desde el celular?**
Sí, la web local es responsive (se adapta a pantallas chicas).

**¿Qué pasa si se corta Internet?**
Nada: la automatización sigue en el ESP32. Al volver, MQTT/HTTP se reconectan.

## Hardware

**¿Qué placas soporta?**
ESP32 clásica, S2, S3, C3 y C6. Ver [Compatibilidad](Compatibilidad.md).
⚠️ TWAI (CAN) solo en ESP32/S2/S3.

**¿Puedo usar un ESP32 de 4 MB?**
No directamente: el firmware (~1,29 MB) y la partición OTA requieren **8 MB**
(`default_8MB.csv`). Los módulos N8/N16 son los recomendados.

**¿Qué sensores soporta?**
SHT31, AHT20, DS18B20, humedad de suelo capacitivo, BH1750, SCD40/41, caudalímetro,
ultrasónico, pluviómetro, anemómetro, pH y EC. Ver [Sensores](Sensores.md).

**¿Puedo agregar un sensor que no está en la lista?**
Hoy **no sin recompilar**: los drivers son compilados. El catálogo permite elegir
entre los soportados y configurar su dirección/pin, pero un modelo nuevo requiere
código. Es el punto de extensión documentado en
[Referencia de código §7](Referencia-de-codigo.md).

**¿Por qué GPIO 34/35/36 son solo entrada?**
Limitación del silicio del ESP32 clásico: esos pines no tienen salida ni pull-up
interno. Ver [Guía de pines](Guia-de-pines.md).

## Configuración

**¿Puedo cambiar los pines sin recompilar?**
Sí, desde `/pins` (se guardan en NVS). Salvo que el firmware esté compilado con
`GH_PINS_LOCKED = 1` (PCB fabricada), en cuyo caso quedan bloqueados.

**¿Qué es `GH_PINS_LOCKED`?**
Un flag en `Version.hpp`: `0` = público (los usuarios configuran sus pines);
`1` = PCB fija (pines bloqueados, pero sensores/actuadores configurables).

**¿Diferencia entre `PinMap.hpp` y `PinConfig`?**
`PinMap.hpp` son **constantes de compilación** (referencia histórica).
`PinConfig` es el **mapa en tiempo de ejecución** guardado en NVS y editable por web.

**¿Qué es NVS?**
Non-Volatile Storage: memoria flash del ESP32 donde se guardan configuración,
pines (`ghpins`), catálogo de sensores (`ghsensors`) y expansores (`ghhw`).

**¿Cómo hago backup de la configuración?**
`GET /api/v1/config/export` para descargar; `POST /api/v1/config/import` para
restaurar. También existe `POST /api/v1/config/rollback` para la versión anterior.

**¿Se puede volver a fábrica?**
Sí: `POST /api/v1/factory-reset`. También hay reset parcial de red y de
automatización.

**¿Cuántos sensores y actuadores soporta?**
Hasta **20 sensores** (`MAX_SENSORS`), **32 actuadores** (`MAX_ACTUATORS`),
**8 válvulas**, **4 zonas** de suelo y **8 zonas** nominales.

## Funcionamiento

**¿Qué es el VPD?**
Déficit de presión de vapor: se calcula desde temperatura + humedad
(`vpd()` en `SensorManager`). Ver [Control PID](Control-PID.md).

**¿Usa control PID?**
Soporta la lógica; en lazos de mucha inercia (suelo, temperatura ambiente) usa
**histéresis**, más robusta. Ver [Control PID](Control-PID.md).

**¿Qué es el modo simulación?**
`simulation: true` genera datos sintéticos sin hardware, para probar la web y las
reglas.

**¿Qué es el Store & Forward?**
El servidor guarda telemetría con un `sequence`; el dispositivo puede reenviar
lo pendiente tras un corte. Ver [Servidor central](Servidor-central.md).

## Operación

**¿Cómo actualizo el firmware?**
Por OTA (ArduinoOTA por WiFi) o desde el servidor central. Con Ethernet
(W5500) funciona por **HTTP**. Ver [OTA](OTA-y-Actualizacion.md).

**¿Qué es el token de API?**
Un secreto de 128 bits generado en el dispositivo que autoriza el control desde
el servidor. Se gestiona en `/api/v1/token/*`. Ver [Seguridad](Seguridad.md).

**¿Cuánto ocupa en flash y RAM?**
Flash ~1,29 MB (~38,6 % de 3,34 MB), RAM ~89 KB (~27 % de 327 KB).

**¿Qué hago si un sensor marca `DISCONNECTED`?**
Ver [Solución de problemas §5](Solucion-de-problemas.md).

**¿Por qué MQTT funciona por Ethernet pero HTTPS no?**
`PubSubClient` acepta cualquier `Client`; `HTTPClient` solo acepta `WiFiClient`.
El OTA por HTTP se resolvió con un GET manual sobre `Client*`; HTTPS sobre W5500
requiere TLS (pendiente).

Ver también: [Solución de problemas](Solucion-de-problemas.md) ·
[Glosario](Glosario.md) · [Enumeraciones y tipos](Enumeraciones-y-tipos.md).
