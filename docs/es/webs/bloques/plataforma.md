# Bloques de plataforma

Este grupo cubre la infraestructura del sitio: **organizaciones** (multi-tenant),
**social-login**, **register**, **seo**, las **pasarelas de pago** y el
**almacenamiento** (`storage-s3`). Las rutas se montan bajo `/api/v1/blocks/<id>`.

---

## 1. `organizaciones` — Organizaciones y membresías

Organizaciones multi-tenant, planes de membresía y **asistencia por QR**.
Categoría `negocio`, slot `content`, permisos `organization.manage`,
`membership.manage` y `attendance.manage`. Todas sus rutas requieren sesión
(`authenticate`). Es el bloque central de la plataforma gimnasio (ver
[Gimnasio multi-tenant](../gym.md)).

### Rutas (`/api/v1/blocks/organizaciones`)

**Organizaciones**

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| GET | `/organizations` | Lista organizaciones con `memberCount`. |
| POST | `/organizations` | Crea una organización (`name`, `slug`, `plan`, `shareInventory`, …). |
| PATCH | `/organizations/:id` | Edita una organización. |
| DELETE | `/organizations/:id` | Elimina organización + sus membresías y asistencias. |

**Planes de membresía**

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| GET | `/plans` | Lista planes. |
| POST | `/plans` | Crea un plan (`name`, `price`, `durationDays`, `features`). |
| PATCH | `/plans/:id` | Edita un plan. |
| DELETE | `/plans/:id` | Elimina un plan. |

**Membresías**

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| GET | `/memberships` | Lista membresías (con `organizationName` y `userName`). |
| POST | `/memberships` | Crea una membresía (genera un `qrcode` UUID). |
| PATCH | `/memberships/:id` | Edita rol/plan/estado/`endDate`, o renueva (`renew: true`). |
| DELETE | `/memberships/:id` | Elimina una membresía y sus asistencias. |

**Asistencia por QR**

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| POST | `/attendance/scan` | Escanea `{qrcode}` y registra la asistencia. |
| GET | `/attendance` | Asistencias (`?organizationId=&membershipId=`). |
| GET | `/attendance/stats` | Conteo por día (`?organizationId=&days=`). |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `defaultPlan` | string | `free` | Plan por defecto. |
| `requireApproval` | boolean | `false` | Requerir aprobación. |
| `shareInventoryByDefault` | boolean | `false` | Compartir inventario por defecto. |

---

## 2. `social-login` — Login social

Inicio de sesión con redes sociales. Categoría `auth`, slot `content`. Modela el
contrato (proveedores + vínculo de cuentas) sin implementar aún el flujo OAuth2
completo; el flujo `authorize → callback → exchange` se documenta en
[Integraciones de terceros](../integraciones.md).

### Rutas (`/api/v1/blocks/social-login`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/providers` | — | Proveedores habilitados (sin secretos). |
| POST | `/verify` | — | Crea/recupera `SocialAccount` y vincula por email. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `providers[]` | array | google/facebook/apple/github | `{id, name, enabled, clientId, clientSecret, redirectUri}`. |
| `autoLinkByEmail` | boolean | `true` | Vincular por email automáticamente. |

---

## 3. `register` — Registro

Registro público **orquestado sobre el núcleo de auth**. Categoría `auth`, slot
`content`. No duplica el registro real (bcrypt + sesión); valida según su `mode`
y reenvía al endpoint interno `POST /api/v1/auth/register`.

### Rutas (`/api/v1/blocks/register`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/settings` | — | Modo y rol por defecto. |
| POST | `/request-invite` | — | Genera un `Invite` (token UUID, +7 días). |
| POST | `/register` | — | Registra y reenvía al auth core (preserva la cookie de sesión). |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `mode` | enum | `open` | `open` / `invite` / `both`. |
| `defaultRole` | enum | `user` | `user` / `editor`. |

---

## 4. `seo` — SEO

Genera `sitemap.xml`, `robots.txt` y metadatos. Categoría `seo` (sin slot, se
integra a nivel de servidor). El sitemap es dinámico: incluye páginas, contenido
publicado, entradas de blog y productos.

### Rutas (`/api/v1/blocks/seo`)

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| GET | `/sitemap.xml` | Sitemap XML dinámico. |
| GET | `/robots.txt` | `robots.txt` con directiva y `Sitemap:` apuntando al sitemap. |
| GET | `/meta` | Metadatos resueltos (título, descripción, OG, robots). |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `siteUrl` | string | — | URL base del sitio. |
| `metaDescription` | string | — | Descripción meta por defecto. |
| `defaultTitle` | string | — | Título por defecto. |
| `ogImageUrl` | string (image) | — | Imagen Open Graph. |
| `twitterHandle` | string | — | Usuario de Twitter/X. |
| `enableSchemaOrg` | boolean | `true` | Habilitar Schema.org. |
| `sitemapExcludePaths[]` | array | `[]` | Rutas excluidas del sitemap. |

---

## 5. Pagos — pasarelas

Las pasarelas de pago son bloques que **proveen** el servicio `payment`
(`provides: ["payment"]`, slot `payment`). No exponen rutas de negocio propias:
se registran en el núcleo `fastify.payments` implementando la interfaz
`createTransaction()` + `verifyWebhook()`, y publican una **config pública** (sin
secretos) para el cliente:

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| GET | `/public-config` | Config pública (entorno, color del botón, claves públicas). |

Los **secretos** (access token, client secret, webhook secret, API keys) se
cifran en reposo con `ENCRYPTION_KEY` (`x-ui.secret`, AES-256-GCM) y nunca se
devuelven al cliente. El webhook se verifica con HMAC-SHA256 sobre la cabecera de
firma de cada proveedor.

| Bloque | Proveedor | Prefijo | Config relevante |
| ------ | --------- | ------- | ---------------- |
| `mercadopago` | Mercado Pago | `/api/v1/blocks/mercadopago` | `publicKey`, `accessToken`, `webhookSecret`, `checkoutMode`, `primaryColor` |
| `stripe` | Stripe | `/api/v1/blocks/stripe` | `publishableKey`, `secretKey`, `webhookSecret`, `primaryColor` |
| `paypal` | PayPal | `/api/v1/blocks/paypal` | `clientId`, `clientSecret`, `webhookId`, `webhookSecret`, `environment` |
| `payway` | Payway / Decidir | `/api/v1/blocks/payway` | `siteId`, `apiKey`, `webhookSecret`, `environment` |
| `modo` | MODO (QR) | `/api/v1/blocks/modo` | `clientId`, `clientSecret`, `webhookSecret`, `environment` |
| `ebanx` | EBANX (Latam) | `/api/v1/blocks/ebanx` | `integrationKey`, `webhookSecret`, `environment` |
| `pos` | Contra entrega / POS | `/api/v1/blocks/pos` | `mode` (`cash`/`terminal`), `terminalId`, `apiKey` |

> El bloque **Mercado Pago** es el **bloque de referencia** de un pago seguro
> (tokenización de tarjeta en el cliente, idempotencia y webhook firmado). Ver
> [Bloque Mercado Pago](../guides/bloque-mercadopago.md) y
> [Pagos](../guides/pagos.md).

---

## 6. `storage-s3` — Almacenamiento de objetos

Driver de almacenamiento para **Amazon S3, Cloudflare R2 y MinIO**. No expone
rutas HTTP: registra el driver `s3` en el núcleo `fastify.storage`
(`provides: ["storage"]`) con `put`/`get`/`delete`. Se activa con
`STORAGE_DRIVER=s3`.

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `endpoint` | string | — | Endpoint (R2/MinIO). |
| `region` | string | `auto` | Región. |
| `accessKeyId` | string | — | Access Key ID. |
| `secretAccessKey` | string (secret) | — | Secret Access Key. |
| `bucket` | string | — | Bucket. |
| `publicBaseUrl` | string | — | URL pública base (opcional). |

---

## 7. Otros bloques de plataforma

Estos bloques no forman parte de los grupos anteriores pero completan la
plataforma. Están listados en el [índice](./index.md).

| Bloque | Qué hace | Nota |
| ------ | -------- | ---- |
| `twilio` | Notificaciones SMS/WhatsApp. | Registra `twilio` en `fastify.notifications` (canales `sms`/`whatsapp`). |
| `push` | Notificaciones push web. | Provee `notification`; expone `GET /public-key` (clave VAPID pública). |
| `workflows` | Automatizaciones. | `GET/POST /workflows`, `PATCH /workflows/:id`. |
| `support` | Soporte / chat en vivo. | `GET/POST /tickets`. |
| `mapa` | Tarjeta de contacto con mapa. | `GET /public-config` (dirección, teléfono, lat/lng). |
| `example` | Bloque de referencia. | `GET /` y `POST /order`; demuestra el sistema de bloques. |

---

## Ver también

- [Gimnasio multi-tenant](../gym.md) — caso de uso completo de `organizaciones`.
- [Integraciones de terceros](../integraciones.md) — proveedores y patrón de
  provider registry.
- [Crear un bloque](../guides/crear-un-bloque.md) — cómo se define `provides`,
  `requiresProviders` y los secretos `x-ui.secret`.
