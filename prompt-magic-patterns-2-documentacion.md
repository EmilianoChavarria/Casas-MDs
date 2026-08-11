# Prompt #2 para Magic Patterns — generar el mapa de pantallas en Markdown

> Usar **después** de que Magic Patterns haya generado los mocks del prompt #1.
> Objetivo: que documente lo que realmente construyó, no lo que se le pidió.

---

## Instrucción

Ya tienes construido todo el prototipo de **Casa Caribe** (plataforma de renta de casas vacacionales de una sola empresa, México). Ahora **no escribas más código**. Genera un **único archivo Markdown** llamado `20-mapa-de-pantallas.md` que documente **todas las pantallas del proyecto**, tanto las que ya existían antes como las que acabas de crear.

Este documento va a ser la referencia que use el equipo de backend (Laravel) para saber qué datos necesita cada pantalla. Precisión sobre extensión.

---

## Regla número uno

**Documenta lo que existe en el código, no lo que debería existir.**

- Recorre los archivos reales del proyecto y lista las pantallas que encuentres.
- Si una pantalla que esperabas no está construida, ponla igual pero marcada `⚠️ No construida`.
- **No inventes campos, botones ni estados que no estén en el código.**
- Si un dato aparece en el mock como texto quemado, documéntalo como dato y anota de dónde debería venir.

---

## Estructura del documento

### 1. Encabezado

Título, una línea de propósito, y la fecha.

### 2. Tabla resumen (al inicio, antes del detalle)

Todas las pantallas en una sola tabla, ordenadas: público → área del huésped → admin.

| # | Pantalla | Ruta | Área | Estado |
|---|---|---|---|---|

Donde **Estado** es uno de: `✅ Construida` · `⚠️ Parcial` · `❌ No construida`.

### 3. Árbol de rutas

Un bloque de código con la estructura de rutas completa en formato **Next.js App Router** (el destino del proyecto), agrupada en `(public)`, `(guest)` y `(admin)`. Convierte las rutas del prototipo (`react-router`) a esa convención: `/property/:slug` → `app/(public)/properties/[slug]/page.tsx`.

### 4. Ficha por pantalla

Una sección por pantalla, **siempre con este mismo esquema**, en este orden:

```markdown
## [Número]. [Nombre de la pantalla]

**Ruta:** `/ruta` · **Área:** Público | Huésped | Admin · **Estado:** ✅ Construida
**Archivo:** `src/pages/Ejemplo.tsx`
**Acceso:** Público | Requiere sesión de huésped | Requiere sesión de admin

### Propósito
Una o dos frases. Qué resuelve para quién.

### Bloques de contenido
Lista ordenada de arriba abajo, tal como se ve en pantalla. Para cada bloque:
qué muestra y qué acciones tiene.

### Datos que muestra
| Campo | Tipo | Origen | Nota |
|---|---|---|---|
Cada dato visible. En **Origen** pon la entidad de negocio (`property.name`,
`booking.total_price`, `season.color`…). Si el mock lo tiene quemado y no sabes
de dónde sale, escribe `⚠️ por definir`.

### Acciones del usuario
| Acción | Control | Resultado esperado |
|---|---|---|
Cada botón, link, toggle o campo interactivo.

### Estados de la pantalla
| Estado | ¿Está maquetado? | Cómo se ve |
|---|---|---|
Obligatorio evaluar los cinco: **carga, vacío, error, éxito, sin permiso**.
Marca `❌` los que no maquetaste — es información valiosa, no un fallo que ocultar.

### Endpoints que consume
Lista tomada de la lista cerrada de más abajo. Si ninguno encaja, escribe
`❌ Sin endpoint definido — falta en la API`.

### Componentes que usa
Nombres de los componentes reales del proyecto.

### Responsive
Qué cambia en móvil respecto a escritorio.

### Notas y pendientes
Decisiones abiertas, deuda visual, cosas que quedaron simuladas.
```

### 5. Sección final: huecos detectados

Tres listas al cierre del documento:

1. **Pantallas sin endpoint** — maquetadas pero sin API que las alimente.
2. **Endpoints sin pantalla** — de la lista de abajo, los que ninguna pantalla consume.
3. **Estados sin maquetar** — dónde falta carga, vacío o error.

---

## Lista cerrada de endpoints — usar solo estos, no inventar

```
# Auth
POST   /api/v1/auth/register · login · logout · refresh · forgot-password · reset-password
GET    /api/v1/auth/me
GET    /api/v1/auth/google/redirect · /callback
POST   /api/v1/auth/google/link      DELETE /api/v1/auth/google/unlink

# Público
GET    /api/v1/properties?checkin=&checkout=&price_min=&price_max=&amenities[]=&guests=
GET    /api/v1/properties/{slug}
GET    /api/v1/properties/{id}/availability?month=      # precio por noche + min_nights
POST   /api/v1/bookings/quote                           # desglose; NO aparta fechas
POST   /api/v1/promotions/validate
POST   /api/v1/bookings
GET    /api/v1/bookings/{id}/status

# Reseñas y favoritos
GET    /api/v1/properties/{slug}/reviews
POST   /api/v1/bookings/{id}/review     PUT /api/v1/reviews/{id}
GET    /api/v1/me/reviews
GET    /api/v1/me/favorites             POST|DELETE /api/v1/me/favorites/{propertyId}

# Chat
GET|POST /api/v1/conversations          GET /api/v1/conversations/{id}
GET    /api/v1/conversations/{id}/messages?before_id=&after_id=&limit=
POST   /api/v1/conversations/{id}/messages
PATCH  /api/v1/conversations/{id}/read
GET    /api/v1/conversations/unread-count

# Admin
GET|POST|PUT|DELETE /api/v1/admin/properties[/{id}]
POST|DELETE         /api/v1/admin/properties/{id}/images[/{imageId}]
GET|POST|PUT|DELETE /api/v1/admin/amenities[/{id}]
GET|POST|PUT|DELETE /api/v1/admin/seasons[/{id}]
GET|POST|PUT|DELETE /api/v1/admin/price-rules[/{id}]
GET|POST|PUT|DELETE /api/v1/admin/holidays[/{id}]
GET    /api/v1/admin/pricing/calendar?property_id=&from=&to=
POST   /api/v1/admin/pricing/preview · /recalculate
GET|POST|PUT|DELETE /api/v1/admin/promotions[/{id}]
PATCH  /api/v1/admin/promotions/{id}/toggle
GET    /api/v1/admin/promotions/{id}/redemptions
PATCH  /api/v1/admin/availability/{propertyId}          # bloquear/desbloquear fechas
GET|PUT /api/v1/admin/bookings[/{id}]
PATCH  /api/v1/admin/bookings/{id}/confirm · /cancel
GET|POST|PUT|DELETE /api/v1/admin/customers[/{id}]
POST   /api/v1/admin/reviews/{id}/reply
PATCH  /api/v1/admin/reviews/{id}/hide · /unhide
GET    /api/v1/admin/conversations?status=&assigned_to=
PATCH  /api/v1/admin/conversations/{id}/assign · /close
GET    /api/v1/admin/reports/occupancy · /revenue · /promotions · /favorites
```

---

## Contexto de negocio que debes reflejar en las fichas

Marca estos puntos donde apliquen — son decisiones tomadas, no sugerencias:

- **El frontend nunca calcula precios.** Toda pantalla con importes los recibe ya calculados del backend.
- **Precio por noche, no por rango.** Una estancia puede cruzar dos temporadas.
- **El precio se congela al confirmar la reserva.** Cambiar una temporada después no altera reservas ya confirmadas.
- **Los impuestos son una lista, no un número.** IVA 16% y ISH 3% son porcentajes; el DSA es **monto fijo por noche**. Deben verse desglosados.
- **Cargos ≠ impuestos.** Los cargos (limpieza) son ingreso del negocio; los impuestos están en tránsito al fisco y no suman a ingresos en los reportes.
- **La dirección exacta y el código de acceso no se muestran al reservar**, solo desde 2 días antes del check-in.
- **Una reserva `pending` expira.** El huésped debe ver cuenta regresiva.
- **El chat guarda en base de datos primero y difunde después.** El WebSocket es transporte, no almacenamiento.
- **Canal de venta único:** este sitio es la única fuente de verdad de la disponibilidad.

Decisiones **pendientes del cliente** — si una pantalla depende de alguna, anótala en "Notas y pendientes" con su clave:

| Clave | Pendiente |
|---|---|
| D1 | Moneda y tipo de cambio (MXN/USD/CAD) |
| D2 | Traducción de contenido asistida (revisar y aprobar) |
| D3 | Cargo por huésped adicional |
| D4 | Descuento por estancia larga |
| D5 | Base de cálculo de impuestos |
| D6 | Cuántos administradores y cómo entran |
| D7 | Política de cancelación y reembolsos |
| D8 | Textos legales |

---

## Formato de salida

- **Un solo archivo `.md`**, en español, listo para pegar en un repositorio de documentación.
- Markdown estándar: encabezados, tablas, listas. Sin HTML.
- Sin capturas ni imágenes.
- Sin código de componentes: es documentación funcional, no técnica de implementación.
- Nombres de archivo y de componente en `código en línea`.
- No repitas el mismo texto en varias fichas: si dos pantallas comparten un bloque, descríbelo una vez y referéncialo.
