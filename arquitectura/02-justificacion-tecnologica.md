# 2. Justificación tecnológica


| Tecnología | Por qué | ¿Cambiaría algo? |
|---|---|---|
| **Next.js** | SSR/ISR necesario para SEO de propiedades públicas (Google indexa mejor que un SPA puro), buen soporte de imágenes optimizadas | No, es la mejor opción para este caso público + SEO |
| **Laravel 12** | Ecosistema maduro (colas, notificaciones, policies), coincide con tu experiencia actual en `notasCreditos` | Ninguno |
| **MySQL 8** | Relacional, transacciones ACID críticas para reservas (evitar doble-booking), soporte de JSON columns si se necesita | Podrías considerar PostgreSQL por mejores índices parciales/exclusion constraints para rangos de fechas — relevante también para validar traslape de **temporadas** (sección 15.3), que en MySQL hay que resolver en la aplicación. Aun así MySQL 8 es suficiente con locking correcto |
| **Laravel Reverb** | Servidor WebSocket oficial de Laravel para el chat huésped↔admin y notificaciones en vivo (sección 16). Self-hosted, sin costo por conexión ni por mensaje, integración nativa con `broadcast()` y Echo. Ya hay experiencia previa con Reverb + Supervisor en este stack | Pusher/Ably como plan B si mantener el proceso cuesta más que la licencia; migrar es cambiar variables de entorno (Reverb habla protocolo Pusher), no reescribir el frontend |
| **Sanctum (elegido sobre JWT)** | Ver sección 7.1 | — |
| **S3/R2** | R2 es más barato (sin egress fees) y compatible con API S3; recomendado si el tráfico de imágenes es alto | Preferencia: **Cloudflare R2** |
| **Redis** | Cache de catálogos + colas de trabajo (necesario para no bloquear el request en emails/pagos) | Ninguno |
| **Leaflet + react-leaflet** (mapas públicos) | Ya implementado en el prototipo; costo de salida mínimo (cambiar de proveedor de tiles es una línea). El mapa público es alto volumen y baja exigencia — no justifica pagar Google. Ver sección 17.9 | Los **tiles** sí requieren un proveedor con API key en producción; el servidor público de OSM prohíbe el uso comercial |
| **Google Places** (solo autocompletado admin) | Precisión de direcciones en México, superior a Nominatim; volumen mínimo (una alta por casa) muy por debajo del free tier | Obligatorio usar *session tokens* y restringir la key por dominio, o la facturación se dispara |
| **Docker + Docker Compose** | Reproducibilidad entre local/dev/prod | Ninguno |
| **GitHub Actions** | Ya lo usas en tu pipeline de `notasCreditos`, reutilizable | Ninguno |
| **VPS (DigitalOcean recomendado)** | Precio equiparable a Hetzner tras el ajuste de jun-2026, con mejor cobertura de regiones (US y EU) y ecosistema integrado (VPC, firewall, MySQL gestionado) | DO > Hetzner > Hostinger |

---

