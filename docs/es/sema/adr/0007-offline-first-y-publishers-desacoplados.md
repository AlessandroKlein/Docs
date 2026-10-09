---
tags:
  - sema
  - adr
  - comunicaciones
---

# 0007. Offline-first con publicadores desacoplados

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0009, D-0010, D-0011, D-0030, D-0036, D-0037, D-0047

## Contexto

Una estación meteorológica suele estar en un lugar sin red estable. Si la lógica de
medición dependiera de Internet, MQTT o un servidor central, una caída de red dejaría
de medir. Al mismo tiempo, cada servicio externo (ThingSpeak, Windy, Weathercloud,
PWSWeather, MQTT, webhook) tiene su propio protocolo, latencia y forma de fallar.

## Decisión

**Offline-first** (D-0011, D-0030): la estación es autónoma; Internet, cloud, MQTT y
servidor son **complementarios**, nunca requisitos.

1. La estación expone API y web local sin depender de ningún servidor (D-0006 → ADR 0010).
2. Cada servicio externo es un **publicador independiente** sobre el modelo canónico
   (D-0010): `Publisher` (`id()`, `enabled()`, `publish()`), orquestados por
   `PublisherManager::publishAll()` después del almacenamiento. Nunca bloquean la
   adquisición: el webhook corta a 2 s (`HTTPClient::setTimeout(2000)`).
3. **Store & Forward** (D-0009): los datos se encolan localmente y se reenvían al
   recuperar la conexión.
4. El **Servidor Central multiestación** (D-0036, D-0037, D-0047) es un componente
   independiente que no le quita autonomía a la estación; sincroniza de forma
   incremental con `device_id`, secuencias y timestamps y **no** controla hardware
   crítico.

## Consecuencias

- ✅ Con la red caída la estación sigue midiendo, guardando y sirviendo su web.
- ✅ Agregar un publicador no toca la adquisición ni el resto de los publicadores.
- ✅ Hoy hay dos publicadores reales: `HttpPublisher` (webhook JSON) y `MqttPublisher`
  (PubSubClient, topic default `sema/measurement`), ambos deshabilitados si su
  configuración está vacía.
- ❌ **Store & Forward no está implementado**: no hay cola persistente de reenvío; si el
  publicador falla, la medición se pierde para ese publicador (queda solo en el
  histórico si hay SD).
- ❌ **Servidor Central fuera de alcance**: es un proyecto separado. SEMA solo se
  autentica contra él con `security.server_key` y versiona el diálogo con
  `SEMA_PROTOCOL_VERSION = 1`.
- ⚠️ `MqttPublisher` reconecta dentro de `publish()` y no reintenta ni encola; el
  webhook no verifica TLS.

## Ver también

- [Decisiones](../Decisiones.md) · [MQTT y WebSocket](../MQTT-y-WebSocket.md) ·
  [Conectividad y red](../Conectividad-y-red.md) · [Comunicaciones remotas](../Comunicaciones-remotas.md)
