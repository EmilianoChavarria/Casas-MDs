# 17. Sistema de diseño del frontend

Tokens, componentes y decisiones visuales derivados del prototipo **Casa Caribe** (Magic Patterns). Complementa la sección 4 (estructura del frontend), que define *dónde* vive cada archivo; esta sección define *qué* contienen.

---

### 17.1 Estado del prototipo

El prototipo es **React 18 + react-router-dom + Tailwind**, con archivos standalone (sin Next.js). El proyecto destino es **Next.js App Router**. No es código listo para producción: es la fuente de verdad **visual**, no estructural.

Qué se toma tal cual y qué se rehace:

| Del prototipo | Decisión |
|---|---|
| `tailwind.config.js` (colores, fuentes, sombras) | ✅ **Se copia sin cambios.** Es el contrato visual |
| Componentes de presentación (`PropertyCard`, `StarRating`, `StatusBadge`, `AvailabilityCalendar`…) | ✅ Se portan casi tal cual, marcando `'use client'` donde toque |
| `index.css` (fuentes, body, Leaflet) | ⚠️ Se porta, **menos la carga de fuentes** (ver 17.4) |
| Routing (`react-router-dom`, `BrowserRouter`) | ❌ Se reemplaza por App Router. `react-router-dom` no se instala |
| Datos mock (`src/data/`) | ❌ Se reemplazan por llamadas a la API (sección 6) |
| `src/store/AppStore.tsx` (Context propio) | ⚠️ Se migra a Zustand según sección 4, salvo `AuthContext` |

**No hay conflicto de librería de componentes.** El prototipo no usa shadcn/MUI/Chakra/Mantine — todo es propio sobre Tailwind, que es exactamente lo que asume la sección 4. Se conserva la decisión de un design system propio en `components/ui/`.

---

### 17.2 Paleta

Tres escalas personalizadas. **No hay modo oscuro** y no está previsto.

| Escala | Rol | Tono ancla |
|---|---|---|
| `lagoon` | Primario de marca, texto, superficies frías | `lagoon-500` `#1caab0` |
| `coral` | Acento y **llamadas a la acción** | `coral-500` `#ef5a35` |
| `sand` | Neutros: superficies, bordes, divisores | `sand-200` `#e7ddc9` |

**Reparto de responsabilidades — regla de uso:**

| Uso | Token |
|---|---|
| Texto de títulos y cuerpo principal | `lagoon-900` |
| Texto secundario | `lagoon-600` |
| Color base del documento | `#12333a` (en `body`, fuera de la paleta) |
| Marca: logo, nav activo, primario del admin | `lagoon-500` |
| **CTA del huésped** (reservar, pagar, buscar, registrarse) | `coral-500` → hover `coral-600` |
| Superficie secundaria (secciones, footer, panel admin) | `sand-50` |
| Borde por defecto (tarjetas, inputs, tablas) | `sand-200` |
| Anillo de foco | `lagoon-100` sobre borde `lagoon-400` |
| Botón deshabilitado | `sand-300` |

⚠️ **`lagoon` vs `coral` no es intercambiable.** `lagoon` es identidad y navegación; `coral` es acción del huésped. Un botón "Reservar" en `lagoon-500` rompe la jerarquía de toda la interfaz pública. El admin usa `lagoon-500` como primario porque ahí la acción no compite con nada.

**Tonos definidos pero sin usar:** `lagoon-200`, `coral-200/300/800/900`, `sand-400/500/600/700/900`. No eliminarlos — dan margen para los estados que aún faltan (17.7).

#### Estados semánticos

**No existen tokens `success`/`error`/`warning`/`info`.** Los estados de reserva se resuelven reutilizando la paleta base:

| Estado | Clases |
|---|---|
| `confirmed` | `bg-lagoon-100 text-lagoon-700` |
| `pending` | `bg-coral-100 text-coral-700` |
| `cancelled` | `bg-gray-200 text-gray-600` |
| `completed` | `bg-sand-200 text-sand-800` |

⚠️ **`error` y `warning` no están definidos** y el backend sí los va a necesitar: validación de formularios, fallo de pago, cupón inválido, propiedad no disponible. Ver 17.7 — hay que decidirlos antes de construir el checkout, no durante.

---

### 17.3 Tipografía

| Familia | Rol | Pesos cargados |
|---|---|---|
| **Poppins** (`font-display`) | Encabezados, wordmark, cifras de precio, códigos de reserva, labels de sección | 500, 600, 700, 800 |
| **Inter** (`font-sans`, default del `body`) | Todo el cuerpo de texto | 400, 500, 600, 700 |

El peso **800 de Poppins se carga pero no se usa** — quitarlo del import ahorra una descarga.

`fontSize` **no está personalizado**: aplica la escala por defecto de Tailwind v3. Escala efectiva en uso:

| Rol | Clases | px |
|---|---|---|
| Hero (Home) | `font-display text-4xl md:text-5xl font-bold leading-tight` | 36 → 48 |
| Título de página | `font-display text-2xl md:text-3xl font-bold` | 24 → 30 |
| Título de sección | `font-display text-2xl md:text-3xl font-bold` | 24 → 30 |
| Subsección | `font-display text-lg font-semibold` | 18 |
| Título de tarjeta | `font-display text-[15px] font-semibold leading-snug` | 15 |
| Label / precio | `font-display text-sm font-semibold` | 14 |
| Body | `text-sm` | 14 |
| Body large | `text-base md:text-lg` | 16 → 18 |
| Caption | `text-xs` | 12 |
| Eyebrow | `text-[11px] font-semibold uppercase tracking-wide` | 11 |
| Botón | `text-sm font-semibold` (primario) / `font-medium` (secundario) | 14 |

Line-height e interletraje explícitos: `leading-tight` (1.25) en heros, `leading-snug` (1.375) en títulos de tarjeta, `leading-relaxed` (1.625) en párrafos largos, `tracking-wide` (0.025em) en eyebrows, `tracking-tight` (−0.025em) solo en el wordmark.

⚠️ **Deuda a corregir al portar:** hay `h2` y `h3` en tarjetas admin con `font-display font-semibold` **sin clase de tamaño**, que caen al default del navegador (≈24px y ≈18.7px). Es accidental, no una decisión. Asignarles tamaño explícito al migrar.

`h5` y `h6` no existen en el prototipo. Si aparecen, usar `text-sm font-semibold`.

---

### 17.4 Carga de fuentes — cambio obligatorio en Next.js

El prototipo carga las fuentes con `@import url()` de Google Fonts dentro del CSS. **En Next.js esto no se porta tal cual.**

Un `@import` en CSS bloquea el render, dispara una petición a un tercero después de que el CSS ya se descargó, y provoca salto de layout al intercambiar la fuente. Justo lo que castiga Core Web Vitals, que la sección 13 marca como riesgo del frontend público.

```ts
// src/app/layout.tsx
import { Poppins, Inter } from 'next/font/google';

const poppins = Poppins({
  subsets: ['latin'],
  weight: ['500', '600', '700'],   // el 800 se cargaba sin usarse
  variable: '--font-display',
  display: 'swap',
});

const inter = Inter({
  subsets: ['latin'],
  weight: ['400', '500', '600', '700'],
  variable: '--font-sans',
  display: 'swap',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="es" className={`${poppins.variable} ${inter.variable}`}>
      <body className="font-sans text-[#12333a] antialiased">{children}</body>
    </html>
  );
}
```

Y el `tailwind.config` pasa a apuntar a las variables:

```js
fontFamily: {
  display: ['var(--font-display)', 'ui-sans-serif', 'system-ui', 'sans-serif'],
  sans:    ['var(--font-sans)',    'ui-sans-serif', 'system-ui', 'sans-serif'],
},
```

`next/font` autoaloja los archivos en el propio dominio: sin petición a `fonts.googleapis.com`, sin salto de layout, y una cosa menos que permitir en la CSP.

---

### 17.5 Espaciado, radios, sombras, breakpoints

**Espaciado:** escala por defecto de Tailwind (1 = 4px). Sin valores arbitrarios. Padding horizontal estándar de página: `px-4 sm:px-6 lg:px-8`. Solo se usa `space-y-*`, nunca `space-x-*`.

**Radios** (defaults de Tailwind, sin personalizar):

| Clase | Uso |
|---|---|
| `rounded-xl` (12px) | Inputs, selects, textareas, links del sidebar |
| `rounded-2xl` (16px) | **Radio estándar de tarjeta** — tarjetas, paneles, secciones, widget de reserva |
| `rounded-3xl` (24px) | Contenedores destacados: buscador, AdminLogin, Confirmation |
| `rounded-full` | Botones pill, avatares, chips, celdas de calendario |

**Sombras** — las únicas dos personalizadas, y cubren todo el sistema:

```js
card: '0 6px 20px -6px rgba(24, 91, 97, 0.18)',   // tarjetas, paneles, nav activo
pop:  '0 12px 40px -8px rgba(24, 91, 97, 0.28)',  // elementos flotantes: buscador, dropdowns, widget
```

Ambas están teñidas de `lagoon` (`rgba(24,91,97,…)`), no de negro puro. Es lo que da la sensación cálida del conjunto — **no sustituirlas por `shadow-lg` genérico** al portar componentes.

**Breakpoints:** defaults de Tailwind. `sm/md/lg` de uso intensivo, `xl` en 3 lugares, `2xl` sin usar. Hay un breakpoint en JS: `PropertyDetail` alterna el calendario entre 1 y 2 meses con `window.innerWidth >= 768` — al portar, eso necesita `'use client'` y guardas de `typeof window`.

---

### 17.6 Inventario de componentes

#### A. Se portan tal cual (`components/`)

| Componente | Props | Nota de migración |
|---|---|---|
| `Header` | `transparent?: boolean` | `'use client'` (menú móvil) |
| `Footer` | — | Server Component |
| `PropertyCard` | `property`, `index?` | `'use client'` (carrusel, favorito, framer-motion). `<img>` → `next/image` |
| `SearchBar` | `compact?: boolean` | `'use client'`. Navegación → `useRouter` de `next/navigation` |
| `AvailabilityCalendar` | `bookedDates`, `blockedDates?`, `selectable?`, `checkIn?`, `checkOut?`, `onSelect?`, `months?` | `'use client'`. **Ver 17.8: le falta mostrar precio por noche** |
| `CalendarLegend` | — | Server Component |
| `PropertyMap` / `SingleLocationMap` | `properties`/`property`, `activeId?`, `height?` | `'use client'` + **`dynamic(..., { ssr: false })`** — Leaflet toca `window` al importarse y revienta el render de servidor |
| `LanguageSelector` | `light?: boolean` | `'use client'` |
| `StarRating` | `rating`, `count?`, `size?: 'sm'\|'md'`, `showCount?` | Server Component |
| `StatusBadge` | `status: BookingStatus` | Server Component |
| `AdminLayout` | `children`, `title` | `'use client'` (drawer) |

#### B. A promover a `components/ui/` (hoy duplicados)

El prototipo define los mismos componentes varias veces dentro de distintas páginas. Al portar se unifican:

| Componente | Duplicado en | Destino |
|---|---|---|
| `Field` / `Input` | `Auth`, `AdminLogin`, `Checkout`, `AdminPropertyForm` (4 copias) | `ui/Input.tsx` + `ui/Field.tsx` |
| `Row` | `PropertyDetail`, `Checkout`, `AdminReservationDetail` (3 copias) | `ui/Row.tsx` |
| `SectionHeading` | `Home` | `ui/SectionHeading.tsx` |
| `EmptyState` | `Profile` | `ui/EmptyState.tsx` |
| `Section` / `Label` / `NumberField` | `AdminPropertyForm` | `ui/` |

Cuatro copias de un input es el momento exacto de extraer que menciona la sección 14 (*"extraer solo cuando se repitan 3+ veces"*). Ya se cumplió.

`tailwind-merge` está instalado y sin usar — es justo lo que hace falta para que estos componentes acepten `className` sin conflictos de clases. Conservarlo.

#### C. Estados transversales

| Estado | Implementación actual |
|---|---|
| Hover | `hover:bg-sand-100`, `hover:bg-coral-600`, `hover:shadow-card`, `group-hover:scale-105` |
| Foco | `focus:border-lagoon-400 focus:ring-2 focus:ring-lagoon-100` — **solo en inputs** |
| Deshabilitado | `disabled:opacity-70`, `disabled:cursor-not-allowed disabled:bg-sand-300` |
| Cargando | Booleano que cambia el texto del botón + `disabled`. Sin spinner ni skeleton |
| Error | ✅ Definido en 17.7.3 — `border-danger-500` + texto `text-xs text-danger-600` + `aria-invalid` |
| Vacío | Borde punteado `border-dashed border-sand-300` |

---

### 17.7 Huecos cerrados — decisiones del sistema de diseño

Lo que el prototipo no resolvió y el backend sí exige. Los seis quedaron decididos; esta subsección es la referencia de implementación.

#### 17.7.1 Tokens semánticos — ✅ marca propia, estados del sistema con defaults de Tailwind

**La regla:** los colores de **marca** salen de las escalas propias (`lagoon`, `coral`, `sand`); los **estados del sistema** salen de las escalas por defecto de Tailwind.

Ya hay precedente en el prototipo: `StatusBadge` usa `bg-gray-200 text-gray-600` para el estado `cancelled`. Esto solo formaliza lo que ya se estaba haciendo.

| Estado | Token | Hex | Uso |
|---|---|---|---|
| **Error** | `red-600` / `red-50` | `#dc2626` / `#fef2f2` | Texto y borde / fondo de alerta |
| **Advertencia** | `amber-500` / `amber-50` | `#f59e0b` / `#fffbeb` | Texto y borde / fondo de alerta |
| **Éxito** | `lagoon-500` / `lagoon-100` | `#1caab0` / `#d5f6f4` | Ya en uso (icono de confirmación, badge `confirmed`) |
| **Info** | `lagoon-400` / `lagoon-50` | `#38c6c8` / `#effcfb` | Ya en uso |

**Por qué no se inventaron escalas nuevas:** la paleta ocupa **todo el rango cálido** — `coral` es un rojo-naranja y `sand` es un ámbar desaturado. Diseñar un rojo y un ámbar propios que se despeguen de ambos exige validación de contraste sobre cada superficie, trabajo que el prototipo no hizo. Los defaults de Tailwind ya vienen con contraste verificado.

⚠️ **La cercanía entre `red-600` y `coral-500` es real y hay que gestionarla por forma, no por color:** el error es texto pequeño oscuro o un borde fino; el CTA es un botón relleno grande. **Nunca un botón rojo relleno junto a uno coral** — ahí sí se pierde la distinción. Un botón destructivo (ej. "dar de baja propiedad") va con borde rojo y fondo transparente, no relleno.

Aliases semánticos en la config, para que la intención esté en el nombre de la clase:

```js
// tailwind.config.js → theme.extend.colors
import colors from 'tailwindcss/colors';

danger:  colors.red,      // text-danger-600, bg-danger-50, border-danger-500
warning: colors.amber,
// success e info NO llevan alias: son lagoon, que ya es un nombre semántico
```

Un `text-red-600` crudo dentro de un componente es señal de que alguien no usó el token.

#### 17.7.2 Foco visible — ✅ `focus-visible`, no `focus`

Hoy solo los inputs tienen anillo de foco: navegar el sitio con teclado es inviable.

```
focus-visible:outline-none
focus-visible:ring-2 focus-visible:ring-lagoon-400
focus-visible:ring-offset-2 focus-visible:ring-offset-white
```

**`focus-visible` y no `focus`** a propósito: el anillo aparece al navegar con teclado, no al hacer clic con el ratón. Con `focus` a secas todos los botones parpadean un anillo al pulsarlos, y es lo que lleva a que alguien lo desactive entero — dejando otra vez el sitio inaccesible.

⚠️ Sobre fondos oscuros (hero de Home, `Auth`, `AdminLogin`, header transparente) el offset cambia a `focus-visible:ring-offset-lagoon-900`, o el anillo se funde con el fondo.

Se aplica a **botones, links, chips, celdas de calendario y controles del carrusel**, no solo a inputs.

#### 17.7.3 Validación de formularios — ✅ inline bajo el campo

El prototipo solo usa `required` nativo. Checkout, alta de propiedad y promociones necesitan error por campo.

```tsx
<input
  aria-invalid={!!error}
  aria-describedby={error ? `${id}-error` : undefined}
  className={cn(
    'rounded-xl border px-4 py-2.5',
    error
      ? 'border-danger-500 focus:border-danger-500 focus:ring-2 focus:ring-danger-100'
      : 'border-sand-200 focus:border-lagoon-400 focus:ring-2 focus:ring-lagoon-100'
  )}
/>
{error && (
  <p id={`${id}-error`} className="mt-1.5 text-xs text-danger-600">{error}</p>
)}
```

`aria-invalid` y `aria-describedby` no son opcionales: sin ellos un lector de pantalla anuncia el campo pero no el motivo del fallo.

**Resumen arriba del formulario solo para errores sin campo dueño** — pago rechazado, propiedad ya no disponible, cupón agotado. Un resumen que repite errores que ya están inline duplica ruido.

Los errores llegan en el envoltorio `errors` de la API (sección 1.4) y `apiClient.ts` los mapea campo→mensaje. Un fallo de validación de Laravel (422) alimenta los errores inline; un 409 o 5xx alimenta el resumen.

#### 17.7.4 Skeletons — ✅ forma del contenido, nunca spinner centrado

```tsx
<div className="h-64 w-full animate-pulse rounded-2xl bg-sand-100" />
```

Se aprovecha `loading.tsx` por segmento de ruta del App Router, más `<Suspense>` para las partes lentas del detalle de propiedad (mapa, calendario).

**Spinner solo dentro de botones** al enviar un formulario — es donde el prototipo ya cambia el texto por `'…'`, y ahí sí funciona porque el ámbito es pequeño y conocido. Un spinner centrado en toda la página no dice nada sobre lo que está por llegar; un skeleton con la forma del listado sí.

#### 17.7.5 Z-index — ✅ escala nombrada

El prototipo usa `z-0/20/30/40/50` sueltos. Con el chat flotante nuevo, el drawer, el dropdown de idioma y el mapa, la colisión era cuestión de tiempo.

```js
// tailwind.config.js → theme.extend.zIndex
zIndex: {
  map:      '0',   // .leaflet-container — ya estaba en z-0
  sticky:  '20',   // header pegado
  dropdown:'30',   // selector de idioma, filtros
  drawer:  '40',   // menú móvil, panel de filtros
  chat:    '45',   // burbuja flotante: sobre el drawer, bajo el modal
  modal:   '50',
  toast:   '60',   // siempre encima
}
```

`z-chat: 45` es la pieza que no existía. Va **sobre** el drawer (el chat debe seguir accesible con el menú abierto) y **bajo** el modal (un diálogo de confirmación no puede quedar tapado por una burbuja).

Regla: **ningún `z-[número]` suelto en componentes.** Si hace falta una capa nueva, se añade a la escala.

#### 17.7.6 Animación — ✅ defaults + `prefers-reduced-motion`

| Uso | Valor |
|---|---|
| Por defecto (colores, transformaciones) | `duration-200 ease-out` |
| Zoom de imagen en `PropertyCard` | `duration-500` — ya en el prototipo, se conserva |
| Entradas con framer-motion | 0.25s, escalonado 0.05s con tope 0.4s — ya en el prototipo |

**Añadido que falta en el prototipo: respetar `prefers-reduced-motion`.**

```tsx
import { useReducedMotion } from 'framer-motion';

const reduce = useReducedMotion();
<motion.div
  initial={reduce ? false : { opacity: 0, y: 12 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: reduce ? 0 : 0.25, delay: reduce ? 0 : delay }}
/>
```

No es una preferencia estética: para personas con sensibilidad vestibular, una cuadrícula de tarjetas entrando escalonadas provoca mareo. El sistema operativo ya expone el ajuste; ignorarlo es el fallo.

Para las transiciones CSS, la contrapartida en `globals.css`:

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

La última línea anula el `scroll-behavior: smooth` que el prototipo pone en `html`.

---

### 17.8 Lo que falta para las secciones 15 y 16

El prototipo cubre el flujo de reserva, pero **no** el motor de precios ni el chat. Componentes por construir desde cero:

**Precios y promociones (sección 15):**

| Componente | Para qué |
|---|---|
| `PriceBreakdown` | Renderiza el `QuoteDTO`: noches, descuentos, cargos, impuestos, total. **Solo muestra, nunca recalcula** |
| `PromoCodeInput` | Aplicar/quitar cupón con feedback de validez |
| `SeasonCalendar` (admin) | Temporadas pintadas sobre el año — usa la columna `color` de la tabla `seasons` |
| `SeasonForm` (admin) | Alta con `adjust_type`, `recurrence: yearly`, prioridad |
| `PricingPreview` (admin) | Simulación de 12 meses antes de guardar |
| Pantalla `/admin/promotions` | **No existe en el prototipo.** `AdminPricing` solo lista precios |

⚠️ **`AvailabilityCalendar` necesita ampliarse.** Hoy muestra estados (disponible/reservado/bloqueado/seleccionado) pero **no el precio por noche ni el mínimo de noches**, que es lo que devuelve `GET /properties/{id}/availability` (sección 6). Cada celda pasa de ser un número a ser día + precio, y con `min_nights` la selección puede quedar rechazada. Es más que un retoque visual.

**Chat (sección 16):**

| Componente | Estado |
|---|---|
| `AdminMessages` | ✅ Existe. Panel dividido, burbujas (`lagoon-500` = host, `max-w-[75%]`, cola con `rounded-br-md`/`rounded-bl-md`), hover de hilo en `lagoon-50` |
| `ChatWidget` (público) | ❌ **No existe.** Burbuja flotante del lado del huésped — usar `shadow-pop`, `z-chat` |
| `TypingIndicator` | ❌ No existe |
| Estado de conexión WS | ❌ No existe. Hace falta un aviso cuando el socket cae y se degrada a polling |
| Contador de no leídos | ❌ No existe en el header |

El estilo de burbuja del admin ya está resuelto y se reutiliza en el widget público invirtiendo los lados.

---

### 17.9 Decisiones sobre conflictos con lo documentado

#### 1. Mapas — ✅ DECIDIDO: híbrido (Leaflet + proveedor de tiles, y Google Places solo para autocompletado)

El prototipo usa `leaflet` + `react-leaflet`. La sección 10 documentaba **Google Maps Platform** para todo. Se resuelve dividiendo por superficie:

| Superficie | Tecnología | Volumen |
|---|---|---|
| Mapa público de resultados (`PropertyMap`) y de detalle (`SingleLocationMap`) | **Leaflet + proveedor de tiles** | Alto — lo carga cada visitante |
| Autocompletado de dirección en `AdminPropertyForm` | **Google Places Autocomplete** | Mínimo — las casas se dan de alta una vez |

⚠️ **Los tiles del prototipo hay que cambiarlos.** Hoy apuntan al servidor público de OpenStreetMap, cuya [política de uso](https://operations.osmfoundation.org/policies/tiles/) establece que se financia con donativos, que el uso comercial debe autoalojar o usar un tercero, y que **pueden bloquear el acceso sin aviso**. Un sitio de renta de casas es uso comercial. Leaflet (la librería) sí es gratis; los tiles no son un recurso ilimitado. Ver [`servicios/13-mapas-tiles.md`](../servicios/13-mapas-tiles.md).

**Por qué híbrido y no todo en un proveedor:**

Los dos usos tienen perfiles opuestos: el mapa público es de alto volumen y baja exigencia (mostrar tiles y pines); el autocompletado es de volumen mínimo y alta exigencia (precisión de dirección). Poner todo en Google cobra por el lado de alto volumen para resolver un problema que solo existe en el de volumen mínimo.

**El argumento decisivo es la reversibilidad:** cambiar de proveedor de tiles en Leaflet es **una línea** (la URL del `TileLayer`). Salir de Google Maps obliga a reescribir ambos componentes de mapa. La opción híbrida deja el componente de alto tráfico en la tecnología con costo de salida cero.

**Orden de ejecución:**

1. **Ahora:** cambiar la URL de tiles a un proveedor con cuenta (MapTiler / Stadia / Geoapify). Es el fix del riesgo de bloqueo.
2. **Al construir `AdminPropertyForm`** (fase 3 del roadmap): añadir Google Places, **con session tokens** y API key restringida por HTTP referrer.

No crear la cuenta de Google antes del paso 2 — evita dejar una key olvidada sin restricciones.

**Descartado:** autoalojar tiles (requiere PostGIS y decenas de GB, desproporcionado); Nominatim para geocodificar (mismo tipo de servicio donado y frágil que los tiles de OSM, con peor cobertura de direcciones en México); prescindir del autocompletado (un pin mal puesto en renta vacacional es un huésped perdido de noche, no un detalle cosmético).

#### 2. Multi-idioma

El prototipo trae i18n propio completo (`LanguageProvider`, `LanguageSelector` en huésped y admin). La sección 14 dice explícitamente: *"no construir multi-tenant ni multi-idioma de propiedades hasta que exista un requerimiento real"*.

El prototipo **ya lo construyó**, así que el costo de la UI está pagado. Pero hay una diferencia que sí cuesta:

- **Traducir la interfaz** (labels, botones, textos fijos): ya hecho, se conserva.
- **Traducir el contenido de las propiedades** (nombre, descripción): **no está hecho** y sí es caro — implica columnas o tabla de traducciones en `properties`, panel de edición por idioma, y `hreflang` en las rutas para no perder SEO.

Si el mercado incluye turistas internacionales (que es el argumento para tener Stripe además de Mercado Pago), lo primero se queda. Lo segundo debería decidirse ahora, porque afecta el esquema de la sección 5 y las rutas SEO de la sección 4.

---

### 17.10 Dependencias

| Paquete | Acción |
|---|---|
| `react`, `react-dom` | Los provee Next.js |
| `react-router-dom` | ❌ **No instalar** — se usa App Router |
| `tailwindcss` | ✅ Fijar versión explícita en `package.json` (el prototipo no la declara) |
| `framer-motion` | ✅ Conservar. Requiere `'use client'` |
| `lucide-react` | ✅ Conservar. Convención con sufijo `Icon`. Tamaño más común `h-4 w-4` |
| `leaflet`, `react-leaflet` | ✅ Conservar (decisión 17.9). Import dinámico con `ssr: false` + **cambiar la URL de tiles** a un proveedor con API key |
| `date-fns` | ✅ Instalado y **sin usar** — el prototipo formatea con `Intl`. Se va a necesitar de verdad para el calendario y los rangos de temporadas |
| `tailwind-merge` | ✅ Instalado y sin usar — necesario para los componentes de `ui/` con `className` (17.6.B) |
| `@radix-ui/react-icons` | ❌ Instalado y sin usar. **Quitar** |
| `zustand` | ➕ Añadir (sección 4). Sustituye a `AppStore.tsx` |
| `laravel-echo`, `pusher-js` | ➕ Añadir para el chat (sección 16) |

---

### 17.11 Checklist de migración a Next.js

1. Copiar `tailwind.config.js` sin cambios, salvo `fontFamily` → variables CSS (17.4).
2. Fuentes con `next/font/google`, eliminar el `@import` de Google Fonts del CSS.
3. Portar `index.css` (body, `.no-scrollbar`, CSS de Leaflet). `.no-scrollbar` está definida y sin usar — conservarla, sirve para el carrusel.
4. Marcar `'use client'`: `Header`, `SearchBar`, `PropertyCard`, `AvailabilityCalendar`, `LanguageSelector`, `AdminLayout`, mapas, y todo lo de chat.
5. `PropertyMap`/`SingleLocationMap` con `dynamic(..., { ssr: false })`.
6. `<img>` → `next/image` en tarjetas, galería y hero (Core Web Vitals, sección 13).
7. `react-router-dom` → App Router: `<Link>` de `next/link`, `useRouter`/`useSearchParams` de `next/navigation`. `ScrollToTop` se elimina — App Router ya lo hace.
8. Rutas: `/property/:slug` → `app/(public)/properties/[slug]/page.tsx`, `/admin/*` → `app/(admin)/dashboard/*`.
9. Unificar los componentes duplicados en `ui/` (17.6.B).
10. Reemplazar `src/data/` mock por los servicios de `services/` contra la API.
11. Añadir a `tailwind.config.js` los tokens de 17.7 (`danger`, `warning`, escala `zIndex`) y el bloque `prefers-reduced-motion` a `globals.css`, **antes** de construir checkout y formularios admin.
12. `window.innerWidth` de `PropertyDetail` → hook con guarda de `typeof window`.

---

Ver también: sección 4 (estructura del frontend), sección 15 (motor de precios), sección 16 (chat).

---

