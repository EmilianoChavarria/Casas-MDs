# 10. Servicios externos recomendados


Cada servicio tiene su propio README con: para qué se usa, justificación, precio y plan gratuito para desarrollo, ruta de creación paso a paso, condiciones de contrato/costos, y configuración concreta (variables de entorno, código). Están en la carpeta `servicios/`:

| Servicio | Cuándo usarlo | Detalle |
|---|---|---|
| **VPS / Hosting (Hetzner)** | Servidor de producción del backend | [`servicios/01-hosting-vps.md`](../servicios/01-hosting-vps.md) |
| **Base de datos (MySQL)** | Self-hosted → gestionado según crecimiento | [`servicios/02-base-datos-mysql.md`](../servicios/02-base-datos-mysql.md) |
| **Cache/Colas (Redis)** | Cache de catálogos, colas de Laravel | [`servicios/03-cache-colas-redis.md`](../servicios/03-cache-colas-redis.md) |
| **Almacenamiento (Cloudflare R2)** | Imágenes de propiedades y backups | [`servicios/04-almacenamiento-r2.md`](../servicios/04-almacenamiento-r2.md) |
| **Stripe** | Pagos con tarjeta internacional | [`servicios/05-pagos-stripe.md`](../servicios/05-pagos-stripe.md) |
| **Mercado Pago** | Pagos locales MX: tarjetas nacionales, OXXO, SPEI | [`servicios/06-pagos-mercadopago.md`](../servicios/06-pagos-mercadopago.md) |
| **Correo (Resend / SES)** | Confirmaciones y notificaciones transaccionales | [`servicios/07-correo-transaccional.md`](../servicios/07-correo-transaccional.md) |
| **Google Maps** | Ubicación de propiedades, autocompletado de dirección | [`servicios/08-google-maps.md`](../servicios/08-google-maps.md) |
| **Cloudflare (DNS/CDN/WAF)** | DNS, CDN, protección DDoS, SSL | [`servicios/09-cloudflare-dns-cdn.md`](../servicios/09-cloudflare-dns-cdn.md) |
| **Sentry** | Monitoreo de errores frontend/backend | [`servicios/10-sentry-monitoreo.md`](../servicios/10-sentry-monitoreo.md) |
| **Dominio** | Registro y gestión del dominio propio | [`servicios/11-dominio.md`](../servicios/11-dominio.md) |
| **WebSockets (Laravel Reverb)** | Chat huésped↔admin y notificaciones en vivo | [`servicios/12-websockets-reverb.md`](../servicios/12-websockets-reverb.md) |

**Sugerencia de pago:** si el mercado objetivo es mexicano, usar Mercado Pago como primario y Stripe como secundario para turistas internacionales.

**Orden de contratación sugerido:** 1) Dominio → 2) Cloudflare (DNS) → 3) VPS (Hetzner) → 4) R2 (mismo panel de Cloudflare) → 5) Stripe/Mercado Pago → 6) Resend → 7) Google Maps → 8) Sentry. Este orden evita bloqueos (ej. no puedes verificar dominio en Resend sin tener antes el DNS en Cloudflare).

**Reverb no se contrata:** es un paquete del propio Laravel, self-hosted en el VPS ya pagado. No añade costo recurrente ni cuenta externa — solo el subdominio `ws.midominio.com` en el DNS de Cloudflare (paso 2) y el proceso bajo Supervisor. Pusher/Ably quedan documentados como plan B, no contratados.

---

