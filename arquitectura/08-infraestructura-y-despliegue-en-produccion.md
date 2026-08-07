# 8. Infraestructura y despliegue en producción


### 8.1 Topología

- 1 VPS (Hetzner CX32 o similar) para Laravel + MySQL + Redis (o MySQL gestionado aparte si el presupuesto lo permite).
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
volumes:
  mysql_data:
```

### 8.3 Otros puntos operativos

- **SSL:** Certbot con renovación automática (cron) o terminación en Cloudflare.
- **Variables de entorno:** `.env` nunca en el repo; usar GitHub Secrets para inyectarlas en CI/CD (igual que ya haces en tu pipeline de `notasCreditos`).
- **Cron:** `php artisan schedule:run` cada minuto (reportes, limpieza de reservas `pending` expiradas).
- **Queue workers + Supervisor:** ya tienes experiencia directa con esto (Reverb/queue:work); mismo patrón aquí para `queue:work` de emails y webhooks de pago.

---

