---
tags:
  - sema
  - api
---

# API REST (referencia completa)

> **Tipo:** Referencia | **Estado:** Estable | **Firmware:** v1.76.0

API HTTP local de SEMA (`include/core/web/HttpServer.hpp`). Puerto **80**.
Todas las respuestas son JSON salvo el dashboard (HTML).

## Autenticación

| Mecanismo | Header / Cookie | Uso |
|-----------|-----------------|-----|
| `X-API-Key` | `X-API-Key: <api_key\|server_key>` | Endpoints de escritura y lectura sensible |
| Sesión web | `Cookie: sema_auth=<token>` | Lectura de config/backup y dashboard |

- `authorized()` acepta `api_key` **o** `server_key`.
- Si **ambas claves están vacías**, `authorized()` devuelve `true` (primera configuración).
- Los endpoints de **escritura** (`PUT /config`, `POST /restart`, `POST /ota`,
  `POST /backup`) exigen `X-API-Key` (no cookie).
- `GET /config` y `GET /backup` aceptan `X-API-Key` **o** cookie de sesión.

---

## Lectura pública

### `GET /api/v1/status`

```json
{ "station": "SEMA-001", "name": "Estación Norte", "firmware": "1.76.0", "uptime_s": 12345 }
```

### `GET /api/v1/health`

```json
{ "status": "HEALTHY", "uptime_s": 12345, "free_heap": 210000,
  "sensors": { "total": 6, "online": 5, "error": 1 } }
```

### `GET /api/v1/system`

```json
{ "id": "SEMA-001", "name": "Estación Norte", "firmware": "1.76.0",
  "hw": "rev0", "config_schema": 1, "protocol": 1 }
```

### `GET /api/v1/sensors`

```json
{
  "catalog": [ { "id": "EXT", "model": "BME280", "interface": "I2C", "healthy": true } ],
  "measurements": [ { "sensor_id": "EXT", "channel_id": "temperature",
                      "measurement": "temperature", "value": 23.4, "unit": "degC",
                      "quality": "VALID", "sequence": 123 } ]
}
```

### `GET /api/v1/history?limit=50`

`limit` 1–100 (default 50).

```json
{ "history": [ { "ts": 1720000000, "sensor": "EXT", "channel": "temperature",
                 "measurement": "temperature", "value": 23.4, "unit": "degC",
                 "quality": "VALID", "seq": 123 } ] }
```

### `GET /api/v1/events`

```json
{ "events": [ { "ts": 123456, "type": "alarm", "source": "EXT", "rule": "high_temp",
                "severity": "WARNING", "value": 41 } ] }
```

### `GET /api/v1/alarms`

Igual que `events`, filtrado a `type == alarm`.

### `GET /api/v1/capabilities`

```json
{ "capabilities": ["wifi", "adc", "pcnt", "dual_core", "can", ...] }
```

### `GET /api/v1/network`

```json
{ "mode": "STA", "connected": true, "ip": "192.168.1.50", "rssi": -55 }
```

### `GET /api/v1/energy`

```json
{ "profile": "always_on", "wake_reason": 0 }
```

### `GET /api/v1/diagnostics`

```json
{ "firmware": "1.76.0", "hw": "rev0", "uptime_s": 12345, "free_heap": 210000,
  "reset_reason": 1, "health": "HEALTHY",
  "history": { "entries": 120, "max": 500 },
  "tasks": 2, "modules": 3, "events": 50,
  "i2c_devices": [ { "address": "0x76", "model": "BME280" } ] }
```

### `GET /api/v1/gpio`

```json
{ "gpio": [ { "id": "relay", "pin": 26, "mode": "output", "value": 0 } ] }
```

---

## Escritura / autenticada

### `GET /api/v1/config` (auth: key o sesión)

Devuelve el JSON completo de configuración (incluye `api_key`/`server_key`).

### `PUT /api/v1/config` (auth: key)

Aplica el JSON completo (transaccional). Re-aplica **sin reinicio** sensores, reglas,
calibración, GPIO y publicadores.

```http
PUT /api/v1/config
X-API-Key: <api_key>
Content-Type: application/json

{ ...config... }
```

Respuesta: `200 {"ok":true}` o `400 {"error":"invalid config"}`.

### `GET /api/v1/backup` (auth: key o sesión)

Respaldo autodescriptivo (config + metadatos `backup_format`, `backup_version`,
`firmware`, `timestamp`).

### `POST /api/v1/backup` (auth: key)

Restaura una config (mismo handler que `PUT /config`).

### `POST /api/v1/gpio` (auth: key o sesión)

Escribe una salida.

```json
{ "pin": 26, "value": 1 }
```

### `GET /api/v1/shift`

Lee el byte del 74HC165 (o el último valor del 74HC595).

```json
{ "type": "74HC165", "value": 0 }
```

### `POST /api/v1/shift` (auth: key o sesión)

Escribe el byte en el 74HC595.

```json
{ "value": 255 }
```

### `POST /api/v1/restart` (auth: key)

Reinicia el ESP32. Respuesta `200 {"ok":true}` antes de reiniciar.

### `POST /api/v1/ota` (auth: key)

Subida de firmware (multipart). Ver [OTA](OTA-y-Actualizacion.md).

---

## Sesión web

### `POST /login`

Form-encoded `password=<api_key|server_key>`. Rate-limited (5 intentos / 60 s).
Éxito → `302` a `/` con `Set-Cookie: sema_auth=<token>; Path=/; HttpOnly; SameSite=Strict`.

### `GET /logout`

Limpia la sesión y redirige a `/`.

### `GET /` (raíz)

Sirve el dashboard (si hay sesión) o el formulario de login.

---

## Códigos de error

| Código | Significado |
|--------|-------------|
| `400` | Body faltante o JSON/config inválida |
| `401` | No autorizado |
| `404` | Ruta no encontrada |
| `429` | Login bloqueado (rate limit) |
| `500` | Error interno (serialización) |
