# Servicio: Analítica de tráfico — Vercel Web Analytics y Speed Insights

> ⏳ **Código listo, falta activar (20-oct-2026).** PR `Casas_front#81`. Al fusionarlo hay que encender **Analytics** y **Speed Insights** en el panel de Vercel; sin eso el código no manda nada.

## ¿Para qué se usa?

Saber cuánta gente entra al sitio, de dónde llega y en qué paso de la reserva se pierde:

- **Web Analytics:** visitantes, páginas vistas, de dónde llegan (Google, Instagram, directo, campañas con `utm_*`), país, dispositivo y navegador.
- **Speed Insights:** qué tan rápido carga cada página para los usuarios reales (Core Web Vitals), no en una prueba de laboratorio.
- **Embudo de reserva:** eventos propios que marcan cada paso del huésped, de ver una casa hasta tener la reserva confirmada.

## Justificación

**Ya está pagado.** El frontend vive en Vercel con el plan Pro (ver `15-hosting-frontend-vercel.md`), que incluye Web Analytics y Speed Insights. No hace falta otra cuenta ni otro proveedor.

**No usa cookies.** Vercel mide sin cookies ni identificadores persistentes del visitante, así que el sitio **no necesita banner de consentimiento**. Google Analytics 4 sí usa cookies: obligaría a pedir consentimiento y a reescribir el aviso de privacidad.

**Por qué no los otros servicios que ya tenemos:**

| Servicio | ¿Sirve para ver el tráfico del sitio? | Por qué |
|---|---|---|
| **Cloudflare** | ❌ No | `www.rivierabnb.com` está en modo **DNS only** (nube gris), porque Vercel lo exige para emitir el certificado SSL. Cloudflare no ve esas visitas. Solo ve `api.rivierabnb.com`, que sí va por su proxy (Cloudflare → Analytics: peticiones, países, ataques bloqueados). |
| **Sentry** | ❌ No | Mide errores y lentitud del front y del back, no visitas. |
| **DigitalOcean** | ❌ No | CPU, memoria y ancho de banda del droplet, más los logs de Nginx. Dice si el servidor aguanta, no qué hacen los usuarios. |

## 💰 Precio

- **Web Analytics y Speed Insights:** incluidos en el plan Pro ($20 USD/mes) que ya se paga.
- **Eventos del embudo (custom events):** cuentan contra la **cuota de eventos del plan Pro**; lo que pase de la cuota se cobra aparte por volumen. Revisar en Vercel → **Usage** el primer mes para ver cuántos genera el tráfico real.

## Qué se mide

### Visitas

Todas las páginas del sitio público, con estas excepciones (no se miden):

| Ruta | Por qué |
|---|---|
| `/dashboard/…`, `/guia/…`, `/anfitrion/…` | Paneles internos: inflarían las cifras con el uso del equipo |
| `/r/…` | Enlaces con token (reseñas) |
| `/establecer-contrasena`, `/recuperar-contrasena` | Llevan token en la URL |

### Embudo de reserva

| # | Evento | Cuándo se manda | Propiedades |
|---|---|---|---|
| 1 | `Casa vista` | Al abrir la ficha de una casa | `casa` (slug) |
| 2 | `Checkout abierto` | Al abrir la página de reservar | `casa`, `huespedes` |
| 3 | `Reserva creada` | Cuando se crea la reserva | `casa`, `metodo` (manual, card…) |
| 4 | `Pago iniciado` | Al abrir el link de pago desde el sitio | `casa` |
| 5 | `Reserva confirmada` | Al ver la reserva ya confirmada, **una vez por reserva y navegador** | `casa` |

Leído en orden, dice dónde se cae la gente: muchas "Casa vista" con pocos "Checkout abierto" apunta a precio o fotos; muchos "Checkout abierto" con pocas "Reserva creada" apunta al formulario.

## Privacidad

Lo que se cuidó para que no salgan datos personales (Vercel además no los admite en los eventos):

- **El código de reserva no sale:** en la URL se reemplaza por `/reservas/[codigo]`. Con el código se abre el estado de la reserva.
- **Se quitan los parámetros de la URL**, salvo los `utm_*`, que dicen de qué campaña llegó la visita.
- **Los eventos llevan solo** el slug de la casa y datos agregados (número de huéspedes, método de pago). Nunca nombre, correo, teléfono ni código.

⚠️ **Aviso de privacidad (D8):** aunque no haya cookies, conviene que mencione que se recogen *estadísticas anónimas de uso del sitio*. Es una línea, pero esos textos los revisó el abogado: pasarla por él.

## Límites conocidos

- **Pagos por Mercado Pago (D17):** quien paga desde el botón del correo no pasa por el sitio, así que ese `Pago iniciado` no se registra. `Reserva confirmada` sí, cuando el huésped abre su reserva.
- **`Reserva confirmada` se cuenta por navegador:** si el huésped abre su reserva en el teléfono y luego en la computadora, cuenta dos veces. Para cifras exactas de reservas confirmadas, el dato bueno es **Reportes** del panel, no la analítica.
- **En local no se registra nada:** Vercel solo mide en sus despliegues.

## Bitácora de configuración

### En el código (`Casas_front#81`)

1. Paquetes `@vercel/analytics` y `@vercel/speed-insights`.
2. `src/components/SiteAnalytics.tsx`: los componentes `<Analytics>` y `<SpeedInsights>` con un `beforeSend` que limpia o descarta cada URL (`sanitizeUrl`), montados en el layout raíz (`src/app/[lang]/layout.tsx`).
3. `src/lib/analytics.ts`: la lista de rutas excluidas, la limpieza de URLs y los eventos del embudo (`track`, `trackOnce`). Si `track` falla, no rompe la reserva.
4. Eventos en: ficha de la casa (`casas/[slug]/page.tsx`), `CheckoutForm.tsx` y `BookingStatusPanel.tsx`.

### En Vercel (después de fusionar)

1. Team `RivieraBnb` → proyecto `rivierabnb-front` → pestaña **Analytics** → **Enable**.
2. Pestaña **Speed Insights** → **Enable**.
3. Esperar el siguiente despliegue y abrir el sitio: las visitas aparecen en minutos.
4. Los eventos del embudo salen en Analytics → **Events**.
5. Al mes, revisar Vercel → **Usage** para confirmar que los eventos caben en la cuota.

## Si algún día no alcanza

Si se necesitan embudos más finos (cohortes, grabaciones de sesión, mapas de calor), las alternativas son **PostHog** (plan gratuito amplio, sin cookies si se configura así) o **Google Analytics 4** (gratis, pero con cookies y banner de consentimiento). Ninguna hace falta hoy.
