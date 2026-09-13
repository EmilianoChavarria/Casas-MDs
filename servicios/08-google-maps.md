# Servicio: Google Maps Platform

> ✅ **Alcance reducido (decisión de la sección 17.9).** Este servicio **ya no cubre los mapas visibles**. Los mapas públicos van con Leaflet + un proveedor de tiles ([`13-mapas-tiles.md`](13-mapas-tiles.md)), porque son la superficie de mayor volumen y menor exigencia, y su costo de salida es una línea de código.
>
> De Google Maps Platform se usa **únicamente Places Autocomplete**, y solo en el formulario admin de alta/edición de propiedades. **No contratar hasta la fase 3 del roadmap** — así no queda una API key olvidada sin restricciones.

## ¿Para qué se usa?

1. **Places Autocomplete** al crear/editar una propiedad en el dashboard admin: autocompletar la dirección y obtener `lat`/`lng` precisos.

~~2. Mapa embebido en la página de detalle~~ → Leaflet ([`13-mapas-tiles.md`](13-mapas-tiles.md))
~~3. Mapa de búsqueda con pines~~ → Leaflet ([`13-mapas-tiles.md`](13-mapas-tiles.md))

### Punto de encuentro de las experiencias (sección 20.8)

El prototipo de experiencias trae un *embed* de Google Maps para el punto de encuentro. **Recomendación: reutilizar `SingleLocationMap` (Leaflet)** — es exactamente el mismo problema que la ubicación de una casa, y la decisión 17.9 ya está tomada.

Si el cliente insiste en Google para esa pantalla, hay que usar la **Maps Embed API**, no la Maps JavaScript API:

| | Maps **Embed** API (`<iframe>`) | Maps **JavaScript** API |
|---|---|---|
| Costo | **Gratis, sin límite** en modo básico (`place`, `view`, `directions`) | ~$7 USD / 1,000 cargas pasada la cuota |
| Cuota Essentials | No la consume | Sí |
| Interactividad | La del iframe de Google | Total (pines propios, capas) |

⚠️ **Es gratis, pero no es gratis de mantener:** añade un segundo stack de mapas al proyecto y una API key más que restringir por *referrer*. El costo aquí no es dinero, es superficie.


## Justificación

Estándar del mercado para geocodificación, con mejor cobertura y precisión de direcciones en México que las alternativas gratuitas — sobre todo en zonas rurales y turísticas, que es justo donde suelen estar las casas de renta vacacional.

**Por qué solo para el autocompletado:** el alta de propiedades es una pantalla interna que se usa una vez por casa y casi nunca se edita — volumen mínimo, muy por debajo de cualquier free tier. Pero la exigencia de precisión es alta: un `lat`/`lng` mal capturado es un huésped perdido buscando la casa de noche. Es el reparto inverso al del mapa público, y por eso cada superficie usa un proveedor distinto.

**Alternativa descartada:** Nominatim (geocodificador de OpenStreetMap). Es un servicio donado con límite de 1 petición/segundo y política de uso restrictiva — la misma fragilidad que los tiles públicos de OSM — y con peor cobertura de direcciones en México.

⚠️ **Riesgo principal a controlar — facturación de Places.** Places Autocomplete cobra **por sesión** solo si se implementan *session tokens*. Sin ellos factura **cada pulsación de tecla**, que es la causa clásica de facturas inesperadas en un formulario que se usa diez veces al mes. Dos medidas obligatorias:
> 1. Usar `AutocompleteSessionToken` en cada sesión de captura.
> 2. Restringir la API key por **HTTP referrer** a los dominios exactos de producción y staging, y limitarla a la API de Places únicamente.


## 💰 Precio y plan gratuito para desarrollo

⚠️ **Cambio importante (desde el 1-mar-2025):** Google eliminó el crédito mensual de $200 USD que este documento mencionaba antes. Ahora el modelo es de **cuotas gratuitas por API (SKU)**, no un crédito compartido:

| Plan | Llamadas gratis por API/mes | Costo si se excede |
|---|---|---|
| **Essentials** (el que aplica a este proyecto) | 10,000 llamadas gratis **por cada API** (Maps JS, Places, Geocoding cuentan cada una por separado) | Desde ~$2–7 USD por 1,000 llamadas según la API, una vez agotada la cuota gratis |
| Pro | 5,000 llamadas gratis por API/mes | Mayor volumen, precio distinto por SKU |
| Enterprise | 1,000 llamadas gratis por API/mes | Volumen alto, precio negociado |

**¿Sirve para desarrollo?** Sí — para un catálogo de propiedades (no un marketplace masivo), las 10,000 llamadas gratis mensuales de cada API bajo el plan Essentials normalmente cubren tanto el desarrollo como el arranque en producción sin generar costo. Aun así requiere **habilitar facturación con tarjeta** desde el inicio (Google no da acceso sin ella, aunque no cobre nada si no se excede la cuota).

**Recomendación:** configurar alertas de presupuesto desde el día uno (ver abajo) para detectar cualquier excedente antes de que genere un cargo relevante.


## Ruta de creación

1. Crear/usar cuenta de Google Cloud: https://console.cloud.google.com
2. Crear un proyecto nuevo (ej. `renta-casas-maps`).
3. Habilitar facturación (requiere tarjeta, aunque el tier gratuito mensual suele cubrir el uso de un proyecto pequeño).
4. **APIs & Services → Library** → habilitar:
   - Maps JavaScript API (mapa embebido)
   - Places API (autocompletado de direcciones)
   - Geocoding API (convertir dirección → lat/lng)
5. **APIs & Services → Credentials → Create Credentials → API Key**.
6. **Restringir la API Key** (importante por seguridad):
   - Restricción de aplicación: "Referencias HTTP (sitios web)" → agregar `https://midominio.com/*` y `https://www.midominio.com/*`.
   - Restricción de API: limitar solo a las 3 APIs habilitadas arriba.
7. Para el uso desde el backend (Geocoding al guardar una propiedad), crear una **segunda API Key** separada, restringida por IP (la IP del VPS), nunca la misma key del frontend.


## Contrato / plan recomendado

- Google Maps Platform usa el plan **Essentials** (pay-as-you-go) con 10,000 llamadas gratis **por cada API** al mes (Maps JavaScript, Places, Geocoding se cuentan cada una por separado; ya no comparten un crédito único de $200 USD como antes de marzo 2025).
- Costos aproximados una vez agotada la cuota gratis de cada API: Maps JavaScript API ~$7 USD/1,000 cargas; Places Autocomplete ~$2.83 USD/1,000 solicitudes (sesión); Geocoding ~$5 USD/1,000 solicitudes (verificar tarifas vigentes en https://mapsplatform.google.com/pricing/, Google las ajusta con frecuencia).
- Para un catálogo de propiedades de una sola empresa (no marketplace masivo), las 10,000 llamadas gratis mensuales de cada API normalmente cubren el uso completo en desarrollo y en el arranque.
- Configurar **alertas de presupuesto** en Google Cloud (Billing → Budgets & alerts) para evitar sorpresas.


## Configuración

> ✅ **Implementado.** Lo que sigue es lo que hay en el código, no una
> propuesta. Rama `feat/ubicacion-y-calendario`.

### ⚠️ Desviación respecto al plan: una sola llave, y de servidor

Este documento proponía **dos llaves**: una pública en el frontend
restringida por *referrer*, y otra de servidor restringida por IP. Se
implementó **solo la segunda**, y todo pasa por Laravel.

El motivo: una llave de Places en el frontend es pública por definición
—se copia de la pestaña de red— y Places se factura por petición.
Restringirla por dominio ayuda, pero no impide que la use un script
**desde ese dominio**. Con la llave en el servidor, el único que puede
gastarla es un admin con sesión, que es exactamente quien da de alta
casas. El coste es un salto de red más por pulsación; a cambio, el gasto
tiene dueño y se puede cortar.

Esto también resuelve el "riesgo principal a controlar" que menciona este
documento más arriba: el endpoint está detrás de `auth:sanctum` + `admin`
y con `throttle`, y el *session token* lo pone el backend.

### Variables de entorno (`.env` de Laravel)
```
GOOGLE_MAPS_API_KEY=AIzaSy_xxxxx
```

Una sola. **No existe `NEXT_PUBLIC_GOOGLE_MAPS_KEY`** y no debe crearse.

APIs a habilitar: **Places API (New)** y **Maps Static API**. No hace
falta Maps JavaScript API ni Geocoding API: el detalle del lugar ya trae
`lat`/`lng`, así que la geocodificación aparte sobra.

Restricción de la llave: **direcciones IP** (la del VPS), y restricción
de API a esas dos.

### Endpoints

Todos bajo `/api/v1/admin/geo`, con sesión de admin:

| Ruta | Qué hace |
|---|---|
| `GET geo/status` | ¿Hay llave configurada? |
| `GET geo/autocomplete?q=&session=` | Sugerencias mientras se escribe |
| `GET geo/places/{placeId}` | Ficha del lugar, desmenuzada, más la zona |
| `GET geo/static-map?lat=&lng=&zoom=` | Imagen del mapa, servida por nosotros |

`geo/places/{placeId}` devuelve además la **zona**: si la colonia ya es
una zona del catálogo llega emparejada, y si no, llega una propuesta para
crearla de un clic. Ver la sección de zonas en `../arquitectura/`.

### Degradación sin llave

Sin `GOOGLE_MAPS_API_KEY`, `geo/status` responde `configured: false` y el
formulario pide la dirección a mano, con latitud y longitud. **El alta de
casas no puede quedar bloqueada porque nadie haya dado de alta la
facturación de Google todavía.**

Los fallos se distinguen: **503** `geo_not_configured` (falta la llave →
captura manual) y **502** `geo_unavailable` (Google falló → se
reintenta). Sin esa distinción las dos cosas llegarían como un 500 y el
formulario no sabría cuál de las dos ofrecer.

### Control de gasto ya implementado

- **Una petición por pausa al teclear**, no por letra. Sin esto, escribir
  una dirección son ~20 cargos.
- **Session token** en cada sesión de captura, puesto por el backend.
- **La ficha de un lugar se cachea una semana**: abrir la misma casa a
  editar diez veces es una sola llamada facturable.

### El mapa: Static Maps, no el SDK

La vista previa del formulario es un **PNG** servido por Laravel, con el
pin movible haciendo clic sobre la imagen (la conversión píxel↔coordenada
es Web Mercator, en `MapPreview.tsx`).

No es un adorno: Google devuelve el punto de la *dirección*, que en una
calle larga cae en el centro de la manzana y no en la casa.

Se eligió Static Maps sobre el SDK de mapas porque la preview solo tiene
que enseñar dónde cayó el pin. Cargar la librería entera para pintar un
cuadro de 640×320 significa traer un mapa interactivo, sus cookies y una
llave pública para nada. **Esto no contradice la decisión 17.9**: los
mapas *públicos* siguen sin ser de Google — hoy son un enlace a Google
Maps por `lat`/`lng` desde la ficha, y Leaflet sigue pendiente
([`13-mapas-tiles.md`](13-mapas-tiles.md)).

⚠️ Static Maps **es un SKU aparte** y consume su propia cuota de 10,000
llamadas gratis al mes. Cada redibujado del mapa (mover el pin, cambiar
el zoom) es una llamada.

Referenciado desde: `../arquitectura/`, secciones 5 y 10.
