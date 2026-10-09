---
tags:
  - sema
  - frontend
---

# Frontend (dashboard web)

> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** 2026-10-08 | **Firmware:** v1.103.0

El dashboard de SEMA es HTML+CSS+JavaScript **embebido en el firmware** (constantes
`PROGMEM` en `src/core/web/HttpServer.cpp`), sin dependencias externas en tiempo de
ejecución: no hay `data/`, ni LittleFS para la web, ni CDN (v1.102.0).

## 1. Cómo está construido

| Componente | Ubicación | Detalle |
|------------|-----------|---------|
| HTML del dashboard | `HttpServer.cpp:171` (`kIndexHtml`) | Estructura del panel + tarjetas Gridstack |
| HTML de login | `HttpServer.cpp:21` (`kLoginHtml`) | Formulario de una contraseña |
| CSS base | `HttpServer.cpp:589` (`kBaseCss`) | Tema oscuro/claro y componentes |
| Navegación | `HttpServer.cpp:646` (`kNav`) | Barra de navegación común entre páginas |
| JS de tema | `HttpServer.cpp:666` (`kThemeJs`) | `toggleTheme()` + persistencia |
| JS de idioma | `HttpServer.cpp:673` (`kI18nJs`) | `applyLang()` / `toggleLang()` |
| Cuerpos de páginas de config | `HttpServer.cpp:715+` (`const String body = R"html(…)`) | Un cuerpo por página |
| Gridstack (CSS/JS) | `src/core/web/GridstackAssets.h` | Gzip embebido: **1 009 B** (CSS) y **22 243 B** (JS) |

Los assets de Gridstack se sirven comprimidos desde PROGMEM:

| Ruta | Contenido | Headers |
|------|-----------|---------|
| `/gridstack.min.css` | CSS gzip | `Content-Encoding: gzip`, `Cache-Control: max-age=3600` |
| `/gridstack-all.min.js` | JS gzip | `Content-Encoding: gzip`, `Cache-Control: max-age=3600` |

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

- **Tarjetas Gridstack** arrastrables y redimensionables; el layout se guarda en
  `system.dashboard_layout` y se sincroniza con `GET/POST /api/v1/dashboard/layout`.
- **Estado**: nombre de estación, firmware, `hw`, uptime.
- **Mediciones en vivo**: tabla de canales con valor, unidad y calidad.
- **Histórico**: últimas mediciones y **gráficos** sobre `<canvas>` (series, ejes y
  rangos; sin librerías externas).
- **Configuración**: formularios por sección (red, seguridad, sistema, viento,
  sensores) en lugar de JSON crudo.

## 4. Tema e idioma

| Aspecto | Implementación |
|---------|----------------|
| Tema oscuro (default) | Clase ausente en `<body>`; paleta estilo GitHub (`#0d1117` fondo, `#e6edf3` texto, `#30363d` bordes, acento `#1f6feb`) |
| Tema claro | Clase `.light` en `<body>` (`toggleTheme()`) |
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

1. Editar las constantes `PROGMEM` en `src/core/web/HttpServer.cpp` (HTML, CSS, JS) o
   `GridstackAssets.h` si se actualiza la librería.
2. Si hay que regenerar los assets gzip, el repo incluye `gen_gridstack.py`.
3. Compilar y flashear (`pio run -e esp32doit-devkit-v1 -t upload`).
4. Los assets se sirven con `Cache-Control: max-age=3600`: al probar cambios de CSS/JS
   hay que forzar recarga (Ctrl+F5).

> **Regla del proyecto**: la interfaz no debe requerir cambiar el firmware para
> configurar la estación. Si algo es configurable, va a un formulario y a la API, no a
> una constante.

---

## Ver también

- [Configuración](Configuracion.md) · [API REST](API-REST.md) · [MQTT y WebSocket](MQTT-y-WebSocket.md)
- [Seguridad](Seguridad.md) · [Guía de desarrollo](Guia-de-desarrollo.md)
