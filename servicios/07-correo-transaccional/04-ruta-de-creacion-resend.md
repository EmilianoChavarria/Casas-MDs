# Ruta de creación (Resend)

1. Crear cuenta en https://resend.com
2. **Domains → Add Domain** (ej. `midominio.com`).
3. Agregar los registros DNS que Resend indique (SPF, DKIM, DMARC recomendado) — esto se hace en Cloudflare si el DNS está ahí (ver `09-cloudflare-dns-cdn.md`).
4. Esperar verificación del dominio (usualmente minutos, hasta 24h).
5. **API Keys → Create API Key** con permiso de solo envío (`Sending access`).

