# Servicio: Google Maps Platform

> ✅ **Alcance reducido (decisión de la sección 17.9).** Este servicio **ya no cubre los mapas visibles**. Los mapas públicos van con Leaflet + un proveedor de tiles ([`13-mapas-tiles.md`](13-mapas-tiles.md)), porque son la superficie de mayor volumen y menor exigencia, y su costo de salida es una línea de código.
>
> De Google Maps Platform se usa **únicamente Places Autocomplete**, y solo en el formulario admin de alta/edición de propiedades. **No contratar hasta la fase 3 del roadmap** — así no queda una API key olvidada sin restricciones.

## ¿Para qué se usa?

1. **Places Autocomplete** al crear/editar una propiedad en el dashboard admin: autocompletar la dirección y obtener `lat`/`lng` precisos.

~~2. Mapa embebido en la página de detalle~~ → Leaflet ([`13-mapas-tiles.md`](13-mapas-tiles.md))
~~3. Mapa de búsqueda con pines~~ → Leaflet ([`13-mapas-tiles.md`](13-mapas-tiles.md))


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


### Variables de entorno
```
# Frontend (Next.js) — clave pública restringida por dominio
NEXT_PUBLIC_GOOGLE_MAPS_KEY=AIzaSy_xxxxx_frontend

# Backend (Laravel) — clave restringida por IP, usada solo para geocoding server-side
GOOGLE_MAPS_SERVER_KEY=AIzaSy_xxxxx_backend
```

### Uso en frontend (resumen)
```tsx
// components/property/PropertyMap.tsx
<APIProvider apiKey={process.env.NEXT_PUBLIC_GOOGLE_MAPS_KEY!}>
  <Map center={{ lat: property.lat, lng: property.lng }} zoom={14}>
    <Marker position={{ lat: property.lat, lng: property.lng }} />
  </Map>
</APIProvider>
```

### Geocoding server-side al guardar propiedad (resumen)
```php
// PropertyService.php
public function geocodeAddress(string $address): array
{
    $response = Http::get('https://maps.googleapis.com/maps/api/geocode/json', [
        'address' => $address,
        'key' => config('services.google_maps.server_key'),
    ]);

    $location = $response->json('results.0.geometry.location');
    return ['lat' => $location['lat'], 'lng' => $location['lng']];
}
```

Referenciado desde: `../arquitectura/`, secciones 5 y 10.
