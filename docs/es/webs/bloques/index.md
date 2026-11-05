# Índice de plugins

La plataforma **Webs** se extiende con **plugins**: paquetes autocontenidos (con
`plugin.json`, servidor y cliente) que agregan funcionalidad al sitio. Cada plugin
expone su API bajo el prefijo `/api/v1/plugins/<id>` y se configura desde
**Plugins → Instalar/Configurar** en el panel de administración.

En el monorepo hay **38 paquetes** (`blocks/*`): 37 plugins funcionales y 1 plugin
de ejemplo. La tabla siguiente los lista con
su `id`, nombre, `category` (campo del manifiesto `plugin.json`) y una descripción
de una línea. Los plugins sin `category` declarada se agrupan en la UI bajo
`otros`.

> **Nota de naming:** el directorio físico `blocks/*`, el prefijo de permisos
> `block.*` y algunos campos internos conservan el nombre «block» de forma
> transitoria (ver [Sistema de plugins](../architecture/plugin-system.md), sección
> «Estado transitorio del rename»).

| `id` | Nombre | `category` | Qué hace |
| ---- | ------ | ---------- | -------- |
| `blog` | Blog / CMS | `contenido` | Publicaciones, categorías y etiquetas. |
| `comments` | Comentarios y valoraciones | `social` | Valoraciones con estrellas y reseñas. |
| `social` | Red social | `social` | Mini red social (publicaciones, multimedia, likes). |
| `chat` | Chat en tiempo real | `comunicacion` | Chat en tiempo real entre usuarios. |
| `chat-ia` | Chat con IA | `comunicacion` | Chat con IA (proveedor configurable). |
| `academia` | Academia / LMS | `educacion` | Cursos, lecciones y progreso. |
| `calendario` | Calendario | `negocio` | Calendario para turnos y actividades. |
| `reservas` | Reservas y turnos | `negocio` | Citas y reservas. |
| `formularios` | Formularios | `contenido` | Creador de formularios y respuestas. |
| `documentos` | Documentos | `contenido` | Documentos y firma electrónica. |
| `notificaciones` | Notificaciones de la app | `comunicacion` | Centro de notificaciones dentro de la app. |
| `ecommerce` | Comercio electrónico | `negocio` | Tienda con productos, pedidos, envíos y cupones. |
| `carrito` | Carrito / Tienda | — | Catálogo de productos con carrito y checkout. |
| `facturacion` | Facturación | `negocio` | Generación de facturas con emisión fiscal (CAE). |
| `marketing` | Marketing | `marketing` | Cupones de descuento, campañas de email y métricas. |
| `analytics` | Analítica | `seo` | Métricas de negocio, ingresos y seguimiento de visitas. |
| `search` | Búsqueda | `seo` | Búsqueda unificada en contenido, publicaciones y productos. |
| `inventario` | Inventario | `negocio` | Control de inventario con ajustes de stock y alertas de stock bajo. |
| `afiliados` | Afiliados | `marketing` | Afiliados y referidos. |
| `organizaciones` | Organizaciones y membresías | `negocio` | Organizaciones multi-tenant, planes de membresía y asistencia por QR. |
| `suscripciones` | Suscripciones | `negocio` | Planes de suscripción con vencimiento y pago. |
| `social-login` | Login social | `auth` | Inicio de sesión con redes sociales. |
| `register` | Registro | `auth` | Registro público orquestado sobre el núcleo de auth. |
| `seo` | SEO | `seo` | Sitemap, robots.txt y etiquetas meta. |
| `mercadopago` | Mercado Pago | — | Pagos con Mercado Pago (plugin de referencia). |
| `stripe` | Stripe | — | Pagos con tarjeta vía Stripe. |
| `paypal` | PayPal | — | Pagos con PayPal. |
| `payway` | Payway / Decidir | — | Pagos con Payway (Decidir 2.0). |
| `modo` | MODO | — | Pagos con MODO (QR). |
| `ebanx` | EBANX | — | Pagos con EBANX (Latam). |
| `pos` | Contra entrega / POS | — | Pagos contra entrega y lectores de tarjeta. |
| `storage-s3` | Almacenamiento S3/R2/MinIO | — | Almacenamiento de objetos (S3, R2, MinIO). |
| `twilio` | Twilio (SMS/WhatsApp) | — | Notificaciones SMS y WhatsApp vía Twilio. |
| `push` | Notificaciones push | — | Notificaciones push web (referencia). |
| `workflows` | Flujos de trabajo | — | Automatizaciones y flujos de trabajo. |
| `support` | Soporte / Chat en vivo | — | Widget flotante de chat y tickets. |
| `mapa` | Mapa / Contacto | — | Tarjeta de contacto con enlace al mapa. |
| `example` | Plugin de ejemplo | — | Plugin de referencia del sistema de plugins. |

> Los plugins `carrito`, los pasarelas de pago, `storage-s3`, `twilio`, `push`,
> `workflows`, `support`, `mapa` y `example` no declaran `category` en su
> `plugin.json`; por eso aparecen con `—` y se agrupan como `otros` en el catálogo.

## Guías por grupo

La documentación de cada plugin (qué hace, prefijo de API, rutas y configuración)
está agrupada en cuatro guías:

| Guía | Plugins |
| ---- | ------- |
| [Contenido y social](./contenido-social.md) | `blog`, `comments`, `social`, `chat`, `chat-ia`, `academia` |
| [Operaciones](./operaciones.md) | `calendario`, `reservas`, `formularios`, `documentos`, `notificaciones` |
| [Comercio](./comercio.md) | `ecommerce`, `carrito`, `facturacion`, `marketing`, `analytics`, `search`, `inventario`, `afiliados` |
| [Plataforma](./plataforma.md) | `organizaciones`, `social-login`, `register`, `seo`, pagos, `storage-s3` |

## Ver también

- [Categorías de plugins](../guides/bloques-categorias.md) — cómo se agrupa el
  catálogo por `category`.
- [Crear un plugin](../guides/crear-un-plugin.md) — referencia del manifiesto
  `plugin.json`.
- [Sistema de plugins](../architecture/plugin-system.md) — SDK, slots y ciclo de
  vida.
- [Gimnasio multi-tenant](../gym.md) — guía de la plataforma de gimnasios.
