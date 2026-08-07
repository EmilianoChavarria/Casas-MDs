# 7. Seguridad


### 7.1 JWT vs Sanctum

**Recomendado: Sanctum.** Para un frontend propio (Next.js, mismo dominio/subdominios) Sanctum con cookies SPA es más simple y seguro (httpOnly cookie, no expones el token a JS, protección CSRF nativa). JWT solo aportaría valor si tuvieras múltiples clientes desacoplados (apps móviles de terceros, microservicios externos) — no es el caso aquí. Si más adelante agregas app móvil nativa, puedes usar tokens personales de Sanctum (Bearer) sin migrar a JWT.

### 7.2 Checklist de seguridad

| Área | Recomendación concreta |
|---|---|
| Roles y permisos | Tabla `roles` + policies de Laravel; opcionalmente `spatie/laravel-permission` si necesitas permisos granulares por módulo |
| Rate limiting | `throttle:api` en rutas públicas (ej. 60/min), más estricto en `/auth/login` (5/min) para evitar fuerza bruta |
| CSRF | Sanctum lo maneja automático en rutas SPA con cookies; API pura con Bearer no lo requiere |
| XSS | Next.js escapa por defecto; nunca usar `dangerouslySetInnerHTML` con contenido de usuario sin sanitizar |
| SQL Injection | Eloquent/Query Builder parametrizado siempre; nunca concatenar SQL crudo |
| CORS | Configurar `config/cors.php` solo con los dominios exactos del frontend (prod y staging) |
| Validación de archivos | FormRequest con `mimes:jpg,png,webp`, `max:5120`, validar dimensiones y re-procesar con Intervention Image antes de subir a S3 (evita payloads maliciosos disfrazados de imagen) |
| Logs | Laravel logging a stack (`daily` + envío a servicio externo tipo Papertrail/Logtail) |
| Auditoría | Tabla `audit_logs` + trait `HasAuditColumns` (ya usado en tu proyecto escolar) para trazabilidad de cambios en propiedades/precios/reservas |
| Backups | Backup automático diario de MySQL (mysqldump a S3/R2) + retención 30 días; probar restauración periódicamente |
| HTTPS | Let's Encrypt (Certbot) o Cloudflare Full (Strict); forzar HTTPS en Nginx (redirect 301) y `Secure` cookies |

---

