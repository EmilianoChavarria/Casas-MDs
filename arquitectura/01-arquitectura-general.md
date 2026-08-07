# 1. Arquitectura general


### 1.1 Visión

No es un marketplace: es un **sistema propietario de gestión y venta de inventario de casas de una sola empresa**, similar a un motor de reservas hotelero (channel manager simplificado) más un frontend público tipo Airbnb.

Arquitectura: **Backend API-first (Laravel) + Frontend desacoplado (Next.js)**, comunicándose por REST/JSON sobre HTTPS. SSR/ISR en Next.js para SEO de las páginas públicas de propiedades.

```
┌─────────────────────┐        HTTPS/JSON        ┌──────────────────────┐
│   Next.js (Vercel    │ ────────────────────────▶│   Laravel 12 API      │
│   o mismo VPS)        │◀──────────────────────── │   (Nginx + PHP-FPM)   │
│  - SSR/ISR público    │                           │  - Auth (Sanctum)     │
│  - Dashboard admin    │                           │  - Controllers/Svc    │
│  - Client Components  │                           │  - Policies/FormReq   │
└──────────┬───────────┘                           └──────────┬───────────┘
           │       ▲                                            │
           │       │  wss:// (chat, notificaciones en vivo)     │
           │       │  ┌──────────────────────┐                  │
           │       └──│  Laravel Reverb (WS)  │◀── broadcast()   │
           │          │  ws.midominio.com      │                  │
           │          └──────────────────────┘                  │
           │ imágenes (URLs firmadas)                           │
           ▼                                                    ▼
   ┌───────────────┐                              ┌───────────────────────┐
   │  S3 / R2       │                              │  MySQL 8 (primaria)   │
   │  (imágenes)    │                              │  + réplica lectura    │
   └───────────────┘                              └───────────────────────┘
                                                              │
                                                              ▼
                                                     ┌───────────────────┐
                                                     │  Redis (cache/queue)│
                                                     └─────────┬──────────┘
                                                               │
                                                     ┌─────────▼──────────┐
                                                     │  Queue Workers      │
                                                     │  (Supervisor)       │
                                                     │  - Emails           │
                                                     │  - Pagos/webhooks   │
                                                     │  - Notificaciones   │
                                                     │  - Recálculo precios│
                                                     └─────────────────────┘

Servicios externos: Stripe/Mercado Pago (pagos), Resend/SES (correo),
Google Maps (geolocalización), Sentry (errores), Cloudflare (CDN/DNS/WAF)
```

### 1.2 Flujo de datos típico (reserva)

1. Cliente busca en Next.js (SSR) → llama `GET /api/v1/properties?checkin&checkout&price&amenities`.
2. Laravel consulta MySQL (con cache Redis para catálogos poco cambiantes) y responde JSON.
3. Cliente selecciona propiedad → ve calendario de disponibilidad **y precios por noche** (`GET /api/v1/properties/{id}/availability`), ya resueltos por temporada y tipo de día (sección 15).
4. Cliente pide desglose → `POST /api/v1/bookings/quote` (sin apartar fechas): noches, descuentos, promociones, cargos e impuestos.
5. Cliente reserva → `POST /api/v1/bookings` (transacción con lock sobre disponibilidad **y sobre el cupón**, si lo hay).
6. Si hay pago: se crea `PaymentIntent` en Stripe/MP, se confirma vía webhook (`POST /api/v1/webhooks/stripe`).
7. Job en cola envía correo de confirmación (Resend/SES) y notifica al admin.
8. Reserva pasa a estado `confirmed`; se actualiza disponibilidad y se **congela** el desglose de precios.

### 1.3 Flujo de datos típico (chat)

1. Huésped abre el widget de chat → `POST /api/v1/conversations` (o recupera la existente por propiedad/reserva).
2. Envía mensaje → `POST /api/v1/conversations/{id}/messages` (HTTP normal: validación, policies, rate limit).
3. Laravel persiste en MySQL y luego emite `broadcast(new MessageSent)`.
4. Reverb empuja el evento por `wss://` al canal `presence-conversation.{id}`; el admin lo ve al instante.
5. Si el destinatario no está presente en el canal, un job con delay manda correo de "mensaje sin leer".

Detalle completo en la sección 16.

### 1.4 Comunicación frontend-backend

- REST/JSON, versión de API en la URL (`/api/v1/...`).
- **WebSockets (Laravel Reverb)** para lo que debe llegar sin recargar: chat y notificaciones en vivo del dashboard. El WS es transporte, no almacenamiento — todo mensaje se persiste en MySQL antes de difundirse (sección 16).
- Auth: Sanctum con **tokens SPA (cookie-based)** para el dashboard admin (mismo dominio o subdominios) y **tokens personales (Bearer)** para clientes/app pública si se requiere movilidad futura. La misma sesión de Sanctum autoriza los canales privados vía `/broadcasting/auth`.
- CORS restringido a los dominios del frontend.
- Todas las respuestas siguen un envoltorio estándar (`data`, `meta`, `errors`) vía Laravel API Resources.

---

