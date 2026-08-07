# 11. Escalabilidad (100 → 100,000 usuarios)


| Etapa | Acciones |
|---|---|
| 100–1,000 usuarios | VPS único, cache Redis de catálogos, CDN para imágenes (ya con R2/Cloudflare), un proceso Reverb en el mismo VPS |
| 1,000–10,000 | Separar MySQL a servidor dedicado, añadir réplica de lectura, cache de queries pesadas (disponibilidad, precios), queue workers escalados horizontalmente, Reverb en contenedor propio con `ulimit -n` ajustado |
| 10,000–100,000 | Balanceador (Nginx/Cloudflare Load Balancer) frente a varias instancias de Laravel, MySQL con réplicas de lectura + posible sharding por región, Redis Cluster, imágenes 100% en CDN con transformaciones on-the-fly, varias instancias de Reverb con `REVERB_SCALING_ENABLED=true` (pub/sub por Redis) tras balanceador con sticky sessions |

Puntos clave transversales:
- **Cache:** Redis para catálogos de propiedades/amenidades (invalidar en cada `update`).
- **CDN:** Cloudflare/R2 para todas las imágenes, nunca servirlas desde el propio VPS.
- **Optimización de consultas:** eager loading (evitar N+1), índices ya definidos en sección 5.
- **Escalado horizontal:** contenedores stateless para `app`, estado (sesión, cache) siempre en Redis, nunca en disco local.
- **Precios:** `price_calendar` materializado (sección 15.6) evita recalcular 30 noches en cada render de calendario. Se reconstruye por cola cuando cambian las reglas, no en el request.
- **Chat:** el historial crece rápido. Paginación **keyset** (`WHERE id < ? ORDER BY id DESC LIMIT 50`), nunca `OFFSET` — con 100k mensajes en un hilo, `OFFSET` degrada de forma lineal. Contadores de no leídos cacheados en Redis, no calculados con `COUNT(*)` en cada carga de la bandeja. Purga programada de conversaciones cerradas con más de 24 meses.
- **Métricas del WebSocket:** vigilar conexiones concurrentes, mensajes/segundo, memoria del proceso Reverb y tasa de reconexión del cliente. Una tasa de reconexión alta suele ser un timeout mal configurado en el proxy, no un problema de carga.

---

