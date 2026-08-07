# Servicio: Registro de Dominio

## ¿Para qué se usa?

Nombre de dominio propio (ej. `midominio.com`) bajo el cual operan el sitio público, el subdominio de API y el CDN de imágenes.


## Justificación

Necesario para identidad de marca, SEO (Next.js SSR depende de un dominio estable e indexable) y para que los certificados SSL y los registros de correo (SPF/DKIM/DMARC, ver `07-correo-transaccional.md`) tengan un dominio verificable.

**Recomendación de registrador:** Cloudflare Registrar (precio al costo, sin markup) si ya se usa Cloudflare para DNS/CDN (ver `09-cloudflare-dns-cdn.md`); alternativa: Namecheap o Google Domains (ahora parte de Squarespace).


## 💰 Precio y plan gratuito para desarrollo

⚠️ Los dominios **no tienen plan gratuito en ningún registrador** — siempre hay que pagar el registro y cada renovación, no es un servicio del tipo "free tier para pruebas".

| Opción | Costo | ¿Necesario para desarrollo? |
|---|---|---|
| **Dominio real (`.com`, `.mx`)** | Ver precios abajo | ❌ No es necesario para desarrollar — solo se necesita antes de salir a producción |
| **`localhost` / dominio local (`.test`, `/etc/hosts`)** | $0 | ✅ Sí — suficiente para todo el desarrollo local, incluyendo probar HTTPS con certificados autofirmados |
| **Subdominio temporal gratuito** (ej. de Cloudflare Pages, Vercel, ngrok) | $0 | ✅ Útil para compartir un preview con el equipo o el cliente antes de comprar el dominio definitivo |

**Recomendación:** no comprar el dominio hasta tener el proyecto listo para producción (o casi); desarrollar con `localhost`/dominio local evita gastar dinero de forma anticipada.


## Ruta de creación

1. Verificar disponibilidad del dominio deseado.
2. **Opción A — Cloudflare Registrar** (recomendado si el DNS ya vive en Cloudflare):
   - Requiere que el dominio ya esté activo/transferido a Cloudflare (no permite registro directo de dominios nuevos en todos los TLD, revisar disponibilidad para `.com`/`.mx`).
3. **Opción B — Namecheap:**
   - Crear cuenta, comprar el dominio, y en la sección **Domain → Nameservers** cambiar a los nameservers de Cloudflare (ver `09-cloudflare-dns-cdn.md`, paso 5).
4. Activar **WHOIS Privacy/Redaction** (gratuito en la mayoría de registradores) para no exponer datos personales/de la empresa públicamente.
5. Activar renovación automática y bloqueo de transferencia (Registrar Lock) para evitar robo de dominio.


## Contrato / plan recomendado

- Dominio `.com` en **Cloudflare Registrar**: ~$10.44 USD/año (precio al costo, sin markup — incluye WHOIS privacy gratis; el registro y la renovación cuestan lo mismo, sin sorpresas al segundo año). Requiere que el DNS del dominio ya viva en Cloudflare.
- Dominio `.com` en **Namecheap**: ~$12–16 USD/año tras el primer año promocional (suele tener precio de renovación más alto que el de registro — revisar antes de comprar).
- Dominio `.mx`: gestionado por NIC México, ~$300–400 MXN/año, requiere datos de persona física o moral mexicana para el registro (puede solicitarse a través de revendedores como Namecheap o directo en https://www.nic.mx).
- Revisar la política de renovación y el precio de renovación antes de comprar en cualquier registrador que no sea Cloudflare.


## Configuración posterior al registro

- Ver `09-cloudflare-dns-cdn.md` para la configuración de nameservers y registros DNS (A, CNAME).
- Ver `07-correo-transaccional.md` para los registros TXT de SPF/DKIM/DMARC necesarios para que los correos transaccionales no caigan en spam.

Referenciado desde: `../../arquitectura/`, secciones 8 y 10.
