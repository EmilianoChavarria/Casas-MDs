# Servicio: Google Maps Platform

## ¿Para qué se usa?
1. Mostrar la ubicación exacta de cada propiedad en su página de detalle (mapa embebido).
2. Autocompletar direcciones al crear/editar una propiedad en el dashboard admin (Places Autocomplete).
3. Opcionalmente, mapa de búsqueda con pines de todas las propiedades disponibles.

## Justificación
Estándar del mercado para geolocalización, mejor cobertura y precisión en México que alternativas gratuitas (OpenStreetMap requiere más esfuerzo de mantenimiento aunque es gratis — considerarlo solo si el presupuesto es muy ajustado).

**Alternativa gratuita:** Mapbox o Leaflet + OpenStreetMap si se quiere evitar el costo de Google, con menor precisión de geocodificación en zonas rurales.

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
- Google Maps Platform usa modelo **pay-as-you-go** con $200 USD de crédito gratuito mensual (recurrente).
- Costos aproximados: Maps JavaScript API ~$7 USD/1,000 cargas; Places Autocomplete ~$2.83 USD/1,000 solicitudes (sesión); Geocoding ~$5 USD/1,000 solicitudes.
- Para un catálogo de propiedades de una sola empresa (no marketplace masivo), el crédito gratuito mensual normalmente cubre el uso completo.
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

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 5 y 10.
