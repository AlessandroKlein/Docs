# Categorías de plugins y submenús del catálogo

El panel **Plugins** (`/admin/plugins`) organiza el catálogo de plugins por
**categorías**. Cada plugin declara una categoría en su manifiesto
(`plugin.json`) mediante el campo `category`, y la UI agrupa el catálogo y lo
navega con **submenús** en la barra lateral.

---

## 1. Navegación lateral

La barra lateral de `/admin/plugins` tiene tres pestañas principales:

| Pestaña        | Contenido                                                        |
| -------------- | ---------------------------------------------------------------- |
| **Instalados** | Plugins instalados, con activar/desactivar, configurar, actualizar y eliminar. |
| **Catálogo**   | Plugins disponibles (locales + remotos) agrupados por categoría. |
| **Importar JSON** | Instalación por lotes pegando un JSON con `id`, `url` y `source`. |

Al abrir **Catálogo**, debajo de la pestaña aparecen las **categorías como
submenús** (una por cada categoría presente). Pulsar una categoría **filtra**
el catálogo para mostrar solo sus plugins; volver a pulsarla quita el filtro.

```text
Plugins
├─ Instalados
├─ Catálogo
│   ├─ negocio
│   ├─ contenido
│   ├─ seo
│   ├─ marketing
│   ├─ comunicacion
│   └─ auth
└─ Importar JSON
```

---

## 2. Categorías

La categoría es un **string libre** declarado en el manifiesto del plugin
(`packages/plugin-sdk/src/manifest.ts`, campo `category`). Las categorías
usadas por los plugins incluidos en el repositorio son:

| Categoría (slug) | Nombre en UI    | Plugins de ejemplo                                   |
| ---------------- | --------------- | ---------------------------------------------------- |
| `negocio`        | Negocio         | `ecommerce`, `reservas`, `calendario`, `facturacion` |
| `contenido`      | Contenido       | `blog`, `academia`, `social`                         |
| `seo`            | SEO             | `seo`, `analytics`                                   |
| `marketing`      | Marketing       | `marketing`, `afiliados`                             |
| `comunicacion`   | Comunicación    | `chat`, `chat-ia`                                    |
| `auth`           | Auth            | `register`, `social-login`                           |

Los plugins **sin categoría** definida se agrupan bajo la etiqueta **`otros`**
en la UI (`const cat = b.category ?? 'otros'`).

> El slug de categoría se muestra con la primera letra en mayúscula
> (`textTransform: capitalize`); por eso `comunicacion` aparece como
> «Comunicacion». El nombre legible (con tilde) es solo para esta tabla.

---

## 3. Declarar la categoría en un plugin

Agregá el campo `category` al `plugin.json` de tu plugin:

```json
{
  "name": "@webs/block-chat",
  "id": "chat",
  "version": "1.0.0",
  "type": "plugin",
  "displayName": { "es": "Chat", "en": "Chat" },
  "description": { "es": "Chat en tiempo real.", "en": "Real-time chat." },
  "author": "Webs",
  "apiVersion": "1.0.0",
  "category": "comunicacion",
  "slots": ["content"]
}
```

El campo es **opcional** y no está restringido a una lista fija: podés usar
cualquier slug (por ejemplo `pago`, `comunicacion`, `negocio`).

> **Nota:** el nombre npm `@webs/block-chat` y el directorio `blocks/*` se
> conservan de forma transitoria (ver [Sistema de plugins](../architecture/plugin-system.md),
> sección «Estado transitorio del rename»).

---

## 4. Cómo se agrupa y filtra el catálogo

En la UI (`apps/web/src/features/plugins/PluginsPage.tsx`):

1. El catálogo disponible (local + remoto, excluyendo instalados) se reduce a
   un mapa `{ categoría: [plugins] }`.
2. La clave de agrupación es `b.category ?? 'otros'`.
3. Los submenús de la barra lateral se generan con las claves de ese mapa.
4. Al seleccionar una categoría, el catálogo filtra las entradas que no
   coinciden (`filter(([cat]) => !activeCategory || cat === activeCategory)`).

La categoría viaja en el objeto de catálogo que devuelve
`GET /api/v1/plugins/catalog` (campo `category` del manifiesto local).

---

## Ver también

- [Marketplace de plugins](./marketplace-de-plugins.md) — instalar/actualizar
  plugins desde la UI.
- [Crear un plugin](./crear-un-plugin.md) — referencia completa del manifiesto
  `plugin.json`.
- [Sistema de plugins](../architecture/plugin-system.md) — SDK, slots y ciclo de
  vida.
