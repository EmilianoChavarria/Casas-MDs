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

-- ── Precios: temporadas y reglas (detalle completo en sección 15) ──
seasons          (id, name, start_date, end_date, recurrence ENUM(none,yearly),
                   adjust_type ENUM(percent,fixed_amount,absolute_price), adjust_value,
                   min_nights, priority, color, is_active)
season_property  (season_id FK, property_id FK)   -- pivot; sin filas = todas las propiedades

price_rules      (id, property_id FK NULL, season_id FK NULL,
                   day_type ENUM(weekday,weekend,holiday),
                   adjust_type ENUM(percent,fixed_amount,absolute_price), adjust_value,
                   min_nights, is_active)

holidays         (id, date, name, country)        -- alimenta day_type = holiday

price_calendar   (property_id FK, date, price, season_id FK NULL, min_nights, computed_at)
                  -- PK (property_id, date); CACHE materializado, no fuente de verdad

-- ── Promociones ──
promotions       (id, name, code UNIQUE NULL, type ENUM(percent,fixed_amount,free_nights),
                   value, max_discount, min_nights, min_amount,
                   booking_starts_at, booking_ends_at,   -- ventana para RESERVAR
                   stay_starts_at, stay_ends_at,         -- ventana de ESTANCIA
                   usage_limit, usage_limit_per_customer, used_count,
                   combinable, priority, is_active)
promotion_property    (promotion_id FK, property_id FK)  -- sin filas = todas
promotion_redemptions (id, promotion_id FK, booking_id FK, customer_id FK,
                        amount_discounted, redeemed_at)

availability     (id, property_id FK, date, status ENUM(available,blocked,booked), booking_id FK NULL)
                  -- UNIQUE (property_id, date)

bookings         (id, property_id FK, customer_id FK, checkin, checkout,
                   guests, status ENUM(pending,confirmed,cancelled,completed),
                   subtotal, discount_total, fees_total, taxes_total,
                   total_price, currency, source, ...)

booking_nights   (id, booking_id FK, date, price, season_id FK NULL, day_type)
                  -- snapshot congelado del precio noche por noche (sección 15.1)

payments         (id, booking_id FK, provider ENUM(stripe,mercadopago),
                   provider_ref, amount, currency, status, paid_at)

-- ── Chat en tiempo real (detalle completo en sección 16) ──
conversations    (id, customer_id FK, property_id FK NULL, booking_id FK NULL,
                   assigned_user_id FK NULL, status ENUM(open,pending,closed),
                   subject, last_message_at, guest_token CHAR(36) NULL)

messages         (id, conversation_id FK, sender_type ENUM(customer,admin,system),
                   sender_id NULL, body TEXT NULL, attachment_url NULL,
                   attachment_meta JSON NULL, client_uuid CHAR(36), read_at, created_at)

configurations   (id, key, value, type)

audit_logs       (id, auditable_type, auditable_id, action, old_values JSON,
                   new_values JSON, user_id, ip, created_at)
```

### 5.2 Relaciones clave

- `properties 1—N property_images`
- `properties N—N amenities` (pivot `property_amenity`)
- `properties 1—N availability` (una fila por día — permite bloqueos manuales y evita traslapes vía `UNIQUE(property_id, date)`)
- `bookings 1—1 payments` (o 1—N si hay pagos parciales)
- `bookings 1—N booking_nights` (desglose congelado del precio)
- `customers 1—N bookings`
- `seasons 1—N price_rules`
- `properties N—N seasons` (pivot `season_property`; **sin filas = la temporada aplica a todas**)
- `properties N—N promotions` (pivot `promotion_property`; misma convención)
- `promotions 1—N promotion_redemptions 1—1 bookings`
- `customers 1—N conversations 1—N messages`
- `conversations N—1 properties` y `N—1 bookings` (ambas opcionales: una conversación puede no estar anclada a nada)

### 5.3 Índices recomendados

- `properties(slug)` UNIQUE — para rutas SEO.
- `properties(status)` — filtrar activas.
- `bookings(property_id, checkin, checkout)` compuesto — validar traslapes rápido.
- `availability(property_id, date)` UNIQUE — ya mencionado, es la pieza clave anti-doble-booking.
- `bookings(status)`, `bookings(customer_id)`.
- Full-text o índice en `properties(name, address)` si se permite búsqueda libre.
- `seasons(is_active, start_date, end_date)` — resolver qué temporada cubre una noche.
- `price_calendar(property_id, date)` PK — lectura del calendario público de un mes.
- `promotions(code)` UNIQUE, `promotions(is_active, booking_starts_at, booking_ends_at)`.
- `promotion_redemptions(promotion_id, customer_id)` — validar `usage_limit_per_customer`.
- `conversations(status, last_message_at DESC)` — bandeja de entrada del admin.
- `messages(conversation_id, id DESC)` — historial con paginación keyset (no `OFFSET`).
- `messages(conversation_id, read_at)` — contar no leídos sin escanear el hilo completo.
- `messages(client_uuid)` UNIQUE — idempotencia ante reenvíos del cliente.

**Estrategia anti doble-booking:** al crear una reserva, usar transacción + `SELECT ... FOR UPDATE` sobre las filas de `availability` del rango de fechas, o directamente intentar `INSERT` de esas fechas como `booked` y capturar el error de UNIQUE constraint si alguna ya existe. Esto es más robusto que solo validar con un `WHERE` antes del insert (condición de carrera).

**Estrategia anti doble-canje de cupón:** el mismo patrón. Dentro de la **misma transacción** de la reserva, `SELECT ... FOR UPDATE` sobre la fila de `promotions` y verificar `used_count < usage_limit` **después** del lock, antes de incrementar. Validarlo en el `quote` no sirve: entre el quote y el pago pasan minutos.

⚠️ **Traslape de temporadas:** MySQL 8 no tiene *exclusion constraints* sobre rangos de fechas (PostgreSQL sí, con `EXCLUDE USING gist`). Por eso el traslape de `seasons` se valida en la capa de aplicación (FormRequest) y la ambigüedad restante se resuelve con la regla determinista de prioridad de la sección 15.3 — nunca se deja al azar del orden de consulta.

---

