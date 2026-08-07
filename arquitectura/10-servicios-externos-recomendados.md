# 10. Servicios externos recomendados


Cada servicio tiene su propia carpeta con un `.md` por sección: para qué se usa, justificación, precio y plan gratuito para desarrollo, ruta de creación paso a paso, condiciones de contrato/costos, y configuración concreta (variables de entorno, código). Ver el [índice principal](../README.md) para el link a cada sección.

| Servicio | Cuándo usarlo | Carpeta |
|---|---|---|
| **VPS / Hosting (Hetzner)** | Servidor de producción del backend | [`01-hosting-vps/`](../01-hosting-vps/) |
| **Base de datos (MySQL)** | Self-hosted → gestionado según crecimiento | [`02-base-datos-mysql/`](../02-base-datos-mysql/) |
| **Cache/Colas (Redis)** | Cache de catálogos, colas de Laravel | [`03-cache-colas-redis/`](../03-cache-colas-redis/) |
| **Almacenamiento (Cloudflare R2)** | Imágenes de propiedades y backups | [`04-almacenamiento-r2/`](../04-almacenamiento-r2/) |
| **Stripe** | Pagos con tarjeta internacional | [`05-pagos-stripe/`](../05-pagos-stripe/) |
| **Mercado Pago** | Pagos locales MX: tarjetas nacionales, OXXO, SPEI | [`06-pagos-mercadopago/`](../06-pagos-mercadopago/) |
| **Correo (Resend / SES)** | Confirmaciones y notificaciones transaccionales | [`07-correo-transaccional/`](../07-correo-transaccional/) |
| **Google Maps** | Ubicación de propiedades, autocompletado de dirección | [`08-google-maps/`](../08-google-maps/) |
| **Cloudflare (DNS/CDN/WAF)** | DNS, CDN, protección DDoS, SSL | [`09-cloudflare-dns-cdn/`](../09-cloudflare-dns-cdn/) |
| **Sentry** | Monitoreo de errores frontend/backend | [`10-sentry-monitoreo/`](../10-sentry-monitoreo/) |
| **Dominio** | Registro y gestión del dominio propio | [`11-dominio/`](../11-dominio/) |

**Sugerencia de pago:** si el mercado objetivo es mexicano, usar Mercado Pago como primario y Stripe como secundario para turistas internacionales.

**Orden de contratación sugerido:** 1) Dominio → 2) Cloudflare (DNS) → 3) VPS (Hetzner) → 4) R2 (mismo panel de Cloudflare) → 5) Stripe/Mercado Pago → 6) Resend → 7) Google Maps → 8) Sentry. Este orden evita bloqueos (ej. no puedes verificar dominio en Resend sin tener antes el DNS en Cloudflare).

---

