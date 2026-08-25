# 20. Experiencias (tours guiados)

Producto **independiente de las casas**: salidas con fecha y hora fija, cupo limitado, precio **por persona** y un guía asignado. Se venden solas — no requieren reserva de casa.

Tres audiencias con panel propio (huésped, administrador, guía) más una **vista pública de captura de reseña** sin sesión.

> Este documento define el módulo completo: modelo de datos, reglas de negocio, API, pantallas, seguridad y su impacto en precios, pagos, notificaciones, servicios y estimación.

---

## 20.1 Por qué es un módulo aparte y no "una casa más"

Tentación razonable: reutilizar `properties` + `bookings` con un flag `type = experience`. **No conviene**, y el motivo es el inventario:

| | Casas | Experiencias |
|---|---|---|
| Unidad de inventario | **La noche** (`availability`, una fila por día) | **La salida** (fecha + hora + cupo) |
| Exclusividad | Una reserva ocupa la casa entera | N reservas comparten la misma salida |
| Precio | Por noche, variable por temporada | **Por persona**, fijo por salida |
| Límite | Capacidad informativa (`guests`) | **Cupo duro que bloquea la venta** |
| Cancelación del operador | Excepcional | **Rutinaria** (mínimo para operar) |
| Personal asignado | Ninguno | **Un guía** por salida |

Un `bookings.property_id` que a veces significa "la casa completa del 3 al 7" y a veces "3 lugares de 12 del sábado a las 9:00" convierte cada consulta de disponibilidad, cada reporte y cada policy en un `if`. **Tablas separadas, motor de reservas con la misma disciplina.**

**Lo que sí se reutiliza:** `customers`, `payments`, el motor de cargos e impuestos, las plantillas de correo, el sistema de diseño, la autenticación y `audit_logs`. La reutilización es de infraestructura, no de esquema de inventario.

---

## 20.2 Modelo de datos

Auditoría estándar en todas (`created_at, updated_at, created_by, updated_by, deleted_at`), igual que en la sección 5.

```sql
-- ── Catálogo ──────────────────────────────────────────────────────────
experiences        (id, name, slug UNIQUE, category ENUM(naturaleza,mar,gastronomia),
                     short_description, description,
                     duration_minutes, default_capacity, default_price, currency,
                     meeting_point_name, meeting_point_address,
                     meeting_lat, meeting_lng, meeting_notes,
                     what_to_bring TEXT NULL, cancellation_policy_id FK NULL,
                     status ENUM(draft,published,archived),
                     rating DECIMAL(2,1) NULL, reviews_count INT DEFAULT 0)
                    -- rating/reviews_count desnormalizados, mismo criterio que 5.4

experience_images   (id, experience_id FK, url, is_cover, order, alt_text)

experience_items    (id, experience_id FK, kind ENUM(included,excluded),
                     label, icon NULL, order)
                    -- "qué incluye" y "qué NO incluye" en una sola tabla:
                    -- son la misma lista con signo contrario

experience_translations (id, experience_id FK, locale CHAR(2),
                     name, short_description, description,
                     status ENUM(draft,machine,reviewed))
                    -- UNIQUE (experience_id, locale) — mismo flujo de D2

-- ── Guías ─────────────────────────────────────────────────────────────
guides             (id, user_id FK NULL UNIQUE, first_name, last_name, email,
                     phone_e164, whatsapp_e164 NULL, languages JSON,
                     certification VARCHAR NULL, certified_until DATE NULL,
                     base_zone VARCHAR, bio TEXT NULL, photo_url NULL,
                     rating DECIMAL(2,1) NULL, reviews_count INT DEFAULT 0,
                     tours_count INT DEFAULT 0,
                     status ENUM(active,inactive), deactivated_at, deactivated_reason)
                    -- user_id NULL = guía que NO entra al panel (ver 20.4)

guide_experience    (guide_id FK, experience_id FK)   -- pivot: qué puede guiar
                    -- sin filas = puede guiar cualquiera

-- ── Salidas (el inventario real) ──────────────────────────────────────
experience_departures (id, experience_id FK, guide_id FK NULL,
                     starts_at DATETIME, timezone VARCHAR,
                     capacity SMALLINT, min_to_operate SMALLINT DEFAULT 1,
                     seats_taken SMALLINT DEFAULT 0,      -- desnormalizado, ver 20.3
                     price_per_person DECIMAL, currency CHAR(3),
                     status ENUM(scheduled,confirmed,cancelled,completed),
                     cancelled_reason NULL, cancelled_at NULL,
                     internal_notes TEXT NULL,
                     review_token CHAR(32) NULL, review_token_expires_at NULL,
                     recurrence_group_id CHAR(36) NULL)   -- une las salidas creadas en lote
                    -- UNIQUE (experience_id, starts_at)
                    -- INDEX (starts_at, status), (guide_id, starts_at)

-- ── Reservas de experiencia ───────────────────────────────────────────
experience_bookings (id, departure_id FK, customer_id FK,
                     seats SMALLINT, unit_price DECIMAL,
                     subtotal, discount_total, taxes_total, total_price, currency,
                     fx_rate DECIMAL NULL, base_currency_total DECIMAL NULL,
                     status ENUM(pending,confirmed,cancelled,expired,completed),
                     payment_method ENUM(card,oxxo,spei) NULL,
                     expires_at DATETIME NULL,
                     terms_version_accepted VARCHAR, terms_accepted_at,
                     code CHAR(8) UNIQUE)                -- código corto para pasar lista
                    -- INDEX (departure_id, status)

experience_attendees (id, experience_booking_id FK, full_name,
                     age_band ENUM(adult,child) NULL,
                     notes_encrypted TEXT NULL)          -- ⚠️ dato sensible, ver 20.9
                    -- opcional: solo si el cliente quiere nombre por persona

-- ── Reseñas ───────────────────────────────────────────────────────────
experience_reviews (id, departure_id FK, experience_id FK, guide_id FK,
                     experience_booking_id FK NULL,
                     rating_experience TINYINT, rating_guide TINYINT,
                     comment TEXT NULL, language CHAR(2),
                     consent_publish BOOLEAN DEFAULT 0,
                     source ENUM(booking_link,departure_link),
                     status ENUM(published,hidden), hidden_reason, hidden_by FK NULL,
                     submitted_ip VARBINARY(16), published_at)
                    -- INDEX (experience_id, status, published_at DESC)
                    -- INDEX (guide_id, status, published_at DESC)
```

### Relaciones clave

- `experiences 1—N experience_departures 1—N experience_bookings`
- `guides 1—N experience_departures` (un guía por salida; una salida sin guía asignado es válida pero **no puede confirmarse**)
- `guides 0..1—1 users` (igual patrón que `customers.user_id`, sección 5.6)
- `experience_departures 1—N experience_reviews N—1 guides`
- `experience_bookings 1—N payments` (**polimórfico**, ver 20.7)

### Índices que no son opcionales

| Índice | Para qué |
|---|---|
| `experience_departures (experience_id, starts_at)` UNIQUE | Impide crear dos salidas idénticas al repetir un lote |
| `experience_departures (starts_at, status)` | Calendario admin y job de mínimo para operar |
| `experience_departures (guide_id, starts_at)` | Panel del guía — es su consulta principal |
| `experience_bookings (departure_id, status)` | Recuento de cupo y lista de asistentes |
| `experience_reviews (guide_id, status)` | Métricas del guía sin `AVG()` sobre todo |

---

## 20.3 El cupo: la regla que sostiene el módulo

> **El cupo por fecha manda sobre el selector de personas.** El selector de la interfaz es una comodidad; la verdad está en el servidor.

`seats_taken` es un **contador desnormalizado** en la salida, como `properties.rating`. Existe porque la tarjeta de reserva pinta el cupo restante de cada fecha del mini calendario: sin él, cada día del mes sería un `SUM()` sobre `experience_bookings`.

**No es la fuente de verdad para vender.** La reserva se hace así, dentro de una transacción:

```php
DB::transaction(function () use ($departureId, $seats) {
    $departure = ExperienceDeparture::whereKey($departureId)
        ->lockForUpdate()          // ← bloquea la fila hasta el commit
        ->firstOrFail();

    if (! in_array($departure->status, ['scheduled', 'confirmed'], true)) {
        throw new DepartureNotBookableException();
    }
    if ($departure->seats_taken + $seats > $departure->capacity) {
        throw new NotEnoughSeatsException($departure->capacity - $departure->seats_taken);
    }

    $departure->increment('seats_taken', $seats);
    // ... crear experience_booking pending + intención de pago
});
```

⚠️ **Sin `lockForUpdate` el módulo vende de más.** Dos personas mirando la última plaza pulsan "Reservar" en el mismo segundo: ambas leen `seats_taken = 11`, ambas ven que cabe una, ambas escriben `12`. Es el mismo fallo de doble-booking de la sección 5, con la diferencia de que aquí **es mucho más probable**: en una casa se compite por un rango de fechas amplio; aquí se compite por la última plaza de una salida concreta, y las últimas plazas se venden con prisa.

**El contador se corrige, no se confía.** Un comando `experiences:reconcile-seats` recalcula `seats_taken` desde `experience_bookings` y reporta discrepancias. Se ejecuta a diario. Cualquier contador desnormalizado sin reconciliación acaba mintiendo.

**Liberación de plazas:** al expirar (mismo job y mismos plazos de la sección 5.5) o al cancelar, se decrementa `seats_taken` dentro de la misma transacción que cambia el estado. Nunca fuera.

---

## 20.4 Los guías

### Un guía puede existir sin cuenta

`guides.user_id` es **nullable**, por la misma razón que `customers.user_id` (sección 5.6): el cliente trabaja con guías externos que quizá nunca entren al panel. El administrador da de alta al guía, lo asigna a salidas, ve sus métricas y sus reseñas — todo sin crear un acceso.

Cuando el guía sí necesita el panel, se le crea el `user` con `role = guide` y se vincula. **El alta de guía y el alta de usuario son dos acciones distintas**, y el modal de alta lo refleja: la casilla "dar acceso al panel" es opcional y dispara la invitación por correo.

✅ **Decidido (D9, 25-ago-2026): los guías sí entran, con panel propio.** Se dan de alta igual que el personal (5.6): el admin crea la cuenta con su correo y la persona puede entrar con Google si coincide. El alta de guía y el alta de usuario siguen siendo dos acciones distintas — un guía externo puntual puede quedarse sin acceso.

### Los guías solo ven lo suyo

Regla dura, con dos capas:

```php
// DeparturePolicy
public function view(User $user, ExperienceDeparture $departure): bool
{
    if ($user->isAdmin()) return true;
    return $user->guide?->id === $departure->guide_id;
}
```

Y **un scope obligatorio en las consultas del panel de guía**, no solo la policy:

```php
// En el controlador del panel de guía — nunca ExperienceDeparture::all()
$departures = ExperienceDeparture::where('guide_id', $user->guide->id)
    ->with('bookings.customer')
    ->upcoming()->get();
```

⚠️ **La policy protege el detalle; el scope protege el listado.** Es el mismo aviso de la sección 5.6 sobre `role_id`: una policy no impide que un listado devuelva filas de más, porque el listado nunca pasa por `authorize()` fila a fila. Se necesitan las dos.

**Lo que un guía NO ve, aunque sea de su salida:** el total pagado por cada reserva, el método de pago, el correo completo del cliente y sus datos de facturación. Ve **nombre, número de personas, si está pagado (sí/no) y las notas operativas**. Eso es lo que necesita para pasar lista y para no darle un cacahuate a quien es alérgico; el resto es información financiera del negocio.

### Dar de baja a un guía

`status = inactive` y `deactivated_at`. **Nunca borrar.** Sus reseñas históricas y sus salidas completadas deben seguir existiendo — son parte del historial del negocio y de las métricas de la experiencia.

⚠️ **Un guía no puede darse de baja si tiene salidas futuras asignadas.** La baja debe listar esas salidas y exigir reasignación antes de completarse. Sin esa comprobación, la baja crea salidas huérfanas que se descubren el día del tour.

---

## 20.5 Mínimo para operar y cancelación de salida

Una salida puede requerir un **mínimo de personas para operar** (`min_to_operate`). Es una regla de rentabilidad: llevar a un guía y una lancha por dos personas puede costar más de lo que ingresa.

**Máquina de estados de la salida:**

```
scheduled ──(seats_taken >= min_to_operate)──> confirmed ──(pasó la fecha)──> completed
    │                                              │
    └────────────(cancelada por admin)─────────────┴──> cancelled
```

| Estado | Significa | ¿Se puede reservar? |
|---|---|---|
| `scheduled` | Publicada, aún no alcanza el mínimo | ✅ Sí |
| `confirmed` | Alcanzó el mínimo, opera seguro | ✅ Sí, hasta el cupo |
| `cancelled` | No opera | ❌ No |
| `completed` | Ya ocurrió | ❌ No — habilita reseñas |

⚠️ **La interfaz pública no debe decir "confirmada" mientras esté en `scheduled`.** Si se cobra por adelantado y la salida se cancela por falta de gente, el huésped que creyó tener plaza garantizada tiene razón en quejarse. La tarjeta de reserva debe mostrarlo: *"Esta salida opera con un mínimo de 4 personas. Si no se alcanza, se cancela 24 h antes y se reembolsa el total."*

**Job `EvaluateDepartureMinimum`** — corre a diario y evalúa las salidas dentro de la ventana de decisión (`departures.decision_hours`, sugerido 24 h antes):

1. `seats_taken >= min_to_operate` → `confirmed`, avisar al guía y a los asistentes.
2. `seats_taken < min_to_operate` → `cancelled`, **reembolso íntegro** a todos, liberar, avisar.

⚠️ **Reembolso íntegro, sin excepción y sin descontar comisión del procesador.** Cancela el operador, no el huésped. Descontar la comisión de Stripe de un reembolso que el cliente no provocó es la clase de detalle que genera una disputa de tarjeta — y en una disputa, quien cancela pierde. Esto interactúa con **D7** y debe quedar por escrito en los términos (**D8**).

Cancelar a mano una salida hace exactamente lo mismo que la rama 2, pidiendo motivo (queda en `cancelled_reason` y en `audit_logs`).

---

## 20.6 El link de reseña — y su problema

**Requisito:** el link es **por tour y por guía**, y el guía lo comparte con el grupo al terminar.

Eso choca de frente con la regla que sostiene las reseñas de casas (sección 5.4): *solo reseña quien se hospedó*, garantizada por `reviews.booking_id UNIQUE` sobre una reserva `completed`. **Un link abierto que cualquiera puede reenviar no tiene esa garantía.** Con la URL, un competidor —o el propio guía— puede dejar reseñas.

Hay dos formas de emitirlo y **hay que elegir** (ver duda **D11**):

| | **A — Token por salida** (lo que pide el prototipo) | **B — Token por reserva** |
|---|---|---|
| Qué comparte el guía | Un solo link para todo el grupo | Un link distinto por reserva (correo/WhatsApp automático) |
| Fricción para el huésped | Mínima | Mínima (le llega a él) |
| ¿Reseña verificada? | ❌ No | ✅ Sí, equivale a `booking_id UNIQUE` |
| ¿`schema.org/AggregateRating`? | ❌ **No emitir** | ✅ Sí |
| Trabajo | Menor | +4–6 h |

✅ **Decidido (D11, 25-ago-2026): los dos.** Cada reserva recibe su link verificado por correo, y el guía conserva el link de grupo para pedirla en persona al terminar el tour. `experience_reviews.source` distingue el origen de cada una.

⚠️ **Solo las verificadas alimentan `schema.org/AggregateRating`.** Las que llegan por el link de grupo se muestran en el sitio y cuentan para las métricas internas del guía, pero **no** entran en los datos estructurados. Mezclarlas es exactamente lo que Google sanciona con acción manual, y perder el rich snippet por unas cuantas reseñas de grupo no compensa.

Esto significa dos promedios distintos en el sistema: el que se enseña en la página (todas las publicadas) y el que se emite en el marcado (solo verificadas). **Conviene que la interfaz de admin los muestre por separado**, o el primer reporte que no cuadre costará una tarde.

**Defensas mínimas del token por salida:**

1. **Se emite al pasar la salida a `completed`**, no antes. Un token que existe desde que se crea la salida circula semanas antes de que nadie haya ido.
2. **Caduca** (`review_token_expires_at`, sugerido 14 días). Coincide con la ventana de respuesta real y cierra la puerta después.
3. **Tope de reseñas = personas confirmadas de esa salida.** Si fueron 8, la novena entrega se rechaza. Es el límite que hace inútil el reenvío masivo del link.
4. **Rate limit por IP** sobre el endpoint público (`throttle:5,60`) y `submitted_ip` guardado, para poder ocultar en bloque un ataque evidente.

⚠️ **`review_token` debe ser aleatorio de 32 bytes (`Str::random(32)`), nunca el `id` de la salida ni un valor derivable.** Un token adivinable es no tener token.

**Publicación automática, ocultación auditada** — mismo criterio que 5.4, y por el mismo motivo: con moderación previa el dueño acaba filtrando las malas, y un listado de puros cincos no lo cree nadie.

**Doble calificación:** `rating_experience` y `rating_guide` son columnas distintas a propósito. Un tour precioso con un guía impuntual y un tour mediocre con un guía excelente son dos diagnósticos operativos opuestos; promediarlos en un número los borra. `guides.rating` sale de `rating_guide`; `experiences.rating` de `rating_experience`. Ambos se recalculan con listener al publicar u ocultar.

**Consentimiento de publicación** (`consent_publish`): sin él la reseña se guarda pero **no se muestra en público**. Sigue contando para las métricas internas del guía. Es requisito de LFPDPPP si el comentario lleva nombre.

⚠️ **Al lanzar no habrá ninguna reseña.** Igual que en 5.4: con `reviews_count = 0` la interfaz **omite el bloque de calificación**, no pinta "0.0 ★".

---

## 20.7 Pagos, precios e impuestos

### Los pagos se comparten; la tabla cambia

`payments` hoy tiene `booking_id FK`. Con experiencias hay **dos cosas pagables**:

```sql
-- Opción elegida: polimórfico
payments (id, payable_type, payable_id, provider, provider_ref,
          amount, currency, status, paid_at)
          -- INDEX (payable_type, payable_id)
```

⚠️ **Es una migración sobre una tabla existente con datos.** Si el módulo de experiencias entra después del lanzamiento, hay que migrar las filas (`payable_type = 'App\Models\Booking'`) y actualizar el webhook, la conciliación y los reportes de ingresos. **Cuesta 4–6 h si se hace desde el inicio, y bastante más si se hace con dinero real en producción.** Es el argumento para decidir *ahora* si el módulo entra, aunque se construya después.

La alternativa —una tabla `experience_payments` aparte— evita la migración pero **duplica el manejo de webhooks, la idempotencia y la conciliación**, que es la parte cara y delicada del módulo de pagos. No compensa.

### Precio

✅ **Cobro completo al reservar (D12).** No hay anticipo ni saldo en el punto de encuentro: el 100% se cobra en línea, lo que asegura el cupo y evita conciliar efectivo después de cada tour.

⚠️ **OXXO y SPEI solo si la salida cae después del vencimiento de la referencia.** Una referencia tarda hasta ~3 días en pagarse; ofrecerla para un tour del sábado aparta un cupo que probablemente expire sin pago. La pasarela debe ocultar esos métodos cuando no dan tiempo, no rechazarlos después.

**Precio por persona fijo por salida** (`departures.price_per_person`), copiado del catálogo al crear la salida y **congelado al reservar** (`experience_bookings.unit_price`). Mismo principio que `booking_nights` en la sección 15: el precio que vio el huésped es el que se cobra, aunque el catálogo cambie mañana.

**El motor de temporadas NO aplica a experiencias.** Una salida es una fecha concreta con un precio concreto; el admin lo fija al crearla. Meter `seasons` y `price_rules` aquí añadiría complejidad para resolver un problema que el formulario de "Nueva salida" ya resuelve capturando el precio.

**Multi-divisa sí aplica** (D1): si el huésped canadiense paga la casa en CAD, también la experiencia. Se reutiliza `exchange_rates` y el congelado de tasa (`fx_rate`, `base_currency_total`). Sin trabajo extra de infraestructura, pero sí de integración.

### Impuestos ⚠️

**Un tour no es hospedaje.** Los tres cargos de D5 no aplican igual:

| Cargo | Casas | Experiencias |
|---|---|---|
| IVA 16% | Aplica | **Aplica** (servicio) |
| ISH (Impuesto Sobre Hospedaje) | Aplica | ❌ **No** — grava el hospedaje, no los servicios turísticos |
| DSA / cuota por noche | Aplica | ❌ No |

⚠️ **Esto hay que confirmarlo con el contador del cliente**, no darlo por bueno desde aquí: la tasa de ISH y su base son estatales, y algunas entidades gravan servicios turísticos conexos. Ver **D10**. Si se aplica ISH a un tour por copiar la configuración de las casas, se está cobrando de más al huésped y declarando mal.

**Consecuencia técnica:** el motor de cargos e impuestos necesita **perfiles fiscales por tipo de producto**, no una configuración global. Es un cambio pequeño si se hace al construir el motor (fase 6) y una refactorización si se hace después.

---

## 20.8 Pantallas

### A. Público — huésped

**Navbar:** nueva entrada **"Experiencias"**. Ruta `/experiencias` (con prefijo de idioma, sección 4).

| Pantalla | Contenido | Notas técnicas |
|---|---|---|
| **Listado** `/experiencias` | Filtros por categoría (naturaleza · mar · gastronomía). Tarjetas: foto, duración, guía, calificación, precio por persona, disponibilidad | Server Component + ISR. La "disponibilidad" es la **próxima salida con cupo**, precalculada — no un `SUM()` por tarjeta |
| **Detalle** `/experiencias/[slug]` | Galería en mosaico · descripción · qué incluye / qué **no** incluye · perfil del guía (bio + métricas de confianza) · punto de encuentro con mapa · reseñas · tarjeta de reserva sticky | SSR. `schema.org/Event` por salida; `AggregateRating` **solo** si se eligió el token por reserva (20.6) |

**Tarjeta de reserva (sticky):** precio por persona → mini calendario de salidas → cupo restante por fecha → selector de personas **limitado al cupo de la fecha elegida** → desglose de total → política de cancelación.

⚠️ **El selector se recalcula al cambiar de fecha, y el servidor vuelve a validar al reservar.** Un límite que solo vive en el `max` del `<input>` no es un límite: se salta con las herramientas del navegador. La defensa real es 20.3.

⚠️ **El cupo restante que se pinta está desactualizado desde el momento en que se renderiza.** Con salidas casi llenas hay que refrescarlo al enfocar la pestaña y traducir el error `NotEnoughSeats` a un mensaje humano con el número real disponible — no a un "algo salió mal".

**Mapa del punto de encuentro:** la sección 17.9 ya decidió que los mapas públicos van con **Leaflet + proveedor de tiles**, no con Google. El prototipo de experiencias trae un *embed* de Google Maps.

> **Recomendación: reutilizar `SingleLocationMap` (Leaflet).** Costo adicional cero, un solo stack de mapas, coherencia visual. El punto de encuentro es exactamente el mismo problema que la ubicación de una casa.
>
> Si el cliente insiste en Google, la **Maps Embed API** (el `<iframe>` de `google.com/maps/embed`) es **gratuita y sin límite de uso** en su modo básico — no es la Maps JavaScript API y no consume su cuota. No añade costo, pero sí un segundo stack de mapas que mantener y una API key más que restringir. Ver [`servicios/08-google-maps.md`](../servicios/08-google-maps.md).

### B. Administración → Experiencias

Dos subsecciones bajo `/dashboard/experiences`.

**B.1 Gestión de tours** (`/dashboard/experiences/departures`)

| Bloque | Detalle |
|---|---|
| Métricas del mes | Salidas · ocupación media · personas confirmadas · promedio de reseñas |
| Calendario de salidas | Por experiencia; cada día muestra `reservados/cupo` y el guía |
| Panel de la salida seleccionada | Guía · límite de personas · hora · barra de ocupación · lista de asistentes · **cancelar salida** |
| Modal **Nueva salida** | Experiencia · fecha · hora · límite · mínimo para operar · guía · precio · **repetición por días de la semana × nº de semanas** · notas internas |
| Acción rápida | **Copiar link de reseña** |

⚠️ **La repetición genera N filas reales, no una regla.** Marcar "martes y jueves × 8 semanas" crea 16 salidas independientes con `recurrence_group_id` común. Una salida individual se edita o cancela sin tocar las demás — que es justo lo que hace falta cuando el guía se enferma un martes.

Al crear el lote hay que **avisar de las colisiones** (`UNIQUE(experience_id, starts_at)`) en vez de fallar entero: *"14 salidas creadas, 2 omitidas porque ya existían"*.

⚠️ **Reducir el límite de personas por debajo de `seats_taken` debe rechazarse.** Un cupo de 8 con 10 reservados es un tour que no cabe en la lancha. El formulario debe impedirlo y explicar por qué, no truncar en silencio.

**B.2 Guías** (`/dashboard/experiences/guides`)

| Bloque | Detalle |
|---|---|
| Modal de alta | Nombre · correo · WhatsApp · idiomas · certificación · zona base · *(opcional)* acceso al panel |
| Listado del equipo | Con estado, nº de tours y calificación |
| Detalle del guía | Métricas · sus tours asignados con estado · sus reseñas · su link de reseñas · **dar de baja** |

⚠️ El teléfono y el WhatsApp se guardan en **E.164** (`+521…`). Un campo de texto libre acaba con nueve formatos distintos y ningún `wa.me` que funcione.

### C. Panel del guía

Entrada propia **"Guía"** en la navbar, visible solo con `role = guide`. Ruta `/guia`.

| Pantalla | Contenido |
|---|---|
| Resumen de carga | Tours de la semana, personas totales, próxima salida |
| Próximos tours | Lista con fecha, hora, ocupación y estado |
| Detalle del tour | Asistentes (personas por reserva, **estado de pago sí/no**, notas operativas) · copiar / enviar link de reseña al grupo |
| Vista previa de la encuesta | Pestaña con la pantalla exacta que recibe el huésped |

La pestaña de vista previa no es adorno: un guía que sabe qué se le va a preguntar al grupo pide la reseña mejor. Es la misma pantalla del punto D, en modo solo lectura y sin token.

⚠️ **"Enviar el link al grupo"** se resuelve con **deep link de WhatsApp** (`https://wa.me/<e164>?text=<url>`) y `mailto:`, abiertos desde el dispositivo del guía. **No** con la WhatsApp Business Cloud API — que es un servicio de pago, requiere verificación de negocio, plantillas aprobadas por Meta y una integración completa. Ver 20.11.

### D. Captura de reseña (link externo) `/r/{token}`

Interfaz **móvil, de un solo paso**, sin sesión y sin navbar:

1. Estrellas 1–5 para **el tour**.
2. Estrellas 1–5 para **el guía** (con su foto y nombre — es a quien están calificando).
3. Comentario opcional.
4. Casilla de **consentimiento de publicación**.
5. Pantalla de agradecimiento con el resumen enviado.

`noindex` en la página. Un formulario de reseña indexado por Google es una invitación abierta.

---

## 20.9 Seguridad y datos personales

### ⚠️ Las notas de asistentes son datos sensibles

"Restricciones alimentarias", "alergia al marisco", "no nada", "usa silla de ruedas" **son datos de salud**. Bajo la LFPDPPP son **datos personales sensibles**, con tres consecuencias que no son opcionales:

1. **Consentimiento expreso** al capturarlos — una casilla explícita en el checkout, no un campo de "notas" ambiguo.
2. **Cifrado en reposo** (`notes_encrypted`, cast `encrypted` de Laravel), igual que `properties.access_instructions` en 19.2. Un volcado de base de datos no debe entregar el historial médico de los huéspedes.
3. **Acceso mínimo:** solo el admin y **el guía de esa salida concreta**. No aparecen en reportes, ni en exportaciones, ni en el listado general de reservas.

El aviso de privacidad (**D8**) debe mencionarlos explícitamente. Añadir a su checklist: *"datos de salud aportados voluntariamente para la seguridad del tour, conservados hasta X, accesibles solo por el guía asignado"*.

### Resto de controles

| Superficie | Control |
|---|---|
| `/r/{token}` | Público sin sesión · `throttle:5,60` por IP · token aleatorio 32 bytes · caducidad · tope por cupo |
| Panel de guía | `role = guide` + policy por `guide_id` + **scope en los listados** (20.4) |
| Datos financieros | El guía nunca ve totales, método de pago ni facturación |
| Baja de guía | Bloqueada si tiene salidas futuras |
| Cancelar salida | Requiere motivo · queda en `audit_logs` · dispara reembolsos |
| Reserva | `lockForUpdate` sobre la salida (20.3) |

---

## 20.10 Notificaciones

Extiende la sección 19. **Cuatro correos nuevos al huésped** y **tres avisos al guía**:

| # | Destinatario | Cuándo | Contenido |
|---|---|---|---|
| E1 | Huésped | Al reservar | Confirmación, salida, punto de encuentro, plazo de pago |
| E2 | Huésped | Al confirmarse el pago | Plaza asegurada + política de cancelación |
| E3 | Huésped | **Salida − 24 h** | Recordatorio: hora, punto de encuentro, qué llevar, contacto del guía |
| E4 | Huésped | Al cancelarse la salida | Motivo, reembolso íntegro y plazo |
| G1 | Guía | Al asignársele una salida | Fecha, hora, experiencia |
| G2 | Guía | **Salida − 24 h** | Roster: personas, notas operativas, estado de pago |
| G3 | Guía | Al cancelarse una salida suya | Motivo |

La solicitud de reseña **no es un correo nuevo**: la entrega el guía en persona al terminar (20.8.C). Si más adelante se quiere automatizar, es el correo E5 y encaja en el mismo job diario.

**Idempotencia:** `notification_log` de 19.4 pasa a polimórfico (`notifiable_type`, `notifiable_id`, `type`) con el mismo `UNIQUE`. Sin eso, el recordatorio de 24 h se envía dos veces al primer despliegue que solape el scheduler.

**Multi-idioma:** 4 correos × 3 idiomas = **12 textos nuevos** de huésped. Los del guía van en **un solo idioma** (el suyo, campo `guides.languages`) — es personal interno, no hace falta triplicarlos.

---

## 20.11 Impacto en servicios y costos

**Resumen: el módulo no obliga a contratar ningún servicio nuevo.** Detalle:

| Servicio | ¿Cambia? | Impacto |
|---|---|---|
| **VPS / MySQL / Redis** ([01](../servicios/01-hosting-vps.md), [02](../servicios/02-base-datos-mysql.md), [03](../servicios/03-cache-colas-redis.md)) | No | Volumen despreciable frente a las casas: decenas de salidas al mes, no millones de filas |
| **Cloudflare R2** ([04](../servicios/04-almacenamiento-r2.md)) | Marginal | + galería por experiencia (~10 fotos) + foto por guía. Con 20 experiencias y 15 guías, **< 1 GB** — dentro del free tier de 10 GB |
| **Stripe / Mercado Pago** ([05](../servicios/05-pagos-stripe.md), [06](../servicios/06-pagos-mercadopago.md)) | Sin costo fijo nuevo | Comisión por transacción, como siempre. ⚠️ Un tour de $800 MXN paga proporcionalmente **más comisión** que una reserva de $10,000: la parte fija (~$3 MXN + %) pesa mucho más en tickets pequeños. Conviene tenerlo en cuenta al fijar precios |
| **Correo** ([07](../servicios/07-correo-transaccional.md)) | Marginal | +7 plantillas; ~3 correos por reserva de experiencia. Resend free = 3,000/mes: sigue sobrando |
| **Google Maps** ([08](../servicios/08-google-maps.md)) | **$0 si se sigue la recomendación** | Leaflet reutilizado. Si se usa Maps **Embed** API: gratis e ilimitada, pero es un segundo stack. La Maps **JavaScript** API sí costaría (~$7 USD/1,000 cargas) — **no usarla** |
| **Tiles de mapa** ([13](../servicios/13-mapas-tiles.md)) | Marginal | Cada detalle de experiencia carga un mapa. Súmalo al cálculo de tiles/mes del proveedor elegido; sigue lejos de agotar cualquier free tier con este volumen |
| **Sentry / Cloudflare / Dominio** | No | — |
| **Reverb / WebSockets** ([12](../servicios/12-websockets-reverb.md)) | No | El módulo **no necesita tiempo real**. Un cupo que se actualiza al refrescar basta; empujarlo por WebSocket es complejidad sin beneficio a este volumen |
| **WhatsApp Business Cloud API** | ❌ **No contratar** | Deep link `wa.me`, gratis. La API de Meta implicaría verificación de negocio, plantillas aprobadas y costo por conversación — **desproporcionado** para "mandar un link al grupo" |

**Costo recurrente adicional del módulo: ~$0 USD/mes.** Lo que cuesta es el desarrollo (20.12).

⚠️ **Lo único que puede mover la aguja** es el volumen de tiles si las experiencias generan mucho tráfico público. Se vigila en el panel del proveedor; el plan de contingencia ya existe: cambiar la URL del `TileLayer` es una línea (17.9).

---

## 20.12 Estimación

| Parte | Horas | Complejidad | Riesgo |
|---|---|---|---|
| Modelo de datos + migraciones + seeders | 10–14 | Media | `payments` polimórfico toca datos existentes |
| CRUD de experiencias (catálogo, imágenes, incluye/no incluye) | 14–18 | Media | Galería en mosaico |
| Guías: CRUD, alta con/sin acceso, baja con validación | 10–14 | Media | Rol nuevo + policies |
| **Motor de salidas: cupo, lock, estados, repetición, mínimo para operar** | **20–26** | **Alta** | Sobreventa concurrente, reconciliación del contador |
| Reserva de experiencia + pagos + webhooks + expiración | 16–22 | **Alta** | Reutiliza pagos, pero polimórfico y con reembolso automático |
| Frontend público: listado, detalle, tarjeta sticky, mapa | 22–28 | Media-Alta | SEO/SSR, cupo desactualizado en cliente |
| Admin: calendario de salidas, panel de salida, modal de nueva salida | 20–26 | Media-Alta | El calendario es la pantalla más pesada del módulo |
| Admin: sección de guías | 8–12 | Media | — |
| **Panel del guía** (resumen, tours, roster, vista previa) | 14–18 | Media | Aislamiento de datos; móvil primero |
| Reseñas: captura pública, tokens, doble rating, agregados | 12–16 | Media-Alta | Vector de spam (20.6) |
| Notificaciones (7 plantillas; 12 textos multi-idioma) | 8–12 | Media | Textos del cliente |
| Impuestos por tipo de producto (perfil fiscal) | 4–8 | Media | Depende de D10 |
| Reportes de experiencias (ocupación, ingresos, guías) | 8–12 | Media | — |
| Pruebas (concurrencia de cupo, policies de guía, tokens) | 10–14 | Media-Alta | Lo de arriba no se valida a mano |
| **Total** | **~176–240 h** | | **~4.5–6 semanas** a tiempo completo, una persona |

**Efecto sobre el total del proyecto:** de **~468–644 h** (sección 13) a **~644–884 h**, es decir **~16–22 semanas**. Es un aumento del **~37%**: no es un añadido menor, es un producto nuevo dentro del mismo sistema.

### Si el calendario aprieta

| Recorte | Ahorro | Consecuencia |
|---|---|---|
| Panel del guía → sustituirlo por un PDF/correo con el roster | −14–18 h | El guía deja de ser usuario del sistema; el admin le manda la lista. Viable al arrancar con 2–3 guías |
| Reseñas de experiencias → diferir | −12–16 h | Se lanza sin calificaciones. Al principio no hay ninguna de todos modos (20.6) |
| Repetición de salidas → capturar una a una | −4–6 h | Trabajo manual del admin para siempre. **Mal recorte**: se paga cada semana |
| Multi-idioma del catálogo de experiencias | −6–8 h | Mismo criterio que D2 |

⚠️ **Lo que no se puede recortar:** el lock de cupo (20.3), el mínimo para operar con su reembolso (20.5) y el aislamiento de datos del guía (20.4). Los tres son correcciones caras si se retrofitan, y visibles para el cliente si fallan.

---

## 20.13 Plan de trabajo

El módulo entra como **fase 15 del roadmap** (sección 12), después del dashboard admin y el chat. Cinco iteraciones de una semana:

⚠️ **Una pieza se adelanta a la fase 2:** `payments` debe nacer **polimórfico** aunque el módulo se construya al final (20.7). Es la única dependencia que no se puede dejar para su turno.

| Sprint | Entregable demostrable | Depende de |
|---|---|---|
| **S1 — Cimientos** | Migraciones, modelos, seeders, `payments` polimórfico, rol `guide`. CRUD de experiencias y de guías en el panel | Fase 2 (BD), fase 9 (auth) |
| **S2 — Motor de salidas** | Crear salidas (con repetición), asignar guía, cupo con lock, estados, job de mínimo para operar, pruebas de concurrencia | S1 |
| **S3 — Venta** | Listado y detalle público, tarjeta sticky, reserva, pago, webhook, expiración, reembolso automático, correos E1–E4 | S2, fase 12 (pagos) |
| **S4 — Operación** | Calendario admin de salidas, panel de salida, sección de guías, panel del guía con roster, avisos G1–G3 | S2 |
| **S5 — Reseñas y cierre** | Captura `/r/{token}`, doble rating, agregados, vista previa de encuesta, reportes, hardening y QA | S3, S4 |

**Orden no negociable:** el motor de salidas (S2) va **antes** que cualquier pantalla de venta. Construir el checkout sobre un cupo sin lock significa reescribir el checkout.

**Se puede paralelizar** S3 y S4 si hay dos personas: comparten S2 y no se tocan entre sí.

### Precondiciones antes de arrancar

| Precondición | Quién | Bloquea |
|---|---|---|
| Respuestas a **D9**, **D10**, **D11**, **D12** | Cliente | S1 (D9), S3 (D10, D12), S5 (D11) |
| Contenido: fotos, descripciones, "qué incluye" de cada experiencia | Cliente | Publicar, no construir |
| Alta de guías: nombres, correos, certificaciones | Cliente | S4 |
| Textos de los 12 correos nuevos en ES/EN/FR | Cliente | S3 |
| **D5 y D8 actualizadas** con el tratamiento fiscal y de datos sensibles | Cliente + contador + abogado | **Producción** |

⚠️ **La captura de contenido no está en las horas de 20.12** — mismo aviso que en la sección 12: la estimación cubre construir el sistema, no llenarlo.

---

## 20.14 Checklist de verificación antes de publicar

- [ ] Reservar dos veces la última plaza en paralelo → una falla con mensaje claro, `seats_taken` correcto.
- [ ] `experiences:reconcile-seats` no reporta discrepancias tras 100 reservas simuladas.
- [ ] Reducir el cupo por debajo de los reservados → rechazado con explicación.
- [ ] Un guía autenticado no ve, ni llamando a la API directamente, salidas de otro guía.
- [ ] Un guía no ve totales ni métodos de pago en el roster.
- [ ] Salida bajo el mínimo → se cancela sola, reembolsa el 100% y avisa a todos.
- [ ] Token de reseña caducado y token con cupo agotado → ambos rechazados.
- [ ] Reseña sin consentimiento → guardada, no visible en público.
- [ ] Experiencia con 0 reseñas → no pinta "0.0 ★".
- [ ] Notas de asistentes cifradas en la BD (verificado con `SELECT` directo).
- [ ] `/r/{token}` responde `noindex`.
- [ ] Baja de guía con salidas futuras → bloqueada.
- [ ] El total de la experiencia lleva IVA y **no** lleva ISH ni DSA (según D10).

---
