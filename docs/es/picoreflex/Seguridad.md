# Seguridad — PicoReflex

> Resumen de la postura de seguridad del firmware (auditoría completa en
> `docs/SEGURIDAD.md` del repo). Firmware auditado: **v0.9.0**.

## 1. Modelo de amenaza

El dispositivo opera en una **red local** (SoftAP propio o Wi-Fi de confianza),
no expuesto directamente a internet. Se audita con criterio de *defensa en
profundidad*, porque el SoftAP y el modo STA en redes ajenas amplían la
superficie a terceros en la misma red o con acceso físico.

## 2. Hallazgos y estado

| # | Hallazgo | Severidad | Estado |
|---|---|---|---|
| 2.1 | XSS reflejado/almacenado por el campo `name` | Alta | ✅ Resuelto (`html_escape` + `/player` eliminado) |
| 2.2 | Endpoints sin autenticación ni rate-limiting | Media | ✅ Resuelto en v0.7.0 (PIN + rate-limiting) |
| 2.3 | Credenciales Wi-Fi (STA) en texto plano | Media | 🟠 Aceptado (persistidas en Flash sin cifrar) |
| 2.4 | Contraseña del AP por defecto `12345678` | Media | 🟠 Configurable desde `/wifi` (v0.9.0) |
| 2.5 | HTTP sin TLS (texto plano) | Baja | 🟠 Aceptado (red local) |
| 2.6 | `smin`/`smax` sin cota superior | Baja | ✅ Resuelto (clamp 50..60000 ms) |
| 2.7 | Flash sin cifrar + checksum no criptográfico | Baja | 🟠 Aceptado (datos de baja sensibilidad) |

### 2.2 Endpoints sin autenticación ni rate-limiting — RESUELTO

- **Estado:** los endpoints de control/configuración (`/start`, `/stop`,
  `/reset`, `/seed`, `/game`, `/display`, `/wifi`, `/battery`, `/config`,
  `/ota`) exigen autenticación por **PIN** (`WEB_PIN`, por defecto `1234`,
  sobrescribible con `-DWEB_PIN`).
- **Mecanismo:** login por cookie `pr_auth` (token FNV-1a) o cabecera
  `Authorization: Bearer <pin>`, con comparación en **tiempo constante**.
- **Rate-limiting:** 60 req/s globales + 5 intentos/s en el login (anti fuerza
  bruta).

### 2.4 Contraseña del AP por defecto `12345678` — CONFIGURABLE

- **Estado (v0.9.0):** la clave del AP es **fija `12345678`** por defecto
  (`AP_PASSWORD_DEFAULT` en `include/config.hpp`), igual para cualquier placa.
  WPA2 por defecto (`AP_OPEN 0`).
- **Configuración:** se puede fijar una clave propia desde `/wifi` (8..15
  caracteres) y se persiste en Flash. En modo AP la clave efectiva se muestra en
  el display OLED (`PASS:xxxx`, línea 7) y por serial.
- **Historial:** v0.7.0 la derivó del RUID; v0.9.0 la restablece como fija a
  petición del usuario (el riesgo se mitiga cambiándola desde `/wifi`).

## 3. Recomendaciones

- **HECHO** — Autenticación PIN + rate-limiting (v0.7.0).
- **HECHO** — Contraseña del AP configurable desde `/wifi` (por defecto `12345678`).
- **HECHO** — Credenciales STA configurables desde la web y persistidas.
- **Pendiente (opcional)** — TLS en el panel web (`pico_mbedtls`, certificado
  autofirmado) para redes con terceros.
- **Pendiente (opcional)** — Cifrado de datos sensibles en Flash (AES-GCM con
  clave derivada del RUID).

## 4. Buenas prácticas de despliegue

- Recompila con `-DWEB_PIN=...` propio si el dispositivo se usa fuera del aula.
- No expongas el dispositivo directamente a internet.
- Compila en `Release` (define `NDEBUG`, apaga el debug de lwIP).
