# Ruta de creación

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

