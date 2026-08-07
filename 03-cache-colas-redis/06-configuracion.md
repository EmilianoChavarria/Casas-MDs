# Configuración


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
