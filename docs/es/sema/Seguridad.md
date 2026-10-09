---
tags:
  - sema
  - seguridad
---

# Seguridad

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

Modelo de seguridad de SEMA tal como está **implementado** en
`src/core/web/HttpServer.cpp`: dos credenciales por header (`X-API-Key`), una
sesión web por cookie, rate limiting en el login y **HTTP sin TLS**. Esta página
también documenta lo que **no** está protegido, que es la parte más importante
para desplegar el equipo.

---

## 1. Resumen del modelo

```text
   navegador  ──► POST /login (usuario + contraseña) ──► cookie sema_auth (1 h deslizante)
   script     ──► X-API-Key: <api_key | server_key | extra_keys[nombre]>
   red local  ──► sin credenciales:
                    · GET /api/v1/status, /health, /system, /sensors, /history,
                      /events, /alarms, /gpio, /diagnostics, /network, /energy …
                    · WebSocket en el puerto 81 (sin autenticación)
```

- **`authorized()`** (`HttpServer.cpp:1465-1494`) valida el header `X-API-Key`
  contra `security.api_key`, `security.server_key` y los valores del mapa
  `security.extra_keys`. Si las tres fuentes están vacías, devuelve `true`.
- **`sessionAuthorized()`** (`HttpServer.cpp:1288-1301`) valida la cookie
  `sema_auth` contra un token por arranque y una ventana deslizante de 1 hora.
- **`webAuthed()`** (`HttpServer.cpp:1280-1286`) es el guardián de casi todo:
  `security.password` vacía ⇒ **`true` sin mirar nada**; si hay contraseña,
  acepta sesión **o** API key.

!!! danger "La `api_key` sola no protege la configuración"
    Los endpoints de configuración usan `webAuthed()`, no `authorized()`. Si dejás
    `security.password` vacía, `PUT /api/v1/config`, `GET /api/v1/config` y
    `GET /api/v1/backup` responden **sin credenciales**, aunque tengas `api_key`
    configurada. Para cerrar la web hay que definir **usuario y contraseña**
    (`security.username` + `security.password`).

## 2. Credenciales

| Clave en la config | Tipo | Uso | Efecto si está vacía |
|--------------------|------|-----|----------------------|
| `security.api_key` | str | `X-API-Key` para la API y las páginas web | No habilita ese camino |
| `security.server_key` | str | Segunda clave, mismo poder (Servidor Central) | No habilita ese camino |
| `security.extra_keys` | str JSON | Mapa `{"nombre":"clave"}`; cada valor sirve como `X-API-Key` | No habilita ese camino |
| `security.username` | str | Usuario del login web | Se usa `admin` |
| `security.password` | str | Contraseña del login web | **Login deshabilitado: la web queda abierta** |

Reglas exactas de `authorized()`:

1. Si `api_key`, `server_key` y `extra_keys` están **las tres** vacías → `true`.
2. Sin header `X-API-Key` → `false` (salvo el caso 1).
3. La primera coincidencia entre `api_key`, `server_key` o cualquier valor de
   `extra_keys` → `true`.
4. Si `extra_keys` no es JSON válido, se ignora ese mapa (no hay error).

Las claves adicionales se generan y revocan con `POST /api/v1/security/keys`
(`action: "generate"|"revoke"`, campo `name`); cada clave nueva son **32
caracteres hex** (16 bytes de `esp_random()`, `HttpServer.cpp:1758-1765`).

## 3. Mapa de endpoints: qué está protegido y qué no

### 3.1 Públicos (sin ninguna verificación)

| Endpoint | Qué expone |
|----------|------------|
| `GET /api/v1/status` | `station`, `name`, `firmware`, `uptime_s` |
| `GET /api/v1/health` | Estado de salud, heap libre, sensores online, tareas |
| `GET /api/v1/system` | Board, flash, pines reservados, pin de SD, SSID y IP, reinicios |
| `GET /api/v1/network` | Modo, IP, RSSI y estado de Ethernet |
| `GET /api/v1/energy` | Perfil energético y motivo de wake |
| `GET /api/v1/diagnostics` | Heap, tareas, módulos, dispositivos I²C detectados |
| `GET /api/v1/capabilities` | Capacidades del chip |
| `GET /api/v1/sensors` | Catálogo y **todas las mediciones** |
| `GET /api/v1/history` | Histórico (`limit` hasta 3000, `format=csv`) |
| `GET /api/v1/events` | Eventos del equipo |
| `GET /api/v1/alarms` | Alarmas registradas |
| `GET /api/v1/gpio` | Estado de cada GPIO gestionado |
| `GET /api/v1/shift` | Valor leído del shift register |
| `GET /api/v1/modbus` | Registros Modbus (si `SEMA_USE_MODBUS=1`) |
| `GET /api/v1/can` | Tramas CAN (si `SEMA_USE_CAN=1`) |
| `GET /api/v1/lora` | Estado LoRa (si `SEMA_USE_LORA=1`) |
| `GET /api/v1/zigbee` | Estado Zigbee (si `SEMA_USE_ZIGBEE=1`) |
| `GET /login`, `POST /login` | Formulario de acceso (con rate limiting, §4) |
| `GET /logout` | Cierra la sesión |
| `GET /gridstack.min.css`, `GET /gridstack-all.min.js` | Assets gzip embebidos |

### 3.2 Protegidos por `webAuthed()` (sesión **o** `X-API-Key`)

| Endpoint | Nota |
|----------|------|
| `GET /api/v1/config`, `PUT /api/v1/config` | Devuelve/reescribe la config completa, **incluidas todas las claves** |
| `POST /api/v1/config/network` | Reinicia el equipo |
| `POST /api/v1/config/system`, `/sensors`, `/io`, `/buses` | Configuración parcial |
| `GET /api/v1/wifi/scan` | SSIDs y RSSI del entorno |
| `POST /api/v1/security/keys` | Genera/revoca claves adicionales |
| `GET /api/v1/update/check` | Consulta GitHub (sale a Internet) |
| `GET /api/v1/backup`, `POST /api/v1/backup` | Backup/restauración de la config |
| `POST /api/v1/restart` | Reinicia el equipo |
| `POST /api/v1/wind/north`, `POST /api/v1/wind/resistors` | Calibración de la veleta |
| `GET`/`POST /api/v1/dashboard/layout` | Layout del dashboard |
| `POST /api/v1/gpio`, `/shift`, `/can`, `/lora`, `/zigbee` | Escritura sobre actuadores y buses |

### 3.3 Protegidos **solo** por `authorized()` (`X-API-Key`, no acepta cookie)

| Endpoint | Consecuencia |
|----------|--------------|
| `POST /api/v1/ota` | `onOtaUpload()` evalúa `authorized()` al comenzar el multipart (`HttpServer.cpp:1942`). La cookie `sema_auth` **no** sirve, y la UI embebida no envía `X-API-Key`: con claves configuradas, la subida desde la web responde `401` (ver [OTA y actualización](OTA-y-Actualizacion.md) §3) |

### 3.4 Páginas HTML

`/`, `/sensors`, `/events` y las cinco páginas `/config/*` llaman a
`webAuthed()` y, sin credenciales, devuelven el formulario de login con HTTP
`200` (`serveAuthedPage()`, `HttpServer.cpp:701-710`). No hay listado de
directorios ni contenido estático adicional: los únicos archivos servidos son
los dos assets de Gridstack.

### 3.5 WebSocket

`ws_.begin()` abre el servidor WebSocket en el **puerto 81** y
`broadcastMeasurements()` emite cada medición a todos los clientes conectados
(`HttpServer.cpp:135`, `143-163`). SEMA **no registra ningún handler
`onEvent`** para el WebSocket ni consulta credenciales antes de transmitir:
cualquier cliente de la red puede conectarse y recibir la telemetría en vivo.

## 4. Login web, sesión y rate limiting

| Aspecto | Valor real | Código |
|---------|------------|--------|
| Endpoint | `POST /login` con `username` y `password` (form-urlencoded) | `HttpServer.cpp:1307-1335` |
| Usuario válido | `security.username` o `"admin"` si está vacío | `HttpServer.cpp:1318` |
| Condición de éxito | `password` configurada no vacía **y** usuario y contraseña coinciden | `HttpServer.cpp:1319` |
| Cookie emitida | `sema_auth=<token>; Path=/; HttpOnly; SameSite=Strict` | `HttpServer.cpp:1324` |
| Token | 16 caracteres hex (`esp_random()` dos veces), **regenerado en cada arranque** | `HttpServer.cpp:123-129` |
| Atributo `Secure` | ❌ Ausente (el servidor es HTTP) | `HttpServer.cpp:1324` |
| `Max-Age`/`Expires` | ❌ Ausentes: es cookie de sesión del navegador | `HttpServer.cpp:1324` |
| Timeout | `kSessionTimeoutMs = 3 600 000` ms (**1 hora**) y es **deslizante**: cada request válido lo renueva | `HttpServer.cpp:47`, `1289-1299` |
| Sesiones simultáneas | Una sola: hay un único `sessionStartMs_` y un único token por arranque | `include/core/web/HttpServer.hpp:99-102` |
| Intento fallido | `401` con el formulario de login | `HttpServer.cpp:1333` |
| Rate limiting | A los **5 intentos fallidos** se bloquea el login durante **60 000 ms**; mientras dura responde `429` en texto plano («Demasiados intentos. Reintentá más tarde.») y el contador se resetea a 0 al bloquear | `HttpServer.cpp:1308-1332` |
| Alcance del bloqueo | Global del dispositivo (no distingue IP ni usuario) | `HttpServer.hpp:100-101` |
| Logout | Pone `sessionStartMs_ = 0` y borra la cookie con `Max-Age=0`; redirige a `/` | `HttpServer.cpp:1337-1342` |

Detalles con impacto real:

- Durante el bloqueo, **un login correcto también recibe `429`**: el chequeo de
  `lockoutUntilMs_` es lo primero que se evalúa.
- El contador de intentos no expira por tiempo: cuatro fallos hoy más uno dentro
  de una semana disparan el bloqueo.
- Como el token es único por arranque y la sesión es global, cualquier persona
  que obtenga la cookie (o el token) queda autenticada mientras haya actividad
  que mantenga viva la ventana deslizante; no hay cierre de sesión por usuario ni
  invalidación al cambiar la contraseña.
- Cambiar `security.password` no invalida la sesión en curso.

## 5. Transporte y confidencialidad

| Tema | Estado real |
|------|-------------|
| HTTP | **Sin TLS**: la web, la API y el login viajan en texto plano (`server_.begin()` sin certificados) |
| Contraseña del login | Viaja en claro en el `POST /login` |
| Claves de API | Viajan en claro en el header `X-API-Key` |
| `GET /api/v1/config` | Devuelve `api_key`, `server_key`, `extra_keys`, `password`, `network.password` y `publishers.mqtt_pass` **sin enmascarar** |
| Backup | Incluye todas esas claves en claro |
| WebSocket (81) | Sin cifrado y sin autenticación |
| `GET /api/v1/update/check` | Sale a Internet con `WiFiClientSecure::setInsecure()` (**no valida el certificado**); solo lee la versión publicada |
| OTA | El binario se sube por HTTP en claro; la verificación `X-SHA256` es **opcional** |

## 6. OTA: sin firma y con verificación opcional

`POST /api/v1/ota` acepta un multipart con el campo `firmware` y escribe la
partición OTA con `Update.begin()`/`write()`/`end(true)`. La única verificación
es la cabecera opcional `X-SHA256` (64 hex) comparada contra el SHA-256 de la
partición recién escrita:

- Sin `X-SHA256` → **no hay verificación de integridad propia** (solo la que hace
  `Update`, que valida el encabezado de imagen).
- ❌ **No hay firma digital** ni verificación de origen del binario: quien pueda
  autenticarse con una `X-API-Key` (o acceder mientras no haya claves) puede
  flashear cualquier imagen válida para ESP32.
- Con `SEMA_DEMO=1` y sin claves configuradas, la subida es completamente
  anónima.

## 7. Limitaciones conocidas

1. HTTP sin TLS en todos los servicios (§5).
2. WebSocket sin autenticación (§3.5).
3. `webAuthed()` abierto cuando `security.password` está vacía (§1).
4. Endpoints de **lectura** de telemetría, histórico, eventos, alarmas,
   diagnósticos, red y GPIO son públicos (§3.1).
5. `server_key` tiene los mismos poderes que `api_key`: da configuración
   completa, no un rol limitado.
6. Sin registro de auditoría de accesos ni de cambios de configuración; el único
   rastro es el log serie y los eventos internos.
7. Sin bloqueo de IP ni límite de tasa en el resto de los endpoints: el rate
   limiting solo existe en `POST /login`.
8. Cookies sin `Secure` (imposible sin TLS) y sin `Max-Age` (expiran al cerrar
   el navegador).
9. OTA sin firma (§6).
10. El login acepta únicamente `application/x-www-form-urlencoded` con los
    campos `username` y `password`; no hay recuperación de contraseña (se cambia
    desde la web autenticada o por serie/NVS).
11. `system.log_level` no tiene efecto, así que no se puede subir el detalle de
    log para auditar por configuración.

## 8. Recomendaciones de despliegue

| Prioridad | Recomendación |
|:---------:|---------------|
| 1 | Definir **`security.password`** (y `security.username`): sin ella la web y los endpoints de configuración quedan abiertos |
| 2 | Definir `security.api_key` con una cadena larga y aleatoria (32+ caracteres) y usarla en todos los clientes automatizados |
| 3 | **No exponer SEMA a Internet.** Mantenerlo en una red de confianza, en una VLAN propia o detrás de una VPN |
| 4 | Si hace falta acceso remoto, publicar a través de un proxy inverso con TLS y autenticación adicional (el TLS nativo no está implementado; ver [Futuro](Futuro.md)) |
| 5 | Restringir por firewall el puerto **81** (WebSocket) si no se usa el dashboard en vivo |
| 6 | Guardar los backups cifrados y fuera del equipo: contienen todas las claves en claro |
| 7 | Rotar `api_key`/`server_key`/`extra_keys` si el equipo estuvo en una red no confiable, y revisar quién conoce la contraseña web |
| 8 | Verificar el SHA-256 del firmware y usar `X-SHA256` en las subidas OTA propias |
| 9 | Cerrar la sesión web (`/logout`) en equipos compartidos y bajar el tiempo de sesión solo recompilando (`kSessionTimeoutMs` no es configurable) |

## 9. Reporte de vulnerabilidades

Según `SECURITY.md` del repositorio de código: el proyecto se publica con fines
de portafolio y referencia, **sin soporte ni mantenimiento activo**. Los reportes
se envían en privado a `aleklein23@gmail.com` (sin abrir issues públicos),
incluyendo descripción, pasos de reproducción y mitigación sugerida. Se acusa
recibo dentro de **7 días**; no hay obligación de corregir, ni bug bounty, ni
proceso formal de divulgación.

---

## Ver también

- [Configuración](Configuracion.md) · [Variables modificables](Variables-modificables.md)
- [OTA y actualización](OTA-y-Actualizacion.md) · [Backup y restauración](Backup-y-restauracion.md)
- [API REST](API-REST.md) · [Conectividad y red](Conectividad-y-red.md) · [Futuro](Futuro.md)
