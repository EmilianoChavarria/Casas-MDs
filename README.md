# Arquitectura Completa — Plataforma de Renta de Casas (mono-empresa)

**Stack:** Next.js (React + TS) · Laravel 12 API REST · MySQL 8 · Sanctum · S3/R2 · Redis · Docker · Nginx · GitHub Actions

Este README es solo el **índice**. El contenido está dividido en dos bloques: la arquitectura del sistema (carpeta [`arquitectura/`](arquitectura/)) y los servicios externos contratables (una carpeta por servicio, en la raíz).

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

## 🧩 Servicios externos

Cada servicio tiene su propia carpeta con un `.md` por sección (¿para qué se usa?, justificación, precio y plan gratuito para desarrollo, ruta de creación, contrato/costos, configuración).

### Resumen rápido

| # | Servicio | Carpeta | ¿Plan free para desarrollo? |
|---|---|---|---|
| 01 | Hosting / VPS — Hetzner Cloud | [`01-hosting-vps/`](01-hosting-vps/) | ❌ No (usar Docker local, gratis) |
| 02 | Base de Datos — MySQL 8 | [`02-base-datos-mysql/`](02-base-datos-mysql/) | ✅ Self-hosted gratis |
| 03 | Cache y Colas — Redis | [`03-cache-colas-redis/`](03-cache-colas-redis/) | ✅ Self-hosted gratis / Upstash free permanente |
| 04 | Almacenamiento de Imágenes — Cloudflare R2 | [`04-almacenamiento-r2/`](04-almacenamiento-r2/) | ✅ Free tier permanente |
| 05 | Pagos — Stripe | [`05-pagos-stripe/`](05-pagos-stripe/) | ✅ Sandbox gratis e ilimitado |
| 06 | Pagos — Mercado Pago | [`06-pagos-mercadopago/`](06-pagos-mercadopago/) | ✅ Usuarios de prueba gratis |
| 07 | Correo Transaccional — Resend / Amazon SES | [`07-correo-transaccional/`](07-correo-transaccional/) | ✅ Resend free permanente ⚠️ SES solo 12 meses |
| 08 | Google Maps Platform | [`08-google-maps/`](08-google-maps/) | ✅ 10,000 llamadas gratis/API/mes (plan Essentials) |
| 09 | Cloudflare — DNS, CDN, WAF | [`09-cloudflare-dns-cdn/`](09-cloudflare-dns-cdn/) | ✅ Plan Free permanente |
| 10 | Monitoreo de Errores — Sentry | [`10-sentry-monitoreo/`](10-sentry-monitoreo/) | ✅ Plan Developer gratis permanente |
| 11 | Registro de Dominio | [`11-dominio/`](11-dominio/) | ❌ Nunca (no se necesita para desarrollar) |

### Índice detallado por servicio

#### 01 · Hosting / VPS — Hetzner Cloud — [`01-hosting-vps/`](01-hosting-vps/)
- [¿Para qué se usa?](01-hosting-vps/01-para-que-se-usa.md)
- [Justificación](01-hosting-vps/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](01-hosting-vps/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](01-hosting-vps/04-ruta-de-creacion.md)
- [Contrato / plan recomendado](01-hosting-vps/05-contrato-plan-recomendado.md)
- [Configuración inicial del servidor](01-hosting-vps/06-configuracion-inicial-del-servidor.md)
- [Variables/credenciales a resguardar en GitHub Secrets](01-hosting-vps/07-variablescredenciales-a-resguardar-en-github-secrets.md)
- [Costos aproximados (mensual)](01-hosting-vps/08-costos-aproximados-mensual.md)

#### 02 · Base de Datos — MySQL 8 — [`02-base-datos-mysql/`](02-base-datos-mysql/)
- [¿Para qué se usa?](02-base-datos-mysql/01-para-que-se-usa.md)
- [Justificación](02-base-datos-mysql/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](02-base-datos-mysql/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación (self-hosted, fase inicial)](02-base-datos-mysql/04-ruta-de-creacion-selfhosted-fase-inicial.md)
- [Ruta de creación (gestionado, fase de escalado)](02-base-datos-mysql/05-ruta-de-creacion-gestionado-fase-de-escalado.md)
- [Contrato / plan recomendado](02-base-datos-mysql/06-contrato-plan-recomendado.md)
- [Configuración](02-base-datos-mysql/07-configuracion.md)

#### 03 · Cache y Colas — Redis — [`03-cache-colas-redis/`](03-cache-colas-redis/)
- [¿Para qué se usa?](03-cache-colas-redis/01-para-que-se-usa.md)
- [Justificación](03-cache-colas-redis/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](03-cache-colas-redis/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](03-cache-colas-redis/04-ruta-de-creacion.md)
- [Contrato / plan recomendado](03-cache-colas-redis/05-contrato-plan-recomendado.md)
- [Configuración](03-cache-colas-redis/06-configuracion.md)

#### 04 · Almacenamiento de Imágenes — Cloudflare R2 — [`04-almacenamiento-r2/`](04-almacenamiento-r2/)
- [¿Para qué se usa?](04-almacenamiento-r2/01-para-que-se-usa.md)
- [Justificación](04-almacenamiento-r2/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](04-almacenamiento-r2/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](04-almacenamiento-r2/04-ruta-de-creacion.md)
- [Contrato / plan recomendado](04-almacenamiento-r2/05-contrato-plan-recomendado.md)
- [Configuración](04-almacenamiento-r2/06-configuracion.md)

#### 05 · Pagos — Stripe — [`05-pagos-stripe/`](05-pagos-stripe/)
- [¿Para qué se usa?](05-pagos-stripe/01-para-que-se-usa.md)
- [Justificación](05-pagos-stripe/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](05-pagos-stripe/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](05-pagos-stripe/04-ruta-de-creacion.md)
- [Contrato / condiciones](05-pagos-stripe/05-contrato-condiciones.md)
- [Configuración](05-pagos-stripe/06-configuracion.md)

#### 06 · Pagos — Mercado Pago — [`06-pagos-mercadopago/`](06-pagos-mercadopago/)
- [¿Para qué se usa?](06-pagos-mercadopago/01-para-que-se-usa.md)
- [Justificación](06-pagos-mercadopago/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](06-pagos-mercadopago/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](06-pagos-mercadopago/04-ruta-de-creacion.md)
- [Contrato / condiciones](06-pagos-mercadopago/05-contrato-condiciones.md)
- [Configuración](06-pagos-mercadopago/06-configuracion.md)

#### 07 · Correo Transaccional — Resend / Amazon SES — [`07-correo-transaccional/`](07-correo-transaccional/)
- [¿Para qué se usa?](07-correo-transaccional/01-para-que-se-usa.md)
- [Justificación](07-correo-transaccional/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](07-correo-transaccional/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación (Resend)](07-correo-transaccional/04-ruta-de-creacion-resend.md)
- [Ruta de creación (alternativa: Amazon SES)](07-correo-transaccional/05-ruta-de-creacion-alternativa-amazon-ses.md)
- [Contrato / plan recomendado](07-correo-transaccional/06-contrato-plan-recomendado.md)
- [Configuración](07-correo-transaccional/07-configuracion.md)

#### 08 · Google Maps Platform — [`08-google-maps/`](08-google-maps/)
- [¿Para qué se usa?](08-google-maps/01-para-que-se-usa.md)
- [Justificación](08-google-maps/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](08-google-maps/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](08-google-maps/04-ruta-de-creacion.md)
- [Contrato / plan recomendado](08-google-maps/05-contrato-plan-recomendado.md)
- [Configuración](08-google-maps/06-configuracion.md)

#### 09 · Cloudflare — DNS, CDN, WAF — [`09-cloudflare-dns-cdn/`](09-cloudflare-dns-cdn/)
- [¿Para qué se usa?](09-cloudflare-dns-cdn/01-para-que-se-usa.md)
- [Justificación](09-cloudflare-dns-cdn/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](09-cloudflare-dns-cdn/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](09-cloudflare-dns-cdn/04-ruta-de-creacion.md)
- [Contrato / plan recomendado](09-cloudflare-dns-cdn/05-contrato-plan-recomendado.md)
- [Configuración adicional](09-cloudflare-dns-cdn/06-configuracion-adicional.md)

#### 10 · Monitoreo de Errores — Sentry — [`10-sentry-monitoreo/`](10-sentry-monitoreo/)
- [¿Para qué se usa?](10-sentry-monitoreo/01-para-que-se-usa.md)
- [Justificación](10-sentry-monitoreo/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](10-sentry-monitoreo/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](10-sentry-monitoreo/04-ruta-de-creacion.md)
- [Contrato / plan recomendado](10-sentry-monitoreo/05-contrato-plan-recomendado.md)
- [Configuración](10-sentry-monitoreo/06-configuracion.md)

#### 11 · Registro de Dominio — [`11-dominio/`](11-dominio/)
- [¿Para qué se usa?](11-dominio/01-para-que-se-usa.md)
- [Justificación](11-dominio/02-justificacion.md)
- [💰 Precio y plan gratuito para desarrollo](11-dominio/03-precio-y-plan-gratuito-para-desarrollo.md)
- [Ruta de creación](11-dominio/04-ruta-de-creacion.md)
- [Contrato / plan recomendado](11-dominio/05-contrato-plan-recomendado.md)
- [Configuración posterior al registro](11-dominio/06-configuracion-posterior-al-registro.md)
