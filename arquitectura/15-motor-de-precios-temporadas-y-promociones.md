# 15. Motor de precios: temporadas y promociones

Esta sección desarrolla el módulo marcado como de **complejidad alta** en la sección 13. Amplía las tablas `seasons` y `price_rules` de la sección 5 con un motor de resolución por noche, temporadas con rangos de fechas automáticos y un subsistema de promociones.

---

### 15.1 Principio de diseño

**El precio nunca se guarda en la reserva "a mano": se calcula noche por noche y se congela al confirmar.**

Una reserva del 20 al 24 de diciembre no tiene "un precio". Tiene 4 noches, cada una con su propio precio resuelto según qué temporada la cubre y qué día de la semana es. El total es la suma, y sobre esa suma se aplican descuentos.

Reglas invariantes:

1. **Resolución por noche (`per-night`)**, nunca por rango completo. Una estancia puede cruzar temporada baja y alta.
2. **Determinismo:** las mismas fechas + mismas reglas activas ⇒ el mismo precio. Sin aleatoriedad ni dependencia del orden de consulta.
3. **Congelación (`price snapshot`):** al confirmar la reserva se guarda el desglose completo en `booking_nights` + `bookings.total_price`. Si el admin cambia una temporada después, las reservas ya confirmadas **no** se recalculan.
4. **Idempotencia del cálculo:** `POST /bookings/quote` y el cálculo interno de `POST /bookings` usan **el mismo** `PricingService`. Nunca dos implementaciones.

---

### 15.2 Pipeline de cálculo

```
                 base_price de la propiedad
                            │
       ┌────────────────────▼────────────────────┐
       │ 1. TEMPORADA (season)                    │  ← rango de fechas, prioridad
       │    override absoluto o ajuste ±%/±$      │
       └────────────────────┬────────────────────┘
                            │
       ┌────────────────────▼────────────────────┐
       │ 2. TIPO DE DÍA (day_type)                │  ← weekday / weekend / holiday
       │    ajuste ±% o precio fijo               │
       └────────────────────┬────────────────────┘
                            │
       ┌────────────────────▼────────────────────┐
       │ 3. AJUSTE POR OCUPACIÓN (opcional)       │  ← huéspedes extra sobre base
       └────────────────────┬────────────────────┘
                            │
                   precio de la noche
                            │
                    Σ (todas las noches)
                            │
       ┌────────────────────▼────────────────────┐
       │ 4. DESCUENTO POR ESTANCIA LARGA          │  ← weekly / monthly
       └────────────────────┬────────────────────┘
                            │
       ┌────────────────────▼────────────────────┐
       │ 5. PROMOCIONES                            │  ← código o automáticas
       └────────────────────┬────────────────────┘
                            │
       ┌────────────────────▼────────────────────┐
       │ 6. CARGOS (limpieza, servicio) + IMPUESTOS│
       └────────────────────┬────────────────────┘
                            │
                     TOTAL a pagar
```

Los pasos 1–3 producen el **precio por noche** (lo que se muestra en el calendario). Los pasos 4–6 solo existen a nivel de la estancia completa.

---

### 15.3 Temporadas (`seasons`)

Reemplaza la definición mínima de la sección 5.

```
seasons (
  id,
  name                 VARCHAR      -- "Temporada alta invierno", "Semana Santa"
  start_date           DATE         -- inclusive
  end_date             DATE         -- inclusive
  recurrence           ENUM(none, yearly)  DEFAULT none
  adjust_type          ENUM(percent, fixed_amount, absolute_price)
  adjust_value         DECIMAL(10,2)       -- 20.00 = +20% si percent
  min_nights           TINYINT NULL        -- estancia mínima obligatoria en esta temporada
  priority             SMALLINT DEFAULT 0  -- mayor gana ante traslape
  color                VARCHAR(7)          -- para pintar el calendario del admin
  is_active            BOOLEAN DEFAULT 1
  ...auditoría
)

season_property (season_id FK, property_id FK)   -- pivot; SIN filas = aplica a TODAS
```

**`adjust_type`:**

| Valor | Significado | Ejemplo con `base_price = 2500` |
|---|---|---|
| `percent` | Porcentaje sobre el precio base | `adjust_value = 20` → 3000 |
| `fixed_amount` | Suma/resta fija | `adjust_value = 500` → 3000 |
| `absolute_price` | Ignora la base, fija el precio | `adjust_value = 3000` → 3000 |

`adjust_value` admite negativos: `percent = -15` es una temporada baja.

**`recurrence = yearly`:** la temporada se repite cada año usando solo mes-día. "Temporada alta invierno" del 15-dic al 05-ene se define una vez y aplica en 2026, 2027, 2028… sin que el admin la reconfigure. El resolver expande la regla al año de la noche evaluada.

⚠️ **Rangos que cruzan el año** (15-dic → 05-ene): con `recurrence = yearly`, cuando `start_date > end_date` en mes-día, el rango se interpreta como *envolvente* — cubre desde el 15-dic hasta el 31-dic **y** desde el 01-ene hasta el 05-ene. Es el caso más común y hay que probarlo explícitamente.

**Resolución de traslapes.** Dos temporadas pueden cubrir la misma noche (ej. "Temporada alta invierno" y "Navidad"). El orden de desempate es fijo y en este orden:

1. Mayor `priority`.
2. Si empatan: la de **rango más corto** (más específica gana — "Navidad" de 5 días vence a "invierno" de 3 semanas).
3. Si empatan: la de `id` mayor (la más reciente).

**Solo gana una temporada por noche.** No se acumulan. Si se quisiera "+20% invierno *y además* +30% navidad", eso se modela como una temporada Navidad con `percent = 56` (el compuesto ya calculado), no encadenando ajustes. Encadenar hace el precio imposible de auditar.

**Validación al crear/editar** (`StoreSeasonRequest`): rechazar si ya existe otra temporada **activa, con la misma `priority`, que se traslape en fechas y comparta al menos una propiedad**. MySQL no tiene exclusion constraints sobre rangos (PostgreSQL sí — ver nota en sección 2), así que esto se valida en la capa de aplicación con una consulta de traslape:

```php
// Traslape clásico: (StartA <= EndB) AND (EndA >= StartB)
Season::query()
    ->where('is_active', true)
    ->where('priority', $priority)
    ->when($id, fn ($q) => $q->whereKeyNot($id))
    ->where('start_date', '<=', $endDate)
    ->where('end_date',   '>=', $startDate)
    ->whereHas('properties', fn ($q) => $q->whereIn('properties.id', $propertyIds))
    ->exists();
```

---

### 15.4 Reglas por tipo de día (`price_rules`)

```
price_rules (
  id,
  property_id  FK NULL          -- NULL = aplica a todas
  season_id    FK NULL          -- NULL = aplica fuera/dentro de cualquier temporada
  day_type     ENUM(weekday, weekend, holiday)
  adjust_type  ENUM(percent, fixed_amount, absolute_price)
  adjust_value DECIMAL(10,2)
  min_nights   TINYINT NULL
  is_active    BOOLEAN
)
```

**Qué cuenta como fin de semana es configurable**, no está hardcodeado. Vive en `configurations`:

```
key: pricing.weekend_days      value: [5,6]     # viernes y sábado (ISO-8601: 1=lunes)
key: pricing.rounding          value: "nearest_10"
key: pricing.currency          value: "MXN"
```

En México la noche cara suele ser viernes y sábado (el huésped se va el domingo), no sábado-domingo. Dejarlo en config evita una migración cuando el negocio cambie de opinión.

**`holiday`** se resuelve contra una tabla `holidays (id, date, name, country)` sembrada con los días festivos oficiales. Un día festivo gana sobre `weekend`, y `weekend` sobre `weekday`.

**Especificidad de `price_rules`:** gana la regla con más campos concretos. `(property_id = 12, season_id = 3)` vence a `(property_id = NULL, season_id = 3)`, que vence a `(NULL, NULL)`.

---

### 15.5 Promociones (`promotions`)

Separadas de las temporadas a propósito: una temporada modifica **el precio publicado** (el huésped ve 3000/noche y eso es el precio); una promoción es **un descuento sobre el total** (el huésped ve 3000/noche, y luego "−10% por código VERANO26").

```
promotions (
  id,
  name                    VARCHAR
  code                    VARCHAR UNIQUE NULL   -- NULL = promoción automática, sin código
  type                    ENUM(percent, fixed_amount, free_nights)
  value                   DECIMAL(10,2)         -- 10 = 10% | 500 = $500 | 1 = 1 noche gratis
  max_discount            DECIMAL(10,2) NULL    -- tope para type=percent
  min_nights              TINYINT NULL
  min_amount              DECIMAL(10,2) NULL    -- subtotal mínimo
  booking_starts_at       DATETIME NULL         -- ventana para RESERVAR
  booking_ends_at         DATETIME NULL
  stay_starts_at          DATE NULL             -- ventana de ESTANCIA válida
  stay_ends_at            DATE NULL
  usage_limit             INT NULL              -- global
  usage_limit_per_customer INT NULL
  used_count              INT DEFAULT 0
  combinable              BOOLEAN DEFAULT 0
  priority                SMALLINT DEFAULT 0
  is_active               BOOLEAN DEFAULT 1
  ...auditoría
)

promotion_property   (promotion_id FK, property_id FK)  -- sin filas = todas
promotion_redemptions (
  id, promotion_id FK, booking_id FK, customer_id FK,
  amount_discounted DECIMAL(10,2), redeemed_at DATETIME
)
```

**Dos ventanas de tiempo distintas** — es la fuente de errores más común y por eso son columnas separadas:

- `booking_*`: *cuándo se puede canjear*. "Black Friday: reserva del 24 al 30 de noviembre".
- `stay_*`: *para qué fechas de estancia sirve*. "…y viaja entre enero y marzo".

Una promo puede tener solo una de las dos, o ambas.

**`combinable`:** si `false` (por defecto), la promoción es exclusiva — se aplica solo la de mayor `priority` y, en empate, la de mayor descuento absoluto para el huésped. Si `true`, se acumula con otras combinables, aplicándose **siempre sobre el subtotal original**, no en cascada (evita que el orden de aplicación cambie el total).

**Consumo de cupones — condición de carrera.** `used_count` se incrementa dentro de la **misma transacción** que crea la reserva, con `SELECT ... FOR UPDATE` sobre la fila de `promotions`, y se verifica que `used_count < usage_limit` **después** del lock. Validar el límite en el `quote` no sirve: entre el quote y el pago pasan minutos. El mismo patrón anti-doble-booking de la sección 5, aplicado a cupones.

Si la reserva se cancela, un listener de `BookingCancelled` decrementa `used_count` y borra la fila de `promotion_redemptions`.

---

### 15.6 Calendario de precios materializado

Renderizar el calendario público de un mes exige 30 resoluciones de precio. Recalcularlas en cada request es caro y repetitivo, porque cambian rara vez.

```
price_calendar (
  property_id FK, date DATE, price DECIMAL(10,2),
  season_id FK NULL, min_nights TINYINT NULL, computed_at DATETIME,
  PRIMARY KEY (property_id, date)
)
```

- Se rellena con el job `RecalculatePriceCalendar` (por cola), disparado por los eventos `SeasonSaved`, `PriceRuleSaved`, `PropertyPriceChanged`, y por `schedule:run` diario para extender el horizonte móvil (12–18 meses hacia adelante).
- Es **cache, no fuente de verdad**. `PricingService` sigue siendo la autoridad; si una fila falta o está obsoleta (`computed_at` anterior al último cambio de reglas), se calcula al vuelo.
- El precio **cobrado** siempre se recalcula con `PricingService` al crear la reserva. `price_calendar` solo alimenta la vista del calendario.

---

### 15.7 `PricingService` — contrato

```php
interface PricingServiceInterface
{
    /** Precio resuelto de UNA noche (pasos 1–3 del pipeline). */
    public function priceForNight(Property $property, CarbonImmutable $date, int $guests = 1): NightPriceDTO;

    /** Desglose completo de una estancia (pipeline entero). */
    public function quote(QuoteRequestDTO $dto): QuoteDTO;
}
```

```php
final readonly class NightPriceDTO
{
    public function __construct(
        public CarbonImmutable $date,
        public float  $basePrice,
        public float  $price,       // precio final de la noche
        public ?int   $seasonId,
        public string $dayType,     // weekday|weekend|holiday
        public ?int   $minNights,
    ) {}
}
```

El `QuoteDTO` devuelve `nights[]`, `subtotal`, `discounts[]`, `fees[]`, `taxes`, `total` y `currency` — todo lo que el frontend necesita para mostrar el desglose sin recalcular nada por su cuenta.

---

### 15.8 Ejemplo de respuesta del quote

`POST /api/v1/bookings/quote`

```json
{
  "data": {
    "property_id": 12,
    "checkin": "2026-12-23",
    "checkout": "2026-12-27",
    "nights": [
      { "date": "2026-12-23", "base_price": 2500, "price": 3900, "season": "Navidad",         "day_type": "weekday" },
      { "date": "2026-12-24", "base_price": 2500, "price": 3900, "season": "Navidad",         "day_type": "holiday" },
      { "date": "2026-12-25", "base_price": 2500, "price": 4290, "season": "Navidad",         "day_type": "holiday" },
      { "date": "2026-12-26", "base_price": 2500, "price": 3900, "season": "Temporada alta",  "day_type": "weekend" }
    ],
    "subtotal": 15990,
    "discounts": [
      { "code": "VERANO26", "label": "Promo 10%", "amount": -1599 }
    ],
    "fees":  [ { "label": "Limpieza", "amount": 800 } ],
    "taxes": { "label": "IVA 16%", "amount": 2334.56 },
    "total": 17525.56,
    "currency": "MXN",
    "min_nights_required": 3
  }
}
```

⚠️ El quote **no** aparta fechas ni consume el cupón. Es solo cálculo. El apartado ocurre en `POST /bookings`.

---

### 15.9 Riesgos y qué probar

| Riesgo | Prueba obligatoria |
|---|---|
| Estancia que cruza dos temporadas | Reserva 28-dic → 03-ene con temporadas distintas a cada lado |
| Temporada `yearly` que envuelve el año | 15-dic → 05-ene debe cubrir ambos extremos, en 2026 y 2027 |
| Traslape con la misma prioridad | Debe rechazarse en la validación, no producir precio ambiguo |
| Doble canje de cupón con `usage_limit = 1` | Dos requests concurrentes: solo una debe confirmarse |
| Precio congelado | Cambiar temporada tras confirmar: la reserva conserva su total |
| Redondeo | Sumar noches redondeadas ≠ redondear la suma; definir cuál y ser consistente |
| DST / zona horaria | Todas las fechas de estancia son `DATE`, nunca `DATETIME` con zona |

---

Ver también: sección 5 (base de datos), sección 6 (endpoints), sección 13 (estimación).

---

