# Integraciones de terceros

Guía de integración con proveedores **reales** para los bloques de Webs. Resumen
de la investigación `docs/investigacion-apis-terceros.md` del monorepo: cubre
facturación, envíos, IA, analytics, calendario, login social y notificaciones,
con foco en el **mercado argentino** y alternativas internacionales.

> Webs ya aplica un **patrón de provider registry** en el backend:
> `fastify.payments.register`, `fastify.storage.register` y
> `fastify.notifications.register`. Cada uno es una **interfaz común + adaptadores
> por proveedor + proveedor por defecto local/mock**. Esta guía sigue el mismo
> patrón para el resto de dominios.

---

## 1. Facturación electrónica (AFIP/ARCA + LATAM)

Proveedores recomendados: **AFIP/ARCA directo** (WSFE v1, control total), o un
**TSD** como **Facturante** (abstrae AFIP y arranca rápido). Internacional: SIAT
(Bolivia), SUNAT (Perú), SII (Chile), SAT/PAC (México).

| Servicio | Endpoint producción | Endpoint homologación | Auth |
| -------- | ------------------- | --------------------- | ---- |
| WSAA (login) | `https://wsaa.afip.gob.ar/ws/services/LoginCms` | `https://wsaahomo.afip.gob.ar/ws/services/LoginCms` | Certificado X.509 + `loginCms` |
| WSFE v1 (emisión) | `https://servicios1.afip.gob.ar/wsfev1/service.asmx` | `https://wswhomo.afip.gob.ar/wsfev1/service.asmx` | TA (`token`+`sign`) en header SOAP |
| WSCDC (consulta CAE) | `https://servicios1.afip.gob.ar/wscdcv1/service.asmx` | `https://wswhomo.afip.gob.ar/wscdcv1/service.asmx` | TA en header SOAP |

**Flujo AFIP:** generar un **TRA** (XML), firmarlo (CMS/PKCS#7), enviarlo a
`loginCms` → se obtiene un **TA** (~12 h) que se cachea y renueva. La emisión usa
`FECAESolicitar` (devuelve **CAE** + vencimiento); la consulta posterior usa
`ConsultarComprobante` de WSCDC.

**Facturante (TSD):** auth con `Usuario` + `Hash` + `Reseller` dentro del
request; operaciones `CrearComprobante`, `DetalleComprobante`, `ListadoComprobantes`;
**webhook** asíncrono para recibir el CAE.

> AFIP pasó a llamarse **ARCA** en octubre de 2024; los webservices siguen
> publicados bajo `afip.gob.ar`.

### Campos clave de una factura

```
{ customer, taxId, vatCondition, type (A/B/C), pointOfSale, number,
  items: [{ concepto, cantidad, importe, iva }], total, currency }
```

### Arquitectura recomendada

`fastify.invoicing.register`:

- Interfaz `InvoicingProvider { authorize(comprobante) → {cae, caefchvto};
  consultarCae(cae); listarPuntosVenta(); tiposComprobante() }`.
- Adaptadores: `AfipWsfeProvider`, `FacturanteProvider`, `MockInvoicingProvider`.
- Cross-cutting: cache de TA/CAE, reintentos, webhook de CAE y guardado del PDF en
  `fastify.storage`.

---

## 2. Envíos / cotización

Argentina: **Andreani**, **OCA**, **Correo Argentino**, y los agregadores
**Envíopack** y **Pickit**. Internacional: **Shippo**, DHL, FedEx, UPS.

| Proveedor | Endpoint base | Auth | Cotizar | Tracking |
| --------- | ------------- | ---- | ------- | -------- |
| Andreani | `https://api.andreani.com` | Bearer (`POST /login`) | `POST /v1/cotizacion` | `GET /v1/tracking/{nro}` |
| OCA | `https://webservice.oca.com.ar/...` | usuario/contraseña e-Pak + CUIT | `Tarifar_Envio_Corporativo` | `Oep_Track.asmx` |
| Envíopack | `https://api.enviopack.com` | API key (Bearer) | `POST /cotizaciones` | `GET /seguimiento` |
| Pickit | `https://apis.pickit.com.ar` | API key/token | `POST /api/cotizar` | `GET /api/rastreo` |
| Shippo | `https://api.goshippo.com` | `ShippoToken <key>` | `POST /shipments` | `GET /tracks/...` |
| DHL Express | `https://express.api.dhl.com/mydhlapi` | Basic (key:secret) | `POST /rates` | `GET /track/shipments` |
| FedEx | `https://apis.fedex.com` | OAuth2 client_credentials | `POST /rate/v1/rates/quotes` | `POST /track/v1/trackingnumbers` |
| UPS | `https://onlinetools.ups.com` | OAuth2 client_credentials | `POST /api/rating/...` | `GET /api/track/v1/details/{nro}` |

### Campos clave

- **Cotización:** origen/destino, `peso` (kg), dimensiones, `valor declarado`,
  operativa, cantidad → `{precio, tiempo, courier, id}`.
- **Tracking:** número de envío → eventos `{fecha, estado, detalle, ubicación}`.

### Arquitectura recomendada

`fastify.shipping.register` con interfaz `ShippingProvider { quote(); createShipment();
track(); cancel() }` y modelos normalizados `Quote` / `Shipment` / `TrackingEvent`.
Un `quote()` multi-courier consulta varios proveedores en paralelo y devuelve el
comparativo. Default: `MockShippingProvider`.

---

## 3. IA / chat

**OpenAI**, **Anthropic Claude**, **Google Gemini**, **Mistral**; locales
**Ollama** y **llama.cpp** (ambos exponen un endpoint OpenAI-compatible).

| Proveedor | Endpoint | Auth |
| --------- | -------- | ---- |
| OpenAI | `POST https://api.openai.com/v1/chat/completions` | `Bearer <key>` |
| Anthropic | `POST https://api.anthropic.com/v1/messages` | `x-api-key` + `anthropic-version: 2023-06-01` |
| Gemini | `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` | `?key=` / `x-goog-api-key` |
| Mistral | `POST https://api.mistral.ai/v1/chat/completions` | `Bearer <key>` |
| Ollama | `POST http://localhost:11434/api/chat` | sin auth |
| llama.cpp | `POST http://localhost:8080/v1/chat/completions` | sin auth |

### Formato común

```
{ model, temperature, max_tokens, stream,
  messages: [{ role: "system"|"user"|"assistant", content: "..." }] }
```

### Arquitectura recomendada

`fastify.ai.register` con `ChatProvider { chat({messages, options, onToken}) }` y
adaptadores por proveedor + `MockAiProvider`. Normalizar mensajes a
`{role, content}` y unificar el streaming con `onToken(delta)` para servirlo como
SSE. El `chat-ia` de Webs ya consume este patrón (ver
[chat-ia](./bloques/contenido-social.md)).

---

## 4. Analytics

**Google Analytics 4 (Measurement Protocol)**, **Plausible**, **Matomo**, **Umami**.

| Proveedor | Endpoint | Auth |
| --------- | -------- | ---- |
| GA4 | `POST https://www.google-analytics.com/mp/collect?measurement_id=G-XXXX&api_secret=...` | `measurement_id` + `api_secret` |
| Plausible | `POST https://plausible.io/api/event` | sin auth (por `domain`); stats con Bearer |
| Matomo | `GET/POST https://<tu-matomo>/matomo.php?idsite=..&rec=1` | `token_auth` (reporting) |
| Umami | `POST https://<tu-umami>/api/send` | `x-umami-api-key` |

### Arquitectura recomendada

`fastify.analytics.register` con `AnalyticsProvider { track(event, props) }`
(fire-and-forget). Los eventos se encolan y flushean en batch; se registran
eventos de negocio (`order_created`, `quote_requested`, `user_signed_up`).

---

## 5. Calendarios

**Google Calendar**, **Microsoft Graph**, **CalDAV** (abierto) y **Cal.com**
(booking).

| Proveedor | Endpoint | Auth |
| --------- | -------- | ---- |
| Google Calendar | `GET/POST https://www.googleapis.com/calendar/v3/calendars/{id}/events` | OAuth2 Bearer |
| Microsoft Graph | `GET/POST https://graph.microsoft.com/v1.0/me/events` | OAuth2 Bearer |
| CalDAV | `PROPFIND` en `.well-known/caldav`; `REPORT` / `PUT` (.ics) | Basic o Bearer |
| Cal.com | `POST https://api.cal.com/v2/bookings` | `Bearer <api_key>` |

### Arquitectura recomendada

`fastify.calendar.register` con `CalendarProvider { listEvents(); createEvent();
updateEvent(); deleteEvent(); getAvailability() }` y el modelo normalizado
`CalendarEvent { id, title, start, end, timezone, attendees, ... }`. Los tokens
OAuth2 se almacenan cifrados por usuario y se refrescan automáticamente.

---

## 6. Login social (OAuth2)

**Google**, **GitHub**, **Facebook Login**, **Apple Sign In** (obligatorio si hay
app iOS con otros logins).

| Proveedor | Autorización | Token | Userinfo |
| --------- | ------------ | ----- | -------- |
| Google | `https://accounts.google.com/o/oauth2/v2/auth` | `https://oauth2.googleapis.com/token` | `https://openidconnect.googleapis.com/v1/userinfo` |
| Facebook | `https://www.facebook.com/v21.0/dialog/oauth` | `https://graph.facebook.com/v21.0/oauth/access_token` | `https://graph.facebook.com/me?fields=id,name,email,picture` |
| Apple | `https://appleid.apple.com/auth/authorize` | `https://appleid.apple.com/auth/token` | claims en `id_token` |
| GitHub | `https://github.com/login/oauth/authorize` | `https://github.com/login/oauth/access_token` | `https://api.github.com/user` |

Flujo **authorization code + PKCE**: redirigir → callback con `code` →
intercambiar → `getUser` → mapear/crear usuario local por `(provider, providerId)`.

### Arquitectura recomendada

Interfaz `SocialAuthProvider { getAuthUrl(state); exchangeCode(code);
getUser(accessToken) }` con adaptadores por proveedor + `MockSocialProvider`. El
bloque `social-login` expone `GET /providers` y `POST /verify` (ver
[social-login](./bloques/plataforma.md)).

---

## 7. Notificaciones

**Twilio** (SMS/WhatsApp), **SendGrid** (email), **Brevo** (email+SMS),
**FCM/APNs** (push), **OneSignal** (push multi-plataforma).

| Proveedor | Endpoint | Auth |
| --------- | -------- | ---- |
| Twilio | `POST https://api.twilio.com/2010-04-01/Accounts/{Sid}/Messages.json` | Basic (`Sid:Token`) |
| SendGrid | `POST https://api.sendgrid.com/v3/mail/send` | `Bearer <API_KEY>` |
| Brevo | `POST https://api.brevo.com/v3/smtp/email` / `.../transactionalSMS/sms` | header `api-key` |
| FCM | `POST https://fcm.googleapis.com/v1/projects/{id}/messages:send` | OAuth2 Bearer |
| APNs | `POST https://api.push.apple.com/3/device/{token}` | `bearer <JWT ES256>` |
| OneSignal | `POST https://onesignal.com/api/v1/notifications` | `Basic <REST_API_KEY>` |

### Arquitectura recomendada

Ya existe `fastify.notifications.register`. Interfaz
`NotificationProvider { sendEmail(); sendSms(); sendWhatsApp(); sendPush() }`
con adaptadores + `MockNotificationProvider`. Cross-cutting: cola con reintentos,
plantillas con variables y webhooks de estado (delivered/failed). En Webs, los
bloques `twilio` y `push` registran sus proveedores en este núcleo.

---

## Patrón de provider registry

Todos los dominios comparten la misma receta:

```
interfaz.ts        → contrato del dominio (quote, track, chat, authorize, ...)
adapters/          → un archivo por proveedor (traduce JSON/SOAP/OAuth)
registry.ts        → fastify.<dominio>.register(provider, config)
provider default   → "local" o "mock" (dev sin dependencias externas)
```

- **Config por entorno:** `PROVIDER=afip|facturante|mock` y credenciales por
  adaptador.
- **Normalización** de request/response a modelos internos del dominio.
- **Secrets** en variables de entorno / gestor de secretos (nunca en código).
- **Retry + timeouts + circuit breaker** para proveedores externos; **webhooks**
  para eventos asíncronos (CAE de Facturante, estado de envío, entregas).

> Fuente: documentación oficial de cada proveedor (relevamiento octubre de 2026).
