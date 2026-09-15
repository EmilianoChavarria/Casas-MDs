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
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
```

Sin los headers `Upgrade` y `Connection`, el handshake WebSocket falla con un 400 y el cliente cae en reconexión infinita — es el error más común al desplegar Reverb detrás de un proxy.

⚠️ **`proxy_read_timeout` debe ser el timeout más laxo de toda la cadena, no el más estricto.** Quien detecta conexiones muertas es el `ping/pong` del protocolo, no el proxy. Con el valor por defecto de 60 s, Nginx cierra la conexión justo cuando toca el ping del servidor (que también sale cada 60 s) y el chat se cae solo cada minuto — con Cloudflare o sin él. Ver la cadena completa de timeouts en la sección 16.8.

### 8.4 Otros puntos operativos

- **SSL:** Certbot con renovación automática (cron) o terminación en Cloudflare. Incluir `ws.midominio.com` en el certificado.
- **Variables de entorno:** `.env` nunca en el repo; usar GitHub Secrets para inyectarlas en CI/CD (igual que ya haces en tu pipeline de `notasCreditos`). Generar credenciales de Reverb **distintas** para producción.
- **Cron:** `php artisan schedule:run` cada minuto. Tareas programadas:
  - `ExpirePendingBookings` **cada 5 minutos** — libera reservas sin pagar según el plazo del medio de pago (sección 5.5). Con lock y relectura de estado, para no cancelar una reserva cuyo webhook de pago está llegando en ese instante.
  - `RecalculatePriceCalendar` — extensión diaria del horizonte de precios.
  - `PurgeOldConversations` — retención diferenciada del chat (sección 16.10.1).
  - `FetchExchangeRates` — actualización diaria del tipo de cambio.
  - Generación de reportes.
- **Queue workers + Supervisor:** ya tienes experiencia directa con esto (Reverb/queue:work); mismo patrón aquí para `queue:work` de emails, webhooks de pago y recálculo del calendario de precios.
- **Proceso Reverb bajo Supervisor:** `autorestart=true`, `numprocs=1`. Si el proceso muere, el chat deja de entregar en vivo (aunque los mensajes se siguen guardando por HTTP) — conviene una alerta sobre ese proceso, no solo sobre el contenedor de la app.


### 8.5 Plantillas de despliegue (fase 17)

El esquema de 8.2 ya existe como archivos en `Casas_back/deploy/`, con su guía en `deploy/README.md`:

| Archivo | Qué resuelve |
|---|---|
| `docker/php/Dockerfile` | **Una sola imagen** para app, cola, scheduler y Reverb: cambia el comando, no el código, así nunca corre la cola con otra versión que la API. Trae `mysqldump` para los respaldos |
| `docker/php/php.ini` | OPcache con `validate_timestamps=0` y sin `expose_php` |
| `docker-compose.prod.yml` | Seis servicios con `name: casas`. MySQL y Redis **sin puertos publicados** |
| `nginx/api.conf`, `nginx/ws.conf` | TLS, redirección 301 y el proxy de Reverb con los timeouts de 8.3. Un solo certificado para `api.` y `ws.` |
| `supervisor/casas.conf` | La alternativa sin Docker |
| `.env.production.example` | Todas las variables de producción |

Diferencias con el boceto de 8.2:
- El frontend no va en el compose; va en Vercel (8.1).
- Nginx no monta el código: todo lo pasa a php-fpm.
- El **primer certificado** se emite con certbot `--standalone`, antes de levantar Nginx, porque sin los `.pem` Nginx no arranca. Las renovaciones van por webroot.

Validado en local con Docker:
- `docker compose config` del archivo de producción;
- `nginx -t` con un certificado de prueba;
- construcción de la imagen.
- la imagen contra un MySQL 8.0.46 en contenedor: todas las migraciones corren y `backup:run --only-db` genera un volcado con todas las tablas. El `mysqldump` de la imagen es el cliente de MariaDB 10.11 (`default-mysql-client` de Debian) y es compatible.

⚠️ **Cada despliegue reinicia `app`, `queue`, `scheduler` y `reverb`.** Con OPcache sin revalidar y la cola cargando el código al arrancar, no reiniciar deja la versión anterior corriendo sin ningún error visible.

---

