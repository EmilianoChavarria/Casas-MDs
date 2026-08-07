# Configuración


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
