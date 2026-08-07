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
POST   /api/v1/bookings
GET    /api/v1/bookings/{id}/status
POST   /api/v1/webhooks/stripe
POST   /api/v1/webhooks/mercadopago
```

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
GET    /api/v1/conversations                                # las del usuario autenticado
POST   /api/v1/conversations                                # { property_id?, booking_id?, subject? }
GET    /api/v1/conversations/{id}
GET    /api/v1/conversations/{id}/messages?before_id=&after_id=&limit=50   # keyset, no offset
POST   /api/v1/conversations/{id}/messages                  # { body, client_uuid, attachment? }
PATCH  /api/v1/conversations/{id}/read                      # marca leídos hasta un message_id
GET    /api/v1/conversations/unread-count

POST   /broadcasting/auth                                   # autorización de canales (Sanctum)
```

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

# Chat (bandeja de administración)
GET                   /api/v1/admin/conversations?status=open&assigned_to=
PATCH                 /api/v1/admin/conversations/{id}/assign
PATCH                 /api/v1/admin/conversations/{id}/close

GET                   /api/v1/admin/reports/occupancy
GET                   /api/v1/admin/reports/revenue
GET                   /api/v1/admin/reports/promotions      # descuento otorgado por promoción
GET                   /api/v1/admin/reports/favorites       # señal de demanda: guardadas vs. reservadas
```

`POST /admin/pricing/preview` es deliberado: deja al admin ver el efecto de una temporada nueva sobre los próximos 12 meses **antes** de guardarla. Un error de configuración de precios es caro y silencioso.

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

