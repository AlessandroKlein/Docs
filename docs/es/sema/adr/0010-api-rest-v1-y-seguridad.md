---
tags:
  - sema
  - adr
  - api
  - seguridad
---

# 0010. API REST `/api/v1` en el Core y seguridad por claves

> **Tipo:** Convención (ADR) | **Estado:** Aceptada | **Fecha:** 2026-10-03
> **Firmware:** v1.103.0 | **Decisión origen:** D-0006, D-0027, D-0035, D-0041, D-0048

## Contexto

La estación tiene que ser operada desde un navegador y, a la vez, consumida por
integraciones externas (servidor central, scripts, dashboards). Exponer todo sin
credenciales permite que cualquiera reinicie la estación o cambie pines; exigir un
servidor para configurarla contradice el offline-first (ADR 0007).

## Decisión

1. **D-0006 / D-0041** — REST + WebSocket forman parte del Core: API HTTP versionada
   en **`/api/v1`** con `status`, `health`, `system`, `config`, `sensors`, `events`,
   `alarms`, `diagnostics`, `network`, `capabilities`, `energy`, `history`, `backup`,
   `restart`, `ota`, `gpio`, `shift`, `wind/*`, `dashboard/layout`, `wifi/scan`,
   `update/check`, `security/keys` y los buses (`modbus`, `can`, `lora`, `zigbee`).
   El **WebSocket** va por el puerto **81** (`/ws`, `ws_` en `HttpServer.hpp`) y
   difunde `{"type":"measurements","data":[…]}`.
2. **D-0027** — la API se describe con esquema versionado (OpenAPI/JSON Schema) para
   generar clientes y validar.
3. **D-0035 / D-0048** — seguridad por roles y capacidades: API Key revocable, RBAC,
   least privilege, rate limiting.

Implementación verificable:

- `authorized()` acepta `X-API-Key` contra `api_key`, `server_key` o cualquier clave de
  `extra_keys` (JSON `{"nombre":"clave"}`); sin ninguna clave configurada devuelve
  `true` (primera configuración).
- `webAuthed()` acepta además la cookie de sesión `sema_auth`; si `security.password`
  está vacío, el acceso web queda abierto.
- Login web (`POST /login`) con usuario (`security.username`, default `admin`) y
  contraseña (`security.password`); **5 intentos fallidos bloquean 60 s** (HTTP 429) y
  la sesión dura **1 h deslizante** (`kSessionTimeoutMs = 3600000`).
- `POST /api/v1/ota` valida `X-API-Key` y, si se envía `X-SHA256` (64 hex), el hash de
  la partición escrita antes de reiniciar.

## Consecuencias

- ✅ Un cliente externo se integra con una sola clave y sin sesión.
- ✅ La web local sigue funcionando sin Internet y sin servidor.
- ⚠️ **Sin TLS/HTTPS**: la clave viaja en claro si la red no es de confianza.
- ⚠️ **RBAC real pendiente**: no hay roles ni permisos por endpoint más allá de
  "autenticado / no autenticado"; `server_key` hoy da los mismos permisos que
  `api_key`.
- ⚠️ **OpenAPI/JSON Schema no generado**: la referencia viva es `API-REST.md` y el
  propio `HttpServer.cpp`.
- ⚠️ `extra_keys` permite revocar borrando la entrada, pero no hay endpoint dedicado
  para listar claves activas.

## Ver también

- [Decisiones](../Decisiones.md) · [API REST](../API-REST.md) · [Seguridad](../Seguridad.md) ·
  [MQTT y WebSocket](../MQTT-y-WebSocket.md)
