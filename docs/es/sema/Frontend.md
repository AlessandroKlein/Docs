---
tags:
  - sema
  - frontend
---

# Frontend (dashboard web)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.114.1

El dashboard de SEMA es HTML+CSS+JavaScript **embebido en el firmware** (constantes
`PROGMEM` en `src/core/web/HttpServer.cpp`), sin dependencias externas en tiempo de
ejecución: no hay `data/`, ni LittleFS para la web, ni CDN (v1.102.0).

## 1. Cómo está construido

| Componente | Ubicación | Detalle |
|------------|-----------|---------|
| HTML del dashboard | `HttpServer.cpp:215` (`kIndexHtml`) | Portada: tarjetas de sensores conectados + predicciones |
| HTML de login | `HttpServer.cpp:23` (`kLoginHtml`) | Formulario de una contraseña |
| CSS base | `HttpServer.cpp:385` (`kBaseCss`) | Tema oscuro/claro y componentes |
| Navegación | `HttpServer.cpp:449` (`kNav`) | Barra de navegación común entre páginas |
| JS de tema | `HttpServer.cpp:469` (`kThemeJs`) | `toggleTheme()` + persistencia |
| JS de idioma | `HttpServer.cpp:476` (`kI18nJs`) | `applyLang()` / `toggleLang()` |
| Cuerpos de páginas de config | `HttpServer.cpp` (`const String body = R"html(…)`) | Un cuerpo por página |
## 2. Páginas

| Ruta | Handler | Contenido |
|------|---------|-----------|
| `/` | `onRoot()` | Dashboard (o login si no hay sesión) |
| `/sensors` | `onSensorsViewPage()` | Vista de mediciones en vivo |
| `/events` | `onEventsPage()` | Eventos y alarmas del `EventLog` |
| `/config/network` | `onNetworkPage()` | WiFi/Ethernet, IP, escaneo de redes |
| `/config/security` | `onSecurityPage()` | Claves API y credenciales del login |
| `/config/system` | `onSystemPage()` | Identidad, hora/NTP, unidades, idioma |
| `/config/wind` | `onWindPage()` | Veleta: resistencias, pull-up, norte |
| `/config/sensors` | `onSensorsPage()` | Alta/edición de sensores y calibración |
| `/login` (GET/POST) | `onLoginPage()` / `onLoginPost()` | Autenticación |
| `/logout` | `onLogout()` | Cierre de sesión |

## 3. Dashboard

Desde v1.114.0 la portada (`/`) es una página **limpia, sin tablero editable**:

- **Sensores conectados**: tarjetas con nombre, valor + unidad y estado; se muestran
  solo los sensores del catálogo con estado `healthy` y las magnitudes derivadas
  (`sensor_id = "DERIVED"`). El catálogo completo sigue en `/config/sensors`.
- **Estado**: nombre de estación, firmware, `hw`, uptime.
- **Predicción** (v1.104.0): sección con el pronóstico a corto plazo
  (`GET /api/v1/forecast`) para temperatura, humedad y lluvia, con filas por
  horizonte (+3h / +6h / +12h / +24h) que muestran valor y % de confianza.
- **Configuración**: formularios por sección (red, seguridad, sistema, viento,
  sensores) en lugar de JSON crudo.

> Hasta v1.113.0 el dashboard usaba un tablero **Gridstack** (tarjetas arrastrables
> y redimensionables) con layout persistido en `system.dashboard_layout` y
> `GET/POST /api/v1/dashboard/layout`. Todo eso se eliminó: la librería
> `GridstackAssets.h` (~1463 líneas) y sus rutas `/gridstack.min.css` /
> `/gridstack-all.min.js` ya no existen, y el layout no se guarda. Ahorro de ~40 KB
> de flash.

### 3.1 Toggle de elementos y bloqueo de pines (v1.105.0–v1.106.0)

- **Elementos habilitables por usuario final**: sensores y buses (Ethernet, Modbus,
  CAN, LoRa, Zigbee) se habilitan/deshabilitan desde la web, persistido en la config (LittleFS),
  también en la variante PCB (`SEMA_PINS_FROM_FILE=1`). El campo `enabled` se aplica
  siempre.
- **Edición de pines solo en modo AP**: `pinsLocked()` = `(SEMA_PINS_FROM_FILE != 0)
  || !isAp()`. En STA los selects de pines quedan solo-lectura y se muestra una nota
  i18n (es/en); en la variante PCB los pines quedan bloqueados siempre.

## 4. Tema e idioma

| Aspecto | Implementación |
|---------|----------------|
| Tema oscuro (default) | Clase ausente en `<body>`; paleta dark OLED (v1.109.0): `--bg:#0F172A`, `--card:#1B2336`, `--bd:#334155`, `--fg:#F8FAFC`, `--muted:#94A3B8`, acento teal/cyan `#22d3ee`, `--ok` verde, `--err` rojo, `--warn` ámbar |
| Tema claro | Clase `.light` en `<body>` (`toggleTheme()`); paleta clara (`--bg:#F1F5F9`, `--card:#FFFFFF`, acento `#0e7490`) |
| Valores numéricos | `ui-monospace,"SF Mono","Cascadia Mono",Consolas,monospace` |
| Detalles visuales | Glow cyan sutil en títulos, focus ring `rgba(34,211,238,.25)`, transición suave de tema |
| Persistencia del tema | `localStorage["sema_theme"]` |
| Idiomas | Español (default) e inglés, definidos en el objeto `I18N` |
| Traducción | Atributos `data-i18n="clave"` en el HTML; `applyLang()` los reemplaza |
| Persistencia del idioma | `localStorage["sema_lang"]` + `document.documentElement.lang` |
| Config asociada | `system.lang` (`"es"`/`"en"`) y `system.units` (`"metric"`/`"imperial"`) |

## 5. Autenticación en la web

1. Sin cookie válida, `/` muestra el formulario de login.
2. `POST /login` (form-encoded) valida `username` (por defecto `admin` si
   `security.username` está vacío) contra `security.password`. Si `security.password`
   está vacía **no hay login**: el acceso queda abierto (primera configuración).
3. Éxito → `302` a `/` con `Set-Cookie: sema_auth=<token>; Path=/; HttpOnly; SameSite=Strict`.
4. La sesión es **deslizante y dura 1 h** (`kSessionTimeoutMs = 3600000`).
5. `/logout` invalida el token. El rate limiting del login es de **5 intentos / 60 s**
   (respuesta `429`).

## 6. Comunicación con el firmware

- **`fetch()`** al mismo origen para estado, mediciones, histórico, eventos y guardado
  de configuración; las escrituras que lo requieren llevan `X-API-Key`.
- **Refresco periódico** del dashboard (lecturas cada pocos segundos): no depende de
  WebSocket.
- **WebSocket** en el **puerto 81** (`WebSocketsServer ws_{81}`) para clientes externos:
  el firmware emite `{"type":"measurements","data":[…]}`
  (`HttpServer::broadcastMeasurements`) solo si hay clientes conectados. Ver
  [MQTT y WebSocket](MQTT-y-WebSocket.md).

## 7. Cómo modificarlo

1. Editar las constantes `PROGMEM` en `src/core/web/HttpServer.cpp` (HTML, CSS, JS).
2. Compilar y flashear (`pio run -e esp32doit-devkit-v1 -t upload`).
3. Al probar cambios de CSS/JS hay que forzar recarga (Ctrl+F5).

> **Regla del proyecto**: la interfaz no debe requerir cambiar el firmware para
> configurar la estación. Si algo es configurable, va a un formulario y a la API, no a
> una constante.

---

## Ver también

- [Configuración](Configuracion.md) · [API REST](API-REST.md) · [MQTT y WebSocket](MQTT-y-WebSocket.md)
- [Seguridad](Seguridad.md) · [Guía de desarrollo](Guia-de-desarrollo.md)
