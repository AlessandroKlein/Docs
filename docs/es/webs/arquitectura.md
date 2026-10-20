# Arquitectura de Webs (visión general)

Este documento es la **visión general consolidada** de la plataforma **Webs**: su
estructura de monorepo, el stack tecnológico, el patrón de bloques/plugins, el
modelo de permisos y el historial de releases. Los detalles por tema están en la
sección [Arquitectura](architecture/overview.md) de la navegación.

## 1. Qué es Webs

Webs es una plataforma web **modular, escalable y multitenant**, de estilo
WordPress: un núcleo (API + panel de administración + sitio público) que se
extiende con **bloques** autocontenidos. Combina un backend
**Node.js + Fastify** con un frontend **React + Vite**, organizados como
**monorepo pnpm**.

```text
Webs (monorepo pnpm, Node ≥ 22, pnpm 11.7)
├── apps/          Aplicaciones (api, web, desktop, mobile)
├── packages/      Paquetes compartidos (@webs/*)
├── blocks/        Bloques de funcionalidad (37 + 1 de ejemplo)
├── templates/     Plantillas de sitio (JSON)
├── docker/        Dockerfiles + nginx + compose
├── scripts/       Helpers (release, backup, docs, setup-finish)
├── infra/         Notas de infraestructura (Kubernetes futuro)
└── docs/          Documentación fuente del monorepo
```

## 2. Estructura del monorepo

| Directorio | Contenido | Detalle |
| ---------- | --------- | ------- |
| `apps/api` | API REST + WebSocket | Fastify 5, `apps/api/src/app.ts` compone plugins y módulos. |
| `apps/web` | Frontend | React 19 + Vite 6; panel admin y sitio público. |
| `apps/desktop` | Companion de escritorio | App desktop (PWA/companion). |
| `apps/mobile` | App móvil | Aplicación móvil / PWA. |
| `packages/core` | `@webs/core` | Tipos, errores versionados (`ErrorCode`), constantes y utilidades. |
| `packages/config` | `@webs/config` | Configuración compartida ESLint + Prettier. |
| `packages/plugin-sdk` | `@webs/plugin-sdk` | SDK de bloques: manifiesto, hooks, slots, adaptadores de proveedores. |
| `packages/i18n` | `@webs/i18n` | Recursos base es/en/pt + fábrica i18next. |
| `packages/ui` | `@webs/ui` | Sistema de temas y tokens CSS. |
| `blocks/*` | Bloques | 38 paquetes (37 bloques + `example-block` de referencia). |
| `templates/*.json` | Plantillas | Presets de sitio: `blog`, `corporativo`, `cursos`, `gym`, `market`, `tienda`. |
| `docs/` | Documentación fuente | Guías/arquitectura/API propias del monorepo. |

Los workspaces se declaran en `pnpm-workspace.yaml` (`apps/*`, `packages/*`,
`blocks/*`); las convenciones de paquete son `@webs/<name>` y
`@webs/block-<id>`.

## 3. Stack tecnológico

| Capa | Tecnología | Versión | Notas |
| ---- | ---------- | ------- | ----- |
| Runtime | Node.js | `>= 22` | `engines.node` en el `package.json` raíz. |
| Monorepo | pnpm workspaces | `11.7.0` | Fijado con `packageManager` + Corepack. |
| API | Fastify | 5 | Plugins `@fastify/jwt`, `@fastify/cookie`, `@fastify/websocket`, `@fastify/cors`, `@fastify/helmet`, `@fastify/rate-limit`, `@fastify/swagger`. |
| ORM | Prisma | 6 | PostgreSQL por defecto; soporta SQLite y MySQL (`schema.*.prisma`). |
| Cache / pub-sub | Redis (ioredis) | 7 | Sincroniza WebSocket entre réplicas, caché y rate limiting global. |
| Colas | BullMQ | 5 | `apps/api/src/lib/queue.ts` (email, reportes, imágenes; fallback sin Redis). |
| Frontend | React + Vite | 19 / 6 | `@tanstack/react-query` + `zustand` para estado/datos. |
| i18n | i18next + react-i18next | — | Locales **es/en/pt**, sobrescritura desde la UI. |
| Tiempo real | WebSocket (`/ws`) | — | `@fastify/websocket`; canal Redis `webs:ws`. |
| Validación | Zod + JSON Schema (Ajv) | 3 / — | Zod para entorno y manifiestos; JSON Schema para API y formularios. |
| Auth | `@fastify/jwt` + `bcryptjs` | — | JWT de acceso + refresh token opaco (ver [Seguridad](seguridad.md)). |

## 4. Patrón de bloques / plugins

Cada bloque es un paquete autocontenido que declara un manifiesto **`block.json`**
y dos puntos de entrada: uno de servidor (`entry`) y uno de cliente
(`clientEntry`). No necesita escribir UI de configuración: el panel genera el
formulario a partir del esquema `settings`.

### 4.1 Manifiesto `block.json`

El contrato se define en `packages/plugin-sdk/src/manifest.ts`
(`BLOCK_API_VERSION = "1.0.0"`):

| Campo | Descripción |
| ----- | ----------- |
| `name` | Nombre de paquete (ej. `@webs/block-suscripciones`). |
| `id` | Slug único (`^[a-z0-9][a-z0-9-_]*$`). |
| `version` | SemVer del bloque. |
| `type` | Reservado a `block`. |
| `displayName` / `description` | Textos localizados por locale BCP-47. |
| `entry` / `clientEntry` | Puntos de entrada server/cliente. |
| `settings` | JSON Schema que genera el formulario de admin (con extensión `x-ui`). |
| `permissions` | Permisos que el bloque declara/requiere. |
| `slots` | Regiones donde puede ubicarse (`header`, `content`, `sidebar`, `payment`, `footer`). |
| `events` | Eventos de negocio que emite (ej. `order.created`). |
| `translations` | Traducciones propias del bloque. |
| `blockDependencies` | Bloques de los que depende (se instalan automáticamente). |
| `category` | Agrupación en el catálogo (ej. `negocio`, `social`, `soporte`). |
| `provides` / `requiresProviders` | Grupos de proveedores que registra/necesita (ej. `payment`). |
| `apiVersion` | Versión del contrato contra la que se escribió. |

### 4.2 `defineBlock` y `defineClientBlock`

```ts
// blocks/<id>/src/server.ts
import { defineBlock } from '@webs/plugin-sdk';

export default defineBlock(
  { /* ...manifesto (block.json) */ },
  {
    register: async (fastify) => {
      fastify.get('/', async () => ({ ok: true }));
    },
    hooks: {
      onInstall: async (ctx) => ctx.logger.info('instalado'),
      onUninstall: async (ctx) => {},
      onActivate: async (ctx) => {},
      onDeactivate: async (ctx) => {},
    },
  },
);
```

```ts
// blocks/<id>/src/client.ts
import { defineClientBlock } from '@webs/plugin-sdk';

export default defineClientBlock(
  { /* ...manifesto */ },
  ({ container, config, api, i18n, theme, slot, locale }) => {
    container.textContent = config.greeting ?? i18n.t('hello');
  },
);
```

El **loader** (`apps/api/src/blocks/loader.ts`) descubre los bloques disponibles,
registra sus permisos (upsert en la tabla `Permission`), instala/activa los
habilitados y resuelve dependencias y proveedores. El contrato de cliente exige
solo montar un `HTMLElement`, por lo que un bloque puede ser React, Vue o DOM
puro.

## 5. Registros de proveedores (provider registries)

La plataforma separa la *interfaz* de un proveedor de su *implementación*, de modo
que los bloques puedan enchufar pasarelas, drivers o adaptadores sin tocar el
núcleo. Los registros centrales viven en el core; los adaptadores viven en
`apps/api/src/integrations/*` o en bloques.

| Grupo | Registro / adaptador | Detalle |
| ----- | -------------------- | ------- |
| **Pagos** | `modules/payments/registry.ts` | `registerPaymentProvider` / `getPaymentProvider` / `listPaymentProviders`; endpoints genéricos de checkout y webhook. Adaptador `PaymentProviderAdapter` (Mercado Pago, Stripe, PayPal, Payway, MODO, EBANX, POS). |
| **Almacenamiento** | `lib/storage.ts` + `plugins/storage.ts` | `registerStorageDriver` / `getStorageDriver`; driver `local` por defecto; bloques S3/R2/MinIO registran drivers extra (`StorageDriver`). |
| **Notificaciones** | `modules/notifications/registry.ts` | `registerNotificationProvider` / `getNotificationProvider`; SMS/WhatsApp/Push (Twilio, push). |
| **Envíos** | `integrations/shipping.ts` | `quoteShipping` con `andreani` / `oca` / `dhl` y fallback a tarifas locales. |
| **IA** | `integrations/ai.ts` | Chat con IA, proveedor conmutable por configuración. |
| **Analytics** | `integrations/analytics.ts` | `internal` / `google-analytics` (GA4) / `plausible` / `matomo`. |
| **Calendario** | `integrations/calendar.ts` | Google Calendar / CalDAV para eventos externos. |
| **Fiscal (tax)** | `integrations/tax.ts` | Emisión de factura con CAE: `afip` / `facturante` / `tsd`. |

El catálogo de bloques usa los campos `provides` / `requiresProviders` para saber
qué bloque cubre cada grupo (`getProviders` en `apps/api/src/blocks/loader.ts`).

## 6. Sistema de permisos y roles

La autorización es **RBAC** (roles + permisos), aplicada con dos decoradores
Fastify (`apps/api/src/plugins/auth.ts`):

- `fastify.authenticate` — exige un token válido (JWT de acceso o API token con
  scopes verificados contra la BD).
- `fastify.authorize(permission)` — exige que el permiso esté en
  `request.user.permissions`.

Características:

- **Roles de sistema** (`super_admin`, `admin`, `editor`, `user`) que no se pueden
  editar ni eliminar; el resto son roles personalizados (CRUD en
  `modules/roles`).
- **Permisos por clave** (ej. `setting.read`, `setting.write`, `content.*`), ligados
  a roles vía `RolePermission`.
- **Permisos de bloques**: cada bloque declara sus propios permisos en `block.json`
  y el loader los registra automáticamente en `Permission`.
- **API tokens (JWT)** con `scopes` acotados y revocables, con rate limiting propio
  (`lib/token-rate-limit.ts`).
- **Sesiones** en PostgreSQL (refresh token hasheado), compartidas entre réplicas.

## 7. Releases versionados (v1.35 → v1.51)

El sistema se versiona con **SemVer** (`VERSIONING.md`) y se publica como tag de
Git. La versión actual se expone en `apps/api/src/modules/system.ts`
(`SYSTEM_VERSION = "1.51.0"`) y en `GET /api/v1/system/version`.

| Versión | Hito principal |
| ------- | -------------- |
| v1.35.0 | Pulido del admin: navegación por categorías, editor Tiptap, cambio de contraseña. |
| v1.36.0 | Fase 1 de gimnasio multi-tenant (organizaciones, membresías, asistencia QR, inventario). |
| v1.37.0 | Desarrollo de 20 bloques (blog, social, chat, ecommerce, facturación, analytics…). |
| v1.38.0 | Panel de administración para los bloques. |
| v1.39.0 | Notificaciones con prioridad/caducidad y centro de notificaciones. |
| v1.40.0 | Fase 2 de gimnasio: rutinas de entrenador/cliente. |
| v1.41.0 | Diseño de la página de inicio (header + menú + hero). |
| v1.42.0 | i18n completa **es/en/pt**, integraciones de terceros, app móvil PWA + desktop. |
| v1.43.0 | Register en login, dashboard, permisos enterprise, sidebar de bloques. |
| v1.44.0 | Red social, chat DM, tienda completa, blog público, perfil ampliado. |
| v1.45.0 | Dashboard de usuario, form builder visual, stock por talle + inventario. |
| v1.46.0 | Configuración de bloques agrupada por categoría. |
| v1.47.0 | Red social completa (historias/feed/direct/follow), blog con paginación. |
| v1.48.0 | **Suscripciones** (planes/contratar/cancelar), checkout, carrito, facturación PDF, calendario mensual. |
| v1.49.0 | Tiempo real vía WebSocket (notificaciones y chat sin recargar). |
| v1.50.0 | **Soporte** (tickets + widget), POS/cobro, asistencia externa, editor de menú, actualizaciones + backups. |
| v1.51.0 | Editor visual del index (drag & drop + preview), inscripción pública a cursos. |

> El `CHANGELOG.md` del monorepo se mantiene por separado y puede ir a la zaga de
> los tags; el historial de tags de Git es la fuente de verdad.

## Ver también

- [Sistema de bloques](architecture/block-system.md) — SDK, slots, loader y ciclo de vida en detalle.
- [Autenticación y RBAC](architecture/auth-rbac.md) — flujo de sesión y permisos por rol.
- [Monorepo](architecture/monorepo.md) — convenciones `@webs/*` y flujo de trabajo.
- [Seguridad (hardening)](seguridad.md) — guía de endurecimiento y hallazgos.
- [Despliegue (Docker)](despliegue.md) — puesta en producción.
