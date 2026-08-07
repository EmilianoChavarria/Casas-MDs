# Configuración


### Variables de entorno (`.env` de Laravel)
```
DB_CONNECTION=mysql
DB_HOST=mysql          # nombre del servicio en docker-compose, o host del gestionado
DB_PORT=3306
DB_DATABASE=rentas
DB_USERNAME=rentas_app
DB_PASSWORD=<generado seguro, guardado en GitHub Secrets>
```

### Buenas prácticas de configuración
- Usuario de aplicación (`rentas_app`) con permisos limitados a la BD específica (nunca usar `root` desde Laravel).
- Charset `utf8mb4` y collation `utf8mb4_unicode_ci` (soporte completo de emojis/acentos).
- Activar `innodb_buffer_pool_size` ajustado a ~70% de la RAM disponible si es self-hosted.
- Backups automáticos diarios vía `mysqldump` programado + subida a Cloudflare R2 (ver `04-almacenamiento-r2.md`):

```bash
# Cron diario (en el VPS)
0 3 * * * docker exec mysql_container mysqldump -u root -p$MYSQL_ROOT_PASSWORD rentas | gzip > /backups/rentas_$(date +\%F).sql.gz
```

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 1, 5 y 8.
