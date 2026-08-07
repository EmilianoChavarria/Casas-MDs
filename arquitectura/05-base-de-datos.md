# 5. Base de datos


### 5.1 Tablas principales (con auditoría estándar: `created_at, updated_at, created_by, updated_by, deleted_at`)

```
users            (id, name, email, password, role_id, is_active, ...)
roles            (id, name, slug)

customers        (id, first_name, last_name, email, phone, document_id, country, ...)

properties       (id, name, slug, description, address, lat, lng,
                   capacity, bedrooms, bathrooms, base_price, status, ...)

property_images  (id, property_id FK, url, is_cover, order, alt_text)

amenities        (id, name, icon, category)
property_amenity (property_id FK, amenity_id FK)   -- pivot

seasons          (id, name, start_date, end_date, priority)
price_rules      (id, property_id FK, season_id FK NULL, day_type ENUM(weekday,weekend,holiday),
                   price, min_nights)

availability     (id, property_id FK, date, status ENUM(available,blocked,booked), booking_id FK NULL)
                  -- UNIQUE (property_id, date)

bookings         (id, property_id FK, customer_id FK, checkin, checkout,
                   guests, status ENUM(pending,confirmed,cancelled,completed),
                   total_price, currency, source, ...)

payments         (id, booking_id FK, provider ENUM(stripe,mercadopago),
                   provider_ref, amount, currency, status, paid_at)

configurations   (id, key, value, type)

audit_logs       (id, auditable_type, auditable_id, action, old_values JSON,
                   new_values JSON, user_id, ip, created_at)
```

### 5.2 Relaciones clave

- `properties 1—N property_images`
- `properties N—N amenities` (pivot `property_amenity`)
- `properties 1—N availability` (una fila por día — permite bloqueos manuales y evita traslapes vía `UNIQUE(property_id, date)`)
- `bookings 1—1 payments` (o 1—N si hay pagos parciales)
- `customers 1—N bookings`
- `seasons 1—N price_rules`

### 5.3 Índices recomendados

- `properties(slug)` UNIQUE — para rutas SEO.
- `properties(status)` — filtrar activas.
- `bookings(property_id, checkin, checkout)` compuesto — validar traslapes rápido.
- `availability(property_id, date)` UNIQUE — ya mencionado, es la pieza clave anti-doble-booking.
- `bookings(status)`, `bookings(customer_id)`.
- Full-text o índice en `properties(name, address)` si se permite búsqueda libre.

**Estrategia anti doble-booking:** al crear una reserva, usar transacción + `SELECT ... FOR UPDATE` sobre las filas de `availability` del rango de fechas, o directamente intentar `INSERT` de esas fechas como `booked` y capturar el error de UNIQUE constraint si alguna ya existe. Esto es más robusto que solo validar con un `WHERE` antes del insert (condición de carrera).

---

