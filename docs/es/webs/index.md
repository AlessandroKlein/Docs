# Documentación de la plataforma Webs

Bienvenido a la documentación técnica de **Webs**, una plataforma web modular, escalable y multitenant construida con **Node.js + Fastify + React** y organizada como monorepo pnpm.

Este directorio describe la *arquitectura* del sistema. Para guías operativas (crear un bloque, desplegar, contribuir) se enlazan los documentos correspondientes en cada sección.

## Índice de contenidos

### Arquitectura

| Documento | Contenido |
|-----------|-----------|
| [Visión general](architecture/overview.md) | Qué es Webs, objetivos, decisiones de stack y mapa de componentes. |
| [Monorepo](architecture/monorepo.md) | Estructura de pnpm workspaces, convenciones `@webs/*` y flujo de trabajo. |
| [Sistema de bloques](architecture/block-system.md) | **Documento central**: manifiesto `block.json`, SDK, slots/placements, loader, registro de cliente y ciclo de vida. |
| [Autenticación y RBAC](architecture/auth-rbac.md) | JWT de acceso + refresh token opaco, flujo de sesión y permisos por rol. |
| [Configuración y temas](architecture/settings-theming.md) | Secciones de ajustes, formularios auto-generados (SchemaForm) y sistema de temas. |
| [Internacionalización](architecture/i18n.md) | i18next, locales base, sobrescritura desde UI y traducciones de bloques. |
| [Escalabilidad](architecture/scalability.md) | API stateless, Redis, PostgreSQL, contenedores, rate limiting y observabilidad. |

### Registros de decisiones de arquitectura (ADR)

| ADR | Decisión |
|-----|----------|
| [ADR-0001 — Stack](adr/0001-stack.md) | Fastify + React + TypeScript + Prisma. |
| [ADR-0002 — Monorepo](adr/0002-monorepo.md) | Monorepo con pnpm workspaces. |
| [ADR-0003 — Sistema de bloques](adr/0003-block-system.md) | Manifiesto + SDK + slots. |
| [ADR-0004 — Autenticación](adr/0004-auth.md) | JWT + refresh token opaco + RBAC. |

### Guías

| Documento | Contenido |
|-----------|-----------|
| [Inicio rápido](guides/getting-started.md) | Arranque y primeros pasos. |
| [Configuración desde la UI](guides/configuracion-desde-ui.md) | Ajustes del sitio sin tocar código. |
| [Footer y copyright](guides/footer-y-copyright.md) | Copyright (año en curso vs rango), enlaces legales, contacto y redes sociales. |
| [Contenido](guides/contenido.md) | CRUD de artículos en `/admin/content` (draft/published/scheduled, localizados). |
| [Categorías de bloques](guides/bloques-categorias.md) | Categorías del catálogo de bloques y submenús. |
| [Marketplace de bloques](guides/marketplace-de-bloques.md) | Instalar y actualizar bloques desde la UI. |

La lista completa de guías está en el directorio `guides/` y en la navegación
lateral de la documentación.

## Convenciones de este repositorio de documentación

- Las rutas se expresan en formato POSIX relativas a la raíz del monorepo (por ejemplo, `apps/api/src/app.ts`).
- Los comandos se muestran tal como están definidos en los `package.json` reales.
- Los diagramas son texto/Markdown para que sean legibles sin herramientas externas.

### Guías y referencia de API

Además de este índice, el repositorio de documentación incluye:

- `docs/guides/` — guías operativas (arranque, crear un bloque, configuración desde UI,
  roles y permisos, despliegue).
- `docs/api/` — referencia de la API (visión general, flujo de autenticación, WebSocket).
