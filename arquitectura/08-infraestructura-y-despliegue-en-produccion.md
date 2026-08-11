# 8. Infraestructura y despliegue en producción


### 8.1 Topología

- 1 VPS (DigitalOcean Basic Droplet 4 vCPU / 8 GB o similar) para Laravel + MySQL + Redis (o MySQL gestionado aparte si el presupuesto lo permite).
- Next.js puede vivir en **Vercel** (más simple, CDN global, ISR nativo) o en el mismo VPS con `pm2`/Docker si prefieres todo autoalojado — dado tu contexto de cPanel/VPS, recomiendo Vercel para el frontend y VPS solo para el backend.

### 8.2 Docker Compose (backend)

```yaml
services:
  app:
    build: ./docker/php
    volumes: ["./backend:/var/www/html"]
    depends_on: [mysql, redis]
  nginx:
    image: nginx:alpine
    ports: ["443:443", "80:80"]
    volumes:
      - ./docker/nginx:/etc/nginx/conf.d
      - ./backend:/var/www/html
      - ./certs:/etc/nginx/certs
    depends_on: [app]
  mysql:
    image: mysql:8
    environment:
      MYSQL_DATABASE: rentas
    volumes: ["mysql_data:/var/lib/mysql"]
  redis:
    image: redis:7-alpine
  queue-worker:
    build: ./docker/php
    command: php artisan queue:work --tries=3
    depends_on: [app, redis]
  reverb:
    build: ./docker/php
    command: php artisan reverb:start --host=0.0.0.0 --port=8080
    depends_on: [app, redis]
    expose: ["8080"]
    restart: unless-stopped
volumes:
  mysql_data:
```

### 8.3 Proxy WebSocket (Nginx)

El chat en tiempo real (sección 16) necesita un subdominio propio, `ws.midominio.com`, apuntando al proceso Reverb:

```nginx
server {
    listen 443 ssl http2;
    server_name ws.midominio.com;

    location / {
        proxy_pass          http://reverb:8080;
        proxy_http_version  1.1;
        proxy_set_header    Upgrade $http_upgrade;
        proxy_set_header    Connection "upgrade";
        proxy_set_header    Host $host;
        proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
        proxy_read_timeout  60s;
    }
}
```

Sin los headers `Upgrade` y `Connection`, el handshake WebSocket falla con un 400 y el cliente cae en reconexión infinita — es el error más común al desplegar Reverb detrás de un proxy.

⚠️ **Cloudflare** soporta WebSockets con el proxy naranja activo, pero corta conexiones inactivas (100 s en plan Free). El cliente debe mantener `ping/pong` activo.

### 8.4 Otros puntos operativos

- **SSL:** Certbot con renovación automática (cron) o terminación en Cloudflare. Incluir `ws.midominio.com` en el certificado.
- **Variables de entorno:** `.env` nunca en el repo; usar GitHub Secrets para inyectarlas en CI/CD (igual que ya haces en tu pipeline de `notasCreditos`). Generar credenciales de Reverb **distintas** para producción.
- **Cron:** `php artisan schedule:run` cada minuto. Tareas programadas:
  - `ExpirePendingBookings` **cada 5 minutos** — libera reservas sin pagar según el plazo del medio de pago (sección 5.5). Con lock y relectura de estado, para no cancelar una reserva cuyo webhook de pago está llegando en ese instante.
  - `RecalculatePriceCalendar` — extensión diaria del horizonte de precios.
  - `PurgeOldConversations` — retención diferenciada del chat (sección 16.8.1).
  - `FetchExchangeRates` — actualización diaria del tipo de cambio.
  - Generación de reportes.
- **Queue workers + Supervisor:** ya tienes experiencia directa con esto (Reverb/queue:work); mismo patrón aquí para `queue:work` de emails, webhooks de pago y recálculo del calendario de precios.
- **Proceso Reverb bajo Supervisor:** `autorestart=true`, `numprocs=1`. Si el proceso muere, el chat deja de entregar en vivo (aunque los mensajes se siguen guardando por HTTP) — conviene una alerta sobre ese proceso, no solo sobre el contenedor de la app.

---

