# Ruta de creación

1. Verificar disponibilidad del dominio deseado.
2. **Opción A — Cloudflare Registrar** (recomendado si el DNS ya vive en Cloudflare):
   - Requiere que el dominio ya esté activo/transferido a Cloudflare (no permite registro directo de dominios nuevos en todos los TLD, revisar disponibilidad para `.com`/`.mx`).
3. **Opción B — Namecheap:**
   - Crear cuenta, comprar el dominio, y en la sección **Domain → Nameservers** cambiar a los nameservers de Cloudflare (ver `../09-cloudflare-dns-cdn/`, paso 5).
4. Activar **WHOIS Privacy/Redaction** (gratuito en la mayoría de registradores) para no exponer datos personales/de la empresa públicamente.
5. Activar renovación automática y bloqueo de transferencia (Registrar Lock) para evitar robo de dominio.

