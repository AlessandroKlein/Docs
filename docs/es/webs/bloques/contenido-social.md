# Bloques de contenido y social

Este grupo cubre la creación y consumo de contenido, la comunidad y la
comunicación: **blog**, **comments**, **social**, **chat**, **chat-ia** y
**academia**. Todos montan sus rutas bajo el prefijo
`/api/v1/blocks/<id>` y se configuran desde la UI (el `settings` del `block.json`
genera un formulario automático).

---

## 1. `blog` — Blog / CMS

Publicaciones (posts) con categorías y etiquetas. Categoría `contenido`, slot
`content`.

### Rutas (`/api/v1/blocks/blog`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/posts` | — | Lista publicaciones (paginadas, `?page=`). |
| GET | `/posts/:id` | — | Detalle de una publicación. |
| POST | `/posts` | ✅ | Crea una publicación. |
| PATCH | `/posts/:id` | ✅ | Edita una publicación. |
| DELETE | `/posts/:id` | ✅ | Elimina una publicación. |
| GET | `/categories` | — | Lista categorías. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `postsPerPage` | number | `10` | Publicaciones por página. |
| `showAuthor` | boolean | `true` | Mostrar autor. |
| `showDate` | boolean | `true` | Mostrar fecha. |
| `defaultCategory` | string | `""` | Categoría por defecto. |
| `allowComments` | boolean | `true` | Permitir comentarios. |

---

## 2. `comments` — Comentarios y valoraciones

Reseñas con estrellas sobre cualquier recurso. Categoría `social`, slot
`content`.

### Rutas (`/api/v1/blocks/comments`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/comments` | — | Lista comentarios (más recientes primero). |
| POST | `/comments` | ✅ | Crea un comentario/valoración. |
| PATCH | `/comments/:id` | ✅ | Edita un comentario. |
| DELETE | `/comments/:id` | ✅ | Elimina un comentario. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `enabled` | boolean | `true` | Habilitar comentarios. |
| `allowAnonymous` | boolean | `true` | Permitir comentarios anónimos. |
| `defaultRating` | integer (1–5) | `5` | Valoración por defecto. |
| `maxRating` | integer (1–5) | `5` | Valoración máxima. |
| `moderation` | enum | `none` | `none` / `basic`. |

---

## 3. `social` — Red social

Mini red social con publicaciones, multimedia y likes. Categoría `social`, slot
`content`.

### Rutas (`/api/v1/blocks/social`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/posts` | — | Feed público (últimos 100, descendente). |
| POST | `/posts` | ✅ | Crea una publicación (recorta a `maxPostLength`). |
| PATCH | `/posts/:id/like` | ✅ | Incrementa `likes`. |
| DELETE | `/posts/:id` | ✅ | Elimina una publicación. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `enabled` | boolean | `true` | Habilitado. |
| `allowImages` | boolean | `true` | Permitir imágenes. |
| `allowVideos` | boolean | `true` | Permitir videos. |
| `maxPostLength` | integer | `500` | Largo máximo de publicación. |
| `moderation` | enum | `basic` | `none` / `basic`. |

---

## 4. `chat` — Chat en tiempo real

Mensajería entre usuarios organizada en **salas** (`room`). Categoría
`comunicacion`, slot `content`. Emite el evento `chat.message` (se puede escuchar
por WebSocket).

### Rutas (`/api/v1/blocks/chat`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/messages` | — | Mensajes de una sala (`?room=`). |
| GET | `/rooms` | — | Salas disponibles (valores distintos de `to`). |
| POST | `/messages` | ✅ | Crea un mensaje y emite `chat.message`. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `room` | string | `general` | Sala por defecto. |
| `allowCommunities` | boolean | `false` | Permitir comunidades. |
| `moderation` | enum | `none` | `none` / `basic`. |
| `maxMessageLength` | number | `500` | Longitud máxima del mensaje. |

---

## 5. `chat-ia` — Chat con IA

Asistente conversacional con proveedor configurable. Categoría `comunicacion`,
slot `content`. Para `ollama` intenta un `POST {ollamaUrl}/api/chat`; para el
resto de proveedores devuelve una respuesta simulada explícita (la integración
real se delega al provider registry de IA, ver [integraciones](../integraciones.md)).

### Rutas (`/api/v1/blocks/chat-ia`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/config` | — | Proveedor/modelo activo y lista de disponibles. |
| POST | `/chat` | ✅ | Envía `{message, history?}` y persiste la conversación en la sala. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `provider` | enum | `openai` | `openai` / `anthropic` / `gemini` / `ollama`. |
| `model` | string | `gpt-4o-mini` | Modelo a usar. |
| `apiKey` | string (password) | — | API key del proveedor. |
| `ollamaUrl` | string | `http://localhost:11434` | URL de Ollama. |
| `room` | string | `general` | Sala donde se persiste la conversación. |

---

## 6. `academia` — Academia / LMS

Cursos, lecciones y progreso de inscripciones. Categoría `educacion`, slot
`content`.

### Rutas (`/api/v1/blocks/academia`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/courses` | — | Lista cursos. |
| POST | `/courses` | ✅ | Crea un curso (con `lessons`). |
| PATCH | `/courses/:id` | ✅ | Edita un curso. |
| DELETE | `/courses/:id` | ✅ | Elimina un curso. |
| GET | `/enrollments` | ✅ | Lista inscripciones. |
| POST | `/enrollments` | ✅ | Inscribe a un alumno (progreso inicial `0`). |
| PATCH | `/enrollments/:id` | ✅ | Actualiza `progress` (0–100) / `completedAt`. |
| DELETE | `/enrollments/:id` | ✅ | Elimina una inscripción. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `allowEnroll` | boolean | `true` | Permitir inscripción. |
| `certificateEnabled` | boolean | `false` | Certificados habilitados. |
| `passingScore` | integer (0–100) | `70` | Nota de aprobación. |
| `defaultLevel` | enum | `básico` | `básico` / `intermedio` / `avanzado`. |

---

## Ver también

- [Operaciones](./operaciones.md) — calendario, reservas, formularios, documentos
  y notificaciones.
- [Integraciones de terceros](../integraciones.md) — proveedores de IA y sus
  endpoints.
