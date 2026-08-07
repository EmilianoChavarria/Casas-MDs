# Servicio: Registro de Dominio

## ¿Para qué se usa?
Nombre de dominio propio (ej. `midominio.com`) bajo el cual operan el sitio público, el subdominio de API y el CDN de imágenes.

## Justificación
Necesario para identidad de marca, SEO (Next.js SSR depende de un dominio estable e indexable) y para que los certificados SSL y los registros de correo (SPF/DKIM/DMARC, ver `07-correo-transaccional.md`) tengan un dominio verificable.

**Recomendación de registrador:** Cloudflare Registrar (precio al costo, sin markup) si ya se usa Cloudflare para DNS/CDN (ver `09-cloudflare-dns-cdn.md`); alternativa: Namecheap o Google Domains (ahora parte de Squarespace).

## Ruta de creación
1. Verificar disponibilidad del dominio deseado.
2. **Opción A — Cloudflare Registrar** (recomendado si el DNS ya vive en Cloudflare):
   - Requiere que el dominio ya esté activo/transferido a Cloudflare (no permite registro directo de dominios nuevos en todos los TLD, revisar disponibilidad para `.com`/`.mx`).
3. **Opción B — Namecheap:**
   - Crear cuenta, comprar el dominio, y en la sección **Domain → Nameservers** cambiar a los nameservers de Cloudflare (ver `09-cloudflare-dns-cdn.md`, paso 5).
4. Activar **WHOIS Privacy/Redaction** (gratuito en la mayoría de registradores) para no exponer datos personales/de la empresa públicamente.
5. Activar renovación automática y bloqueo de transferencia (Registrar Lock) para evitar robo de dominio.

## Contrato / plan recomendado
- Dominio `.com`: ~$10–14 USD/año (Cloudflare Registrar, precio al costo) vs ~$12–16 USD/año en Namecheap tras el primer año promocional.
- Dominio `.mx`: gestionado por NIC México, ~$300–400 MXN/año, requiere datos de persona física o moral mexicana para el registro (puede solicitarse a través de revendedores como Namecheap o directo en https://www.nic.mx).
- Revisar la política de renovación y el precio de renovación (suele ser mayor al precio del primer año) antes de comprar.

## Configuración posterior al registro
- Ver `09-cloudflare-dns-cdn.md` para la configuración de nameservers y registros DNS (A, CNAME).
- Ver `07-correo-transaccional.md` para los registros TXT de SPF/DKIM/DMARC necesarios para que los correos transaccionales no caigan en spam.

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 8 y 10.
