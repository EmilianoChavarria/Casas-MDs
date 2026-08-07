# 11. Escalabilidad (100 → 100,000 usuarios)


| Etapa | Acciones |
|---|---|
| 100–1,000 usuarios | VPS único, cache Redis de catálogos, CDN para imágenes (ya con R2/Cloudflare) |
| 1,000–10,000 | Separar MySQL a servidor dedicado, añadir réplica de lectura, cache de queries pesadas (disponibilidad, precios), queue workers escalados horizontalmente |
| 10,000–100,000 | Balanceador (Nginx/Cloudflare Load Balancer) frente a varias instancias de Laravel, MySQL con réplicas de lectura + posible sharding por región, Redis Cluster, imágenes 100% en CDN con transformaciones on-the-fly |

Puntos clave transversales:
- **Cache:** Redis para catálogos de propiedades/amenidades (invalidar en cada `update`).
- **CDN:** Cloudflare/R2 para todas las imágenes, nunca servirlas desde el propio VPS.
- **Optimización de consultas:** eager loading (evitar N+1), índices ya definidos en sección 5.
- **Escalado horizontal:** contenedores stateless para `app`, estado (sesión, cache) siempre en Redis, nunca en disco local.

---

