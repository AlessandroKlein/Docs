---
tags:
  - sema
  - frontend
---

# Frontend (guía de estilo)

> **Tipo:** Guía | **Estado:** Estable | **Firmware:** v1.60.0

El dashboard web de SEMA es HTML estático embebido en el firmware
(`HttpServer::kIndexHtml` PROGMEM) + JavaScript vanilla.

## Estilo visual

- Tema oscuro estilo GitHub (`#0d1117` fondo, `#e6edf3` texto, `#30363d` bordes).
- Acento azul `#1f6feb` (botones) y `#58a6ff` (gráfico).
- Tipografía `system-ui, sans-serif`.

## Secciones del dashboard

1. **Estado** — nombre, estación, versión, uptime (auto-refresco 5 s).
2. **Sensores** — tabla de mediciones en vivo (5 s).
3. **Histórico** — tabla de las últimas 20 mediciones + **gráfico** de temperatura
   (canvas, últimas 100 muestras).
4. **Configuración** — formulario (nombre, WiFi, hostname, claves) + cerrar sesión.

## Comportamiento

- Las peticiones usan `fetch()` (mismo origen) y reciben la cookie de sesión.
- El gráfico se dibuja con `<canvas>` (sin librerías externas).
- Los timestamps se muestran como fecha local si son epoch UTC (`ts > 1000000000`),
  o como "uptime N s" si son datos antiguos.

## Login

- Formulario centrado con un único campo de contraseña.
- `POST /login` → redirige al dashboard; error → vuelve al login.

## Cómo modificar

El HTML/CSS/JS vive en `src/core/web/HttpServer.cpp` (constantes `PROGMEM`). Para
cambiar el estilo, editar `kIndexHtml` / `kLoginHtml` y recompilar.
