# Bloques de comercio

Este grupo cubre la venta y el crecimiento del negocio: **ecommerce**,
**carrito**, **facturacion**, **marketing**, **analytics**, **search**,
**inventario** y **afiliados**. Las rutas se montan bajo `/api/v1/blocks/<id>`.

---

## 1. `ecommerce` — Comercio electrónico

Tienda con productos, pedidos, envíos y cupones. Categoría `negocio`, slot
`content` y `requiresProviders: ["payment"]` (necesita una pasarela de pago).
Es el bloque central de comercio: `facturacion` y `marketing` declaran
`blockDependencies: ["ecommerce"]`.

### Rutas (`/api/v1/blocks/ecommerce`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/products` | — | Productos del `settings.products`. |
| POST | `/products` | — | Crea un producto persistido. |
| GET | `/products/persisted` | — | Productos guardados en base de datos. |
| PATCH | `/products/:id` | — | Edita un producto. |
| DELETE | `/products/:id` | — | Elimina un producto. |
| GET | `/orders` | — | Lista pedidos. |
| POST | `/orders` | — | Crea un pedido desde `{items: [{id, qty}]}`. |
| PATCH | `/orders/:id` | — | Cambia `status` (`pending`/`paid`/`preparing`/`shipped`/`cancelled`). |
| POST | `/orders/:id/apply-coupon` | ✅ | Aplica un cupón y descuenta el `%` del total. |
| GET | `/shipping/quotes` | — | Cotiza envío (`?provider=&weight=&destination=`). |
| POST | `/shipments` | ✅ | Crea un envío para un pedido. |
| GET | `/shipments` | — | Lista envíos (`?orderId=`). |
| PATCH | `/shipments/:id` | ✅ | Actualiza el estado del envío. |
| GET | `/coupons` | — | Lista cupones. |
| POST | `/coupons` | ✅ | Crea un cupón (`{code, discountPercent, validUntil}`). |
| DELETE | `/coupons/:id` | ✅ | Elimina un cupón. |
| GET | `/dashboard` | — | Métricas: `revenue`, `avgTicket`, cantidad de pedidos. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `shippingProviders` | enum | `local` | Proveedor de envío (`local`/`andreani`/`oca`/`correo`/`dhl`). |
| `shippingRates[]` | array | tarifas AR | Tarifas por proveedor (`baseCost`, `costPerKg`, `days`). |
| `shippingApiKey` | string (secret) | — | API key de envíos. |
| `shippingFlat` | number | `0` | Tarifa de envío fija. |
| `products[]` | array | `[]` | Productos: `{id, title, price, stock, category}`. |

---

## 2. `carrito` — Carrito / Tienda

Catálogo de productos con carrito y checkout. Es una vista simplificada de
ecommerce: expone el catálogo definido en su propia configuración y delega el
pago a una pasarela (`requiresProviders: ["payment"]`, slot `content`).

### Rutas (`/api/v1/blocks/carrito`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/catalog` | — | Catálogo de `settings.catalog`. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `title` | string | `Tienda` | Título de la tienda. |
| `catalog[]` | array | `[]` | Productos: `{id, title, price, stock}`. |

---

## 3. `facturacion` — Facturación

Generación de facturas con emisión fiscal (CAE). Categoría `negocio`, slot
`content`, depende de `ecommerce`. El ciclo de vida es `draft → issued → paid` y
la emisión (`/issue`) hoy simula el CAE; el proveedor fiscal real se configura
con `taxProvider` (ver [integraciones](../integraciones.md)).

### Rutas (`/api/v1/blocks/facturacion`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/invoices` | — | Lista facturas (`?status=`). |
| POST | `/invoices` | ✅ | Crea una factura en `draft`. |
| PATCH | `/invoices/:id` | ✅ | Transiciona `status` (respetando el orden del ciclo). |
| POST | `/invoices/:id/issue` | ✅ | Emite la factura: setea `issued` + `cae`. |
| DELETE | `/invoices/:id` | ✅ | Elimina una factura. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `businessName` | string | — | Razón social. |
| `taxId` | string | — | CUIT/CIF. |
| `taxRate` | number | `21` | IVA (%). |
| `taxProvider` | enum | `local` | `local` / `afip` / `facturante` / `tsd`. |
| `arcaCert` | string (secret) | — | Certificado ARCA. |

---

## 4. `marketing` — Marketing

Cupones de descuento, campañas de email y métricas. Categoría `marketing`, slot
`content`, depende de `ecommerce`.

### Rutas (`/api/v1/blocks/marketing`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/coupons` | — | Lista cupones. |
| POST | `/coupons` | ✅ | Crea un cupón. |
| PATCH | `/coupons/:id` | ✅ | Edita un cupón. |
| DELETE | `/coupons/:id` | ✅ | Elimina un cupón. |
| GET | `/campaigns` | — | Lista campañas de email. |
| POST | `/campaigns` | ✅ | Crea una campaña. |
| GET | `/stats` | — | Métricas de marketing. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `defaultDiscountPercent` | number | `10` | Descuento por defecto (%). |
| `pointsPerPurchase` | number | `1` | Puntos por compra. |
| `emailProvider` | enum | `internal` | `internal` / `mailchimp` / `resend` / `sendgrid`. |
| `emailApiKey` | string (secret) | — | API key del proveedor de email. |

---

## 5. `analytics` — Analítica

Métricas de negocio, ingresos y seguimiento de visitas. Categoría `seo`, slot
`footer`.

### Rutas (`/api/v1/blocks/analytics`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| POST | `/track` | — | Registra una visita/evento. |
| GET | `/stats` | — | Estadísticas generales. |
| GET | `/summary` | — | Resumen. |
| GET | `/revenue` | — | Ingresos. |
| GET | `/top-products` | — | Productos más vendidos. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `enabled` | boolean | `true` | Registrar visitas. |
| `provider` | enum | `internal` | `internal` / `google-analytics` / `plausible` / `matomo`. |
| `trackingId` | string (secret) | — | ID de seguimiento. |
| `trackPageViews` | boolean | `true` | Registrar vistas de página. |
| `anonymizeIp` | boolean | `true` | Anonimizar IP. |

---

## 6. `search` — Búsqueda

Búsqueda unificada en contenido, publicaciones y productos. Categoría `seo`, slot
`header`.

### Rutas (`/api/v1/blocks/search`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/search` | — | Busca `?q=` y devuelve `{results: {content, posts, products}}`. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `minChars` | number | `2` | Mínimo de caracteres para buscar. |
| `maxResults` | number | `20` | Resultados por tipo. |
| `indexContent` | boolean | `true` | Indexar contenido. |
| `indexProducts` | boolean | `true` | Indexar productos. |

---

## 7. `inventario` — Inventario

Control de inventario con ajustes de stock y alertas de stock bajo. Categoría
`negocio`, slot `content`, permiso `inventory.manage`. Se vincula al stock de la
tienda vía `productId` y soporta `organizationId` (multi-tenant).

### Rutas (`/api/v1/blocks/inventario`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/inventory` | ✅ | Ítems (`?organizationId=`). |
| POST | `/inventory` | ✅ | Crea un ítem (`sku`, `quantity`, `minQuantity`, `unitPrice`, …). |
| PATCH | `/inventory/:id` | ✅ | Edita un ítem. |
| DELETE | `/inventory/:id` | ✅ | Elimina un ítem. |
| POST | `/inventory/:id/adjust` | ✅ | Ajusta stock (`{delta, reason}`), valida no quedar negativo. |
| GET | `/inventory/low` | ✅ | Ítems con stock bajo (≤ `minQuantity` o umbral). |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `lowStockThreshold` | number | `5` | Umbral de stock bajo. |
| `trackBySku` | boolean | `true` | Control por SKU. |

---

## 8. `afiliados` — Afiliados

Programa de afiliados y referidos. Categoría `marketing`, slot `content`.

### Rutas (`/api/v1/blocks/afiliados`)

| Método | Ruta | Auth | Descripción |
| ------ | ---- | ---- | ----------- |
| GET | `/affiliates` | ✅ | Lista afiliados. |
| POST | `/affiliates` | ✅ | Crea un afiliado. |
| PATCH | `/affiliates/:id` | ✅ | Edita un afiliado. |
| DELETE | `/affiliates/:id` | ✅ | Elimina un afiliado. |
| GET | `/referrals` | ✅ | Lista referidos. |
| POST | `/referrals` | ✅ | Crea un referido. |
| PATCH | `/referrals/:id` | ✅ | Edita un referido. |

### Configuración (`settings`)

| Campo | Tipo | Default | Descripción |
| ----- | ---- | ------- | ----------- |
| `defaultCommission` | number | `0` | Comisión por defecto. |
| `commissionPercent` | number | `10` | Comisión porcentual. |
| `commissionFixed` | number | `0` | Comisión fija. |
| `minWithdrawal` | number | `50` | Retiro mínimo. |
| `cookieDays` | number | `30` | Días de cookie de atribución. |

---

## Ver también

- [Plataforma](./plataforma.md) — pagos, storage y organizaciones.
- [Integraciones de terceros](../integraciones.md) — proveedores de facturación,
  envíos, analytics y notificaciones.
