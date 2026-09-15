# 5. Base de datos


### 5.1 Tablas principales (con auditoría estándar: `created_at, updated_at, created_by, updated_by, deleted_at`)

```
-- ── Identidad y autenticación (detalle en 5.6) ──
users            (id, name, email UNIQUE, email_verified_at,
                   password NULL,          -- NULL en cuentas creadas por OAuth
                   role_id FK, locale, is_active, ...)
roles            (id, name, slug)          -- admin, staff, guide, guest, cohost
                  -- cohost = dueño externo, solo lectura de sus casas (5.13)

social_accounts  (id, user_id FK, provider ENUM(google), provider_user_id,
                   email, avatar_url, created_at)
                  -- UNIQUE (provider, provider_user_id)

customers        (id, user_id FK NULL UNIQUE, first_name, last_name, email,
                   phone, document_id, country, locale, ...)
                  -- user_id NULL = reserva creada por el admin, sin cuenta

properties       (id, name, slug, description, address, lat, lng,
                   capacity, bedrooms, bathrooms,
                   base_price, base_currency CHAR(3) DEFAULT 'MXN',  -- D1
                   included_guests SMALLINT,                          -- D3
                   extra_guest_fee DECIMAL NULL,                      -- D3, por noche
                   cancellation_policy_id FK NULL,                    -- D7, NULL = general
                   status,
                   rating DECIMAL(2,1) NULL, reviews_count INT DEFAULT 0, ...)
                  -- rating y reviews_count: desnormalizados, ver 5.4
                  -- base_currency: el admin captura en la moneda que quiera;
                  --                el huésped siempre ve conversión (5.10)

favorites        (user_id FK, property_id FK, created_at)
                  -- PRIMARY KEY (user_id, property_id); ver 5.8

reviews          (id, booking_id FK UNIQUE, property_id FK, customer_id FK,
                   rating TINYINT, comment TEXT NULL, language CHAR(2),
                   host_reply TEXT NULL, host_replied_at,
                   status ENUM(published,hidden), hidden_reason, hidden_by FK NULL,
                   published_at, ...)

property_images  (id, property_id FK, url, is_cover, order, alt_text)

amenities        (id, name, icon, category)
property_amenity (property_id FK, amenity_id FK)   -- pivot

-- ── Co-anfitriones: dueños externos (detalle en 5.13) ──
property_cohost  (property_id FK, user_id FK, commission_percent DECIMAL(5,2),
                   created_at, updated_at)
                  -- PRIMARY KEY (property_id, user_id)
                  -- El porcentaje es del PAR, no de la persona: el mismo
                  -- dueño puede tener dos casas con tratos distintos, y
                  -- una casa puede tener dos dueños (un matrimonio).

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

-- ── Descuento por estancia larga (D4) ──
length_of_stay_discounts (id, property_id FK NULL, min_nights,
                   adjust_type ENUM(percent,fixed_amount), adjust_value,
                   priority, is_active)
                  -- property_id NULL = regla general; una fila por escalón
                  -- (p.ej. 7 noches -10%, 28 noches -25%). Gana el escalón
                  -- de mayor min_nights que cumpla la estancia, no se suman.

-- ── Políticas de cancelación (D7) ──
cancellation_policies (id, name, is_default BOOLEAN DEFAULT 0,
                   refund_rules JSON, notes)
                  -- refund_rules: [{ "hours_before": 720, "refund_percent": 100 },
                  --                 { "hours_before": 168, "refund_percent": 50 }]
                  -- La propiedad apunta a una; sin apuntar, hereda la default.

-- ── Tipo de cambio (D1) ──
exchange_rates   (id, base CHAR(3), quote CHAR(3), rate DECIMAL(12,6),
                   provider, fetched_at)
                  -- UNIQUE (base, quote, DATE(fetched_at)) — una tasa por día

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
                   guests, status ENUM(pending,confirmed,cancelled,expired,completed),
                   payment_method ENUM(card,oxxo,spei) NULL,
                   expires_at DATETIME NULL,        -- ver 5.5
                   subtotal, extra_guests_total, discount_total,
                   fees_total, taxes_total,
                   total_price, currency, source,
                   fx_rate DECIMAL(12,6) NULL,      -- congelada al confirmar (D1)
                   base_currency_total DECIMAL NULL, -- normalizado a MXN
                   cancellation_policy_snapshot JSON, -- la vigente al reservar (D7)
                   cohost_user_id FK NULL,          -- de quién era la casa AL RESERVAR
                   cohost_commission_percent DECIMAL(5,2) NULL,
                   cohost_commission_base DECIMAL NULL,
                   cohost_commission_amount DECIMAL NULL,
                   cohost_payout_amount DECIMAL NULL,
                  -- los cuatro CONGELADOS y en MONEDA BASE, no en la de
                  -- la reserva: al dueño se le liquida en pesos (5.13)
                   invoice_requested BOOLEAN DEFAULT 0, invoice_notes TEXT NULL,
                   terms_version_accepted VARCHAR, terms_accepted_at DATETIME, ...)
                  -- facturación: ver 5.7 · términos aceptados: ver 5.9

booking_nights   (id, booking_id FK, date, price, season_id FK NULL, day_type)
                  -- snapshot congelado del precio noche por noche (sección 15.1)

payments         (id, payable_type, payable_id,   -- POLIMÓRFICO: bookings o
                   provider ENUM(stripe,externo),  -- experience_bookings
                   provider_ref, amount, currency, status, paid_at)
                  -- UNIQUE (provider, provider_ref): los webhooks se
                  --   reintentan, y el mismo evento no debe crear dos pagos
                  -- 'externo': cobro fuera del sistema (efectivo, transferencia
                  --            directa, o reserva previa al lanzamiento)
                  -- INDEX (payable_type, payable_id) · ver sección 20.7

-- ── Chat en tiempo real (detalle completo en sección 16) ──
conversations    (id, customer_id FK, property_id FK NULL, booking_id FK NULL,
                   assigned_user_id FK NULL, status ENUM(open,pending,closed),
                   subject, last_message_at, guest_token CHAR(36) NULL)

messages         (id, conversation_id FK, sender_type ENUM(customer,admin,system),
                   sender_id NULL, body TEXT NULL, attachment_url NULL,
                   attachment_meta JSON NULL, client_uuid CHAR(36), read_at, created_at)

-- ── Experiencias / tours guiados (esquema completo en la sección 20.2) ──
experience_categories  (id, slug, name, order, is_active)      -- + traducciones
experiences            (id, name, slug, category_id FK, duration_minutes,
                         min_to_operate, decision_hours, meeting_lat/lng, ...)
experience_items       (id, experience_id FK, kind ENUM(included,excluded,guide_gear), label)
guides                 (id, user_id FK NULL UNIQUE, ..., bio, status)
experience_departures  (id, experience_id FK, guide_id FK NULL, starts_at,
                         capacity, min_to_operate, decision_hours, seats_taken,
                         status, review_token, is_private, private_token)
experience_private_requests (id, experience_id FK, group_size, preferred_date,
                         status, departure_id FK NULL, ...)
experience_bookings    (id, departure_id FK, customer_id FK, seats, total_price, status)
experience_attendees   (id, experience_booking_id FK, full_name, notes_encrypted)
experience_reviews     (id, departure_id FK, experience_id FK, guide_id FK,
                         rating_experience, rating_guide, consent_publish, status)

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
- `experiences 1—N experience_departures 1—N experience_bookings` (**el inventario es la salida, no el día** — sección 20.1)
- `guides 0..1—1 users` (mismo patrón nullable que `customers.user_id`, y por el mismo motivo: sección 20.4)
- `properties N—N users` (pivote `property_cohost`, **con atributo**: `commission_percent` — mismo patrón que `fee_property`; ver 5.13)
- `bookings N—0..1 users` por `cohost_user_id` (**congelado**: de quién era la casa al reservar, no de quién es hoy)
- `payments N—1 payable` (polimórfico: `bookings` o `experience_bookings`, sección 20.7)
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
- `bookings(cohost_user_id, status)` — el panel del dueño lista sus reservas por estado, y es su consulta más frecuente.

**Estrategia anti doble-booking:** al crear una reserva, usar transacción + `SELECT ... FOR UPDATE` sobre las filas de `availability` del rango de fechas, o directamente intentar `INSERT` de esas fechas como `booked` y capturar el error de UNIQUE constraint si alguna ya existe. Esto es más robusto que solo validar con un `WHERE` antes del insert (condición de carrera).

#### ⚠️ Supuesto: canal de venta único

**Toda la estrategia anterior protege contra carreras *dentro de este sistema*.** No protege contra ventas en plataformas externas: si la misma casa estuviera publicada en Airbnb o Booking, alguien podría reservar allí y este sistema seguiría vendiendo esas noches, porque nadie le avisó.

**Decisión: se asume canal único** — las casas se venden exclusivamente por este sitio. El cliente lo confirmó.

Si ese supuesto cambia, la solución estándar es sincronización **iCal** en ambos sentidos:

- **Exportar:** una URL `.ics` por propiedad que las plataformas externas consultan.
- **Importar:** un job periódico que lee los `.ics` externos y bloquea esas fechas con un estado nuevo `blocked_external` en `availability` (distinto de `blocked`, para que el admin no lo pueda desbloquear a mano y provocar un choque).

⚠️ Incluso con iCal, la sincronización **no es en tiempo real**: las plataformas refrescan cada 2–4 horas, así que queda una ventana de riesgo. La única alternativa de tiempo real son las APIs de partner de cada plataforma, que requieren aprobación comercial.

**Costo de equivocarse en este supuesto: acotado.** iCal es aditivo —una tabla, un job y un estado más— y se estima en 30–45 h. No obliga a rehacer el motor de reservas, a diferencia de multi-divisa. Por eso asumir canal único es una apuesta de bajo riesgo.

**Estrategia anti doble-canje de cupón:** el mismo patrón. Dentro de la **misma transacción** de la reserva, `SELECT ... FOR UPDATE` sobre la fila de `promotions` y verificar `used_count < usage_limit` **después** del lock, antes de incrementar. Validarlo en el `quote` no sirve: entre el quote y el pago pasan minutos.

### 5.4 Reseñas

**Regla que sostiene todo el diseño: solo reseña quien se hospedó.** `reviews.booking_id` es FK **único** y la reserva debe estar en estado `completed`. Una estancia, una reseña.

Eso elimina el spam **por diseño**, no por moderación. Sin cuenta con reserva completada no hay forma de escribir una reseña, así que no hace falta una cola de aprobación ni un administrador revisando texto.

**Publicación automática, ocultación excepcional.** Las reseñas se publican solas. El admin puede **ocultar** una concreta (datos personales, insultos, confusión evidente), pero queda registrado quién lo hizo y por qué (`hidden_by`, `hidden_reason`) y la fila se conserva.

⚠️ **Por qué no hay aprobación previa:** el dueño de las casas es también el dueño del sitio. Con moderación previa, tarde o temprano se rechazan las reseñas malas — y un listado donde todo son cinco estrellas no lo cree nadie. Las reseñas dejarían de aportar la confianza que justifica construirlas. La ocultación auditada da el control necesario sin invitar al sesgo sistemático.

**Ventana para reseñar:** configurable en `configurations` (`reviews.window_days`, sugerido 30 días tras el `checkout`). Sin límite, aparecen reseñas de estancias de hace dos años que ya no describen la casa actual.

#### `rating` y `reviews_count` desnormalizados

Viven como columnas en `properties` y se recalculan con un listener al publicar u ocultar una reseña. **No se calculan con `AVG()` en cada consulta:** el listado público muestra decenas de tarjetas y cada una necesitaría una agregación — exactamente el coste que la sección 11 marca como problema de escalabilidad.

`rating` es `NULL`, no `0`, cuando no hay reseñas. Son cosas distintas y la interfaz debe distinguirlas.

⚠️ **Al lanzar no habrá ninguna reseña.** Nadie ha completado una estancia todavía. Mostrar "0.0 ★" o cinco estrellas vacías resta credibilidad justo al arrancar. La interfaz debe **omitir el bloque de calificación** cuando `reviews_count = 0`, no pintarlo en cero.

#### Reseñas y multi-idioma

Un huésped francés escribirá su reseña en francés. **No se traducen automáticamente:** el contenido generado por usuarios traducido a máquina cae en el mismo problema de política de spam de Google que se resolvió en D2 — y ahí sí no hay forma de que un humano revise cada reseña.

Se guarda `reviews.language` con el idioma detectado y se muestra la reseña **en su idioma original**, etiquetada. Si se quiere ofrecer traducción, que sea a petición del lector y en el cliente, nunca contenido indexable.

#### Reseñas y SEO

Con reseñas reales se puede emitir `schema.org/AggregateRating` en las páginas de propiedad, lo que habilita **estrellas en los resultados de Google**. Es de las pocas mejoras de SEO con efecto visible inmediato en la tasa de clics, y encaja con la razón por la que el frontend público usa SSR.

⚠️ Solo con reseñas verificadas. Emitir datos estructurados de calificaciones inventadas o sin respaldo es motivo de acción manual de Google.

**Índices:** `reviews(booking_id)` UNIQUE, `reviews(property_id, status, published_at DESC)`, `reviews(customer_id)`.

`property_id` está desnormalizado en `reviews` a propósito: permite listar las reseñas de una casa sin pasar por `bookings`.

### 5.5 Expiración de reservas `pending`

Una reserva sin pagar aparta fechas. **El plazo lo determina el medio de pago**, no un número elegido a ojo:

| Medio | Plazo | Motivo |
|---|---|---|
| Tarjeta (Stripe / MP) | **30 minutos** | El cobro es inmediato; apartar más solo bloquea inventario |
| **OXXO / SPEI** | **El vencimiento real del voucher** que emite Stripe (típicamente ~3 días) | La persona tiene que ir físicamente a la tienda o hacer la transferencia |

⚠️ **`expires_at` se deriva del vencimiento que devuelve el proveedor, no de una constante.** Si Stripe emite un voucher válido 3 días y el sistema expira a las 48 h, alguien puede pagar en OXXO una reserva que ya se liberó — y quizá se revendió. Tomando la fecha del proveedor y añadiendo un pequeño margen, ese escenario deja de ser posible por construcción.

**Job cada 5 minutos** (`ExpirePendingBookings`): busca `status = pending AND expires_at < now()`, y por cada una, dentro de una transacción:

1. `SELECT ... FOR UPDATE` sobre la reserva y **releer su estado**.
2. Si sigue `pending`: marcarla `expired`, liberar las filas de `availability` y decrementar `used_count` del cupón si lo hubo.

⚠️ **El paso 1 no es opcional.** El webhook de pago y el job de expiración pueden ejecutarse en el mismo instante: el huésped paga justo cuando el plazo vence. Sin bloquear y releer, se cancela una reserva que acaba de pagarse. El webhook debe hacer la comprobación simétrica: si la reserva ya está `expired`, **no confirmarla** — hay que reembolsar y avisar.

**`expired` es un estado propio, distinto de `cancelled`.** Nadie canceló nada: se agotó el plazo. Mezclarlos ensucia los reportes, porque una tasa de abandono en el checkout y una tasa de cancelación miden problemas distintos.

**En la interfaz:** el huésped debe ver una cuenta regresiva del apartado. Un checkout que expira en silencio se percibe como un fallo del sitio, no como una regla.

### 5.6 Identidad: `users` vs. `customers`

Decisión: **una sola tabla de identidad (`users`) con rol, y `customers` como ficha del huésped.** Toda persona que inicia sesión —personal o huésped— vive en `users`. `customers` guarda los datos de contacto y facturación de quien reserva.

```
users ──1:0..1──> customers          (customers.user_id, nullable y único)
users ──1:N─────> social_accounts    (proveedores OAuth vinculados)
customers ──1:N─> bookings
```

**Por qué `customers.user_id` es nullable — y es la razón principal de este diseño:** el admin puede dar de alta una reserva de alguien que llamó por teléfono. Se crea el `customer` sin `user`. Si esa persona más adelante entra con Google usando el mismo correo, **se vincula a su ficha existente y ve su historial de reservas**. Con un modelo donde huésped y cuenta son lo mismo, esa reserva telefónica quedaría huérfana para siempre.

⚠️ **Actualización (28-ago-2026): reservar desde el sitio exige sesión.** El cliente pidió cerrar el checkout a cuentas, así que `POST /bookings` pasó al grupo `auth:sanctum` (detalle en 06). Esto NO cambia el esquema: `customers.user_id` sigue siendo nullable por el motivo del párrafo anterior —el alta telefónica desde el panel—, y la vinculación por correo al registrarse sigue funcionando igual. Lo que se cerró es una puerta del sitio público, no el modelo.

**Por qué `users.password` es nullable:** una cuenta creada con "Continuar con Google" nunca tuvo contraseña. Guardar un hash falso o una cadena vacía es peor — ver las reglas de la sección 7.3.

**Por qué `social_accounts` es tabla y no una columna `users.google_id`:** permite vincular varios proveedores a la misma cuenta y añadir Apple o Facebook después sin migrar. `UNIQUE (provider, provider_user_id)` impide que dos cuentas locales reclamen la misma identidad de Google.

⚠️ **`role_id` no es opcional en las consultas del panel.** Con huéspedes y personal en la misma tabla, cualquier listado de administración debe filtrar por rol. Un `User::all()` en una pantalla admin lista también a los huéspedes. Mitigación: un *global scope* o un modelo `Staff` con `where('role_id', '!=', guest)` de fábrica, en vez de confiar en que nadie lo olvide.

**Índices adicionales:** `users(email)` UNIQUE, `customers(user_id)` UNIQUE, `customers(email)`, `social_accounts(provider, provider_user_id)` UNIQUE.

#### Alta del personal: solo por invitación (D6)

**Decisión del cliente:** el administrador da de alta al personal con su correo. Esa persona puede entrar con Google, y si el correo coincide, se vincula a la cuenta que ya existe. **Sin correo dado de alta previamente, no hay acceso al panel.**

Es la forma correcta, y no solo la más cómoda: convierte el panel en un sistema **cerrado por invitación**. La alternativa —permitir que cualquiera se registre y luego darle rol— deja una ventana en la que existen cuentas sin rol definido en la misma tabla que los administradores.

```
users.role_id        -- asignado por quien invita, nunca por quien se registra
users.invited_at     -- alta creada desde el panel, sin contraseña todavía
users.password       -- NULL hasta que la persona la fija o entra con Google
```

⚠️ **La vinculación exige `email_verified = true` de Google** (regla de 7.1.1). Sin esa comprobación, quien logre crear una cuenta de Google con el correo de un administrador entra al panel con sus permisos. Es la diferencia entre "vinculamos por correo" y un *account takeover*.

⚠️ **El rol nunca se deduce del correo ni del dominio.** Se asigna explícitamente al invitar. Una regla del tipo "si el correo es de tal dominio, es admin" es una escalada de privilegios esperando a que alguien registre un alias.

### 5.7 Facturación (CFDI) — fuera del alcance inicial

**Decisión: el sistema no timbra facturas.** El huésped que la necesite marca una casilla en el checkout, se levanta un aviso al admin y la factura se emite por fuera (portal del contador o del SAT), con los datos de la reserva.

```
bookings.invoice_requested  BOOLEAN
bookings.invoice_notes      TEXT NULL     -- RFC y razón social si el huésped los deja
```

**Por qué no se integra un PAC ahora:**

- El perfil de huésped es mayoritariamente **turista de Canadá y Estados Unidos, sin RFC**, que nunca pedirá factura. Facturar a extranjeros se resuelve con el RFC genérico `XEXX010101000`, pero es un caso aparte y de volumen bajo.
- **CFDI 4.0 exige coincidencia exacta** de nombre, RFC, régimen fiscal y código postal del receptor contra la constancia de situación fiscal del SAT. Un dato mal capturado y el timbrado se rechaza. Automatizarlo obliga a pedir y validar esos cuatro campos **en el checkout** — fricción justo en la pantalla donde se decide la compra.
- Arrastra cancelación de CFDI y notas de crédito por cada reembolso, encadenado a la política de cancelación (D7), que aún no está definida.
- Con multi-divisa, el CFDI debe llevar la moneda de la operación y su tipo de cambio.

**Ruta de actualización si el volumen lo justifica:** integrar un PAC (Facturama, SW Sapien, Finkok) con una tabla `invoices` que guarde `uuid_sat`, serie, folio, datos fiscales del receptor, `uso_cfdi`, moneda, tipo de cambio y las URL del XML y el PDF. Estimado 30–45 h. **No requiere migrar nada** de lo anterior — `invoice_requested` sigue siendo el disparador.

⚠️ Confirmar con el contador (duda **D5**) cuántas facturas se emiten al mes. Si son muchas, esta decisión debe revisarse antes del lanzamiento: emitirlas a mano es trabajo que crece con las ventas.

### 5.8 Favoritos

```
favorites (user_id FK, property_id FK, created_at)
           PRIMARY KEY (user_id, property_id)   -- impide duplicados sin código extra
```

Persisten en servidor, así que sobreviven al cambio de dispositivo: alguien explora en el móvil y reserva desde la computadora encontrando lo que guardó.

**El motivo de fondo para llevarlos a base de datos es la señal de demanda.** Una casa que se guarda mucho y se reserva poco está diciendo algo — gusta, pero algo la frena: precio, fotos o disponibilidad. Ese dato solo existe si los favoritos se almacenan del lado del servidor, y alimenta `GET /admin/reports/favorites`.

#### ⚠️ El visitante anónimo

Requerir sesión introduce un problema de experiencia: alguien que está explorando pulsa el corazón y se topa con "inicia sesión". Interrumpir a un visitante por una función secundaria puede costar más de lo que aporta.

**Manejo obligatorio, no opcional:**

1. Al pulsar el corazón sin sesión, guardar la intención en `localStorage` y abrir el modal de acceso (con el botón de Google visible — son dos clics).
2. Tras iniciar sesión, **aplicar automáticamente** el favorito pendiente y quedarse en la misma página.

Sin el paso 1, el visitante inicia sesión, vuelve, y la casa que quería guardar no está marcada. Se pierde la intención justo después de haber pagado el costo de registrarse — que es la peor combinación posible.

**Índice:** la clave primaria compuesta ya cubre la consulta por usuario. Para el reporte de demanda, `favorites(property_id)`.

### 5.9 Aceptación de términos

```
bookings.terms_version_accepted   VARCHAR    -- ej. "2026-08-01"
bookings.terms_accepted_at        DATETIME
```

Cada reserva registra **qué versión de los términos aceptó el huésped y cuándo**. Los documentos legales se versionan por fecha y las versiones anteriores se conservan accesibles.

**Es el mismo principio que el precio congelado (15.1) y la política de cancelación congelada (D7):** ante una disputa por una cancelación, hay que poder demostrar qué condiciones estaban vigentes **en el momento de reservar**, no las de hoy. Sin este registro, cambiar los términos reescribe retroactivamente lo que cada huésped aceptó — que es justo lo que una disputa pone a prueba.

Las páginas viven en `/legal/{privacidad,terminos,cookies}`, en los tres idiomas, enlazadas desde el pie de página y desde el checkout. **El contenido lo aporta el cliente o su abogado** (duda **D8**); el desarrollo construye la estructura, el versionado y el registro de aceptación.

### 5.10 Moneda de captura y conversión (D1)

**Decisión del cliente:** el administrador captura el precio **en la moneda que quiera** —MXN, USD o CAD— y puede cambiar de una a otra. **El huésped siempre ve el precio convertido automáticamente** a su moneda.

```
properties.base_price      DECIMAL     -- el número que capturó el admin
properties.base_currency   CHAR(3)     -- en qué moneda lo capturó
```

Tres consecuencias que conviene tener claras antes de escribir el motor:

**1. `base_currency` no es una preferencia de visualización, es parte del precio.** Un `base_price = 150` significa cosas muy distintas con `MXN` o con `USD`. Guardar el número sin su moneda es el error clásico que aparece meses después, cuando alguien captura una casa en dólares.

**2. La moneda base de reportes es MXN, y es otra cosa.** Cada reserva guarda `fx_rate` y `base_currency_total` para que los ingresos se puedan sumar en una sola moneda sin volver a consultar el tipo de cambio de hace seis meses. Sin ese campo, un reporte anual mezcla tres monedas o depende de una tasa que ya cambió.

**3. La tasa se congela al confirmar.** Es el mismo principio del precio congelado (15.1): el total que aceptó el huésped no puede moverse porque el dólar subió al día siguiente.

⚠️ **El redondeo se aplica al mostrar, nunca al guardar.** Convertir, redondear, y volver a convertir para el cobro produce un total distinto del que vio el huésped. Se guarda el importe exacto y solo se redondea la presentación.

⚠️ **Cambiar `base_currency` de una propiedad no convierte el precio.** Si el admin pasa una casa de MXN a USD, `base_price = 2500` pasaría a significar 2,500 dólares. La interfaz debe ofrecer explícitamente convertir el importe o dejarlo tal cual, y no elegir por él.

---

### 5.11 Cargo por huésped adicional (D3)

```
properties.included_guests    SMALLINT    -- cuántos entran en el precio base
properties.extra_guest_fee    DECIMAL     -- por persona y POR NOCHE
```

El cargo entra al desglose **por noche**, junto al precio de la noche, y no como una línea suelta al final: así una estancia que cruza dos temporadas cobra el extra de cada noche que corresponde, y el congelado de `booking_nights` sigue siendo un reflejo fiel de lo cobrado.

⚠️ **`included_guests` no sustituye a `capacity`.** Son cosas distintas: `capacity` es el máximo legal y físico de la casa; `included_guests` es cuántos entran en el precio. Una casa puede admitir 8 e incluir 4.

---

### 5.12 Política de cancelación (D7)

**Decisión del cliente:** una política general, y la posibilidad de fijar una distinta para casas concretas.

```
cancellation_policies (id, name, is_default, refund_rules JSON, notes)
properties.cancellation_policy_id   FK NULL   -- NULL = hereda la default
```

Es el mismo patrón de herencia que ya usan las temporadas y las promociones en este esquema: **sin fila, aplica la general**. Evita duplicar la política en las 50 casas y evita que cambiar la general obligue a editarlas una por una.

**`refund_rules` es JSON a propósito.** Una política es una lista de escalones (`720 h antes → 100%`, `168 h antes → 50%`) cuyo número varía por política. Modelarlo como columnas obliga a inventar `refund_1`, `refund_2`… y a migrar cuando alguien quiera tres escalones.

⚠️ **La política se congela en la reserva** (`cancellation_policy_snapshot`). Si el cliente endurece la política en noviembre, quien reservó en agosto conserva la que aceptó. Sin el snapshot, un cambio de configuración reescribe retroactivamente el contrato de todas las reservas vivas — exactamente lo que una disputa de tarjeta pone a prueba.

---

⚠️ **Traslape de temporadas:** MySQL 8 no tiene *exclusion constraints* sobre rangos de fechas (PostgreSQL sí, con `EXCLUDE USING gist`). Por eso el traslape de `seasons` se valida en la capa de aplicación (FormRequest) y la ambigüedad restante se resuelve con la regla determinista de prioridad de la sección 15.3 — nunca se deja al azar del orden de consulta.

---

### 5.13 Co-anfitriones: dueños externos

Un dueño externo publica su casa en el sistema y queda como **co-anfitrión**: ve sus reservas, su ocupación y lo que le corresponde, pero **no da de alta ni modifica nada** — el administrador mantiene el control de qué se publica y a qué precio (1.1). De cada reserva, un porcentaje pactado se lo queda el administrador como comisión.

#### Por qué un pivote y no `properties.cohost_user_id`

Porque **el porcentaje es del par casa-dueño**, no de la persona ni de la casa. El mismo dueño puede tener dos casas negociadas a porcentajes distintos, y una casa puede tener dos dueños (un matrimonio, unos hermanos). Una columna en `properties` obligaría a inventar una tabla aparte para el porcentaje en cuanto aparezca el primer caso. Es el mismo patrón de `fee_property`, el otro pivote del proyecto con un atributo por par.

#### ⚠️ Por qué la comisión se congela en la reserva

Las cinco columnas `cohost_*` de `bookings` se escriben al crear la reserva y **no se vuelven a tocar**. No se calculan al leer el reporte.

El motivo es el mismo que el de `cancellation_policy_snapshot` y `booking_nights`: **renegociar no puede reescribir el pasado**. Si el trato pasa del 15% al 18%, las reservas ya liquidadas tienen que seguir diciendo 15%. Calculando al leer, un cambio de porcentaje reescribiría en silencio el histórico de liquidaciones, y el fallo se descubriría cuando el co-anfitrión reclame una cifra que ya no coincide con lo que cobró.

Por la misma razón se guardan **los importes resueltos y no solo el porcentaje**: la base sobre la que se aplica es configurable (`cohost.commission_base`, ver **D14**), así que con solo el porcentaje, cambiar esa regla recalcularía todo el pasado.

⚠️ **Los cuatro importes van en MONEDA BASE**, no en la de la reserva. Al dueño se le liquida en pesos; guardarlos en la moneda del huésped obligaría a sumar dólares con pesos al totalizar el reporte — exactamente el fallo que ya se corrigió en la conciliación de pagos.

⚠️ `cohost_user_id` es **nullable**: la mayoría de las casas son del negocio y no tienen dueño externo. Sin co-anfitrión, las cinco columnas quedan en `NULL` y el alta de la reserva no cambia en nada.

#### El aislamiento es un filtro, no una policy

Un co-anfitrión no debe ver el negocio de otro. Esa separación **vive en las consultas**: todo endpoint de `/host/*` arranca resolviendo las casas del usuario por el pivote y filtra por ahí (ver 6 y 7). Es el mismo criterio que el panel del guía (20.4) y por el mismo motivo: **una policy no protege un listado**, porque un listado nunca pasa por `authorize()` fila a fila.

Las reservas del dueño se filtran además por `cohost_user_id` —el valor congelado— y no por el pivote: si una casa cambia de dueño, el nuevo no hereda las liquidaciones del anterior.

#### Qué no sale del servidor

**El correo y el teléfono del huésped no se exponen al co-anfitrión**, solo su nombre de pila. Con los datos de contacto, el dueño puede cerrar la siguiente reserva por fuera del sistema y saltarse al administrador. Es la misma línea de privacidad que ya siguen las reseñas públicas (5.4). Tampoco ve la comisión del procesador de pagos —la absorbe el administrador— ni nada de las casas que no son suyas.

