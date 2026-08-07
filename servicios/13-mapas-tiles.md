# Servicio: Tiles de Mapa (Leaflet) — MapTiler / Stadia / Geoapify

## ¿Para qué se usa?

Servir las imágenes de fondo (*tiles*) de los mapas que renderiza Leaflet:

1. **Mapa de resultados de búsqueda** (`PropertyMap`) — pines de todas las propiedades disponibles, con precio, en `/search`.
2. **Mapa de ubicación** (`SingleLocationMap`) — posición de una propiedad en su página de detalle.

No cubre el autocompletado de direcciones del dashboard admin — eso va con Google Places, ver [`08-google-maps.md`](08-google-maps.md) y la sección 17.9.

## Justificación

**Leaflet es gratis; los tiles no son gratis en producción.** Es la distinción que decide este servicio.

El prototipo apunta al servidor público de tiles de OpenStreetMap. Su [Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/) dice explícitamente que ese servidor se financia con donativos y tiene capacidad limitada, que **el uso comercial debe autoalojar o usar un proveedor tercero**, y que el acceso **puede retirarse sin previo aviso** si degrada el servicio. Un sitio de renta vacacional es uso comercial.

Es decir: el prototipo funciona hoy, pero apoyado en infraestructura donada que no está destinada a eso. Dejarlo así en producción es aceptar que el mapa puede dejar de cargar un día cualquiera, sin aviso ni soporte.

**Por qué un proveedor de tiles y no Google Maps para esta superficie:** el mapa público es la superficie de **mayor volumen** del sistema (lo carga cada visitante en páginas SSR indexadas) y la de **menor exigencia funcional** (mostrar fondo y pines). Google cobraría justo por el lado caro. Además, cambiar de proveedor de tiles en Leaflet es **una línea de código** — la URL del `TileLayer` — mientras que salir de Google Maps obliga a reescribir ambos componentes. El costo de salida es la razón principal.

**Descartado — autoalojar tiles:** requiere PostGIS, importar el planet de OSM (decenas de GB) y un pipeline de renderizado. Desproporcionado frente a un free tier.

## 💰 Precio y plan gratuito para desarrollo

| Opción | Notas |
|---|---|
| **Servidor público de OSM** | $0, sin cuenta. ✅ Válido en **desarrollo local**. ❌ **No usar en producción** — contra su política de uso |
| **MapTiler** | Free tier permanente con API key; estilos vectoriales y raster, buena calidad visual |
| **Stadia Maps** | Free tier permanente; incluye estilos Alidade/Outdoors. Históricamente generoso para volumen bajo |
| **Geoapify** | Free tier permanente con API key; también ofrece geocodificación |
| **Thunderforest / Jawg** | Alternativas equivalentes con free tier y API key |

⚠️ **Verificar los límites vigentes al momento de contratar.** Los free tiers de tiles cambian con frecuencia y varían según si el uso es comercial. Confirmar en la página de precios del proveedor elegido antes de fijarlo, y revisar que el plan gratuito **permita uso comercial** — algunos lo restringen a proyectos personales o sin ánimo de lucro.

**Recomendación:** servidor público de OSM en desarrollo local (cero fricción), proveedor con API key desde el primer despliegue a un dominio real.

## Ruta de creación

1. Elegir proveedor y crear cuenta (MapTiler: https://cloud.maptiler.com · Stadia: https://client.stadiamaps.com · Geoapify: https://myprojects.geoapify.com).
2. Crear un proyecto/API key.
3. **Restringir la key por dominio** (HTTP referrer): `midominio.com`, `www.midominio.com` y el dominio de staging. La key viaja en el bundle del navegador — la restricción por dominio es la única protección real.
4. Copiar la URL de tiles del estilo elegido.
5. Guardarla en `.env.local` del frontend.

## Contrato / plan recomendado

- Empezar en el free tier. Al tráfico inicial del proyecto (fase 100–1,000 usuarios de la sección 11) no se agota.
- Vigilar el consumo en el panel del proveedor durante el primer mes en producción para tener una cifra real de tiles/mes antes de decidir un plan pago.
- Si el free tier queda corto: comparar precios entre los proveedores listados antes de subir de plan. El costo de cambiar es una línea de código, así que no hay lock-in que justifique aceptar un mal precio.

## Configuración

### Variables de entorno (`.env.local` de Next.js)
```
NEXT_PUBLIC_MAP_TILE_URL=https://api.maptiler.com/maps/streets-v2/{z}/{x}/{y}.png?key=<API_KEY>
NEXT_PUBLIC_MAP_ATTRIBUTION=&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://www.maptiler.com/">MapTiler</a>
```

⚠️ La atribución **no es opcional**: los datos son de OpenStreetMap (licencia ODbL) y los proveedores exigen además su propio crédito. Quitarla incumple la licencia y los términos del servicio.

### Componente
```tsx
'use client';
import { MapContainer, TileLayer } from 'react-leaflet';

<MapContainer center={[20.2114, -87.4654]} zoom={13} style={{ borderRadius: '1rem' }}>
  <TileLayer
    url={process.env.NEXT_PUBLIC_MAP_TILE_URL!}
    attribution={process.env.NEXT_PUBLIC_MAP_ATTRIBUTION!}
  />
  {/* pines */}
</MapContainer>
```

Es el único cambio respecto al prototipo: sustituir la URL de tiles de OSM por la del proveedor. El resto de `PropertyMap.tsx` y `SingleLocationMap.tsx` queda igual.

### Next.js — import dinámico obligatorio
```tsx
import dynamic from 'next/dynamic';

const PropertyMap = dynamic(() => import('@/components/map/PropertyMap'), {
  ssr: false,
  loading: () => <div className="h-full w-full animate-pulse rounded-2xl bg-sand-100" />,
});
```

Leaflet accede a `window` al importarse; sin `ssr: false` rompe el render de servidor.

Referenciado desde: `../arquitectura/`, secciones **17.9** y 4.
