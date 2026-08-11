# Servicio: Base de Datos — MySQL 8

## ¿Para qué se usa?

Almacena todo el modelo relacional del sistema: propiedades, reservas, disponibilidad, clientes, pagos, auditoría (ver sección 5 del documento principal).


## Justificación

- Transaccional (ACID), crítico para evitar doble-booking en `bookings`/`availability`.
- Ecosistema maduro con Laravel/Eloquent (migraciones, seeders, ya es tu stack actual en `notasCreditos`).
- Dos modalidades posibles:

| Modalidad | Cuándo usarla |
|---|---|
| **Self-hosted en el mismo VPS (Docker)** | Fase inicial (100–1,000 usuarios); menor costo, tú controlas backups |
| **Gestionado (ej. DigitalOcean Managed MySQL, Amazon RDS)** | A partir de crecimiento medio/alto; failover automático, backups gestionados, réplicas de lectura con 1 clic |

**Recomendación:** iniciar self-hosted en Docker dentro del mismo VPS de DigitalOcean (ver `01-hosting-vps.md`) y migrar a un servicio gestionado cuando se llegue a la fase de escalado (sección 11 del doc principal, >10,000 usuarios).


## 💰 Precio y plan gratuito para desarrollo

| Opción | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Self-hosted (Docker local o en el VPS)** | $0 | ✅ Sí, 100% gratis e ilimitado en el tiempo — es la opción recomendada para todo el desarrollo |
| **DigitalOcean Managed MySQL** | Desde $15 USD/mes, sin plan gratuito propio | ⚠️ Parcial — no tiene free tier permanente, pero cuentas nuevas de DigitalOcean reciben ~$200 USD en créditos (60 días), suficientes para probar el servicio gestionado sin pagar de tu bolsillo |

**Recomendación:** no hay razón para pagar una base de datos gestionada durante el desarrollo; usar siempre MySQL self-hosted vía Docker (gratis) y reservar el servicio gestionado para producción en la fase de escalado.


## Ruta de creación (self-hosted, fase inicial)

Ya está definida en el `docker-compose.yml` del backend (ver sección 8.2 del documento principal). No requiere cuenta externa.


## Ruta de creación (gestionado, fase de escalado)

1. Crear cuenta en el proveedor elegido (ej. DigitalOcean: https://cloud.digitalocean.com).
2. Databases → Create Database Cluster → MySQL 8.
3. Elegir región cercana al VPS de la app (para minimizar latencia).
4. Configurar "Trusted Sources" para que solo el VPS del backend pueda conectarse (firewall a nivel de base de datos).
5. Copiar cadena de conexión y credenciales.


## Contrato / plan recomendado

- Self-hosted: sin costo adicional (incluido en el VPS), sin límite de tiempo.
- Gestionado (ejemplo DigitalOcean): plan básico desde $15 USD/mes (1 vCPU/1GB) hasta ~$60 USD/mes con réplica de lectura incluida (precios ago-2026, verificar vigencia en https://www.digitalocean.com/pricing/managed-databases).
- Revisar políticas de backup automático (retención típica: 7 días en plan básico).


## Configuración


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

Referenciado desde: `../arquitectura/`, secciones 1, 5 y 8.
