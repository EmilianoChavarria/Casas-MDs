# Prompt para Magic Patterns — Casa Caribe (v2, solo maquetación)

> Pegar tal cual. Está escrito para continuar el prototipo existente, no para empezar de cero.

---

## Contexto

Estoy extendiendo el prototipo **Casa Caribe**: plataforma de renta de casas vacacionales de **una sola empresa** (no es marketplace) en Quintana Roo, México. Público tipo Airbnb + panel de administración interno.

**Stack del prototipo (respetar sin cambios):** React + TypeScript + Tailwind + `lucide-react` + `framer-motion` + `leaflet`/`react-leaflet`.

**Esto es SOLO maquetación.** No conectar APIs, no lógica de negocio, no cálculos reales: todo con datos mock en el archivo del componente, estados simulados con `useState`, y todas las variantes visuales visibles.

---

## Sistema de diseño — usar exactamente esto, no inventar paleta

**Colores** (escalas propias en `tailwind.config.js`):

| Escala | Rol | Ancla |
|---|---|---|
| `lagoon` | marca, texto, navegación, primario del **admin** | `lagoon-500` `#1caab0` |
| `coral` | **solo CTA del huésped** (reservar, pagar, buscar, registrarse) | `coral-500` `#ef5a35` |
| `sand` | neutros: superficies, bordes, divisores | `sand-200` `#e7ddc9` |

- Texto base del documento: `#12333a`. Títulos `lagoon-900`, secundario `lagoon-600`.
- Superficie secundaria: `sand-50`. Borde por defecto: `sand-200`.
- **`lagoon` y `coral` NO son intercambiables.** Un botón "Reservar" en lagoon rompe la jerarquía.
- **No hay modo oscuro** y no debe agregarse.
- Estados del sistema con defaults de Tailwind vía alias: `danger` = `red`, `warning` = `amber`. Éxito e info = `lagoon`.
- ⚠️ `red-600` y `coral-500` se parecen: distinguir **por forma, no por color**. Botón destructivo = borde rojo + fondo transparente, nunca relleno rojo junto a un CTA coral.

**Badges de estado de reserva:**
`confirmed` → `bg-lagoon-100 text-lagoon-700` · `pending` → `bg-coral-100 text-coral-700` · `cancelled` → `bg-gray-200 text-gray-600` · `completed` → `bg-sand-200 text-sand-800` · `expired` → `bg-gray-200 text-gray-600` con borde punteado.

**Tipografía:** Poppins (`font-display`) en encabezados, wordmark, cifras de precio, códigos de reserva y labels de sección. Inter (`font-sans`) en todo el cuerpo. Escala Tailwind por defecto; **siempre clase de tamaño explícita** (nunca un `h2`/`h3` sin `text-*`).

**Forma:** `rounded-xl` inputs · `rounded-2xl` tarjetas y paneles (radio estándar) · `rounded-3xl` contenedores destacados · `rounded-full` pills, chips, avatares y celdas de calendario.

**Sombras — solo estas dos, teñidas de lagoon, nunca `shadow-lg` genérico:**
```
card: 0 6px 20px -6px rgba(24, 91, 97, 0.18)
pop:  0 12px 40px -8px rgba(24, 91, 97, 0.28)
```

**Espaciado:** escala Tailwind, sin valores arbitrarios. Padding de página `px-4 sm:px-6 lg:px-8`. Solo `space-y-*`.

**Escala de z-index nombrada, sin `z-[número]` suelto:**
`map:0` · `sticky:20` · `dropdown:30` · `drawer:40` · `chat:45` · `modal:50` · `toast:60`.

**Accesibilidad y movimiento (obligatorio en todo lo nuevo):**
- Foco con `focus-visible:ring-2 focus-visible:ring-lagoon-400 focus-visible:ring-offset-2` en **botones, links, chips, celdas de calendario y controles de carrusel**, no solo inputs. Sobre fondos oscuros: `ring-offset-lagoon-900`.
- Validación **inline bajo el campo**: `border-danger-500` + `<p class="mt-1.5 text-xs text-danger-600">` + `aria-invalid` + `aria-describedby`. Resumen arriba del formulario **solo** para errores sin campo dueño (pago rechazado, propiedad ya no disponible, cupón agotado).
- Carga = **skeleton con la forma del contenido** (`animate-pulse rounded-2xl bg-sand-100`). Spinner **solo dentro de botones**. Nunca spinner centrado de página.
- Vacío = borde `border-dashed border-sand-300` + icono + acción.
- Respetar `prefers-reduced-motion`. Transiciones `duration-200 ease-out`; zoom de imagen en tarjeta `duration-500`.
- Mobile-first. Breakpoints `sm/md/lg`.
- Todo el texto de interfaz debe poder verse en **es / en / fr** (ya existe `LanguageSelector`).

---

## Parte 1 — Lo que YA EXISTE en el prototipo: no rehacer, solo ajustar donde se indica

Pantallas y componentes ya construidos: **Home** (hero + buscador + secciones), **Listado de propiedades** (filtros + mapa Leaflet), **Detalle de propiedad** (galería, calendario, widget de reserva, mapa), **Checkout**, **Confirmación**, **Auth** (login/registro), **Perfil del huésped**, **AdminLogin**, **AdminLayout** (sidebar + drawer móvil), **Dashboard admin**, **Propiedades admin**, **Alta/edición de propiedad**, **Reservaciones admin**, **Detalle de reservación**, **Precios admin** (solo lista), **Mensajes admin** (panel dividido con burbujas).

Componentes reutilizables ya resueltos: `Header`, `Footer`, `PropertyCard`, `SearchBar`, `AvailabilityCalendar`, `CalendarLegend`, `PropertyMap`, `SingleLocationMap`, `LanguageSelector`, `StarRating`, `StatusBadge`, `AdminLayout`.

**Ajustes puntuales a lo existente (no rediseñar, extender):**

1. **Unificar duplicados en `components/ui/`** — `Field`/`Input` está copiado en 4 pantallas (Auth, AdminLogin, Checkout, AdminPropertyForm); `Row` en 3. Extraer a `ui/Input.tsx`, `ui/Field.tsx`, `ui/Row.tsx`, `ui/SectionHeading.tsx`, `ui/EmptyState.tsx`, `ui/Section.tsx`, `ui/Label.tsx`, `ui/NumberField.tsx`, aceptando `className` vía `tailwind-merge`.
2. **`AvailabilityCalendar` cambia de contenido, no solo de estilo.** Cada celda pasa de ser un número a ser **día + precio de esa noche**. Añadir: aviso de **mínimo de noches** por temporada, estado de **selección rechazada** por no cumplir el mínimo, y leyenda ampliada (disponible / reservado / bloqueado / seleccionado / mínimo no cumplido).
3. **Añadir contador de no leídos** al `Header` público y al `AdminLayout`.
4. Botones y links del prototipo hoy no tienen anillo de foco: aplicar `focus-visible` a todos.
5. Estado de carga hoy es solo cambio de texto del botón: añadir skeletons por pantalla.

---

## Parte 2 — Lo que HAY QUE CREAR

### A. Chat en tiempo real (el lado público no existe)

- **`ChatWidget`** — burbuja flotante en el sitio público, `shadow-pop`, `z-chat` (sobre el drawer, bajo el modal). Cerrada = burbuja con badge de no leídos; abierta = ventana.
- **`ChatWindow`** con `MessageList` (scroll invertido, cargar historial hacia arriba), `MessageBubble`, `MessageComposer` (textarea + adjuntar imagen/PDF + contador de 2,000 caracteres máx.), `TypingIndicator` ("escribiendo…").
- Reutilizar el estilo de burbuja del admin invirtiendo los lados: host = `lagoon-500`, `max-w-[75%]`, cola con `rounded-br-md`/`rounded-bl-md`.
- **Estados de mensaje visibles:** `sending` (gris, optimistic UI), enviado, entregado, **leído**, y **fallido con reintentar**.
- **Mensajes de sistema** dentro del hilo, con estilo distinto de huésped y admin ("Reserva confirmada", "Pago recibido").
- **Aviso de conexión perdida**: banda discreta "Sin conexión — reintentando…" cuando el socket cae.
- **Formulario de visitante sin cuenta** (nombre + correo) antes del primer mensaje.
- Miniatura de adjunto dentro de la burbuja + visor ampliado.

**Admin — bandeja de chat completa** (hoy solo hay panel dividido): filtros por `estado` (open/pending/closed) y `asignado a`, ordenada por último mensaje, indicador de en línea, acciones **asignar** y **cerrar** conversación, chips de contexto (propiedad / reserva vinculada).

### B. Motor de precios y promociones (nada de esto existe)

- **`PriceBreakdown`** — renderiza el desglose completo: lista de **noches con su precio individual y su temporada**, subtotal, descuentos, cargos, **impuestos como lista** (IVA %, ISH %, **DSA como monto fijo por noche**), total y moneda. Debe verse claramente que hay tres impuestos distintos y que uno no es porcentaje. Colapsable en móvil.
- **`PromoCodeInput`** — aplicar/quitar cupón con estados: válido (chip verde-lagoon con monto descontado), inválido, expirado, agotado, no aplica a esta propiedad.
- **Pantalla `/admin/promotions`** — **no existe**. Lista con badges de activa/inactiva, toggle rápido, columna "usos / límite". Formulario con: nombre, código (o "promoción automática sin código"), tipo (`porcentaje` / `monto fijo` / `noches gratis`), valor, tope de descuento, mínimo de noches, mínimo de importe, **dos ventanas de fecha separadas y bien diferenciadas visualmente — "cuándo se puede reservar" vs. "para qué fechas de estancia sirve"**, límite global, límite por cliente, `combinable`, prioridad, propiedades aplicables (vacío = todas). Vista de **canjes** de una promoción.
- **`SeasonCalendar` (admin)** — 12 meses del año en una vista, con las temporadas pintadas por su color. Traslapes visibles.
- **`SeasonForm`** — nombre, rango de fechas, `recurrence: yearly` (repetir cada año), tipo de ajuste (`porcentaje` / `monto fijo` / `precio absoluto`), valor (admite negativos = temporada baja), mínimo de noches, **prioridad**, color, propiedades aplicables. **Alerta de conflicto**: "se traslapa con «Navidad» con la misma prioridad" antes de guardar.
- **`PricingPreview`** — simulación de los próximos 12 meses con las reglas nuevas **antes** de guardar, mostrando precio antes/después.
- **Reglas por tipo de día** — pantalla para `weekday` / `weekend` / `holiday` con ajuste por tipo, y un ajuste de configuración de **qué días cuentan como fin de semana** (por defecto viernes y sábado, no sábado y domingo).
- **Días festivos (`holidays`)** — CRUD simple con lista por año.
- **Configuración de cargos e impuestos** — cargos (limpieza por estancia, huésped extra, mascota) separados visualmente de impuestos, con tipo de cálculo `porcentaje` o `monto fijo por noche` y tasas editables. Debe leerse que los cargos son ingreso y los impuestos no.
- **Acción "recalcular calendario de precios"** con estado en proceso.

### C. Checkout y reserva

- **Cuenta regresiva del apartado** visible durante todo el checkout ("tu reserva se libera en 14:32"), más estado de **reserva expirada**.
- **Selector de método de pago**: tarjeta / OXXO / SPEI, cada uno con su bloque de instrucciones y referencia.
- **Selector de moneda** MXN / USD / CAD, con nota de tipo de cambio.
- **Aceptación de términos** con enlace y versión del documento.
- **Resumen de errores arriba del formulario** para: pago rechazado, propiedad ya no disponible, cupón agotado.
- **Pantalla de cancelación del huésped** con desglose del reembolso (qué se devuelve del alojamiento, la limpieza y los impuestos) y confirmación en dos pasos.

### D. Área del huésped

- **Mis reservas** — próximas / pasadas / canceladas, con acciones por estado.
- **Detalle de reserva del huésped** — desglose congelado, política de cancelación aplicable, instrucciones de llegada **que solo aparecen a partir de 2 días antes del check-in** (antes: mensaje "se enviarán 2 días antes de tu llegada").
- **Escribir reseña** — estrellas + texto, disponible solo tras el checkout, con aviso de ventana de tiempo.
- **Mis reseñas** — lista con opción de editar dentro de la ventana.
- **Favoritos** — cuadrícula de propiedades guardadas + estado vacío. Corazón activo/inactivo en `PropertyCard`.
- **Preferencias de correo** — baja de la solicitud de reseña (solo ese correo, los transaccionales no se dan de baja).
- **Reseñas en el detalle de propiedad** — promedio, conteo, distribución por estrellas, lista paginada y **respuesta del anfitrión** anidada.

### E. Admin — lo que falta

- **Alta manual de reserva** (reservas por teléfono/transferencia): selección de fechas, huésped nuevo o existente, origen de la reserva y pago registrado como "externo".
- **Bloquear/desbloquear fechas** de una propiedad desde un calendario.
- **Moderación de reseñas** — ocultar con motivo (nunca borrar), mostrar de nuevo, responder.
- **Reportes**: ocupación, ingresos (**con los impuestos separados como dinero en tránsito, no como ingreso**), descuento otorgado por promoción, y **favoritos vs. reservas como señal de demanda**.
- **Cola de revisión de traducciones** — "3 traducciones pendientes de aprobar": vista lado a lado ES / EN / FR por propiedad, editar y **aprobar**; solo lo aprobado se publica.
- **Estado de correos dentro del detalle de reservación ya existente** — bloque compacto de **solo lectura, sin acciones**: los 7 correos del ciclo (solicitud, recordatorio de pago, pago confirmado, instrucciones de llegada, recordatorio de check-in, solicitud de reseña, cancelación) con enviado / pendiente / no aplica y la fecha. **No es una pantalla nueva ni tiene botón de reenviar**: solo responde "¿ya le llegó?" cuando el huésped llama.
- **Reembolsos** — pantalla de reembolso total o parcial con nota de que la comisión del procesador no se recupera.
- **Ampliar el formulario de propiedad** con: autocompletado de dirección, **instrucciones de acceso y código de cerradura marcados como campo sensible**, huéspedes incluidos en el precio + cargo por huésped adicional, descuentos por estancia semanal y mensual, política de cancelación asignable, y descripción en un solo idioma con aviso de traducción automática posterior.

### F. Transversales

- Páginas legales: **aviso de privacidad, términos y condiciones, cookies**.
- **404** y **error de servidor**.
- Skeletons de listado, detalle, calendario y bandeja de chat.
- Toasts (`z-toast`).
- Modal de confirmación destructiva (borde rojo, fondo transparente).

---

## Reglas de salida

1. Un archivo por componente, TypeScript + Tailwind, mismo estilo que el prototipo actual.
2. Datos mock realistas y en español (Tulum, Playa del Carmen, Cancún; precios en MXN; nombres de temporada "Navidad", "Semana Santa", "Temporada alta invierno").
3. **Mostrar todas las variantes de estado en la maqueta** (cargando, vacío, error, éxito, deshabilitado) — es el objetivo del ejercicio.
4. No instalar librerías nuevas de UI (nada de shadcn, MUI, Chakra ni Mantine): el design system es propio sobre Tailwind.
5. No inventar tokens de color nuevos ni modo oscuro.
6. El frontend **nunca calcula precios**: `PriceBreakdown` solo muestra los números del mock.
