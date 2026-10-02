---
tags:
  - frontend
  - diseño
---

# Frontend (guía de estilo)

> **Tipo:** Web | **Estado:** Estable | **Fecha:** 2026-10-02

Sistema de diseño compartido entre la **web local del ESP32** (`WebAssets.hpp`) y el
**dashboard central del servidor**. Vista previa interactiva en
[`docs/frontend-preview.html`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/frontend-preview.html).

## 1. Paleta de color

| Token | Valor | Uso |
|-------|-------|-----|
| `--bg` | `#0f172a` | fondo |
| `--panel` | `#1e293b` | tarjetas/paneles |
| `--panel2` | `#334155` | bordes/secundario |
| `--text` | `#e2e8f0` | texto |
| `--muted` | `#94a3b8` | texto secundario |
| `--accent` | `#22c55e` | acento (verde) |
| `--warn` | `#f59e0b` | advertencia |
| `--danger` | `#ef4444` | error/peligro |

Tema claro mediante `[data-theme="light"]` (toggle persistido en `localStorage`).

## 2. Tipografía

- **Fraunces** (serif) → títulos de marca.
- **JetBrains Mono** → datos técnicos, clases, códigos.
- **system-ui** → cuerpo de texto.

## 3. Componentes

| Componente | Clase | Uso |
|-----------|-------|-----|
| Tarjeta de sensor | `.tile` | etiqueta + valor + unidad |
| Grilla | `.grid` | `repeat(auto-fill, minmax(150px, 1fr))` |
| Estado | `.dot` (online/offline/warn/alarm) | presencia |
| Botón | `button` (+ `.secondary`, `.danger`) | acciones |
| Tabla | `.table` (responsive en móvil) | datos |
| Chip | `.chip` | selección múltiple |
| Modal | `.modal` / `.modal-box` | formularios |
| Login | `.card` | tarjeta de acceso |
| Alarma | `ul.alarms` / `.critical` | listado |

## 4. Responsive

- `max-width: 640px` en la web local.
- Grillas `auto-fill` adaptables.
- **Tablas**: scroll horizontal + tarjetas apiladas en `max-width: 640px`
  (con `data-label`).

## 5. Ejemplo de estructura

```text
Dashboard
├── header
├── main
│   ├── Temperature
│   ├── Humidity
│   ├── Soil
│   └── Irrigation
└── sidebar
```
