# 6. API REST (resumen de endpoints)


### Auth
```
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
GET    /api/v1/auth/me
POST   /api/v1/auth/refresh
```

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

