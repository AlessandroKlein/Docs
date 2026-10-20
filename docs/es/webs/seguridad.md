# Seguridad de Webs

Guía de **endurecimiento (hardening)** y resumen de los hallazgos de auditoría de
la plataforma Webs. Describe qué medidas ya están aplicadas y qué **debés cambiar
sí o sí** antes de exponer una instalación en producción.

## 1. Autenticación (medidas aplicadas)

| Mecanismo | Detalle |
| --------- | ------- |
| **JWT de acceso** | Firmado con `JWT_SECRET`, TTL **15 minutos** (`ACCESS_TOKEN_TTL_SECONDS = 60 * 15`), payload con `type: 'access'` y `permissions`. |
| **Refresh token** | Opaco (`randomBytes(48)` en hex), guardado **hasheado (SHA-256)** en `Session`, en cookie `httpOnly` + `SameSite=Lax` + `Secure`. TTL 30 días (90 con *recordarme*). **Rotación** en cada refresh. |
| **bcryptjs** | Coste **10** para password hash (`bcrypt.hash(input.password, 10)`). Nunca en texto plano. |
| **2FA TOTP** | `otplib` (Google Authenticator/Authy); 8 códigos de respaldo de un solo uso, hasheados. |
| **2FA SMS (OTP)** | Código vía proveedor de notificaciones (ej. Twilio), activado por usuario. |
| **Passkeys (WebAuthn)** | FIDO2 con `@simplewebauthn/server`; login sin contraseña y registro/gestión de llaves. `rpId` por defecto `localhost` (override con `WEBAUTHN_RP_ID`). |
| **Rate limiting** | Login **10/15 min**, registro **5/15 min**, recuperar/restablecer contraseña **5/15 min** por IP (Redis `rate-limiter-flexible`); global `@fastify/rate-limit` 300/min. |
| **Bloqueo por intentos** | Tras **5 intentos fallidos** la cuenta se bloquea **15 min** (`MAX_FAILED_LOGIN_ATTEMPTS = 5`, `LOCKOUT_DURATION_MS = 15 min`). |
| **2FA obligatoria para admins** | Con `require2FAForAdmin` en Ajustes → Seguridad, los roles `super_admin`/`admin` deben activar 2FA o passkey. |

## 2. Hardening requerido en producción

Estos son los puntos que **todavía usan valores de desarrollo** y debés corregir
antes de publicar:

| # | Ítem | Estado actual | Acción requerida |
| - | ---- | ------------- | ---------------- |
| 1 | `JWT_SECRET` | `change-me-in-production-please-0123456789` en `docker-compose.yml` (y `change-me-in-production` en `.env.example`; default de dev en `env.ts`) | Generar uno fuerte: `openssl rand -hex 32` y setearlo en el entorno real. |
| 2 | Contraseñas de base de datos | `docker-compose.yml` usa `webs/webs` (Postgres) y `root`/`webs`/`webs` (MySQL) | Cambiarlas y reflejarlas en `DATABASE_URL`; no exponer los puertos 5432/3306 a internet. |
| 3 | `CORS_ORIGIN` | `http://localhost:5173` (default y en compose) | Poner el(los) origen(es) real(es), separados por coma. **No usar `*`** (la API usa `credentials: true`). |
| 4 | `ENCRYPTION_KEY` | Opcional (`env.ts`); sin ella los campos secretos se guardan **sin cifrar** | Setearla: `openssl rand -hex 32`. Cifra los campos marcados `x-ui.secret` (p. ej. la **contraseña SMTP**) con AES-256-GCM. |
| 5 | `POST /api/v1/blocks/organizaciones/attendance/external` | Endpoint **público** y `X-Scanner-Key` **opcional** (`blocks/organizaciones/src/server.ts`) | Exigir `X-Scanner-Key` y validarla (o restringir el acceso por red/firewall) antes de producción. |
| 6 | HTTPS | Cookies `Secure` solo si el protocolo es `https` o se fuerza vía ajuste | Servir todo bajo **HTTPS** (reverse proxy/nginx); activar `secureCookies` y HSTS en Ajustes → Seguridad. |

### Detalle del punto 5 (asistencia externa)

El endpoint de asistencia por escáner está pensado para que un dispositivo físico
o un sistema automático registre asistencia sin pasar por la sesión del admin:

```ts
// blocks/organizaciones/src/server.ts (resumen actual)
fastify.post('/attendance/external', async (request) => {
  const qrcode = body?.qrcode;
  const scannerKey = headers?.['x-scanner-key'];   // opcional hoy
  // ... valida membresía y crea AttendanceRecord (method: 'external[:id]')
});
```

Hoy **no** valida la presencia ni el valor de `X-Scanner-Key`. En producción hay
que (a) exigir el header y compararlo contra una clave configurada, o (b) colocar
este endpoint detrás de una red/VPN privada. El resto de rutas de `organizaciones`
sí usan `authenticate`.

## 3. Buenas prácticas ya aplicadas

- **Sin secretos hardcodeados en código**: los secretos viven en variables de
  entorno, validadas por **Zod** (`apps/api/src/config/env.ts`); los defaults son
  de desarrollo y la API advierte en producción si `JWT_SECRET` usa el valor por
  defecto.
- **Secretos en ajustes con `x-ui.secret`**: los campos sensibles del esquema
  (contraseña SMTP, teléfono, dirección, WhatsApp) se marcan con `x-ui.secret:
  true` y se cifran en reposo con AES-256-GCM (`apps/api/src/lib/crypto.ts`);
  además se renderizan como input de contraseña con `x-ui.widget: 'password'`.
- **Copias de seguridad con re-verificación de contraseña**: `GET
  /api/v1/system/backup` exige el permiso `setting.read` **y** re-verifica la
  contraseña del usuario (`bcrypt.compare`) antes de exportar; la exportación
  **omite** `passwordHash` y los secretos de 2FA. El dump de BD (`POST
  /api/v1/system/backup`) exige `setting.write`.
- **Validación de entradas**: JSON Schema (Ajv) en body/query/params de toda la
  API; sanitización de HTML (`sanitize-html`) contra XSS en contenido enriquecido
  y plantillas de correo.
- **Cabeceras de seguridad**: Helmet (oculta `X-Powered-By`), CSP y HSTS
  configurables; mínimo privilegio para la cuenta de runtime
  (`scripts/create-least-privilege-user.sql`).

## Ver también

- [Autenticación y RBAC](architecture/auth-rbac.md) — flujo de sesión y permisos.
- [Guía de seguridad](guides/seguridad.md) — medidas organizadas por categoría.
- [Despliegue (Docker)](despliegue.md) — cómo aplicar estas variables en producción.
