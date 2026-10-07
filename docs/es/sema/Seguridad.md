---
tags:
  - sema
  - seguridad
---

# Seguridad

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.62.0

Modelo de seguridad de SEMA.

## Credenciales

| Clave | Uso |
|-------|-----|
| `security.api_key` | Clave de la web/API local; también es la contraseña del login |
| `security.server_key` | Clave del Servidor Central → SEMA |

- `authorized()` acepta `api_key` **o** `server_key` vía header `X-API-Key`.
- Si **ambas están vacías**, se permite el acceso (primera configuración).
- Las claves se devuelven **solo** a peticiones autenticadas (`GET /config`, `GET /backup`).

## Endpoints protegidos

| Endpoint | Requiere |
|----------|----------|
| `PUT /api/v1/config` | `X-API-Key` |
| `POST /api/v1/backup` | `X-API-Key` |
| `POST /api/v1/restart` | `X-API-Key` |
| `POST /api/v1/ota` | `X-API-Key` |
| `GET /api/v1/config` | `X-API-Key` **o** sesión |
| `GET /api/v1/backup` | `X-API-Key` **o** sesión |
| `POST /api/v1/gpio` | `X-API-Key` **o** sesión |

## Login web

- `POST /login` con la contraseña (`api_key` o `server_key`).
- Éxito → cookie `sema_auth=<token>; Path=/; HttpOnly; SameSite=Strict`.
- El token de sesión se genera por arranque con `esp_random()`.

## Mitigaciones

| Mecanismo | Detalle |
|-----------|---------|
| Rate limiting | 5 intentos fallidos → bloqueo de 60 s (`429`) |
| Expiración de sesión | sesión deslizante de 1 h; el logout la invalida |
| `SameSite=Strict` | mitiga CSRF contra `PUT /config` |
| Cookie `HttpOnly` | evita lectura del token por JavaScript |
| Backup autenticado | el respaldo (con claves) exige auth |

## Recomendaciones

- Cambiar `api_key`/`server_key` de sus defaults.
- Usar claves largas y únicas.
- Para exponer SEMA a Internet, usar una VPN o proxy con TLS (el TLS nativo está
  previsto como mejora futura; ver [Futuro](Futuro.md)).

## Limitaciones conocidas

- La web local es **HTTP** (sin TLS); la contraseña viaja en claro en el POST de
  login dentro de la red local.
- El `server_key` permite config de riesgo vía API (por diseño, para el Servidor Central).
