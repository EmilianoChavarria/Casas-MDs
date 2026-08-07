# Servicio: Cache y Colas — Redis

## ¿Para qué se usa?

1. **Cache** de catálogos poco cambiantes (propiedades, amenidades, temporadas) para reducir carga a MySQL.
2. **Colas (queues)** de Laravel: envío de correos, procesamiento de webhooks de pago, generación de reportes — todo lo que no debe bloquear el request HTTP.
3. Backend de **sesiones** (si se usa Sanctum SPA) para que el estado no dependa del disco local (requisito para escalado horizontal, sección 11).


## Justificación

Redis es el estándar de facto en el ecosistema Laravel para cache y colas; en memoria, muy rápido, soporta expiración automática (TTL) ideal para cache de catálogos.


## 💰 Precio y plan gratuito para desarrollo

| Opción | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Self-hosted (Docker local o en el VPS)** | $0 | ✅ Sí, gratis e ilimitado — recomendado para desarrollo diario |
| **Upstash** | Free tier permanente: 256 MB y 500,000 comandos/mes, **sin tarjeta de crédito** | ✅ Sí, se puede usar indefinidamente para dev sin pagar nada |
| **Redis Cloud** | Free tier permanente: 30 MB | ✅ Sí, para pruebas muy pequeñas |

**Recomendación:** self-hosted vía Docker para desarrollo diario; Upstash es una buena alternativa gratuita si se quiere probar la integración con un Redis "en la nube" sin salir de la capa gratuita.


## Ruta de creación


**Fase inicial (self-hosted, recomendado):** ya está definido como servicio en el `docker-compose.yml` (sección 8.2 del documento principal) — no requiere cuenta externa ni configuración adicional más allá del contenedor.

**Fase de escalado (gestionado, opcional):**
1. Crear cuenta en Upstash (https://upstash.com) o Redis Cloud (https://redis.com/redis-enterprise-cloud) — ambos tienen tier gratuito de entrada.
2. Crear base de datos Redis, elegir región cercana al VPS.
3. Copiar `REDIS_URL` / host, puerto y password.


## Contrato / plan recomendado

- Self-hosted: sin costo adicional.
- Upstash: free tier permanente de 256 MB y 500,000 comandos/mes (sin tarjeta); plan pago pay-as-you-go desde $0.20 USD por 100,000 comandos + $0.25 USD/GB-mes de storage por encima de 1GB, ancho de banda gratis hasta 200GB/mes — conveniente cuando el tráfico aún es variable. Planes fijos desde $10 USD/mes.
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

Referenciado desde: `../arquitectura/`, secciones 1, 8 y 11.
