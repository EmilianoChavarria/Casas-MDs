# Servicio: Cache y Colas — Redis

## ¿Para qué se usa?
1. **Cache** de catálogos poco cambiantes (propiedades, amenidades, temporadas) para reducir carga a MySQL.
2. **Colas (queues)** de Laravel: envío de correos, procesamiento de webhooks de pago, generación de reportes — todo lo que no debe bloquear el request HTTP.
3. Backend de **sesiones** (si se usa Sanctum SPA) para que el estado no dependa del disco local (requisito para escalado horizontal, sección 11).

## Justificación
Redis es el estándar de facto en el ecosistema Laravel para cache y colas; en memoria, muy rápido, soporta expiración automática (TTL) ideal para cache de catálogos.

## Ruta de creación

**Fase inicial (self-hosted, recomendado):** ya está definido como servicio en el `docker-compose.yml` (sección 8.2 del documento principal) — no requiere cuenta externa ni configuración adicional más allá del contenedor.

**Fase de escalado (gestionado, opcional):**
1. Crear cuenta en Upstash (https://upstash.com) o Redis Cloud (https://redis.com/redis-enterprise-cloud) — ambos tienen tier gratuito de entrada.
2. Crear base de datos Redis, elegir región cercana al VPS.
3. Copiar `REDIS_URL` / host, puerto y password.

## Contrato / plan recomendado
- Self-hosted: sin costo adicional.
- Upstash: tier gratuito hasta 10,000 comandos/día, plan pago desde ~$0.20 USD por 100K comandos (pay-as-you-go) — conveniente cuando el tráfico aún es variable.
- Redis Cloud: plan gratuito 30MB, planes pagos desde ~$5 USD/mes.

**Recomendación:** mantener self-hosted hasta que el volumen de colas/cache justifique un servicio gestionado con alta disponibilidad.

## Configuración

### Variables de entorno (`.env` de Laravel)
```
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
REDIS_HOST=redis        # nombre del servicio en docker-compose, o host gestionado
REDIS_PASSWORD=<generado seguro>
REDIS_PORT=6379
```

### Supervisor (persistencia del worker de colas — ya tienes experiencia previa con esto en Reverb/queue:work)
```ini
[program:rentas-queue-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
numprocs=2
user=deploy
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/queue-worker.log
```

### Cache con invalidación
```php
// Al leer catálogo
Cache::remember('amenities:all', now()->addHours(6), fn () => Amenity::all());

// Al modificar (en el Service correspondiente)
Cache::forget('amenities:all');
```

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 1, 8 y 11.
