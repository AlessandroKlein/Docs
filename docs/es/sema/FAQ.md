---
tags:
  - sema
  - faq
---

# Preguntas frecuentes (FAQ)

> **Tipo:** Soporte | **Estado:** Estable | **Firmware:** v1.71.0

## ¿Qué es SEMA?

Un **Sistema de Estación Meteorológica Autónoma** (ESP32) modular y configurable
desde la web. Principio: *el hardware define las capacidades; la configuración web
define cómo se utilizan*.

## ¿Cómo accedo a la web?

`http://<ip>/` o `http://sema-001.local/` (mDNS). Si hay claves configuradas, primero
pide login.

## ¿Cómo configuro los sensores?

Por `PUT /api/v1/config` (JSON `sensors[]`) o por el dashboard. Si `sensors[]` está
vacío, se usa el catálogo por defecto.

## ¿Qué sensores soporta?

15 tipos en 5 interfaces: ver [Sensores](Sensores.md).

## ¿Cómo publica los datos?

- **MQTT** (topic `sema/measurement`), **Webhook HTTP** y **WebSocket** local.
- Ver [MQTT y WebSocket](MQTT-y-WebSocket.md).

## ¿Cómo actualizo el firmware?

Por OTA (`POST /api/v1/ota`) o USB. Ver [OTA](OTA-y-Actualizacion.md).

## ¿Cómo hago backup?

`GET /api/v1/backup` (descarga) y `POST /api/v1/backup` (restaura).

## ¿Soporta batería / bajo consumo?

Sí: perfiles energéticos, deep sleep y wake por timer/lluvia.

## ¿Puedo conectar actuadores (relés)?

Sí, por GPIO standalone (`gpio[]`) y MCP23017. Ver
[Actuadores y salidas](Actuadores-y-Salidas.md).

## ¿Tiene control PID?

Hoy el control es por **umbrales/reglas** e histéresis; el PID está previsto como
mejora futura. Ver [Control PID](Control-PID.md).

## ¿Es seguro?

Tiene `api_key`/`server_key`, login con cookie, rate limiting, expiración de sesión y
`SameSite`. La web local es HTTP (TLS es futuro). Ver [Seguridad](Seguridad.md).

## ¿Dónde está el código?

[github.com/AlessandroKlein/SEMA](https://github.com/AlessandroKlein/SEMA).

## ¿Cómo contribuyo?

Ver [Guía de desarrollo](Guia-de-desarrollo.md) y las reglas de trabajo del repo.
