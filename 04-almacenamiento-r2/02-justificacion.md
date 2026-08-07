# Justificación

- **R2 sobre S3:** compatible con la API de S3 (Laravel lo soporta nativamente con el driver `s3`), pero **sin costos de egress** (transferencia de salida gratuita), lo cual es clave porque las imágenes de propiedades generan mucho tráfico de lectura.
- Integración directa con Cloudflare CDN (ver `../09-cloudflare-dns-cdn/`) para servir las imágenes con caché en el borde, cerca del usuario final.

**Cuándo usar S3 en su lugar:** si el equipo ya opera fuertemente en AWS (ej. usa Amazon SES para correo, Lambda, etc.) puede convenir mantener todo en un solo proveedor por simplicidad operativa. Para este proyecto, R2 es la opción con mejor costo-beneficio.

