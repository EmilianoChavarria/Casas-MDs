# 6. API REST (resumen de endpoints)


### Auth
```
POST   /api/v1/auth/register                # faltaba: alta de huésped
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
GET    /api/v1/auth/me
POST   /api/v1/auth/refresh
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
POST   /api/v1/auth/email/verify/{id}/{hash}
```

### Auth con Google (OAuth)
```
GET    /api/v1/auth/google/redirect?redirect_to=/profile   # inicia el flujo
GET    /api/v1/auth/google/callback                        # retorno de Google
POST   /api/v1/auth/google/link                            # vincular (autenticado)
DELETE /api/v1/auth/google/unlink                          # desvincular
```

**Flujo completo** (Next.js desacoplado + Laravel API):

1. El navegador va a `GET /auth/google/redirect` — no es fetch, es navegación real.
2. Socialite redirige a Google con `state` (CSRF).
3. Google retorna a `/auth/google/callback`.
4. Laravel verifica, crea o vincula el `user`, y **abre sesión de Sanctum (cookie)**.
5. Redirige a `NEXT_PUBLIC_APP_URL + redirect_to`.

⚠️ **`redirect_to` debe validarse contra una lista blanca de rutas relativas.** Aceptar cualquier URL convierte el endpoint en un *open redirect*: `?redirect_to=https://sitio-falso.com` manda al usuario a una página de phishing **desde tu propio dominio**, con la confianza que eso implica. Solo se aceptan rutas que empiecen con `/` y no con `//`.

⚠️ **Cookie entre subdominios:** si Next.js vive en `midominio.com` y la API en `api.midominio.com`, hace falta `SESSION_DOMAIN=.midominio.com` y `SANCTUM_STATEFUL_DOMAINS` con ambos. Sin eso el callback abre sesión y el frontend no la ve.

### Público
```
GET    /api/v1/properties?checkin=&checkout=&price_min=&price_max=&amenities[]=&guests=
GET    /api/v1/properties/{slug}
GET    /api/v1/properties/{id}/availability?month=2026-08   # incluye precio por noche y min_nights
POST   /api/v1/bookings/quote                               # desglose de precio; NO aparta fechas
POST   /api/v1/promotions/validate                          # { code, property_id, checkin, checkout }
GET    /api/v1/bookings/{id}/status
POST   /api/v1/webhooks/stripe
```

### Huésped con sesión
```
POST   /api/v1/bookings                                     # ⚠️ exige sesión (ver abajo)
GET    /api/v1/me/bookings                                  # historial de la cuenta
```

⚠️ **Reservar exige sesión.** Es una decisión del cliente (28-ago-2026) y **se aparta de 5.6**, donde reservar como invitado era deliberado para no perder reservas por la fricción del registro. `POST /bookings` vive ahora en el grupo `auth:sanctum`; revertirlo es sacarlo de ese grupo, nada más.

`customers.user_id` **sigue siendo nullable** y no es contradicción: el admin captura reservas por teléfono de gente sin cuenta. Lo que cambió es quién puede reservar **desde el sitio**, no qué reservas puede haber en la base.

`GET /me/bookings` filtra por `customers.user_id`, no por `bookings.user_id`: la reserva cuelga de la ficha de cliente, y esa ficha es la que se vincula a la cuenta. Así una reserva telefónica aparece en el historial en cuanto la persona se registra con el mismo correo, sin tocar las reservas.

⚠️ **Con sesión, el dueño de la reserva lo decide la cuenta, NO el correo del formulario.** `BookingService::resolveCustomer()` busca primero la ficha de `user_id` y solo cae al correo si no existe; y nunca adopta una ficha que ya es de otra cuenta. Sin esa regla, escribir el correo de otra persona colgaría la reserva de SU ficha y saldría en el historial de esa persona. Reservar a nombre de un tercero es legítimo; heredar su historial no.

### Reseñas (sección 5.4)
```
GET    /api/v1/properties/{slug}/reviews?page=      # público, solo published
POST   /api/v1/bookings/{id}/review                 # solo el huésped, reserva completed
PUT    /api/v1/reviews/{id}                         # editar dentro de la ventana
GET    /api/v1/me/reviews                           # reseñas del huésped autenticado

GET    /api/v1/me/favorites                         # favoritos del huésped
POST   /api/v1/me/favorites/{propertyId}
DELETE /api/v1/me/favorites/{propertyId}

POST   /api/v1/admin/reviews/{id}/reply             # respuesta del anfitrión
PATCH  /api/v1/admin/reviews/{id}/hide              # { reason } — nunca borra la fila
PATCH  /api/v1/admin/reviews/{id}/unhide
```

`POST /bookings/{id}/review` valida tres cosas: que la reserva sea del huésped autenticado, que esté en estado `completed`, y que esté dentro de `reviews.window_days` desde el `checkout`. El `UNIQUE` sobre `booking_id` impide la segunda reseña sin necesidad de comprobarlo en código.

### Chat (sección 16)
```
# Cliente con sesión — filtran por pertenencia: una ajena responde 404
GET    /api/v1/me/conversations                             # con unread_count y último mensaje
POST   /api/v1/me/conversations                             # { body, client_uuid, subject?, property_slug?, booking_code? }
GET    /api/v1/me/conversations/{id}                        # conversación + últimos 50 mensajes
GET    /api/v1/me/conversations/{id}/messages?before_id=|after_id=   # keyset, no offset
POST   /api/v1/me/conversations/{id}/messages               # { body?, client_uuid, attachment? } — throttle 30/min
POST   /api/v1/me/conversations/{id}/read                   # marca leído lo que le llegó

# Visitante sin cuenta, por su liga (16.4) — sin socket, consulta cada 5 s
GET    /api/v1/conversations/guest/{token}
GET    /api/v1/conversations/guest/{token}/messages?after_id=
POST   /api/v1/conversations/guest/{token}/messages         # throttle 20/min
POST   /api/v1/conversations/guest/{token}/read

POST   /api/v1/experiences/{slug}/private-requests          # abre la conversación: { conversation_id, token, has_account }

POST   /api/broadcasting/auth                               # autorización de canales (Sanctum, cookie del SPA)
```

Mensajes idempotentes por `client_uuid`: un reintento devuelve el mensaje que ya existe. Adjuntos solo imagen (JPG/PNG/WebP, 8 MB).

`after_id` es el parámetro de reconexión: tras una caída del WebSocket, el cliente pide lo que se perdió. El WS **no** reenvía historial.

### Admin
```
GET|POST|PUT|DELETE  /api/v1/admin/properties[/{id}]
POST                  /api/v1/admin/properties/{id}/images
DELETE                /api/v1/admin/properties/{id}/images/{imageId}
GET|POST|PUT|DELETE  /api/v1/admin/amenities[/{id}]

# Precios y temporadas
GET|POST|PUT|DELETE  /api/v1/admin/seasons[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/price-rules[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/holidays[/{id}]
GET                   /api/v1/admin/pricing/calendar?property_id=&from=&to=   # precios resueltos
POST                  /api/v1/admin/pricing/preview        # simular reglas antes de guardar
POST                  /api/v1/admin/pricing/recalculate    # encola RecalculatePriceCalendar

# Promociones
GET|POST|PUT|DELETE  /api/v1/admin/promotions[/{id}]
PATCH                 /api/v1/admin/promotions/{id}/toggle
GET                   /api/v1/admin/promotions/{id}/redemptions

PATCH                 /api/v1/admin/availability/{propertyId}   # bloquear/desbloquear fechas
GET|PUT               /api/v1/admin/bookings[/{id}]
PATCH                 /api/v1/admin/bookings/{id}/confirm
PATCH                 /api/v1/admin/bookings/{id}/cancel
GET|POST|PUT|DELETE  /api/v1/admin/customers[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/users[/{id}]

# Co-anfitriones: dueños externos (5.13)
GET                   /api/v1/admin/cohosts                      # dueños, sus casas y porcentajes
POST                  /api/v1/admin/cohosts                      # alta por invitación, role=cohost
POST                  /api/v1/admin/cohosts/{id}/properties      # { property_id, commission_percent }
DELETE                /api/v1/admin/cohosts/{id}/properties/{propertyId}
DELETE                /api/v1/admin/cohosts/{id}                 # desactiva la cuenta

# Chat (bandeja de administración)
# Administradores y personal; guías no.
GET                   /api/v1/admin/conversations?status=&search=&mine=   # paginada, con unread_count
GET                   /api/v1/admin/conversations/unread-count
GET                   /api/v1/admin/conversations/{id}           # con cliente, anclas y solicitud privada
GET                   /api/v1/admin/conversations/{id}/messages?before_id=|after_id=
POST                  /api/v1/admin/conversations/{id}/messages  # quien responde una sin dueño se la queda
POST                  /api/v1/admin/conversations/{id}/read
PATCH                 /api/v1/admin/conversations/{id}           # { status: open|pending|closed, assigned_user_id }

GET                   /api/v1/admin/reports/occupancy
GET                   /api/v1/admin/reports/revenue
GET                   /api/v1/admin/reports/promotions      # descuento otorgado por promoción
GET                   /api/v1/admin/reports/favorites       # señal de demanda: guardadas vs. reservadas
```

`POST /admin/pricing/preview` es deliberado: deja al admin ver el efecto de una temporada nueva sobre los próximos 12 meses **antes** de guardarla. Un error de configuración de precios es caro y silencioso.

### Experiencias — público (sección 20)
```
GET    /api/v1/experience-categories                         # las encendidas, traducidas (?lang=)
GET    /api/v1/experiences?category={slug}&date_from=        # sin salidas privadas
GET    /api/v1/experiences/{slug}
POST   /api/v1/experiences/{slug}/private-requests           # solicitud de salida privada; no aparta nada
GET    /api/v1/experiences/private/{token}                   # la salida privada, solo con su liga (20.5.1)
GET    /api/v1/experiences/{slug}/departures?month=2026-09   # fecha, hora, cupo restante, precio
GET    /api/v1/experiences/{slug}/reviews?page=              # solo published + consent_publish
GET    /api/v1/guides/{slug}                                 # perfil público del guía
POST   /api/v1/experience-bookings/quote                     # desglose; NO aparta cupo
POST   /api/v1/experience-bookings                           # { departure_id, seats, customer }
GET    /api/v1/experience-bookings/{id}/status
```

`POST /experience-bookings` es el único que aparta cupo, y lo hace con `lockForUpdate` sobre la salida (20.3). Devuelve **422 `not_enough_seats`** con el número real disponible en el cuerpo, para que la interfaz pueda corregir el selector en vez de mostrar un error genérico.

⚠️ `GET /experiences/{slug}/departures` es la consulta que alimenta el mini calendario. Devuelve `seats_left`, no `seats_taken`: el frontend no debe tener que restar, y el cupo total es información interna.

### Reseñas de experiencia — link externo, sin sesión
```
GET    /api/v1/r/{token}                     # datos del tour y del guía para pintar el formulario
POST   /api/v1/r/{token}                     # { rating_experience, rating_guide, comment?, consent }
```

Ambas rutas van con `throttle:5,60` por IP y fuera de `auth:sanctum`. El `GET` devuelve **404 para un token caducado o inexistente** — nunca un mensaje que confirme que el token existió.

### Experiencias — panel del guía (`role = guide`)
```
GET    /api/v1/guide/summary                        # carga de la semana
GET    /api/v1/guide/departures?from=&to=           # SOLO las suyas (scope, no policy)
GET    /api/v1/guide/departures/{id}                # roster, punto de encuentro, cosas necesarias (gear)
POST   /api/v1/guide/departures/{id}/complete       # finaliza y emite review_url (la del QR); 409 si no ha empezado
GET    /api/v1/guide/profile
PUT    /api/v1/guide/profile                        # SOLO bio (presentación), languages, whatsapp_e164
POST   /api/v1/guide/profile/photo
GET    /api/v1/guide/reviews                        # sus reseñas
```

Una salida de otro guía responde **404, no 403**: un 403 confirmaría que el id existe.

⚠️ Estos endpoints **filtran por `guide_id` en la consulta**, no solo con una policy. Un listado nunca pasa por `authorize()` fila a fila (20.4).

### Panel del co-anfitrión (`role = cohost`)
```
GET    /api/v1/host/summary                         # sus casas, llegadas próximas, mes en curso
GET    /api/v1/host/bookings?status=                # SOLO las de sus casas (scope, no policy)
GET    /api/v1/host/calendar?from=&to=              # ocupación, SIN precios
GET    /api/v1/host/report?from=&to=                # noches, generado, comisión, lo que le toca
PUT    /api/v1/host/profile                         # sus propios datos; el correo NO se cambia aquí
```

⚠️ Mismo criterio que el panel del guía: **estos endpoints filtran por pertenencia en la consulta**, no con una policy. Las reservas se filtran por el `cohost_user_id` **congelado** en la reserva, no por el pivote: si una casa cambia de dueño, el nuevo no hereda las liquidaciones del anterior (5.13).

⚠️ **Controladores propios, no los de `admin/`.** Los de administración no filtran por dueño —a un administrador le corresponde verlo todo— así que reutilizarlos aquí enseñaría el negocio de los demás. Se duplican a propósito, más pequeños.

⚠️ **Privacidad del huésped:** `host/bookings` devuelve **solo el nombre de pila**. El correo y el teléfono no salen del servidor: con ellos, el dueño puede cerrar la siguiente reserva por fuera del sistema. Tampoco viaja la comisión del procesador de pagos, que absorbe el administrador.

Los importes van en **moneda base**, aunque el huésped haya pagado en dólares: es lo que se le liquida al dueño.

### Experiencias — administración
```
GET|POST|PUT|DELETE  /api/v1/admin/experiences[/{id}]           # DELETE archiva; items { included, excluded, guide_gear } van en POST/PUT
POST|DELETE           /api/v1/admin/experiences/{id}/images[/{image}]   # 422 al borrar la única foto de una publicada

GET|POST              /api/v1/admin/experience-categories
PUT                   /api/v1/admin/experience-categories/{id}
PATCH                 /api/v1/admin/experience-categories/{id}/toggle

GET|POST|PUT          /api/v1/admin/guides[/{id}]                # show trae métricas, próximas salidas y reseñas
POST                  /api/v1/admin/guides/{id}/photo
POST                  /api/v1/admin/guides/{id}/invite           # crea el user con role=guide
POST                  /api/v1/admin/guides/{id}/deactivate       # 409 si tiene salidas futuras

GET                   /api/v1/admin/experience-departures?experience_id=&from=&to=   # máx. 2 meses; trae paid_seats
GET                   /api/v1/admin/experience-departures/metrics?month=YYYY-MM
POST                  /api/v1/admin/experience-departures        # fecha + hora de Cancún; repetición o privada
GET                   /api/v1/admin/experience-departures/{id}   # con reservas y asistentes
PATCH                 /api/v1/admin/experience-departures/{id}   # guía, cupo (422 bajo lo reservado), notas
POST                  /api/v1/admin/experience-departures/{id}/cancel     # { reason } — reembolsa y avisa
POST                  /api/v1/admin/experience-departures/{id}/complete   # 409 si no ha empezado

GET                   /api/v1/admin/experience-private-requests?status=
PATCH                 /api/v1/admin/experience-private-requests/{id}      # { status, admin_notes }

PATCH                 /api/v1/admin/experience-reviews/{id}/hide
PATCH                 /api/v1/admin/experience-reviews/{id}/unhide

GET                   /api/v1/admin/reports/experiences          # ocupación e ingresos
GET                   /api/v1/admin/reports/guides               # tours, ocupación, calificación
```

`POST /admin/experience-departures` acepta `repeat_weekdays: [2,4]` (ISO, 1 = lunes) y `repeat_until` (máx. 6 meses) y responde con el resumen del lote (`created`, `skipped` por colisión de `UNIQUE(experience_id, starts_at)`), no con un error si alguna fecha ya existía. Con `is_private` crea una sola salida con su `private_url`; si trae `private_request_id`, la solicitud pasa a `scheduled`.

Cupo, mínimo, horas para decidir y precio **se copian de la experiencia** a la salida al crearla (20.5); los que vengan en la petición los reemplazan solo para esa salida.

### Ejemplo de respuesta (`GET /api/v1/properties/{slug}`)

```json
{
  "data": {
    "id": 12,
    "name": "Casa Palmar",
    "slug": "casa-palmar",
    "description": "Casa frente al mar en Tulum...",
    "capacity": 8,
    "bedrooms": 4,
    "bathrooms": 3,
    "base_price": 2500,
    "currency": "MXN",
    "amenities": [
      { "id": 1, "name": "Alberca", "icon": "pool" },
      { "id": 2, "name": "WiFi", "icon": "wifi" }
    ],
    "images": [
      { "url": "https://cdn.example.com/1.jpg", "is_cover": true }
    ],
    "location": { "lat": 20.2114, "lng": -87.4654, "address": "Tulum, Q. Roo" }
  },
  "meta": { "currency": "MXN" }
}
```

Todos los listados usan paginación estándar de Laravel (`data`, `links`, `meta`).

---

