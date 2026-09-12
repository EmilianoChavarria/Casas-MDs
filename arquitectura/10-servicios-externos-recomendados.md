# 10. Servicios externos recomendados


Cada servicio tiene su propio README con: para qué se usa, justificación, precio y plan gratuito para desarrollo, ruta de creación paso a paso, condiciones de contrato/costos, y configuración concreta (variables de entorno, código). Están en la carpeta `servicios/`:

| Servicio | Cuándo usarlo | Detalle |
|---|---|---|
| **VPS / Hosting (DigitalOcean)** | Servidor de producción del backend | [`servicios/01-hosting-vps.md`](../servicios/01-hosting-vps.md) |
| **Base de datos (MySQL)** | Self-hosted → gestionado según crecimiento | [`servicios/02-base-datos-mysql.md`](../servicios/02-base-datos-mysql.md) |
| **Cache/Colas (Redis)** | Cache de catálogos, colas de Laravel | [`servicios/03-cache-colas-redis.md`](../servicios/03-cache-colas-redis.md) |
| **Almacenamiento (Cloudflare R2)** | Imágenes de propiedades y backups | [`servicios/04-almacenamiento-r2.md`](../servicios/04-almacenamiento-r2.md) |
| **Stripe** | **Única pasarela**: tarjeta en cualquier moneda y, en México, OXXO y SPEI | [`servicios/05-pagos-stripe.md`](../servicios/05-pagos-stripe.md) |
| **Correo (Resend / SES)** | Confirmaciones y notificaciones transaccionales | [`servicios/07-correo-transaccional.md`](../servicios/07-correo-transaccional.md) |
| **Tiles de mapa (MapTiler/Stadia/Geoapify)** | Mapas públicos con Leaflet: resultados de búsqueda y detalle de propiedad | [`servicios/13-mapas-tiles.md`](../servicios/13-mapas-tiles.md) |
| **Google Maps** | **Solo** Places Autocomplete en el alta de propiedades del admin (ver sección 17.9) | [`servicios/08-google-maps.md`](../servicios/08-google-maps.md) |
| **Google OAuth** | "Continuar con Google" en registro e inicio de sesión del huésped | [`servicios/14-auth-google-oauth.md`](../servicios/14-auth-google-oauth.md) |
| **Cloudflare (DNS/CDN/WAF)** | DNS, CDN, protección DDoS, SSL | [`servicios/09-cloudflare-dns-cdn.md`](../servicios/09-cloudflare-dns-cdn.md) |
| **Sentry** | Monitoreo de errores frontend/backend | [`servicios/10-sentry-monitoreo.md`](../servicios/10-sentry-monitoreo.md) |
| **Dominio** | Registro y gestión del dominio propio | [`servicios/11-dominio.md`](../servicios/11-dominio.md) |
| **WebSockets (Laravel Reverb)** | Chat huésped↔admin y notificaciones en vivo | [`servicios/12-websockets-reverb.md`](../servicios/12-websockets-reverb.md) |

### ✅ Decidido: una sola pasarela, Stripe

Mercado Pago queda **descartado** por decisión del cliente. Stripe cubre las tres formas de pago previstas:

| Método | Moneda | Nota |
|---|---|---|
| Tarjeta | MXN, USD, CAD | — |
| **OXXO** | Solo MXN | Voucher en efectivo. Requiere cuenta de Stripe México |
| **SPEI** | Solo MXN | Transferencia bancaria (`customer_balance`). Misma cuenta |

**Lo que se gana:** un solo panel para conciliar, un solo modelo de webhooks que mantener, un solo juego de credenciales que rotar. La integración de pagos es la parte más delicada del sistema y duplicarla duplica el riesgo, no solo el trabajo.

**Lo que se pierde, y conviene saberlo:**

- ⚠️ **Comisión algo mayor en tarjeta nacional.** Stripe México cobra ~3.6% + $3 MXN frente al ~3.49% + $4 de Mercado Pago. En tickets de varios miles de pesos la diferencia es de decenas de pesos por reserva, pero existe.
- ⚠️ **Tasa de aprobación.** Mercado Pago suele aprobar algo más en tarjetas nacionales mexicanas, por su relación con los bancos locales. Si aparecen rechazos que no se explican, es lo primero que hay que medir.
- **Meses sin intereses (MSI):** Stripe los ofrece en México, pero hay que habilitarlos explícitamente. No vienen activados de fábrica.

**Cómo se revertiría:** el código habla con una interfaz `PaymentGateway`, no con Stripe. Añadir otra pasarela es una clase nueva y una línea en el contenedor de servicios; nada del flujo de reservas cambia.

**Orden de contratación sugerido:** 1) Dominio → 2) Cloudflare (DNS) → 3) VPS (DigitalOcean) → 4) R2 (mismo panel de Cloudflare) → 5) Stripe → 6) Resend → 7) Tiles de mapa → 8) Sentry → 9) Google Maps (**hasta la fase 3**, cuando se construya el alta de propiedades). Este orden evita bloqueos (ej. no puedes verificar dominio en Resend sin tener antes el DNS en Cloudflare).

**Google Maps va al final a propósito:** solo se necesita para el autocompletado del formulario admin. Crear la cuenta antes deja una API key sin uso y sin restricciones dando vueltas — el escenario exacto de la factura sorpresa.

### El módulo de experiencias no añade ningún servicio

El módulo de tours guiados (sección 20) **no obliga a contratar nada nuevo**. Su costo recurrente adicional es **~$0 USD/mes**; lo que cuesta es el desarrollo. Resumen de impacto (detalle en 20.11):

| Servicio | Impacto |
|---|---|
| R2 | + galerías de experiencias y fotos de guías: **< 1 GB**, dentro del free tier |
| Correo | +7 plantillas; sigue muy por debajo de los 3,000/mes de Resend free |
| Tiles de mapa | Un mapa más por página de detalle; se suma al conteo del proveedor |
| Stripe / MP | Sin costo fijo nuevo. ⚠️ La comisión **pesa más en tickets pequeños**: un tour de $800 MXN deja proporcionalmente menos que una reserva de $10,000 |
| Google Maps | **$0** si se reutiliza Leaflet. La Maps **Embed** API también es gratis; la Maps **JavaScript** API no — no usarla |
| Reverb | No se usa: el módulo no necesita tiempo real |

⚠️ **WhatsApp Business Cloud API: no contratar.** "Enviar el link de reseña al grupo" se resuelve con un deep link `wa.me` abierto desde el teléfono del guía, gratis. La API de Meta exige verificación de negocio, plantillas aprobadas y cobra por conversación — desproporcionado para este uso.

**Reverb no se contrata:** es un paquete del propio Laravel, self-hosted en el VPS ya pagado. No añade costo recurrente ni cuenta externa — solo el subdominio `ws.midominio.com` en el DNS de Cloudflare (paso 2) y el proceso bajo Supervisor. Pusher/Ably quedan documentados como plan B, no contratados.

---

