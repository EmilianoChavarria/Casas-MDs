# Ruta de creación

1. Crear cuenta en https://dash.cloudflare.com/sign-up (si no se creó ya para R2).
2. **Add a site** → ingresar `midominio.com`.
3. Elegir plan **Free**.
4. Cloudflare escaneará los registros DNS existentes; revisar y confirmar.
5. Cambiar los **nameservers** del dominio (en el registrador, ver `11-dominio.md`) a los que indique Cloudflare (ej. `ana.ns.cloudflare.com`, `bob.ns.cloudflare.com`).
6. Esperar propagación (minutos a 24h).
7. Configurar registros DNS necesarios:
   ```
   A       @                 <IP del VPS>          Proxied (naranja)
   A       api               <IP del VPS>          Proxied (naranja)
   CNAME   cdn               <bucket>.r2.dev        Proxied (naranja)
   CNAME   www               midominio.com          Proxied (naranja)
   ```
8. **SSL/TLS → Overview** → seleccionar modo **Full (Strict)** (requiere certificado válido en el origen, ver Certbot en `01-hosting-vps.md`, o usar Cloudflare Origin Certificate).
9. **SSL/TLS → Origin Server → Create Certificate** para generar un certificado de origen válido por 15 años, instalarlo en Nginx del VPS (más simple que gestionar Let's Encrypt manualmente).
10. **Security → WAF** → activar reglas gestionadas básicas (incluidas en Free).
11. **Speed → Optimization** → activar Auto Minify y Brotli.

