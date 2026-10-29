# API HTTP — PicoReflex

El panel web escucha en **HTTP :80**. Los endpoints de control/configuración
exigen **autenticación por PIN** (ver la sección **Autenticación** más abajo).

## Autenticación

- PIN por defecto: `1234` (macro `WEB_PIN`, sobrescribible con `-DWEB_PIN=...`).
- Dos formas de autenticarse:
  - **Cookie:** `POST /login` con `pin=<PIN>` → establece la cookie `pr_auth`
    (token FNV-1a).
  - **Cabecera:** `Authorization: Bearer <PIN>` en cada petición.
- Sin sesión válida, los endpoints protegidos devuelven `401` (o el formulario
  de login en el navegador).
- **Rate-limiting:** 60 peticiones/s globales y 5 intentos/s en el login.

## Endpoints

| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| GET | `/` | — | Panel principal (HTML/CSS embebido) |
| GET/POST | `/login` | — | Formulario de PIN; establece `pr_auth` |
| GET | `/logout` | — | Borra la sesión |
| GET | `/api/status` | ✅ | JSON con estadísticas, estado del sistema y configuración |
| POST | `/start` | ✅ | Inicia una ronda |
| POST | `/stop` | ✅ | Detiene la ronda |
| POST | `/reset` | ✅ | Reinicia estadísticas |
| POST | `/seed` | ✅ | Fija la semilla del PRNG |
| POST | `/game` | ✅ | Modo, dificultad, estímulos, duración, cuenta regresiva |
| POST | `/display` | ✅ | OLED encendido/apagado + controlador SSD1306/SH1106 |
| POST | `/wifi` | ✅ | SSID/contraseña (STA) + clave del AP |
| POST | `/battery` | ✅ | Calibración de la curva de descarga |
| POST | `/config` | ✅ | Aplica parámetros (compatibilidad) |
| GET | `/ota` | ✅ | Página de actualización de firmware |
| POST | `/ota/upload` | ✅ | Sube el binario `.uf2`/`.bin` (streaming + CRC32) |

## Parámetros

### `/game`

| Parámetro | Valores |
|---|---|
| `mode` | `0` = Free Play, `1` = Contrarreloj, `2` = Muerte súbita, `3` = Simón Dice |
| `difficulty` | `0` = Fácil (1000 ms), `1` = Normal (500 ms), `2` = Difícil (250 ms) |
| `stimulus_min_ms` / `stimulus_max_ms` | Rango entre estímulos (clamp 50..60000 ms) |
| `round_seconds` | Duración de la ronda (modo Contrarreloj) |
| `start_delay_s` | Cuenta regresiva inicial (segundos) |

### `/wifi`

| Parámetro | Notas |
|---|---|
| `ssid` | SSID de la red STA (0..32 caracteres) |
| `password` | Contraseña STA (0..64 caracteres) |
| `ap_pass` | Clave del AP de respaldo (8..15 caracteres; vacío = `12345678` por defecto) |

### `/battery`

| Parámetro | Notas |
|---|---|
| `vfull` | Tensión (mV) con 100 % de carga |
| `vmid` | Tensión (mV) con 50 % de carga (inflexión) |
| `vempty` | Tensión (mV) con 0 % de carga (corte) |
| `loadcomp` | Compensación por carga (mV) sumada a la lectura |

### `/display`

| Parámetro | Notas |
|---|---|
| `oled` | `1` = encendido, `0` = apagado |
| `controller` | `0` = SSD1306 (0.96"), `1` = SH1106 (1.3"); se persiste en Flash |

### `/ota/upload`

| Cabecera | Notas |
|---|---|
| `X-CRC32` | CRC32 del binario (hex, opcional) |
| `Content-Length` | Tamaño total del binario |

El binario se escribe por streaming en el slot de **staging**; al terminar, un
commit lo promueve a aplicación activa (ver `Arquitectura.md` §9).

## Notas

- `/config` se mantiene por compatibilidad; los parámetros individuales
  (`seed`, `mode`, `difficulty`, `smin`, `smax`, `round`, `name`) se aplican
  igual que sus rutas dedicadas.
- Todos los valores numéricos se parsean con `strtol` (sin `atoi`) y se validan
  por rango.
