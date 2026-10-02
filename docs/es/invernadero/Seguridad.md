---
tags:
  - invernadero
  - seguridad
---

# Seguridad

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-02

## 1. Autenticación local (web)

- Web local con login de administrador + token de sesión (expira 1 h).
- Contraseña derivada del UID en el primer arranque, persistida en NVS separado.

## 2. Token de API (servidor central)

- Token aleatorio de 128 bits en NVS separado (nunca se exporta en el JSON).
- Endpoints: `POST /api/v1/token/rotate|revoke`, `GET /api/v1/token/status`.
- `requireAuth()` acepta sesión de admin **o** token (`Authorization: Bearer`).

## 3. Servidor central

- JWT (Bearer) con `sub`, `roles`, `perms`, `scope`.
- Contraseñas `password_hash()` (bcrypt).
- RBAC granular (8 roles + permisos por recurso).
- **Rate limiting por IP** y **CORS configurable**.
- MQTT con usuario/clave + TLS en producción.

## 4. TLS

- Arquitectura preparada para TLS desde V8.
- Obligatorio para comunicación remota de producción; mTLS + certificados en V9.

## 5. Invariantes

1. El dispositivo funciona sin servidor.
2. El servidor nunca es necesario para una función automática crítica.
3. La automatización se ejecuta localmente.
4. Las comunicaciones son una capa independiente de la lógica de control.
5. El servidor administra; el ESP32 mide, decide, controla y protege.

> Ver [`docs/SECURITY.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/SECURITY.md).
