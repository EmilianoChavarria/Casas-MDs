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
GET    /api/v1/properties/{id}/availability?month=2026-08
POST   /api/v1/bookings
GET    /api/v1/bookings/{id}/status
POST   /api/v1/webhooks/stripe
POST   /api/v1/webhooks/mercadopago
```

### Admin
```
GET|POST|PUT|DELETE  /api/v1/admin/properties[/{id}]
POST                  /api/v1/admin/properties/{id}/images
DELETE                /api/v1/admin/properties/{id}/images/{imageId}
GET|POST|PUT|DELETE  /api/v1/admin/amenities[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/seasons[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/price-rules[/{id}]
PATCH                 /api/v1/admin/availability/{propertyId}   # bloquear/desbloquear fechas
GET|PUT               /api/v1/admin/bookings[/{id}]
PATCH                 /api/v1/admin/bookings/{id}/confirm
PATCH                 /api/v1/admin/bookings/{id}/cancel
GET|POST|PUT|DELETE  /api/v1/admin/customers[/{id}]
GET|POST|PUT|DELETE  /api/v1/admin/users[/{id}]
GET                   /api/v1/admin/reports/occupancy
GET                   /api/v1/admin/reports/revenue
```

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

