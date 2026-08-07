# 5. Base de datos


### 5.1 Tablas principales (con auditoría estándar: `created_at, updated_at, created_by, updated_by, deleted_at`)

```
-- ── Identidad y autenticación (detalle en 5.4) ──
users            (id, name, email UNIQUE, email_verified_at,
                   password NULL,          -- NULL en cuentas creadas por OAuth
                   role_id FK, locale, is_active, ...)
roles            (id, name, slug)          -- admin, staff, guest

social_accounts  (id, user_id FK, provider ENUM(google), provider_user_id,
                   email, avatar_url, created_at)
                  -- UNIQUE (provider, provider_user_id)

customers        (id, user_id FK NULL UNIQUE, first_name, last_name, email,
                   phone, document_id, country, locale, ...)
                  -- user_id NULL = reserva creada por el admin, sin cuenta

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

### 5.4 Identidad: `users` vs. `customers`

Decisión: **una sola tabla de identidad (`users`) con rol, y `customers` como ficha del huésped.** Toda persona que inicia sesión —personal o huésped— vive en `users`. `customers` guarda los datos de contacto y facturación de quien reserva.

```
users ──1:0..1──> customers          (customers.user_id, nullable y único)
users ──1:N─────> social_accounts    (proveedores OAuth vinculados)
customers ──1:N─> bookings
```

**Por qué `customers.user_id` es nullable — y es la razón principal de este diseño:** el admin puede dar de alta una reserva de alguien que llamó por teléfono. Se crea el `customer` sin `user`. Si esa persona más adelante entra con Google usando el mismo correo, **se vincula a su ficha existente y ve su historial de reservas**. Con un modelo donde huésped y cuenta son lo mismo, esa reserva telefónica quedaría huérfana para siempre.

**Por qué `users.password` es nullable:** una cuenta creada con "Continuar con Google" nunca tuvo contraseña. Guardar un hash falso o una cadena vacía es peor — ver las reglas de la sección 7.3.

**Por qué `social_accounts` es tabla y no una columna `users.google_id`:** permite vincular varios proveedores a la misma cuenta y añadir Apple o Facebook después sin migrar. `UNIQUE (provider, provider_user_id)` impide que dos cuentas locales reclamen la misma identidad de Google.

⚠️ **`role_id` no es opcional en las consultas del panel.** Con huéspedes y personal en la misma tabla, cualquier listado de administración debe filtrar por rol. Un `User::all()` en una pantalla admin lista también a los huéspedes. Mitigación: un *global scope* o un modelo `Staff` con `where('role_id', '!=', guest)` de fábrica, en vez de confiar en que nadie lo olvide.

**Índices adicionales:** `users(email)` UNIQUE, `customers(user_id)` UNIQUE, `customers(email)`, `social_accounts(provider, provider_user_id)` UNIQUE.

⚠️ **Traslape de temporadas:** MySQL 8 no tiene *exclusion constraints* sobre rangos de fechas (PostgreSQL sí, con `EXCLUDE USING gist`). Por eso el traslape de `seasons` se valida en la capa de aplicación (FormRequest) y la ambigüedad restante se resuelve con la regla determinista de prioridad de la sección 15.3 — nunca se deja al azar del orden de consulta.

---

