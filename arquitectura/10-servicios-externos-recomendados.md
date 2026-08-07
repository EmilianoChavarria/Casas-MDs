# 10. Servicios externos recomendados


Cada servicio tiene su propio README con: para qué se usa, justificación, precio y plan gratuito para desarrollo, ruta de creación paso a paso, condiciones de contrato/costos, y configuración concreta (variables de entorno, código). Están en la carpeta `servicios/servicios/`:

| Servicio | Cuándo usarlo | Detalle |
|---|---|---|
| **VPS / Hosting (Hetzner)** | Servidor de producción del backend | [`servicios/servicios/01-hosting-vps.md`](../servicios/servicios/01-hosting-vps.md) |
| **Base de datos (MySQL)** | Self-hosted → gestionado según crecimiento | [`servicios/servicios/02-base-datos-mysql.md`](../servicios/servicios/02-base-datos-mysql.md) |
| **Cache/Colas (Redis)** | Cache de catálogos, colas de Laravel | [`servicios/servicios/03-cache-colas-redis.md`](../servicios/servicios/03-cache-colas-redis.md) |
| **Almacenamiento (Cloudflare R2)** | Imágenes de propiedades y backups | [`servicios/servicios/04-almacenamiento-r2.md`](../servicios/servicios/04-almacenamiento-r2.md) |
| **Stripe** | Pagos con tarjeta internacional | [`servicios/servicios/05-pagos-stripe.md`](../servicios/servicios/05-pagos-stripe.md) |
| **Mercado Pago** | Pagos locales MX: tarjetas nacionales, OXXO, SPEI | [`servicios/servicios/06-pagos-mercadopago.md`](../servicios/servicios/06-pagos-mercadopago.md) |
| **Correo (Resend / SES)** | Confirmaciones y notificaciones transaccionales | [`servicios/servicios/07-correo-transaccional.md`](../servicios/servicios/07-correo-transaccional.md) |
| **Google Maps** | Ubicación de propiedades, autocompletado de dirección | [`servicios/servicios/08-google-maps.md`](../servicios/servicios/08-google-maps.md) |
| **Cloudflare (DNS/CDN/WAF)** | DNS, CDN, protección DDoS, SSL | [`servicios/servicios/09-cloudflare-dns-cdn.md`](../servicios/servicios/09-cloudflare-dns-cdn.md) |
| **Sentry** | Monitoreo de errores frontend/backend | [`servicios/servicios/10-sentry-monitoreo.md`](../servicios/servicios/10-sentry-monitoreo.md) |
| **Dominio** | Registro y gestión del dominio propio | [`servicios/servicios/11-dominio.md`](../servicios/servicios/11-dominio.md) |

**Sugerencia de pago:** si el mercado objetivo es mexicano, usar Mercado Pago como primario y Stripe como secundario para turistas internacionales.

**Orden de contratación sugerido:** 1) Dominio → 2) Cloudflare (DNS) → 3) VPS (Hetzner) → 4) R2 (mismo panel de Cloudflare) → 5) Stripe/Mercado Pago → 6) Resend → 7) Google Maps → 8) Sentry. Este orden evita bloqueos (ej. no puedes verificar dominio en Resend sin tener antes el DNS en Cloudflare).

---

