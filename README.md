# Arquitectura Completa — Plataforma de Renta de Casas (mono-empresa)

**Stack:** Next.js (React + TS) · Laravel 12 API REST · MySQL 8 · Sanctum · S3/R2 · Redis · Docker · Nginx · GitHub Actions

Este README es solo el **índice**. El contenido está dividido en dos bloques: la arquitectura del sistema (carpeta [`arquitectura/`](arquitectura/)) y los servicios externos contratables (un `.md` por servicio, en [`servicios/servicios/`](servicios/servicios/)).

---

## 📐 Arquitectura — [`arquitectura/`](arquitectura/)

| # | Sección | Archivo |
|---|---|---|
| 1 | Arquitectura general | [`01-arquitectura-general.md`](arquitectura/01-arquitectura-general.md) |
| 2 | Justificación tecnológica | [`02-justificacion-tecnologica.md`](arquitectura/02-justificacion-tecnologica.md) |
| 3 | Estructura del backend Laravel | [`03-estructura-del-backend-laravel.md`](arquitectura/03-estructura-del-backend-laravel.md) |
| 4 | Estructura del frontend Next.js (App Router) | [`04-estructura-del-frontend-nextjs-app-router.md`](arquitectura/04-estructura-del-frontend-nextjs-app-router.md) |
| 5 | Base de datos | [`05-base-de-datos.md`](arquitectura/05-base-de-datos.md) |
| 6 | API REST (resumen de endpoints) | [`06-api-rest-resumen-de-endpoints.md`](arquitectura/06-api-rest-resumen-de-endpoints.md) |
| 7 | Seguridad | [`07-seguridad.md`](arquitectura/07-seguridad.md) |
| 8 | Infraestructura y despliegue en producción | [`08-infraestructura-y-despliegue-en-produccion.md`](arquitectura/08-infraestructura-y-despliegue-en-produccion.md) |
| 9 | DevOps | [`09-devops.md`](arquitectura/09-devops.md) |
| 10 | Servicios externos recomendados | [`10-servicios-externos-recomendados.md`](arquitectura/10-servicios-externos-recomendados.md) |
| 11 | Escalabilidad (100 → 100,000 usuarios) | [`11-escalabilidad-100-100000-usuarios.md`](arquitectura/11-escalabilidad-100-100000-usuarios.md) |
| 12 | Roadmap por fases | [`12-roadmap-por-fases.md`](arquitectura/12-roadmap-por-fases.md) |
| 13 | Estimación aproximada | [`13-estimacion-aproximada.md`](arquitectura/13-estimacion-aproximada.md) |
| 14 | Buenas prácticas aplicadas | [`14-buenas-practicas-aplicadas.md`](arquitectura/14-buenas-practicas-aplicadas.md) |

---

## 🧩 Servicios externos — [`servicios/servicios/`](servicios/servicios/)

Un solo `.md` por servicio, con todas sus secciones (¿para qué se usa?, justificación, precio y plan gratuito para desarrollo, ruta de creación, contrato/costos, configuración).

| # | Servicio | Archivo | ¿Plan free para desarrollo? |
|---|---|---|---|
| 01 | Hosting / VPS — Hetzner Cloud | [`01-hosting-vps.md`](servicios/servicios/01-hosting-vps.md) | ❌ No (usar Docker local, gratis) |
| 02 | Base de Datos — MySQL 8 | [`02-base-datos-mysql.md`](servicios/servicios/02-base-datos-mysql.md) | ✅ Self-hosted gratis |
| 03 | Cache y Colas — Redis | [`03-cache-colas-redis.md`](servicios/servicios/03-cache-colas-redis.md) | ✅ Self-hosted gratis / Upstash free permanente |
| 04 | Almacenamiento de Imágenes — Cloudflare R2 | [`04-almacenamiento-r2.md`](servicios/servicios/04-almacenamiento-r2.md) | ✅ Free tier permanente |
| 05 | Pagos — Stripe | [`05-pagos-stripe.md`](servicios/servicios/05-pagos-stripe.md) | ✅ Sandbox gratis e ilimitado |
| 06 | Pagos — Mercado Pago | [`06-pagos-mercadopago.md`](servicios/servicios/06-pagos-mercadopago.md) | ✅ Usuarios de prueba gratis |
| 07 | Correo Transaccional — Resend / Amazon SES | [`07-correo-transaccional.md`](servicios/servicios/07-correo-transaccional.md) | ✅ Resend free permanente ⚠️ SES solo 12 meses |
| 08 | Google Maps Platform | [`08-google-maps.md`](servicios/servicios/08-google-maps.md) | ✅ 10,000 llamadas gratis/API/mes (plan Essentials) |
| 09 | Cloudflare — DNS, CDN, WAF | [`09-cloudflare-dns-cdn.md`](servicios/servicios/09-cloudflare-dns-cdn.md) | ✅ Plan Free permanente |
| 10 | Monitoreo de Errores — Sentry | [`10-sentry-monitoreo.md`](servicios/servicios/10-sentry-monitoreo.md) | ✅ Plan Developer gratis permanente |
| 11 | Registro de Dominio | [`11-dominio.md`](servicios/servicios/11-dominio.md) | ❌ Nunca (no se necesita para desarrollar) |
