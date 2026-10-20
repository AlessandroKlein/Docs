# Gimnasio multi-tenant

La plataforma **Webs** permite montar un **gimnasio (o cadena de gimnasios)**
multi-tenant usando el bloque `organizaciones` como núcleo y una serie de bloques
complementarios. La plantilla `gym` arranca todo de una vez.

## Qué incluye

| Bloque | Rol en el gimnasio |
| ------ | ------------------ |
| `organizaciones` | Sedes/organizaciones, planes de membresía y asistencia por QR. |
| `reservas` | Clases y turnos (servicios con duración y precio). |
| `calendario` | Agenda de clases y actividades. |
| `ecommerce` + `carrito` + `mercadopago` | Tienda (suplementos, merch) y cobro. |
| `facturacion` | Emisión de facturas (CAE). |
| `inventario` | Stock de productos (con soporte por organización). |
| `academia` | Rutinas y cursos (LMS). |
| `chat` + `social` | Comunidad y mensajería. |
| `analytics` + `notificaciones` + `support` | Métricas, avisos y soporte. |

---

## 1. Organizaciones (sedes)

El bloque `organizaciones` implementa el **multi-tenancy**: cada sede es una
fila en `organization` con `name`, `slug`, `plan`, `active`, `shareInventory` y
`logoUrl`.

```text
POST /api/v1/blocks/organizaciones/organizations
{ "name": "Sede Palermo", "slug": "palermo", "plan": "pro", "active": true }
```

- `shareInventory` controla si la sede **comparte inventario** con otras (ver
  sección 4).
- `GET /organizations` devuelve cada sede con su `memberCount` (conteo de
  membresías activas).
- `DELETE /organizations/:id` elimina en cascada las membresías y los registros
  de asistencia de esa sede.

---

## 2. Planes de membresía

Los planes se definen en `membershipPlan` y se administran con
`/api/v1/blocks/organizaciones/plans`:

```json
{
  "name": "Full",
  "price": 25000,
  "durationDays": 30,
  "features": ["acceso 24hs", "clases ilimitadas", "nutricionista"],
  "active": true
}
```

| Ruta | Descripción |
| ---- | ----------- |
| `GET /plans` | Lista planes. |
| `POST /plans` | Crea un plan. |
| `PATCH /plans/:id` | Edita nombre, precio, duración o features. |
| `DELETE /plans/:id` | Elimina un plan. |

Una **membresía** vincula un usuario con una sede y un plan. Al crearla se genera
automáticamente un `qrcode` (UUID) único:

```json
POST /api/v1/blocks/organizaciones/memberships
{
  "organizationId": "<sede>",
  "userId": "<usuario>",
  "role": "member",
  "plan": "Full",
  "status": "active",
  "endDate": "2026-11-19"
}
```

`PATCH /memberships/:id` con `{ "renew": true }` **renueva** la membresía:
recalcula `endDate` sumando la `durationDays` del plan y vuelve a poner
`status: "active"`.

---

## 3. Asistencia por QR

Cada membresía lleva un `qrcode`. El flujo de ingreso es:

1. El socio muestra su QR (generado a partir del `qrcode` de la membresía).
2. El front de la sede hace `POST /attendance/scan` con `{ qrcode }`.
3. El bloque valida que la membresía exista, esté `active` y no haya vencido.
4. Crea un `attendanceRecord` (`method: "qr"`) y devuelve la organización.

```json
POST /api/v1/blocks/organizaciones/attendance/scan
{ "qrcode": "6f0e4c5f-..." }

→ { "ok": true, "membership": {...}, "organization": "Sede Palermo", "record": {...} }
```

Respuestas de error:

| Error | Causa |
| ----- | ----- |
| `QR vacío` | No se envió `qrcode`. |
| `Membresía no encontrada` | El QR no corresponde a ninguna membresía. |
| `Membresía <status>` | La membresía no está `active` (p. ej. `cancelled`). |
| `Membresía vencida` | `endDate` anterior a hoy. |

Métricas de asistencia:

```text
GET /attendance             → registros (filtros organizationId/membershipId)
GET /attendance/stats       → { total, byDay: {"2026-10-19": 42, ...} }
```

---

## 4. Inventario (por sede)

El bloque `inventario` soporta `organizationId` en cada ítem, lo que permite
separar (o compartir) stock entre sedes:

```json
POST /api/v1/blocks/inventario/inventory
{ "sku": "PROT-001", "name": "Proteína 1kg", "quantity": 120,
  "minQuantity": 10, "unitPrice": 18000, "organizationId": "<sede>" }
```

| Ruta | Descripción |
| ---- | ----------- |
| `GET /inventory?organizationId=` | Ítems de una sede. |
| `POST /inventory` | Crea un ítem. |
| `POST /inventory/:id/adjust` | Ajusta stock (`{delta, reason}`). |
| `GET /inventory/low` | Ítems por debajo del umbral de stock bajo. |

El ajuste `delta` puede ser positivo (entrada) o negativo (salida) y **valida que
no quede stock negativo**.

---

## 5. Aplicar la plantilla `gym`

La plantilla `templates/gym.json` instala y configura todos los bloques del
gimnasio, define el nombre/tema y crea el menú:

```bash
# UI: Plantillas → Aplicar plantilla → Gimnasio
# API:
curl -X POST http://localhost:4000/api/v1/templates/gym/apply
```

Contenido resumido de la plantilla:

```json
{
  "id": "gym",
  "name": "Gimnasio",
  "blocks": [
    { "id": "organizaciones", "config": { "multiTenant": true, "shareInventory": false } },
    { "id": "reservas", "config": { "provider": "local", "reminderEmail": true } },
    { "id": "calendario", "config": { "provider": "local" } },
    { "id": "ecommerce", "config": { "shippingProvider": "flat" } },
    { "id": "inventario", "config": { "lowStockThreshold": 5, "trackBySku": true } },
    { "id": "carrito", "config": {} },
    { "id": "mercadopago", "config": {} },
    { "id": "facturacion", "config": { "afipMode": "factura", "arcaEnvironment": "homol" } },
    { "id": "academia", "config": { "allowEnroll": true } },
    { "id": "chat", "config": { "allowCommunities": true, "moderation": "basic" } },
    { "id": "social", "config": { "allowImages": true } },
    { "id": "analytics", "config": { "provider": "internal" } },
    { "id": "notificaciones", "config": {} },
    { "id": "support", "config": {} }
  ],
  "settings": {
    "site": { "name": "Mi Gimnasio", "tagline": "Entrená sin límites" },
    "appearance": { "primaryColor": "#16a34a", "defaultTheme": "light" }
  },
  "menu": [
    { "label": "Inicio", "path": "/", "order": 0 },
    { "label": "Clases", "path": "/clases", "order": 1 },
    { "label": "Planes", "path": "/planes", "order": 2 },
    { "label": "Tienda", "path": "/tienda", "order": 3 },
    { "label": "Rutinas", "path": "/rutinas", "order": 4 },
    { "label": "Comunidad", "path": "/comunidad", "order": 5 }
  ]
}
```

> La aplicación es *best-effort*: si algún bloque no está disponible, se omite con
> una advertencia y el resto se aplica igual. Los cambios de `config` de bloques
> ya cargados toman efecto al reiniciar la API. Ver [Plantillas](guides/plantillas.md).

---

## Ver también

- [Organizaciones y membresías](./bloques/plataforma.md) — referencia completa de
  rutas y settings.
- [Comercio](./bloques/comercio.md) — ecommerce, inventario y facturación.
- [Integraciones de terceros](./integraciones.md) — facturación fiscal y pasarelas.
