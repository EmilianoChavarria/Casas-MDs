# Arquitectura Completa — Plataforma de Renta de Casas (mono-empresa)

**Stack:** Next.js (React + TS) · Laravel 12 API REST · MySQL 8 · Sanctum · S3/R2 · Redis · Docker · Nginx · GitHub Actions

---

## 1. Arquitectura general

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

## 2. Justificación tecnológica

| Tecnología | Por qué | ¿Cambiaría algo? |
|---|---|---|
| **Next.js** | SSR/ISR necesario para SEO de propiedades públicas (Google indexa mejor que un SPA puro), buen soporte de imágenes optimizadas | No, es la mejor opción para este caso público + SEO |
| **Laravel 12** | Ecosistema maduro (colas, notificaciones, policies), coincide con tu experiencia actual en `notasCreditos` | Ninguno |
| **MySQL 8** | Relacional, transacciones ACID críticas para reservas (evitar doble-booking), soporte de JSON columns si se necesita | Podrías considerar PostgreSQL por mejores índices parciales/exclusion constraints para rangos de fechas, pero MySQL 8 es suficiente con locking correcto |
| **Sanctum (elegido sobre JWT)** | Ver sección 7.1 | — |
| **S3/R2** | R2 es más barato (sin egress fees) y compatible con API S3; recomendado si el tráfico de imágenes es alto | Preferencia: **Cloudflare R2** |
| **Redis** | Cache de catálogos + colas de trabajo (necesario para no bloquear el request en emails/pagos) | Ninguno |
| **Docker + Docker Compose** | Reproducibilidad entre local/dev/prod | Ninguno |
| **GitHub Actions** | Ya lo usas en tu pipeline de `notasCreditos`, reutilizable | Ninguno |
| **VPS (Hetzner recomendado)** | Mejor relación costo/rendimiento que DigitalOcean para este tamaño de proyecto | Hetzner > DO > Hostinger |

---

## 3. Estructura del backend Laravel

```
app/
├── Console/
│   └── Commands/
├── DTOs/
│   ├── PropertyDTO.php
│   ├── BookingDTO.php
│   └── AvailabilityRangeDTO.php
├── Events/
│   ├── BookingConfirmed.php
│   ├── BookingCancelled.php
│   └── PaymentReceived.php
├── Exceptions/
│   ├── PropertyNotAvailableException.php
│   └── PaymentFailedException.php
├── Http/
│   ├── Controllers/
│   │   └── Api/V1/
│   │       ├── Admin/
│   │       │   ├── PropertyController.php
│   │       │   ├── AmenityController.php
│   │       │   ├── SeasonController.php
│   │       │   ├── PricingController.php
│   │       │   ├── BookingAdminController.php
│   │       │   ├── UserController.php
│   │       │   ├── CustomerController.php
│   │       │   └── ReportController.php
│   │       ├── Auth/
│   │       │   └── AuthController.php
│   │       └── Public/
│   │           ├── PropertySearchController.php
│   │           ├── AvailabilityController.php
│   │           ├── BookingController.php
│   │           └── PaymentWebhookController.php
│   ├── Middleware/
│   │   ├── EnsureUserIsAdmin.php
│   │   └── LogApiRequests.php
│   ├── Requests/
│   │   ├── Property/StorePropertyRequest.php
│   │   ├── Property/UpdatePropertyRequest.php
│   │   └── Booking/StoreBookingRequest.php
│   └── Resources/
│       ├── PropertyResource.php
│       ├── BookingResource.php
│       └── AvailabilityResource.php
├── Jobs/
│   ├── SendBookingConfirmationEmail.php
│   ├── ProcessPaymentWebhook.php
│   └── GenerateOccupancyReport.php
├── Listeners/
│   ├── NotifyAdminOnNewBooking.php
│   └── ReleaseHoldOnPaymentFailed.php
├── Models/
│   ├── Property.php
│   ├── PropertyImage.php
│   ├── Amenity.php
│   ├── Booking.php
│   ├── Customer.php
│   ├── Season.php
│   ├── PriceRule.php
│   ├── Availability.php
│   ├── Payment.php
│   ├── AuditLog.php
│   └── User.php
├── Notifications/
│   ├── BookingConfirmedNotification.php
│   └── BookingCancelledNotification.php
├── Policies/
│   ├── PropertyPolicy.php
│   └── BookingPolicy.php
├── Repositories/
│   ├── Contracts/
│   │   ├── PropertyRepositoryInterface.php
│   │   └── BookingRepositoryInterface.php
│   └── Eloquent/
│       ├── PropertyRepository.php
│       └── BookingRepository.php
├── Services/
│   ├── PropertyService.php
│   ├── AvailabilityService.php
│   ├── PricingService.php
│   ├── BookingService.php
│   ├── PaymentService.php
│   └── ReportService.php
├── Traits/
│   ├── HasAuditColumns.php
│   └── ApiResponder.php
└── Helpers/
    └── DateRangeHelper.php
```

**Patrón de capas:** Controller → Service → Repository → Model.
El Controller solo valida (FormRequest) y delega. El Service contiene reglas de negocio (ej. "no permitir reservar si hay traslape de fechas"). El Repository abstrae Eloquent para poder testear con mocks o cambiar de ORM si algún día fuera necesario.

Ejemplo de contrato:

```php
interface BookingRepositoryInterface
{
    public function hasOverlap(int $propertyId, Carbon $checkin, Carbon $checkout): bool;
    public function create(BookingDTO $dto): Booking;
}
```

---

## 4. Estructura del frontend Next.js (App Router)

```
src/
├── app/
│   ├── (public)/
│   │   ├── page.tsx                  # Home
│   │   ├── properties/
│   │   │   ├── page.tsx              # Listado + filtros (Server Component)
│   │   │   └── [slug]/
│   │   │       ├── page.tsx          # Detalle (SSR/ISR)
│   │   │       └── BookingWidget.tsx # Client Component
│   │   └── layout.tsx
│   ├── (admin)/
│   │   ├── dashboard/
│   │   │   ├── page.tsx
│   │   │   ├── properties/
│   │   │   ├── bookings/
│   │   │   ├── customers/
│   │   │   ├── seasons-pricing/
│   │   │   └── reports/
│   │   └── layout.tsx                # Protegido por middleware
│   ├── api/                          # (opcional) BFF routes / proxy
│   ├── layout.tsx
│   └── middleware.ts                 # Protección de rutas admin
├── components/
│   ├── ui/                           # botones, inputs, modal (design system)
│   ├── property/
│   │   ├── PropertyCard.tsx
│   │   ├── PropertyGallery.tsx
│   │   └── AvailabilityCalendar.tsx
│   └── booking/
│       └── BookingForm.tsx
├── hooks/
│   ├── useAvailability.ts
│   └── useBooking.ts
├── services/
│   ├── apiClient.ts                  # fetch/axios con interceptores
│   ├── propertyService.ts
│   └── bookingService.ts
├── store/                            # Zustand
│   ├── authStore.ts
│   └── bookingStore.ts
├── context/
│   └── AuthContext.tsx
├── types/
│   ├── property.ts
│   └── booking.ts
└── styles/
    └── globals.css                   # Tailwind
```

**Decisiones:**
- **Server Components** para listado/detalle público (SEO, menos JS al cliente).
- **Client Components** solo donde hay interactividad (calendario, formulario de reserva).
- **Zustand** sobre Redux: menor boilerplate, suficiente para el estado del admin (filtros, carrito de reserva).
- `apiClient.ts` centraliza baseURL, manejo de tokens y refresh, y parseo de errores.

---

## 5. Base de datos

### 5.1 Tablas principales (con auditoría estándar: `created_at, updated_at, created_by, updated_by, deleted_at`)

```
users            (id, name, email, password, role_id, is_active, ...)
roles            (id, name, slug)

customers        (id, first_name, last_name, email, phone, document_id, country, ...)

properties       (id, name, slug, description, address, lat, lng,
                   capacity, bedrooms, bathrooms, base_price, status, ...)

property_images  (id, property_id FK, url, is_cover, order, alt_text)

amenities        (id, name, icon, category)
property_amenity (property_id FK, amenity_id FK)   -- pivot

seasons          (id, name, start_date, end_date, priority)
price_rules      (id, property_id FK, season_id FK NULL, day_type ENUM(weekday,weekend,holiday),
                   price, min_nights)

availability     (id, property_id FK, date, status ENUM(available,blocked,booked), booking_id FK NULL)
                  -- UNIQUE (property_id, date)

bookings         (id, property_id FK, customer_id FK, checkin, checkout,
                   guests, status ENUM(pending,confirmed,cancelled,completed),
                   total_price, currency, source, ...)

payments         (id, booking_id FK, provider ENUM(stripe,mercadopago),
                   provider_ref, amount, currency, status, paid_at)

configurations   (id, key, value, type)

audit_logs       (id, auditable_type, auditable_id, action, old_values JSON,
                   new_values JSON, user_id, ip, created_at)
```

### 5.2 Relaciones clave

- `properties 1—N property_images`
- `properties N—N amenities` (pivot `property_amenity`)
- `properties 1—N availability` (una fila por día — permite bloqueos manuales y evita traslapes vía `UNIQUE(property_id, date)`)
- `bookings 1—1 payments` (o 1—N si hay pagos parciales)
- `customers 1—N bookings`
- `seasons 1—N price_rules`

### 5.3 Índices recomendados

- `properties(slug)` UNIQUE — para rutas SEO.
- `properties(status)` — filtrar activas.
- `bookings(property_id, checkin, checkout)` compuesto — validar traslapes rápido.
- `availability(property_id, date)` UNIQUE — ya mencionado, es la pieza clave anti-doble-booking.
- `bookings(status)`, `bookings(customer_id)`.
- Full-text o índice en `properties(name, address)` si se permite búsqueda libre.

**Estrategia anti doble-booking:** al crear una reserva, usar transacción + `SELECT ... FOR UPDATE` sobre las filas de `availability` del rango de fechas, o directamente intentar `INSERT` de esas fechas como `booked` y capturar el error de UNIQUE constraint si alguna ya existe. Esto es más robusto que solo validar con un `WHERE` antes del insert (condición de carrera).

---

## 6. API REST (resumen de endpoints)

### Auth
```
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
GET    /api/v1/auth/me
POST   /api/v1/auth/refresh
```

### Público
```
GET    /api/v1/properties?checkin=&checkout=&price_min=&price_max=&amenities[]=&guests=
GET    /api/v1/properties/{slug}
GET    /api/v1/properties/{id}/availability?month=2026-08
POST   /api/v1/bookings
GET    /api/v1/bookings/{id}/status
POST   /api/v1/webhooks/stripe
POST   /api/v1/webhooks/mercadopago
```

### Admin
```
GET|POST|PUT|DELETE  /api/v1/admin/properties[/{id}]
POST                  /api/v1/admin/properties/{id}/images
DELETE                /api/v1/admin/properties/{id}/images/{imageId}
GET|POST|PUT|DELETE  /api/v1/admin/amenities[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/seasons[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/price-rules[/{id}]
PATCH                 /api/v1/admin/availability/{propertyId}   # bloquear/desbloquear fechas
GET|PUT               /api/v1/admin/bookings[/{id}]
PATCH                 /api/v1/admin/bookings/{id}/confirm
PATCH                 /api/v1/admin/bookings/{id}/cancel
GET|POST|PUT|DELETE  /api/v1/admin/customers[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/users[/{id}]
GET                   /api/v1/admin/reports/occupancy
GET                   /api/v1/admin/reports/revenue
```

### Ejemplo de respuesta (`GET /api/v1/properties/{slug}`)

```json
{
  "data": {
    "id": 12,
    "name": "Casa Palmar",
    "slug": "casa-palmar",
    "description": "Casa frente al mar en Tulum...",
    "capacity": 8,
    "bedrooms": 4,
    "bathrooms": 3,
    "base_price": 2500,
    "currency": "MXN",
    "amenities": [
      { "id": 1, "name": "Alberca", "icon": "pool" },
      { "id": 2, "name": "WiFi", "icon": "wifi" }
    ],
    "images": [
      { "url": "https://cdn.example.com/1.jpg", "is_cover": true }
    ],
    "location": { "lat": 20.2114, "lng": -87.4654, "address": "Tulum, Q. Roo" }
  },
  "meta": { "currency": "MXN" }
}
```

Todos los listados usan paginación estándar de Laravel (`data`, `links`, `meta`).

---

## 7. Seguridad

### 7.1 JWT vs Sanctum

**Recomendado: Sanctum.** Para un frontend propio (Next.js, mismo dominio/subdominios) Sanctum con cookies SPA es más simple y seguro (httpOnly cookie, no expones el token a JS, protección CSRF nativa). JWT solo aportaría valor si tuvieras múltiples clientes desacoplados (apps móviles de terceros, microservicios externos) — no es el caso aquí. Si más adelante agregas app móvil nativa, puedes usar tokens personales de Sanctum (Bearer) sin migrar a JWT.

### 7.2 Checklist de seguridad

| Área | Recomendación concreta |
|---|---|
| Roles y permisos | Tabla `roles` + policies de Laravel; opcionalmente `spatie/laravel-permission` si necesitas permisos granulares por módulo |
| Rate limiting | `throttle:api` en rutas públicas (ej. 60/min), más estricto en `/auth/login` (5/min) para evitar fuerza bruta |
| CSRF | Sanctum lo maneja automático en rutas SPA con cookies; API pura con Bearer no lo requiere |
| XSS | Next.js escapa por defecto; nunca usar `dangerouslySetInnerHTML` con contenido de usuario sin sanitizar |
| SQL Injection | Eloquent/Query Builder parametrizado siempre; nunca concatenar SQL crudo |
| CORS | Configurar `config/cors.php` solo con los dominios exactos del frontend (prod y staging) |
| Validación de archivos | FormRequest con `mimes:jpg,png,webp`, `max:5120`, validar dimensiones y re-procesar con Intervention Image antes de subir a S3 (evita payloads maliciosos disfrazados de imagen) |
| Logs | Laravel logging a stack (`daily` + envío a servicio externo tipo Papertrail/Logtail) |
| Auditoría | Tabla `audit_logs` + trait `HasAuditColumns` (ya usado en tu proyecto escolar) para trazabilidad de cambios en propiedades/precios/reservas |
| Backups | Backup automático diario de MySQL (mysqldump a S3/R2) + retención 30 días; probar restauración periódicamente |
| HTTPS | Let's Encrypt (Certbot) o Cloudflare Full (Strict); forzar HTTPS en Nginx (redirect 301) y `Secure` cookies |

---

## 8. Infraestructura y despliegue en producción

### 8.1 Topología

- 1 VPS (Hetzner CX32 o similar) para Laravel + MySQL + Redis (o MySQL gestionado aparte si el presupuesto lo permite).
- Next.js puede vivir en **Vercel** (más simple, CDN global, ISR nativo) o en el mismo VPS con `pm2`/Docker si prefieres todo autoalojado — dado tu contexto de cPanel/VPS, recomiendo Vercel para el frontend y VPS solo para el backend.

### 8.2 Docker Compose (backend)

```yaml
services:
  app:
    build: ./docker/php
    volumes: ["./backend:/var/www/html"]
    depends_on: [mysql, redis]
  nginx:
    image: nginx:alpine
    ports: ["443:443", "80:80"]
    volumes:
      - ./docker/nginx:/etc/nginx/conf.d
      - ./backend:/var/www/html
      - ./certs:/etc/nginx/certs
    depends_on: [app]
  mysql:
    image: mysql:8
    environment:
      MYSQL_DATABASE: rentas
    volumes: ["mysql_data:/var/lib/mysql"]
  redis:
    image: redis:7-alpine
  queue-worker:
    build: ./docker/php
    command: php artisan queue:work --tries=3
    depends_on: [app, redis]
volumes:
  mysql_data:
```

### 8.3 Otros puntos operativos

- **SSL:** Certbot con renovación automática (cron) o terminación en Cloudflare.
- **Variables de entorno:** `.env` nunca en el repo; usar GitHub Secrets para inyectarlas en CI/CD (igual que ya haces en tu pipeline de `notasCreditos`).
- **Cron:** `php artisan schedule:run` cada minuto (reportes, limpieza de reservas `pending` expiradas).
- **Queue workers + Supervisor:** ya tienes experiencia directa con esto (Reverb/queue:work); mismo patrón aquí para `queue:work` de emails y webhooks de pago.

---

## 9. DevOps

### 9.1 Git Flow simplificado

```
main        → producción (protegida, solo vía PR + CI verde)
develop     → integración / QA
feature/*   → features individuales
hotfix/*    → fixes urgentes desde main
```

### 9.2 Ambientes

| Ambiente | Propósito | Deploy |
|---|---|---|
| Local | Docker Compose local | manual |
| Desarrollo | rama `develop`, DB de prueba | automático al hacer push a `develop` |
| QA | mirror de prod con datos anonimizados | automático al mergear a `qa` (opcional) |
| Producción | rama `main` | automático tras PR aprobado + CI verde |

### 9.3 GitHub Actions (esqueleto, reutilizando tu experiencia con FTP/CI ya construida)

```yaml
name: Deploy Backend
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: composer install && php artisan test
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /var/www/backend
            git pull origin main
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            sudo supervisorctl restart all
```

A diferencia de tu pipeline actual por FTP (cPanel shared hosting), aquí al tener VPS con acceso SSH puedes hacer deploy por `git pull` + `artisan migrate`, mucho más robusto que sincronizar archivos por FTP.

---

## 10. Servicios externos recomendados

Cada servicio tiene su propio README con: para qué se usa, justificación, ruta de creación paso a paso, condiciones de contrato/costos, y configuración concreta (variables de entorno, código). Están en la carpeta `servicios/`:

| Servicio | Cuándo usarlo | Detalle |
|---|---|---|
| **VPS / Hosting (Hetzner)** | Servidor de producción del backend | [`servicios/01-hosting-vps.md`](servicios/01-hosting-vps.md) |
| **Base de datos (MySQL)** | Self-hosted → gestionado según crecimiento | [`servicios/02-base-datos-mysql.md`](servicios/02-base-datos-mysql.md) |
| **Cache/Colas (Redis)** | Cache de catálogos, colas de Laravel | [`servicios/03-cache-colas-redis.md`](servicios/03-cache-colas-redis.md) |
| **Almacenamiento (Cloudflare R2)** | Imágenes de propiedades y backups | [`servicios/04-almacenamiento-r2.md`](servicios/04-almacenamiento-r2.md) |
| **Stripe** | Pagos con tarjeta internacional | [`servicios/05-pagos-stripe.md`](servicios/05-pagos-stripe.md) |
| **Mercado Pago** | Pagos locales MX: tarjetas nacionales, OXXO, SPEI | [`servicios/06-pagos-mercadopago.md`](servicios/06-pagos-mercadopago.md) |
| **Correo (Resend / SES)** | Confirmaciones y notificaciones transaccionales | [`servicios/07-correo-transaccional.md`](servicios/07-correo-transaccional.md) |
| **Google Maps** | Ubicación de propiedades, autocompletado de dirección | [`servicios/08-google-maps.md`](servicios/08-google-maps.md) |
| **Cloudflare (DNS/CDN/WAF)** | DNS, CDN, protección DDoS, SSL | [`servicios/09-cloudflare-dns-cdn.md`](servicios/09-cloudflare-dns-cdn.md) |
| **Sentry** | Monitoreo de errores frontend/backend | [`servicios/10-sentry-monitoreo.md`](servicios/10-sentry-monitoreo.md) |
| **Dominio** | Registro y gestión del dominio propio | [`servicios/11-dominio.md`](servicios/11-dominio.md) |

**Sugerencia de pago:** si el mercado objetivo es mexicano, usar Mercado Pago como primario y Stripe como secundario para turistas internacionales.

**Orden de contratación sugerido:** 1) Dominio → 2) Cloudflare (DNS) → 3) VPS (Hetzner) → 4) R2 (mismo panel de Cloudflare) → 5) Stripe/Mercado Pago → 6) Resend → 7) Google Maps → 8) Sentry. Este orden evita bloqueos (ej. no puedes verificar dominio en Resend sin tener antes el DNS en Cloudflare).

---

## 11. Escalabilidad (100 → 100,000 usuarios)

| Etapa | Acciones |
|---|---|
| 100–1,000 usuarios | VPS único, cache Redis de catálogos, CDN para imágenes (ya con R2/Cloudflare) |
| 1,000–10,000 | Separar MySQL a servidor dedicado, añadir réplica de lectura, cache de queries pesadas (disponibilidad, precios), queue workers escalados horizontalmente |
| 10,000–100,000 | Balanceador (Nginx/Cloudflare Load Balancer) frente a varias instancias de Laravel, MySQL con réplicas de lectura + posible sharding por región, Redis Cluster, imágenes 100% en CDN con transformaciones on-the-fly |

Puntos clave transversales:
- **Cache:** Redis para catálogos de propiedades/amenidades (invalidar en cada `update`).
- **CDN:** Cloudflare/R2 para todas las imágenes, nunca servirlas desde el propio VPS.
- **Optimización de consultas:** eager loading (evitar N+1), índices ya definidos en sección 5.
- **Escalado horizontal:** contenedores stateless para `app`, estado (sesión, cache) siempre en Redis, nunca en disco local.

---

## 12. Roadmap por fases

| Fase | Contenido |
|---|---|
| 1 | Arquitectura y setup de repos/CI/CD |
| 2 | Base de datos (migraciones + seeders) |
| 3 | Backend core (propiedades, amenidades, disponibilidad, precios) |
| 4 | Frontend público (listado, detalle, calendario) |
| 5 | Autenticación (admin + cliente) |
| 6 | Motor de reservas (anti doble-booking, estados) |
| 7 | Pagos (Stripe/MP + webhooks) |
| 8 | Dashboard admin completo + reportes |
| 9 | Hardening de seguridad + backups |
| 10 | Producción, monitoreo (Sentry), lanzamiento |

---

## 13. Estimación aproximada

| Módulo | Horas | Complejidad | Prioridad | Riesgo principal |
|---|---|---|---|---|
| Setup arquitectura + CI/CD | 16–24 | Media | Alta | Config. de entornos |
| Modelo de datos + migraciones | 12–16 | Media | Alta | Cambios tardíos de esquema |
| CRUD propiedades/amenidades/imágenes | 30–40 | Media | Alta | Manejo de imágenes/S3 |
| Motor de disponibilidad y precios | 30–40 | **Alta** | Alta | Condiciones de carrera, reglas de temporada |
| Reservas (flujo completo) | 24–32 | Alta | Alta | Doble-booking, estados inconsistentes |
| Pagos + webhooks | 24–32 | **Alta** | Alta | Idempotencia de webhooks, reconciliación |
| Frontend público (SEO/SSR) | 40–50 | Media-Alta | Alta | Rendimiento de imágenes, Core Web Vitals |
| Dashboard admin | 40–50 | Media | Media | Volumen de pantallas |
| Auth y permisos | 12–16 | Baja-Media | Alta | — |
| Reportes | 12–20 | Media | Baja-Media | Consultas agregadas costosas |
| Seguridad/hardening | 12–16 | Media | Alta | — |
| Despliegue producción | 12–16 | Media | Alta | Primer deploy real |
| **Total estimado** | **~264–352 h** | | | (~7–9 semanas a tiempo completo, una persona) |

Dependencias críticas: el motor de disponibilidad/precios debe cerrarse antes de construir reservas; pagos depende de reservas; dashboard admin puede avanzar en paralelo al frontend público una vez la API esté estable.

---

## 14. Buenas prácticas aplicadas

- **SOLID** en Services (una responsabilidad por servicio: `PricingService` no toca disponibilidad, `AvailabilityService` no calcula precios).
- **Repository Pattern** para desacoplar Eloquent de la lógica de negocio (facilita tests unitarios con mocks).
- **Service Layer** entre Controllers y Repositories — Controllers delgados.
- **DTOs** para pasar datos entre capas sin exponer el Model de Eloquent directamente a la lógica de negocio.
- **RESTful** estricto (recursos en plural, verbos HTTP correctos, códigos de estado consistentes).
- **PSR-12** vía `laravel/pint` en el pipeline de CI.
- **DRY/KISS/YAGNI:** no construir multi-tenant ni multi-idioma de propiedades hasta que exista un requerimiento real; empezar simple y extraer abstracciones solo cuando se repitan 3+ veces.
- **Clean Architecture parcial:** no es necesario un desacople extremo (hexagonal completo) para este tamaño de proyecto; el patrón Controller→Service→Repository ya da suficiente testabilidad y mantenibilidad sin sobre-ingeniería.
