# Servicio: Cloudflare — DNS, CDN, WAF

## ¿Para qué se usa?

1. **DNS** del dominio principal (`midominio.com`) y subdominios (`api.midominio.com`, `cdn.midominio.com`).
2. **CDN/Proxy** frente al VPS del backend y frente al bucket de R2 (imágenes) — cachea contenido estático cerca del usuario.
3. **WAF (Web Application Firewall)** y protección DDoS básica gratuita frente al backend.
4. Emisión/gestión de certificados SSL de borde (modo Full Strict).


## Justificación

Es prácticamente obligatorio en este stack porque R2 (almacenamiento) requiere una cuenta de Cloudflare de todas formas (ver `04-almacenamiento-r2.md`); consolidar DNS + CDN + WAF en el mismo proveedor simplifica la operación y es gratuito en su plan base.


## 💰 Precio y plan gratuito para desarrollo

| Plan | Costo | ¿Sirve para dev/producción inicial? |
|---|---|---|
| **Free** | $0/mes, **de forma permanente** (no es una prueba con límite de tiempo) | ✅ Sí — incluye DNS, CDN, SSL, protección DDoS L3/L4 y WAF gestionado básico; suficiente para desarrollo y para el arranque en producción |
| **Pro** | $20 USD/mes (facturación anual) o $25 USD/mes (mensual) | Solo necesario al escalar (reglas WAF avanzadas, mejor caché de imágenes) |

**Recomendación:** quedarse en el plan Free indefinidamente hasta que el tráfico o los requisitos de seguridad justifiquen el salto a Pro — no hace falta pagar nada para desarrollo ni para el lanzamiento inicial.


## Ruta de creación

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
   A       ws                <IP del VPS>          Proxied (naranja)
   CNAME   cdn               <bucket>.r2.dev        Proxied (naranja)
   CNAME   www               midominio.com          Proxied (naranja)
   ```
   El registro `ws` es el subdominio del WebSocket de Laravel Reverb (chat en tiempo real, sección 16). El proxy naranja soporta WebSockets sin configuración extra, pero **corta las conexiones inactivas a los 100 s en plan Free** — el `ping/pong` que trae el protocolo Pusher lo resuelve; ver `12-websockets-reverb.md` para los valores concretos.

   > Dejarlo en gris (proxy off) esquiva ese límite, pero expone la IP del origen y deja el WebSocket sin protección DDoS. No compensa: mantener naranja y ajustar los timeouts.
8. **SSL/TLS → Overview** → seleccionar modo **Full (Strict)** (requiere certificado válido en el origen, ver Certbot en `01-hosting-vps.md`, o usar Cloudflare Origin Certificate).
9. **SSL/TLS → Origin Server → Create Certificate** para generar un certificado de origen válido por 15 años, instalarlo en Nginx del VPS (más simple que gestionar Let's Encrypt manualmente).
10. **Security → WAF** → activar reglas gestionadas básicas (incluidas en Free).
11. **Speed → Optimization** → activar Auto Minify y Brotli.


## Contrato / plan recomendado

- **Plan Free:** $0/mes, permanente — incluye DNS, CDN básico, SSL, protección DDoS L3/L4, WAF gestionado limitado. Suficiente para el arranque del proyecto.
- **Plan Pro:** $20 USD/mes facturado anual, o $25 USD/mes facturado mensualmente — reglas WAF más avanzadas, mejor caché de imágenes, útil al escalar (fase de 10,000+ usuarios, sección 11 del doc principal).


## Configuración adicional


### Page Rules / Cache Rules para imágenes (fase inicial)
```
URL: cdn.midominio.com/*
Configuración: Cache Level = Cache Everything, Edge TTL = 1 mes
```

### Reglas de Firewall recomendadas
- Rate limiting en `api.midominio.com/api/v1/auth/login` (mitiga fuerza bruta a nivel de borde, complementa el `throttle` de Laravel — ver sección 7 del documento principal).
- Bloqueo geográfico opcional si el negocio es 100% doméstico (revisar antes de activar, ya que puede bloquear turistas legítimos reservando desde el extranjero).

Referenciado desde: `../arquitectura/`, secciones 1, 7, 8 y 10.
