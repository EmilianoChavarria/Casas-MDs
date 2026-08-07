# Ruta de creación


**Fase inicial (self-hosted, recomendado):** ya está definido como servicio en el `docker-compose.yml` (sección 8.2 del documento principal) — no requiere cuenta externa ni configuración adicional más allá del contenedor.

**Fase de escalado (gestionado, opcional):**
1. Crear cuenta en Upstash (https://upstash.com) o Redis Cloud (https://redis.com/redis-enterprise-cloud) — ambos tienen tier gratuito de entrada.
2. Crear base de datos Redis, elegir región cercana al VPS.
3. Copiar `REDIS_URL` / host, puerto y password.

