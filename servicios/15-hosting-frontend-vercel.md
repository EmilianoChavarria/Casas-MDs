# Servicio: Hosting del frontend — Vercel

> ✅ **En producción (14-oct-2026).** `https://rivierabnb.com` y `https://www.rivierabnb.com` en vivo, desplegados desde `main` del repo `Casas_front`, conectados al backend real (`https://api.rivierabnb.com`).

## ¿Para qué se usa?

Hosting del frontend Next.js — build, CDN global, despliegue automático en cada push a `main`, y Preview Deployments por cada Pull Request.

## Justificación

Es la opción estándar para Next.js (mismo equipo que mantiene el framework): SSR/ISR nativos sin configurar nada aparte, CDN incluido, y el despliegue es automático por Git — no hace falta un pipeline propio como sí hace falta para el backend (ver `01-hosting-vps.md`).

## 💰 Precio

Se contrató el plan **Pro ($20 USD/mes)**, no el gratuito (Hobby). Motivo real: al crear el Team dedicado al proyecto (`RivieraBnb`) dentro de la cuenta personal del operador, Vercel exigió Pro para poder crear el Team — no fue una elección activa de necesitar las funciones de Pro, fue un requisito de la plataforma en ese flujo.

## Cuenta usada

Se intentó crear una cuenta nueva con el correo dedicado (`rivierabnb56@gmail.com`), pero Vercel vincula la verificación por **número de teléfono**, no por correo — el número del operador ya tenía una cuenta personal de Vercel asociada, y no dejó crear una segunda cuenta con el mismo número. Se resolvió entrando con la cuenta personal existente y creando un **Team** aparte (`RivieraBnb`) para mantener el proyecto organizado y con facturación propia, sin mezclarlo con proyectos personales.

## Bitácora de configuración

### Importar el proyecto

1. Dentro del Team `RivieraBnb` → **Add New** → **Project** → **Import Git Repository**.
2. Repo: `EmilianoChavarria/Casas_front`, rama `main`.
3. Nombre del proyecto: `rivierabnb-front` (para que combine con el nombre usado en Sentry).
4. Root Directory: `./` (no es monorepo). Framework detectado solo: **Next.js**.

### Variables de entorno (Production)

Pegadas de una vez con "paste the .env contents" en el formulario de importación, ámbito **Production** (en vez de "Production and Preview" — no hay nadie usando Preview todavía):

```
NEXT_PUBLIC_API_URL=https://api.rivierabnb.com
NEXT_PUBLIC_IMAGE_HOST=pub-49874da19a0947adac2f895572cea0ac.r2.dev
NEXT_PUBLIC_CHARGE_IN_BASE=true
NEXT_PUBLIC_SENTRY_DSN=https://c4ae2bad56697d3b74b92c45b10ed861@o4512234105929729.ingest.us.sentry.io/4512234142367744
SENTRY_DSN=https://c4ae2bad56697d3b74b92c45b10ed861@o4512234105929729.ingest.us.sentry.io/4512234142367744
SENTRY_ENVIRONMENT=production
SENTRY_TRACES_SAMPLE_RATE=0.1
SENTRY_ORG=rivierabnb
SENTRY_PROJECT=rivierabnb-front
NEXT_PUBLIC_MAP_TILE_URL=https://tiles.stadiamaps.com/tiles/alidade_bright/{z}/{x}/{y}{r}.png
NEXT_PUBLIC_MAP_ATTRIBUTION=&copy; <a href="https://stadiamaps.com/">Stadia Maps</a> &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>
```

`SENTRY_AUTH_TOKEN` queda sin poner — pendiente, igual que en local (ver `10-sentry-monitoreo.md`); sin ella el build funciona bien, solo no sube los source maps.

No se aceptó la integración opcional de Sentry que ofrece Vercel al crear el proyecto — ya está configurado a mano por variables de entorno, y la integración hubiera podido duplicar o desalinear esa configuración.

### Dominio propio

El DNS de `rivierabnb.com` vive en **Cloudflare** (ver `09-cloudflare-dns-cdn.md`). En el proyecto de Vercel → **Settings → Domains → Add Existing**:

1. Se agregó `rivierabnb.com`, con la opción **"Redirect apex domains to www"** activada (recomendación de Vercel, evita problemas de SSL en el dominio raíz).
2. Vercel agregó automáticamente `www.rivierabnb.com` como el dominio que sirve de verdad el contenido.
3. Registros creados en Cloudflare → DNS → Records, **ambos en modo "DNS only" (nube gris, sin proxy)** — con el proxy naranja activado, Vercel no puede emitir el certificado SSL:

   ```
   CNAME   @     68096b1335b366e0.vercel-dns-016.com
   CNAME   www   68096b1335b366e0.vercel-dns-016.com
   ```

4. Confirmado en vivo: `rivierabnb.com` responde `308` → `https://www.rivierabnb.com`, y este último responde `200`.

⚠️ El valor exacto del target CNAME (`68096b1335b366e0.vercel-dns-016.com`) lo genera Vercel por proyecto — si se recrea el proyecto o se agrega el dominio de nuevo, puede salir un valor distinto; siempre usar el que muestre el panel de Vercel en el momento, no copiar este literal a ciegas.

## Pendiente

- `SENTRY_AUTH_TOKEN` sigue vacío — sin source maps reales en los stack traces de Sentry hasta que se genere.

Referenciado desde: `../arquitectura/08-infraestructura-y-despliegue-en-produccion.md` (sección 8.1, donde ya se recomendaba Vercel para el frontend).

## Analítica

El plan Pro incluye Web Analytics y Speed Insights. Cómo se usan, qué se mide y qué se cuidó de privacidad: ver [`16-analitica-trafico.md`](16-analitica-trafico.md).
