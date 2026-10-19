# Categorías de bloques y submenús del catálogo

El panel **Bloques** (`/admin/blocks`) organiza el catálogo de bloques por
**categorías**. Cada bloque declara una categoría en su manifiesto
(`block.json`) mediante el campo `category`, y la UI agrupa el catálogo y lo
navega con **submenús** en la barra lateral.

---

## 1. Navegación lateral

La barra lateral de `/admin/blocks` tiene tres pestañas principales:

| Pestaña        | Contenido                                                        |
| -------------- | ---------------------------------------------------------------- |
| **Instalados** | Bloques instalados, con activar/desactivar, configurar, actualizar y eliminar. |
| **Catálogo**   | Bloques disponibles (locales + remotos) agrupados por categoría. |
| **Importar JSON** | Instalación por lotes pegando un JSON con `id`, `url` y `source`. |

Al abrir **Catálogo**, debajo de la pestaña aparecen las **categorías como
submenús** (una por cada categoría presente). Pulsar una categoría **filtra**
el catálogo para mostrar solo sus bloques; volver a pulsarla quita el filtro.

```text
Bloques
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

La categoría es un **string libre** declarado en el manifiesto del bloque
(`packages/plugin-sdk/src/manifest.ts`, campo `category`). Las categorías
usadas por los bloques incluidos en el repositorio son:

| Categoría (slug) | Nombre en UI    | Bloques de ejemplo                                  |
| ---------------- | --------------- | --------------------------------------------------- |
| `negocio`        | Negocio         | `ecommerce`, `reservas`, `calendario`, `facturacion` |
| `contenido`      | Contenido       | `blog`, `academia`, `social`                        |
| `seo`            | SEO             | `seo`, `analytics`                                  |
| `marketing`      | Marketing       | `marketing`, `afiliados`                            |
| `comunicacion`   | Comunicación    | `chat`, `chat-ia`                                   |
| `auth`           | Auth            | `register`, `social-login`                          |

Los bloques **sin categoría** definida se agrupan bajo la etiqueta **`otros`**
en la UI (`const cat = b.category ?? 'otros'`).

> El slug de categoría se muestra con la primera letra en mayúscula
> (`textTransform: capitalize`); por eso `comunicacion` aparece como
> «Comunicacion». El nombre legible (con tilde) es solo para esta tabla.

---

## 3. Declarar la categoría en un bloque

Agregá el campo `category` al `block.json` de tu bloque:

```json
{
  "name": "@webs/block-chat",
  "id": "chat",
  "version": "1.0.0",
  "type": "block",
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

---

## 4. Cómo se agrupa y filtra el catálogo

En la UI (`apps/web/src/features/blocks/BlocksPage.tsx`):

1. El catálogo disponible (local + remoto, excluyendo instalados) se reduce a
   un mapa `{ categoría: [bloques] }`.
2. La clave de agrupación es `b.category ?? 'otros'`.
3. Los submenús de la barra lateral se generan con las claves de ese mapa.
4. Al seleccionar una categoría, el catálogo filtra las entradas que no
   coinciden (`filter(([cat]) => !activeCategory || cat === activeCategory)`).

La categoría viaja en el objeto de catálogo que devuelve
`GET /api/v1/blocks/catalog` (campo `category` del manifiesto local).

---

## Ver también

- [Marketplace de bloques](./marketplace-de-bloques.md) — instalar/actualizar
  bloques desde la UI.
- [Crear un bloque](./crear-un-bloque.md) — referencia completa del manifiesto
  `block.json`.
- [Sistema de bloques](../architecture/block-system.md) — SDK, slots y ciclo de
  vida.
