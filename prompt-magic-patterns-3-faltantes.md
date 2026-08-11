# Prompt #3 para Magic Patterns — construir lo que falta

> Escrito contra el handoff técnico que generaste del prototipo actual.
> Reemplaza al prompt #1: aquí ya se sabe exactamente qué existe.

---

## Contexto

Tienes construido el prototipo **Casa Caribe** — plataforma de renta de casas vacacionales de **una sola empresa** (no es marketplace), en Quintana Roo, México. Ya documentaste su estado: 16 pantallas, 13 componentes compartidos, tokens `lagoon`/`coral`/`sand`, Poppins + Inter, sombras `card`/`pop`.

Ahora hay que **completarlo**. Todo lo de abajo es **solo maquetación**: datos mock en el propio archivo, estados simulados con `useState`, sin llamadas a API y sin lógica de negocio real.

**No rehagas nada de lo que ya existe.** Extiende los archivos actuales donde se indique y crea solo lo que falta.

---

## Reglas del sistema de diseño — ya establecidas, respetar sin excepción

- Paleta: `lagoon` (marca, navegación, primario del **admin**), `coral` (**solo CTA del huésped**: reservar, pagar, buscar, registrarse), `sand` (superficies y bordes). **No son intercambiables.**
- **No hay modo oscuro.** No agregarlo.
- Sombras: solo `shadow-card` y `shadow-pop`. **Nunca `shadow-lg` genérico** — están teñidas de lagoon y eso da la sensación cálida del conjunto.
- Radios: `rounded-xl` inputs · `rounded-2xl` tarjetas y paneles · `rounded-3xl` contenedores destacados · `rounded-full` pills, chips, avatares, celdas de calendario.
- Tipografía: `font-display` (Poppins) en encabezados, cifras de precio, códigos de reserva y labels de sección. Inter en el cuerpo. **Siempre clase de tamaño explícita.**
- Espaciado: escala Tailwind, sin valores arbitrarios. `space-y-*`, nunca `space-x-*`. Padding de página `px-4 sm:px-6 lg:px-8`.
- Iconos: `lucide-react`, convención con sufijo `Icon`, tamaño más común `h-4 w-4`.
- Mobile-first. Todo texto de interfaz debe pasar por el sistema i18n existente (`src/i18n/`), en es / en / fr.
- No instalar ninguna librería de componentes (nada de shadcn, MUI, Chakra, Mantine). El design system es propio.

---

## PARTE 0 — Añadir a la config y al CSS **antes** de construir pantallas

Sin esto no hay con qué pintar una validación, y añadirlo después obliga a retocar cada formulario.

### 0.1 Tokens semánticos en `tailwind.config.js`

Hoy no existen `success` / `error` / `warning` / `info`. La regla es: **la marca sale de las escalas propias; los estados del sistema salen de los defaults de Tailwind.**

```js
import colors from 'tailwindcss/colors';
// theme.extend.colors
danger:  colors.red,      // text-danger-600, bg-danger-50, border-danger-500
warning: colors.amber,
// success e info NO llevan alias: son lagoon, que ya es un nombre semántico
```

⚠️ `red-600` y `coral-500` se parecen. **Distinguir por forma, no por color:** el error es texto pequeño o borde fino; el CTA es un botón relleno. Un botón destructivo va con **borde rojo y fondo transparente**, nunca relleno rojo junto a un CTA coral.

### 0.2 Escala de z-index nombrada

Hoy hay `z-0/20/30/40/50` sueltos. Con el chat flotante nuevo la colisión es cuestión de tiempo.

```js
zIndex: {
  map: '0', sticky: '20', dropdown: '30',
  drawer: '40', chat: '45', modal: '50', toast: '60',
}
```

`z-chat: 45` va **sobre** el drawer (el chat sigue accesible con el menú móvil abierto) y **bajo** el modal (un diálogo de confirmación no puede quedar tapado por una burbuja). A partir de aquí: **ningún `z-[número]` suelto en componentes.**

### 0.3 Foco visible — hoy solo lo tienen los inputs

Navegar el sitio con teclado es imposible. Aplicar a **botones, links, chips, celdas de calendario y controles del carrusel**:

```
focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-lagoon-400
focus-visible:ring-offset-2 focus-visible:ring-offset-white
```

**`focus-visible`, no `focus`.** Con `focus` a secas todos los botones parpadean un anillo al hacer clic con el ratón, y eso es lo que lleva a que alguien lo desactive entero. Sobre fondos oscuros (hero de Home, `Auth`, `AdminLogin`, header transparente): `focus-visible:ring-offset-lagoon-900`.

### 0.4 `prefers-reduced-motion` en `index.css`

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

La última línea anula el `scroll-behavior: smooth` que hoy está en `html`. Y en los `motion.*` de framer-motion, usar `useReducedMotion()` para desactivar el escalonado de entrada de las tarjetas. No es estético: una cuadrícula entrando escalonada marea a personas con sensibilidad vestibular.

### 0.5 Patrón de validación de formularios — hoy **no existe ninguno**

Solo hay `required` nativo. Checkout, alta de propiedad y promociones necesitan error por campo:

```tsx
<input
  aria-invalid={!!error}
  aria-describedby={error ? `${id}-error` : undefined}
  className={error
    ? 'border-danger-500 focus:border-danger-500 focus:ring-2 focus:ring-danger-100'
    : 'border-sand-200 focus:border-lagoon-400 focus:ring-2 focus:ring-lagoon-100'}
/>
{error && <p id={`${id}-error`} className="mt-1.5 text-xs text-danger-600">{error}</p>}
```

`aria-invalid` y `aria-describedby` no son opcionales: sin ellos un lector de pantalla anuncia el campo pero no el motivo del fallo. **Resumen arriba del formulario solo para errores sin campo dueño** (pago rechazado, propiedad ya no disponible, cupón agotado) — repetir inline lo que ya está inline solo duplica ruido.

### 0.6 Skeletons — hoy la carga es solo cambiar el texto del botón

```tsx
<div className="h-64 w-full animate-pulse rounded-2xl bg-sand-100" />
```

Skeleton **con la forma del contenido** para listado, detalle, calendario y bandeja de chat. **Spinner solo dentro de botones.** Un spinner centrado de página no dice nada sobre lo que está por llegar.

### 0.7 Limpieza de dependencias

- `tailwind-merge` está instalado y sin usar: es exactamente lo que hace falta para que los componentes de `ui/` acepten `className` sin conflictos. Usarlo.
- `date-fns` está instalado y sin usar (hoy se formatea con `Intl`): se va a necesitar de verdad para el calendario de precios y los rangos de temporadas.
- `@radix-ui/react-icons` está instalado y sin usar: **quitarlo**.
- `.no-scrollbar` está definida en `index.css` y sin aplicar: usarla en el carrusel de `PropertyCard`.
- `UserCircleIcon` se importa en `Header.tsx` y no se usa: quitar el import.

---

## PARTE 1 — Corregir componentes existentes

### 1.1 `AvailabilityCalendar` — cambia de contenido, no solo de estilo

Hoy la celda es un número con estado (`available`, `booked`, `blocked`, pasado, seleccionado, en rango). **Cada celda pasa a ser día + precio de esa noche.**

Props nuevos: `nightPrices: { date: string; price: number; minNights?: number }[]`, `currency: string`, y para el modo admin `editable?: boolean` con `onToggleBlock?(dates: string[])`.

Estados nuevos que hay que maquetar:
- Precio bajo el número del día, en `font-display text-xs`.
- **Mínimo de noches no cumplido**: la selección queda rechazada, con mensaje explicando el mínimo de esa temporada.
- Celda de temporada especial: franja de color (viene de `season.color`).
- Ampliar `CalendarLegend` con las entradas nuevas.

⚠️ **Se evalúa la noche, no el día del calendario.** La "noche del viernes" es la que se duerme viernes→sábado. Una estancia vie→dom son **dos** noches, viernes y sábado.

### 1.2 Extraer los duplicados a `src/components/ui/`

Están copiados por el proyecto: `Input` en `Checkout.tsx` y `AdminPropertyForm.tsx`; `Field` en `Auth.tsx` y `AdminLogin.tsx`; `Row` tres veces (`PropertyDetail`, `Checkout`, `AdminReservationDetail`).

Unificar en `ui/Input.tsx`, `ui/Field.tsx`, `ui/Row.tsx`, `ui/SectionHeading.tsx`, `ui/EmptyState.tsx`, `ui/Section.tsx`, `ui/Label.tsx`, `ui/NumberField.tsx` — todos aceptando `className` vía `tailwind-merge`, y con el patrón de error de 0.5 incorporado.

### 1.3 `Header` y `AdminLayout`

Añadir **badge de mensajes no leídos** en ambos. En el header público va junto al icono de sesión; en el admin, sobre el link de Mensajes del sidebar.

### 1.4 Ruta `*` (fallback)

Hoy el comodín renderiza `Home`. Eso hace que cualquier URL rota devuelva la portada como si fuera válida. Reemplazar por una **pantalla 404** propia.

---

## PARTE 2 — Componentes nuevos

### 2.1 `PriceBreakdown`

Renderiza el desglose completo de una cotización. **El frontend nunca calcula precios**: solo muestra lo que recibe.

Bloques, en este orden: lista de **noches con su precio individual y su temporada** (colapsable), subtotal, descuentos, cargos, impuestos, total, moneda.

⚠️ **Los impuestos son una lista, no un número.** Tres cargas simultáneas y **el DSA no es un porcentaje**:

```
IVA 16%   percent           $2,494.56
ISH 3%    percent           $  467.73
DSA       fixed_per_night   $  144.00
```

El huésped tiene que verlas desglosadas para poder facturar, y el tipo debe ser visible porque nadie puede deducir de dónde salen $144 que no son porcentaje de nada.

⚠️ **Cargos ≠ impuestos.** La limpieza es ingreso del negocio; los impuestos están en tránsito al fisco. Deben leerse como cosas distintas, no como una lista uniforme.

### 2.2 `PromoCodeInput`

Aplicar y quitar cupón. Estados a maquetar: vacío, validando, **válido** (chip con el monto descontado), inválido, expirado, **agotado**, no aplica a esta propiedad, no alcanza el mínimo de noches.

### 2.3 `BookingCountdown`

Una reserva `pending` expira. Cuenta regresiva visible durante todo el checkout ("tu reserva se libera en 14:32"), con estado de aviso al acercarse el límite y **estado de reserva expirada**. Un checkout que expira en silencio se percibe como un fallo del sitio, no como una regla.

### 2.4 Chat — el lado público **no existe**

- `ChatWidget` — burbuja flotante, `shadow-pop`, `z-chat`. Cerrada: burbuja con badge de no leídos. Abierta: ventana.
- `ChatWindow`, `MessageList` (scroll invertido, cargar historial hacia arriba), `MessageBubble`, `MessageComposer` (textarea + adjuntar imagen/PDF + contador de 2,000 caracteres máx.), `TypingIndicator`.
- Reutilizar el estilo de burbuja que ya resolviste en `AdminMessages` (host `lagoon-500`, `max-w-[75%]`, cola con `rounded-br-md` / `rounded-bl-md`) **invirtiendo los lados**.
- Estados de mensaje: `sending` (gris, optimistic UI), enviado, entregado, **leído**, y **fallido con reintentar**.
- **Mensajes de sistema** dentro del hilo, con estilo distinto de huésped y admin ("Reserva confirmada", "Pago recibido").
- **Aviso de conexión perdida**: banda discreta "Sin conexión — reintentando…".
- **Formulario de visitante sin cuenta** (nombre + correo) antes del primer mensaje: un interesado que aún no reserva no tiene cuenta.
- Miniatura de adjunto en la burbuja + visor ampliado.

### 2.5 `Toast` y `ConfirmDialog`

Toasts en `z-toast`. Diálogo de confirmación destructiva en `z-modal`, con el botón de borde rojo y fondo transparente (nunca relleno).

---

## PARTE 3 — Pantallas nuevas del huésped

| Ruta | Pantalla | Contenido |
|---|---|---|
| `/bookings/:id` | **Detalle de reserva del huésped** | Desglose congelado (`PriceBreakdown`), estado, política de cancelación aplicable, acciones según estado. ⚠️ **La dirección exacta y el código de acceso NO se muestran al reservar**: aparecen solo desde 2 días antes del check-in. Antes de eso, mensaje "se enviarán 2 días antes de tu llegada". Es decisión de seguridad: un código de cerradura en un correo queda meses expuesto en la bandeja del huésped |
| `/bookings/:id/cancel` | **Cancelar reserva** | Desglose del reembolso (qué se devuelve del alojamiento, la limpieza y los impuestos), política aplicable congelada al reservar, confirmación en dos pasos |
| `/bookings/:id/review` | **Escribir reseña** | Estrellas + texto. Solo tras el checkout, con aviso de ventana de tiempo para escribirla. `Profile.tsx` ya tiene el prop `showReview` en `BookingRow`: enlazar desde ahí |
| `/me/reviews` | **Mis reseñas** | Lista con opción de editar dentro de la ventana |
| `/favorites` | **Favoritos** | Cuadrícula de propiedades guardadas + estado vacío. Hoy el corazón de `PropertyCard` es un `useState` local que no lleva a ninguna parte |
| `/legal/privacidad` · `/legal/terminos` · `/legal/cookies` | **Páginas legales** | Contenido de relleno, pero con la estructura real y control de versión visible del documento |
| `*` | **404** | Reemplaza el fallback actual a `Home` |

**Ampliar `PropertyDetail`** con bloque de reseñas completo: promedio, conteo, **distribución por estrellas**, lista paginada y **respuesta del anfitrión** anidada bajo la reseña.

**Ampliar `Profile`** con preferencias de correo: baja de la solicitud de reseña. Solo ese correo lleva opción de baja — los transaccionales (confirmación, instrucciones de llegada) no, porque el huésped los necesita para completar un servicio que contrató.

**Ampliar `Checkout`** con: `PriceBreakdown`, `PromoCodeInput`, `BookingCountdown`, **selector de método de pago** (tarjeta / OXXO / SPEI, cada uno con su bloque de instrucciones y referencia), **selector de moneda** (MXN / USD / CAD con nota de tipo de cambio), **aceptación de términos con versión del documento**, y el **resumen de errores** de 0.5 para pago rechazado / propiedad ya no disponible / cupón agotado.

---

## PARTE 4 — Pantallas nuevas de administración

### 4.1 `/admin/calendar` — ya existe, **hacerla editable**

Hoy es solo lectura. Añadir: selector de propiedad, **selección de rango por clic-arrastre**, bloquear y desbloquear con motivo (mantenimiento / uso propio / otro), y **celda `booked` que enlaza al detalle de la reserva** mostrando el nombre del huésped.

⚠️ Un día `booked` **no se puede desbloquear a mano** — hay una reserva detrás. Debe verse deshabilitado con su razón.

### 4.2 Precios y temporadas — `AdminPricing` hoy solo lista precios

| Ruta | Pantalla |
|---|---|
| `/admin/seasons` | **`SeasonCalendar`**: los 12 meses del año en una vista, con las temporadas pintadas por su color. Traslapes visibles |
| `/admin/seasons/new` · `/:id/edit` | **`SeasonForm`**: nombre, rango de fechas, `recurrence: yearly` (repetir cada año), tipo de ajuste (`porcentaje` / `monto fijo` / `precio absoluto`), valor (admite negativos = temporada baja), mínimo de noches, **prioridad**, color, propiedades aplicables (vacío = todas). **Alerta de conflicto** antes de guardar: "se traslapa con «Navidad», que tiene la misma prioridad" |
| — | **`PricingPreview`**: simulación de los próximos 12 meses con las reglas nuevas **antes** de guardar, mostrando precio antes/después. Un error de configuración de precios es caro y silencioso |
| `/admin/price-rules` | Reglas por **tipo de día**: `weekday` / `weekend` / `holiday`, con ajuste por tipo y especificidad por propiedad y temporada. Incluir el ajuste de **qué días cuentan como fin de semana** (por defecto **viernes y sábado**, no sábado y domingo) |
| `/admin/holidays` | CRUD de días festivos, listado por año. Un festivo gana sobre `weekend`, y `weekend` sobre `weekday` |
| `/admin/taxes-fees` | **Cargos** (limpieza por estancia, huésped extra, mascota) visualmente **separados** de **impuestos** (tipo `porcentaje` o `monto fijo por noche`, tasas editables). Debe leerse que los cargos son ingreso del negocio y los impuestos no |
| — | Acción **"recalcular calendario de precios"** con estado en proceso |

### 4.3 `/admin/promotions` — no existe

Lista con badges activa/inactiva, toggle rápido y columna "usos / límite".

Formulario: nombre, código (o "promoción automática sin código"), tipo (`porcentaje` / `monto fijo` / `noches gratis`), valor, tope de descuento, mínimo de noches, mínimo de importe, límite global, límite por cliente, `combinable`, prioridad, propiedades aplicables.

⚠️ **Dos ventanas de fecha separadas y visualmente bien diferenciadas** — es la fuente de error más común:
- *cuándo se puede canjear*: "Black Friday, reserva del 24 al 30 de noviembre"
- *para qué fechas de estancia sirve*: "…y viaja entre enero y marzo"

Más una vista de **canjes** de cada promoción.

### 4.4 Resto del admin

| Ruta | Pantalla |
|---|---|
| `/admin/reservations/new` | **Alta manual de reserva** — el cliente seguirá recibiendo reservas por teléfono y transferencia. Fechas, huésped nuevo o existente, origen de la reserva, y pago registrado como **"externo"** |
| `/admin/reviews` | **Moderación**: ocultar **con motivo** (nunca borrar la fila), volver a mostrar, y responder como anfitrión |
| `/admin/reports` | Ocupación · Ingresos (⚠️ **con los impuestos separados como dinero en tránsito, no como ingreso** — mezclarlos declara dinero que se debe entregar al fisco) · Descuento otorgado por promoción · **Favoritos vs. reservas como señal de demanda** (una casa que se guarda mucho y se reserva poco está diciendo algo: precio, fotos o disponibilidad) |
| `/admin/translations` | **Cola de revisión de traducciones**: "3 traducciones pendientes de aprobar". Vista lado a lado ES / EN / FR por propiedad, editar y **aprobar**. Solo lo aprobado se publica |

**Ampliar `AdminMessages`** (hoy es un panel dividido simple): filtros por estado (`open` / `pending` / `closed`) y por asignado, orden por último mensaje, indicador de en línea, acciones **asignar** y **cerrar**, chips de contexto (propiedad o reserva vinculada) y contador de no leídos por hilo.

**Ampliar `AdminReservationDetail`** con un bloque **de solo lectura, sin acciones**, del estado de los 7 correos del ciclo (solicitud, recordatorio de pago, pago confirmado, instrucciones de llegada, recordatorio de check-in, solicitud de reseña, cancelación): enviado / pendiente / no aplica, con fecha. **No es pantalla nueva y no lleva botón de reenviar** — solo responde "¿ya le llegó?" cuando el huésped llama.

**Ampliar `AdminPropertyForm`** con: autocompletado de dirección, **instrucciones de acceso y código de cerradura marcados como campo sensible**, huéspedes incluidos en el precio + cargo por huésped adicional, descuentos por estancia semanal y mensual, política de cancelación asignable, y aviso de que la descripción se captura en un solo idioma y se traduce después.

**Actualizar el sidebar de `AdminLayout`**: hoy tiene 6 links y ahora hacen falta más. Agruparlos — Operación (Dashboard, Calendario, Reservaciones, Mensajes) · Catálogo (Propiedades, Reseñas, Traducciones) · Precios (Precios, Temporadas, Reglas, Festivos, Promociones, Impuestos) · Reportes.

---

## Reglas de negocio que deben notarse en la maqueta

- **El frontend nunca calcula precios.** Toda pantalla con importes los recibe ya calculados.
- **Precio por noche, nunca por rango.** Una estancia puede cruzar temporada baja y alta.
- **El precio se congela al confirmar.** Cambiar una temporada después no altera reservas ya confirmadas.
- **Solo gana una temporada por noche.** No se acumulan.
- **Canal de venta único**: este sitio es la única fuente de verdad de la disponibilidad.
- **El chat guarda en base de datos primero y difunde después.** Si el socket falla, el mensaje existe igual y aparece al recargar.

---

## Formato de salida

1. Un archivo por componente o pantalla, TypeScript + Tailwind, mismo estilo que el código actual.
2. Rutas nuevas registradas en `App.tsx` respetando la convención existente de `react-router-dom`.
3. Textos por el sistema i18n existente (`src/i18n/`), no cadenas sueltas.
4. Datos mock realistas y en español: Tulum, Playa del Carmen, Cancún; precios en MXN; temporadas "Navidad", "Semana Santa", "Temporada alta invierno".
5. **Mostrar todas las variantes de estado**: carga, vacío, error, éxito, deshabilitado. Es el objetivo del ejercicio, no un extra.
6. Al terminar, lista qué construiste y qué decidiste no construir, con el motivo.
