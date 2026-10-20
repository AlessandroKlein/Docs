# Despliegue con Docker

Cómo levantar **Webs** en producción con Docker Compose, ejecutar la instalación,
aplicar plantillas, actualizar y sacar copias de seguridad.

## 1. Arranque rápido

```bash
# Desde la raíz del monorepo
docker compose up -d --build
# equivalente: make up
```

Luego abrí `http://localhost:5173` y seguí el **asistente de instalación**. Los
servicios y puertos definidos en `docker-compose.yml`:

| Servicio | Imagen/base | Puerto (host) | Notas |
| -------- | ----------- | ------------- | ----- |
| `postgres` | `postgres:16-alpine` | `5432` | `webs`/`webs`/`webs` (usuario/clave/BD). Healthcheck `pg_isready`. |
| `mysql` | `mysql:8.4` | `3306` | Alternativa (elegís el motor en el asistente). |
| `redis` | `redis:7-alpine` | `6379` | Cache/pub-sub/colas. Healthcheck `redis-cli ping`. |
| `api` | `docker/Dockerfile.api` | `4000` | Node 22-slim; expone `/api/v1` y `/ws`. |
| `web` | `docker/Dockerfile.web` (nginx) | `5173 → 80` | Build estático; proxea `/api`, `/ws` y `/docs` hacia `api`. |

> Antes de producción, revisá el [hardening de seguridad](seguridad.md): cambiar
> `JWT_SECRET`, contraseñas de base de datos, `CORS_ORIGIN`, `ENCRYPTION_KEY` y
> habilitar HTTPS.

## 2. Asistente de instalación

El contenedor `api` arranca en **modo instalación** (sin base de datos) y expone
el asistente. La instalación se dispara con:

```http
POST /api/v1/setup/install
Content-Type: application/json

{
  "database": { "provider": "postgresql", "host": "postgres", "port": 5432,
                "name": "webs", "user": "webs", "password": "webs" },
  "site": { "name": "Mi sitio" },
  "admin": { "email": "admin@example.com", "password": "********" },
  "smtp": { "enabled": false },
  "template": "gym"
}
```

El endpoint `install` (`apps/api/src/modules/setup/service.ts`) hace:

1. Guarda el payload en `DATA_DIR/setup-payload.json`.
2. Crea el esquema con `prisma db push --schema <engine>.prisma --skip-generate`.
3. Regenera el cliente con `prisma generate --schema <engine>.prisma`.
4. Ejecuta `scripts/setup-finish.mjs` (seed + admin + ajustes) en un proceso nuevo.
5. Marca la instalación como completada (y plantilla pendiente si se eligió).

Endpoints auxiliares del asistente:

| Endpoint | Uso |
| -------- | --- |
| `GET /api/v1/setup/status` | Estado de instalación, proveedores soportados (`postgresql`/`sqlite`/`mysql`). |
| `POST /api/v1/setup/test-db` | Prueba la conexión (`prisma db pull --print`), sin modificar nada. |
| `GET /api/v1/setup/templates` | Lista de plantillas disponibles. |

## 3. Plantillas

Al terminar la instalación podés aplicar una plantilla que preconfigura bloques,
menú y apariencia. Las plantillas son JSON en `templates/`:

| Plantilla | Enfoque |
| --------- | ------- |
| `gym` | Gimnasio multi-tenant (organizaciones, membresías, asistencia QR, rutinas). |
| `tienda` | Tienda/ecommerce con bloques reales (carrito, checkout, facturación). |
| `blog` | Sitio de contenidos. |
| `corporativo` | Sitio institucional. |
| `cursos` | Academia / LMS (cursos y lecciones). |
| `market` | Marketplace. |

## 4. Actualización

Las releases se publican como tags de Git (ver [Arquitectura](arquitectura.md#7-releases-versionados-v135-v151)).
Para actualizar una instalación existente:

```bash
git pull
docker compose up -d --build   # reconstruye imágenes con el código nuevo
docker compose logs -f api      # verificá el arranque
```

- El esquema se mantiene con `prisma db push`/`migrate`; el asistente y el build
  ya generan el cliente Prisma del motor elegido.
- Los datos viven en los volúmenes (`pgdata`, `mysqldata`, `appdata`): un
  `docker compose down -v` **borra todo** (equivale a `make reset`).

## 5. Copias de seguridad

| Endpoint | Método | Permiso | Qué hace |
| -------- | ------ | ------- | -------- |
| `/api/v1/system/backup?password=…` | `GET` | `setting.read` | Exporta un JSON con el contenido del sitio (usuarios sin `passwordHash` ni secretos 2FA, roles, ajustes, páginas, contenido, productos, pedidos, pagos, inventario, membresías, suscripciones). **Re-verifica la contraseña** del usuario. |
| `/api/v1/system/backup` | `POST` | `setting.write` | Respaldo de la base de datos vía `scripts/backup.mjs` (SQLite: copia; Postgres/MySQL: dump). Devuelve la ruta del archivo. |

El `GET` descarga un adjunto `webs-backup-<fecha>.json`; el `POST` genera el dump
en disco.

## 6. pnpm vía el runtime

El monorepo fija la versión con `packageManager: "pnpm@11.7.0"` en el
`package.json` raíz, y Node lo resuelve con **Corepack** (incluido en Node ≥ 16.9):

```bash
corepack enable
corepack prepare pnpm@11.7.0 --activate
pnpm --version
```

Los Dockerfiles hacen exactamente esto (`corepack enable && corepack prepare
pnpm@11.7.0 --activate`) antes de `pnpm install --frozen-lockfile`. En entornos
restringidos sin binario global, pnpm se puede invocar con el runtime de Node
apuntando al entry JS del paquete (`node <ruta>/pnpm/bin/pnpm.cjs …`); la API en
producción **no** necesita pnpm en runtime: ejecuta la CLI de Prisma y los scripts
(`backup.mjs`, `setup-finish.mjs`) directamente con `node`.

## 7. Variables de entorno clave

| Variable | Obligatoria | Producción |
| -------- | ----------- | ---------- |
| `NODE_ENV` | no | `production`. |
| `HOST` / `PORT` | no | `0.0.0.0` / `4000`. |
| `DATABASE_URL` | **sí** | PostgreSQL/MySQL/SQLite con credenciales fuertes. |
| `REDIS_URL` | no | Recomendado (WebSocket/caché/rate-limit global). |
| `JWT_SECRET` | sí (mín. 16) | `openssl rand -hex 32`. |
| `CORS_ORIGIN` | no | Origen(es) real(es), separados por coma. |
| `ENCRYPTION_KEY` | recomendada | `openssl rand -hex 32` (cifrado de secretos). |
| `SSL_ENABLED` / `SSL_CERT_PATH` / `SSL_KEY_PATH` | no | TLS para la API si no hay proxy. |
| `SEED_ADMIN_EMAIL` / `SEED_ADMIN_PASSWORD` | no | Credenciales del superadmin del seed. |

## Ver también

- [Guía de despliegue](guides/despliegue.md) — build, escalado horizontal, reverse proxy y Kubernetes.
- [Seguridad (hardening)](seguridad.md) — variables y protecciones obligatorias.
- [Inicio rápido](guides/getting-started.md) — arranque en local.
