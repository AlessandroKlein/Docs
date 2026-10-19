# Contenido (`/admin/content`)

El módulo de **contenido** permite crear, editar, ver y eliminar artículos
desde el panel de administración, en la ruta `/admin/content`. Está pensado
para contenido editorial (páginas, artículos, entradas) con títulos y cuerpos
**localizados** y un ciclo de vida de publicación.

El backend es un CRUD completo en `apps/api/src/modules/content` expuesto en
`/api/v1/content`, con validación de esquema, paginación y filtros.

---

## 1. Permisos necesarios

| Acción                      | Permiso(s)        |
| --------------------------- | ----------------- |
| Ver contenido (listar/ver)  | `content.read`    |
| Crear/editar/eliminar       | `content.write`   |
| Publicar (estado published) | `content.publish` |

> Para publicar (crear con estado `published` o cambiar un contenido a
> `published`) se exige además `content.publish`. Un usuario con
> `content.write` pero sin `content.publish` puede crear borradores y programar,
> pero no publicar.

---

## 2. Estados del ciclo de vida

Cada contenido tiene un `status` con tres valores:

| Estado       | Etiqueta UI | Significado                                                   |
| ------------ | ----------- | ------------------------------------------------------------- |
| `draft`      | Borrador    | No visible públicamente. Es el valor por defecto al crear.     |
| `published`  | Publicado   | Visible. Se registra `publishedAt` con la fecha del momento.   |
| `scheduled`  | Programado  | Estado reservado para publicación programada (aún no visible). |

Al pasar un contenido a `published`, la API guarda `publishedAt` automáticamente
(`new Date()`). El estado se cambia desde el formulario o con un `PATCH`.

---

## 3. Títulos y cuerpos localizados

Los campos `title` y `body` son **objetos localizados**: un mapa
`{ locale: texto }` con una clave por idioma habilitado (`es`, `en`, `pt`).

```json
{
  "title": { "es": "Bienvenidos", "en": "Welcome" },
  "body":  { "es": "<p>Hola</p>", "en": "<p>Hello</p>" }
}
```

- El editor de la UI guarda el texto en la clave `es` y **preserva** las demás
  claves ya existentes del contenido.
- El listado muestra el texto en `es` (o el primer valor disponible).
- El `body` se **sanitiza** contra XSS antes de guardarse
  (`sanitizeLocalized`, con `sanitize-html`).

---

## 4. Crear contenido

En `/admin/content` pulsá **Nuevo** (o usá el submenú lateral
«Nuevo contenido»). El formulario pide:

| Campo   | Descripción                                                        |
| ------- | ------------------------------------------------------------------ |
| **Slug**    | Identificador único, obligatorio. Patrón `^[a-z0-9][a-z0-9-]*$`. |
| **Título**  | Obligatorio. Se guarda como `title.es`.                           |
| **Estado**  | `draft` / `published` / `scheduled` (por defecto `draft`).         |
| **Cuerpo**  | Editor WYSIWYG (texto enriquecido), opcional.                      |

Reglas de negocio:

- `slug` y `title` son obligatorios; sin ellos la UI muestra
  «Slug y título son obligatorios».
- El `slug` es **único por tipo** de contenido (`type`, por defecto `page`).
  Duplicarlo devuelve `409 Conflict` («Ya existe contenido con ese slug»).
- El `type` por defecto es `page`.

Ejemplo de creación vía API:

```http
POST /api/v1/content
Content-Type: application/json

{
  "slug": "quienes-somos",
  "title": { "es": "Quiénes somos" },
  "body": { "es": "<p>Somos Webs.</p>" },
  "status": "draft"
}
```

---

## 5. Listar, buscar y filtrar

La pestaña «Contenido» muestra una tabla paginada con columnas: **Título**,
**Slug**, **Estado** (badge), **Actualizado** y **Acciones**.

- **Buscar**: filtra por slug (parámetro `q`, búsqueda parcial insensible a
  mayúsculas).
- **Estado**: filtra por `draft` / `published` / `scheduled` o «Todos».
- **Paginación**: `GET /api/v1/content` devuelve `items`, `total`, `page` y
  `pageSize`; la UI muestra «Mostrando X de Y resultados».

Parámetros del listado: `type`, `status`, `q`, `page`, `pageSize`.

```http
GET /api/v1/content?status=draft&q=quienes&page=0&pageSize=10
```

---

## 6. Ver, editar y eliminar

Cada fila ofrece tres acciones:

| Acción    | Comportamiento                                                   |
| --------- | ---------------------------------------------------------------- |
| **Ver**   | Abre una vista previa (modal) con título y cuerpo renderizado.    |
| **Editar**| Carga el contenido en el editor para modificar y **Guardar** (`PATCH`). |
| **Eliminar** | Pide confirmación (`ConfirmDialog`) y ejecuta `DELETE` (irreversible). |

La edición es **parcial** (`PATCH /api/v1/content/:id`): solo se envían los
campos modificados (`slug`, `title`, `body`, `status`, `seo`).

---

## 7. Referencia de API

| Método | Ruta                | Permiso        | Descripción                          |
| ------ | ------------------- | -------------- | ------------------------------------ |
| `GET`  | `/api/v1/content`   | `content.read` | Listado paginado y filtrado.         |
| `POST` | `/api/v1/content`   | `content.write`| Crear (exige `content.publish` si es `published`). |
| `GET`  | `/api/v1/content/:id` | `content.read` | Detalle por id.                      |
| `PATCH`| `/api/v1/content/:id` | `content.write` | Actualización parcial.               |
| `DELETE`| `/api/v1/content/:id` | `content.write` | Eliminar (204).                      |

Campos admitidos en crear/actualizar: `type`, `slug`, `title`, `body`, `status`
y `seo` (metadatos opcionales: `title`, `description`, `keywords`,
`ogImageUrl`, `canonicalUrl`).

---

## Ver también

- [Configuración desde la UI](./configuracion-desde-ui.md)
- [Roles y permisos](./roles-permisos.md) — asignación de `content.*`.
- [API: resumen](../api/overview.md) — convenciones de paginación y errores.
