# 7. Seguridad


### 7.1 JWT vs Sanctum

**Recomendado: Sanctum.** Para un frontend propio (Next.js, mismo dominio/subdominios) Sanctum con cookies SPA es más simple y seguro (httpOnly cookie, no expones el token a JS, protección CSRF nativa). JWT solo aportaría valor si tuvieras múltiples clientes desacoplados (apps móviles de terceros, microservicios externos) — no es el caso aquí. Si más adelante agregas app móvil nativa, puedes usar tokens personales de Sanctum (Bearer) sin migrar a JWT.

### 7.1.1 Autenticación con Google (OAuth) — reglas obligatorias

Se integra **Laravel Socialite** con scopes no sensibles (`openid`, `email`, `profile`). Costo $0 y sin proceso de verificación de Google — ver [`servicios/14-auth-google-oauth.md`](../servicios/14-auth-google-oauth.md).

Estas cinco reglas no son recomendaciones; omitir cualquiera abre una vía de apropiación de cuentas:

| # | Regla | Qué pasa si se omite |
|---|---|---|
| 1 | **Verificar el claim `email_verified` de Google antes de vincular por correo.** Si viene `false`, no vincular: pedir confirmación por correo | Un atacante crea una cuenta en un proveedor con el correo de la víctima y entra como ella. Es el vector clásico de *account takeover* por OAuth |
| 2 | **Rechazar login con contraseña si `users.password IS NULL`** — cuentas creadas por OAuth | Comparar contra `null` tiene comportamiento indefinido según la implementación; en el peor caso una contraseña vacía valida |
| 3 | **Validar `redirect_to` contra lista blanca**: solo rutas relativas que empiecen con `/` y no con `//` | *Open redirect*: `?redirect_to=https://sitio-falso.com` envía al usuario a phishing **desde tu dominio**, heredando su credibilidad |
| 4 | **No permitir desvincular Google si es el único método de acceso.** Exigir establecer contraseña primero | El usuario se queda fuera de su propia cuenta, sin forma de recuperarla salvo soporte manual |
| 5 | **Conservar el `state` de OAuth** (Socialite lo hace, no desactivarlo) y aplicar `throttle` al callback | Sin `state`, un atacante puede forzar que la víctima vincule *su* cuenta de Google a la sesión del atacante |

**Sobre la vinculación automática por correo.** Con Google el `email_verified` es fiable —verifica tanto Gmail como Workspace— así que vincular automáticamente cuando coincide el correo *y* el claim es `true` es aceptable y evita cuentas duplicadas. Pero el claim **se lee, no se asume**. La regla general: *nunca confiar en un correo que el proveedor no declare verificado.*

**Cuentas OAuth y verificación de correo:** un usuario creado por Google llega con el correo ya verificado por Google, así que `email_verified_at` se marca en el momento. No se le manda correo de verificación — sería redundante y añade fricción justo donde OAuth la estaba quitando.

**Separación de roles:** un usuario con rol `guest` nunca debe pasar `EnsureUserIsAdmin`, aunque su cuenta se haya creado por Google. El proveedor de identidad autentica *quién eres*, no *qué puedes hacer*.

**Privacidad:** usar OAuth implica que Google conoce que la persona usa este sitio. Debe declararse en el aviso de privacidad del cliente.

### 7.2 Checklist de seguridad

| Área | Recomendación concreta |
|---|---|
| Roles y permisos | Tabla `roles` + policies de Laravel; opcionalmente `spatie/laravel-permission` si necesitas permisos granulares por módulo |
| Rate limiting | `throttle:api` en rutas públicas (ej. 60/min), más estricto en `/auth/login` (5/min) para evitar fuerza bruta, en `/conversations/{id}/messages` (30/min, anti-flood) y en `/promotions/validate` (10/min, evita enumerar códigos por fuerza bruta) |
| Canales WebSocket | Siempre `private`/`presence`, **nunca públicos** — un canal público lo lee cualquiera que adivine el ID. Autorización en `routes/channels.php` validando `customer_id` contra el usuario de Sanctum; `/broadcasting/auth` protegido por la misma sesión (sección 16.5) |
| Secretos de Reverb | `REVERB_APP_KEY` sí va al frontend; `REVERB_APP_SECRET` **jamás** — todo lo prefijado `NEXT_PUBLIC_` queda visible en el bundle |
| Contenido del chat | Guardar texto plano y renderizar como texto en React; `body` máx. 2,000 caracteres; adjuntos por el mismo pipeline validado de imágenes, con URL firmada y expiración |
| Acceso de visitante al chat | `guest_token` UUID v4 en cookie `httpOnly` + `Secure` + `SameSite=Lax`, expira a 30 días y se invalida al vincular la cuenta |
| Manipulación de precios | El precio **nunca** se acepta desde el cliente. `POST /bookings` recalcula el total en el servidor con `PricingService` e ignora cualquier importe enviado en el request (sección 15.1) |
| Abuso de cupones | `usage_limit` verificado con `SELECT ... FOR UPDATE` dentro de la transacción de reserva, más `usage_limit_per_customer` contra `promotion_redemptions` (sección 15.5) |
| Cambios de precios y promociones | Auditar en `audit_logs` toda alta/edición de `seasons`, `price_rules` y `promotions` — es dinero, y un cambio silencioso es difícil de rastrear después |
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

