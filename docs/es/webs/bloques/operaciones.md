# Bloques de operaciones

Este grupo cubre la operación diaria del negocio: **calendario**, **reservas**,
**formularios**, **documentos** y **notificaciones**. Las rutas se montan bajo
`/api/v1/blocks/<id>`.

---

## 1. `calendario` — Calendario

Calendario de turnos y actividades. Categoría `negocio`, slot `content`. El
proveedor `local` persiste en la base de datos; `google` y `caldav` están
modelados como adaptadores futuros (ver [integraciones](../integraciones.md)).

### Rutas (`/api/v1/blocks/calendario`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/events` | — | Lista eventos, con filtro por rango `?from=&to=`. |
| POST | `/events` | ✅ | Crea un evento. |
| PATCH | `/events/:id` | ✅ | Edita un evento. |
| DELETE | `/events/:id` | ✅ | Elimina un evento. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `provider` | enum | `local` | `local` / `google` / `caldav`. |
| `timezone` | string | `America/Argentina/Buenos_Aires` | Zona horaria. |
| `workingHoursStart` | string | `09:00` | Inicio de jornada. |
| `workingHoursEnd` | string | `18:00` | Fin de jornada. |
| `calendarApiKey` | string (secret) | — | API key del calendario externo. |

---

## 2. `reservas` — Reservas y turnos

Citas y reservas sobre **servicios** definidos en la configuración. Categoría
`negocio`, slot `content`. Con `provider=local`, cada reserva se refleja además
como evento del calendario local.

### Rutas (`/api/v1/blocks/reservas`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/services` | — | Servicios ofrecidos (de `settings.services`). |
| GET | `/bookings` | — | Reservas, con filtro por día `?date=`. |
| POST | `/bookings` | ✅ | Crea una reserva (y refleja evento en calendario local). |
| PATCH | `/bookings/:id` | ✅ | Cambia estado (`confirm` / `cancel`). |
| DELETE | `/bookings/:id` | ✅ | Elimina una reserva. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `provider` | enum | `local` | `local` / `google` / `caldav`. |
| `reminderEmail` | boolean | `false` | Recordatorio por email. |
| `leadTimeHours` | number | `24` | Antelación mínima (horas). |
| `cancelPolicyHours` | number | `24` | Política de cancelación (horas). |
| `services[]` | array | `[]` | Servicios: `{id, name, duration, price}`. |

---

## 3. `formularios` — Formularios

Creador de formularios y captura de respuestas. Categoría `contenido`, slot
`content`. Las respuestas son anónimas por defecto (`allowAnonymous`); las
mutaciones administrativas (crear/editar/borrar formularios) siempre requieren
sesión.

### Rutas (`/api/v1/blocks/formularios`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/forms` | — | Lista formularios. |
| POST | `/forms` | ✅ | Crea un formulario (`{title, fields}`). |
| PATCH | `/forms/:id` | ✅ | Edita un formulario. |
| DELETE | `/forms/:id` | ✅ | Elimina un formulario. |
| GET | `/forms/:id/responses` | — | Respuestas de un formulario. |
| POST | `/forms/:id/responses` | * | Envía una respuesta (`{data: {...}}`). |

\* Requiere sesión solo si `allowAnonymous=false`.

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `allowAnonymous` | boolean | `true` | Permitir respuestas anónimas. |
| `successMessage` | string | `¡Gracias por tu respuesta!` | Mensaje de éxito. |
| `notificationsEmail` | string (email) | — | Email de notificaciones. |

---

## 4. `documentos` — Documentos

Documentos con firma electrónica. Categoría `contenido`, slot `content`. Con
`signatureProvider` distinto de `local`, la firma se delegaría a un proveedor
externo (DocuSign, etc.); hoy la firma es local.

### Rutas (`/api/v1/blocks/documentos`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/documents` | — | Lista documentos. |
| POST | `/documents` | ✅ | Crea un documento (`{name, content, userId?}`). |
| PATCH | `/documents/:id` | ✅ | Edita nombre/contenido/`signed`. |
| DELETE | `/documents/:id` | ✅ | Elimina un documento. |
| POST | `/documents/:id/sign` | ✅ | Marca `signed: true`. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `requireSignature` | boolean | `true` | Requerir firma. |
| `auditTrail` | boolean | `false` | Registrar auditoría. |
| `signatureProvider` | enum | `local` | `local` / `docusign` / `external`. |

---

## 5. `notificaciones` — Notificaciones de la app

Centro de notificaciones in-app (bandeja de entrada). Categoría `comunicacion`,
slot `header`. Emite el evento `notification.created` al enviar una.

### Rutas (`/api/v1/blocks/notificaciones`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/inbox` | — | Notificaciones (`?userId=` o todas). |
| POST | `/send` | ✅ | Crea una notificación y emite `notification.created`. |
| PATCH | `/read/:id` | ✅ | Marca una como leída. |
| PATCH | `/read-all` | ✅ | Marca todas (o las de un `userId`) como leídas. |
| DELETE | `/:id` | ✅ | Elimina una notificación. |
| GET | `/providers` | — | Proveedores del `settings` (email/sms/whatsapp/push). |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `enabled` | boolean | `true` | Habilitado. |
| `providers[]` | array | email/sms/whatsapp/push | Proveedores con `{id, name, enabled}`. |

> Los **proveedores reales de envío** (Twilio para SMS/WhatsApp, push web) se
> registran en el núcleo `fastify.notifications` mediante bloques que implementan
> la interfaz `NotificationProvider`. Ver [Integraciones de terceros](../integraciones.md).

---

## Ver también

- [Contenido y social](./contenido-social.md) — blog, comentarios, red social,
  chat y academia.
- [Comercio](./comercio.md) — ecommerce, inventario y afiliados.
