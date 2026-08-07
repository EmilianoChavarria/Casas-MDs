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
           │                                                    │
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
                                                     └─────────────────────┘

Servicios externos: Stripe/Mercado Pago (pagos), Resend/SES (correo),
Google Maps (geolocalización), Sentry (errores), Cloudflare (CDN/DNS/WAF)
```

### 1.2 Flujo de datos típico (reserva)

1. Cliente busca en Next.js (SSR) → llama `GET /api/v1/properties?checkin&checkout&price&amenities`.
2. Laravel consulta MySQL (con cache Redis para catálogos poco cambiantes) y responde JSON.
3. Cliente selecciona propiedad → ve calendario de disponibilidad (`GET /api/v1/properties/{id}/availability`).
4. Cliente reserva → `POST /api/v1/bookings` (transacción con lock optimista sobre disponibilidad).
5. Si hay pago: se crea `PaymentIntent` en Stripe/MP, se confirma vía webhook (`POST /api/v1/webhooks/stripe`).
6. Job en cola envía correo de confirmación (Resend/SES) y notifica al admin.
7. Reserva pasa a estado `confirmed`; se actualiza disponibilidad.

### 1.3 Comunicación frontend-backend

- REST/JSON, versión de API en la URL (`/api/v1/...`).
- Auth: Sanctum con **tokens SPA (cookie-based)** para el dashboard admin (mismo dominio o subdominios) y **tokens personales (Bearer)** para clientes/app pública si se requiere movilidad futura.
- CORS restringido a los dominios del frontend.
- Todas las respuestas siguen un envoltorio estándar (`data`, `meta`, `errors`) vía Laravel API Resources.

---

