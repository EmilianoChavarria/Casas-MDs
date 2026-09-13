# Servicio: Almacenamiento de Imágenes — Cloudflare R2

## ¿Para qué se usa?

Almacenar las imágenes de las propiedades (`property_images`) y los backups de MySQL. Se sirven a través de CDN hacia el frontend.


## Justificación

- **R2 sobre S3:** compatible con la API de S3 (Laravel lo soporta nativamente con el driver `s3`), pero **sin costos de egress** (transferencia de salida gratuita), lo cual es clave porque las imágenes de propiedades generan mucho tráfico de lectura.
- Integración directa con Cloudflare CDN (ver `09-cloudflare-dns-cdn.md`) para servir las imágenes con caché en el borde, cerca del usuario final.

**Cuándo usar S3 en su lugar:** si el equipo ya opera fuertemente en AWS (ej. usa Amazon SES para correo, Lambda, etc.) puede convenir mantener todo en un solo proveedor por simplicidad operativa. Para este proyecto, R2 es la opción con mejor costo-beneficio.


## 💰 Precio y plan gratuito para desarrollo

Cloudflare R2 tiene un **free tier permanente** (no es una prueba por tiempo limitado):

| Recurso | Incluido gratis cada mes | ¿Sirve para desarrollo? |
|---|---|---|
| Storage (Standard) | 10 GB-mes | ✅ Sí, de sobra para un catálogo de propiedades en desarrollo |
| Class A operations (escrituras: subir/listar) | 1,000,000 /mes | ✅ Sí |
| Class B operations (lecturas) | 10,000,000 /mes | ✅ Sí |
| Egress (transferencia de salida) | Siempre $0, incluso fuera del free tier | ✅ Sí |

**Recomendación:** usar la misma cuenta/bucket de R2 para desarrollo (con un bucket separado, ej. `rentas-casas-media-dev`) sin preocuparse por costo — es muy difícil salir del free tier en fase de desarrollo.


## Ruta de creación

1. Crear/usar cuenta de Cloudflare (la misma que se usará para DNS, ver `09-cloudflare-dns-cdn.md`).
2. Panel → **R2 Object Storage** → Create bucket (ej. `rentas-casas-media`).
3. Ir a **Manage R2 API Tokens** → Create API Token con permisos "Object Read & Write" limitado a ese bucket.
4. Copiar `Access Key ID`, `Secret Access Key` y el `Account ID` (forman el endpoint).
5. (Opcional pero recomendado) Conectar el bucket a un dominio propio vía **R2 → bucket → Settings → Public Access → Connect Domain** (ej. `cdn.midominio.com`) para servir imágenes con tu propio dominio en vez de la URL genérica de R2.


## Contrato / plan recomendado

- **Free tier:** 10 GB storage + 1M Class A ops + 10M Class B ops por mes, gratis de forma permanente.
- **Storage (excedente):** $0.015 USD/GB-mes.
- **Class A operations** (escrituras, ej. subir imagen): $4.50 USD por millón (excedente sobre el 1M gratis).
- **Class B operations** (lecturas): $0.36 USD por millón (excedente sobre el 10M gratis).
- **Egress: $0 siempre**, dentro o fuera del free tier (esta es la diferencia clave frente a S3, que cobra ~$0.09/GB de salida).
- Para un catálogo de propiedades con cientos de imágenes y tráfico moderado, el costo mensual esperado en producción es de unos pocos dólares (o $0 si no se supera el free tier).


## Configuración

> ✅ **Implementado.** Lo que sigue es lo que hay en el código, no una
> propuesta. Rama `feat/almacenamiento-r2` del backend.

### Qué se guarda en la base de datos: la ruta, no la URL

`property_images.path` guarda `properties/12/01j8x….webp` y
`zones.image_path` guarda `zones/01j8x….webp`. La URL pública se arma al
leer, en `App\Services\Media\ImageStorage`.

⚠️ Esto **cambia respecto al borrador anterior de este documento**, que
guardaba `Storage::url($path)` en la columna. Guardar la URL completa es
cómodo un día y caro el resto: conectar un dominio propio al bucket,
cambiar de proveedor o pasar de pruebas a producción invalidaría todas
las filas a la vez, y arreglarlo sería un reemplazo de cadenas sobre
datos reales. Con la ruta guardada, ese cambio es una variable de
entorno.

`ImageStorage::url()` acepta también URLs absolutas y las devuelve tal
cual, porque algunas fotos de zona son de banco de imágenes y nunca
pasaron por el bucket. Se reconocen por el `http` de delante, sin
necesidad de una columna que marque el origen.

### Variables de entorno (`.env` de Laravel)
```
FILESYSTEM_DISK=r2

R2_ACCESS_KEY_ID=<access key>
R2_SECRET_ACCESS_KEY=<secret key>
R2_BUCKET=casa-caribe
R2_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com
R2_URL=https://fotos.midominio.com   # dominio publico conectado al bucket
```

`R2_ENDPOINT` es el de la **API** (lleva credenciales). `R2_URL` es el
dominio desde el que el navegador **descarga** las fotos. No son el
mismo, y confundirlos deja todas las fotos rotas con un error que no
explica por qué.

En desarrollo no hace falta nada de esto: con `FILESYSTEM_DISK=public`
las fotos van a `storage/app/public` y las sirve el propio Laravel. Una
sola vez:

```
php artisan storage:link
```

### El frontend tiene que reconocer el dominio

`next.config.ts` deriva los dominios de imagen permitidos de
`NEXT_PUBLIC_API_URL`. Con las fotos en otro dominio hay que añadirlo:

```
NEXT_PUBLIC_IMAGE_HOST=fotos.midominio.com
```

Sin esto Next rechaza las imágenes con un error de host no configurado.

### `config/filesystems.php`

Tres particularidades de R2 que hay que respetar o las subidas fallan de
formas poco claras:

1. **R2 no admite ACLs.** Por eso el disco **no** declara `'visibility'`:
   en cuanto se declara, Flysystem manda una cabecera de ACL en cada
   subida y R2 responde con un error que no menciona la palabra ACL por
   ningún lado.
2. **La región siempre es `auto`.** R2 no tiene regiones al estilo de S3.
3. **El endpoint es por cuenta, no por bucket** —
   `https://<cuenta>.r2.cloudflarestorage.com/<bucket>`—, así que se usa
   estilo de ruta. Queda en `R2_USE_PATH_STYLE` por si una cuenta
   concreta responde al estilo de subdominio.

Además `'throw' => true`: si una subida falla hay que enterarse en el
momento. En `false`, Laravel devuelve `false` y la casa se queda con una
foto que no existe.

### Normalización al subir

Toda foto se reencodea a **WebP**, con el lado mayor limitado a 2560 px,
antes de guardarse. No es cosmética:

- **Descarta los metadatos**, y con ellos las coordenadas GPS que los
  teléfonos incrustan en cada foto. Publicar el EXIF de la foto de una
  recámara es publicar la ubicación exacta de la casa con precisión de
  metros — incluidas las casas que todavía son borrador.
- **Aplica la orientación** que venía en esos metadatos antes de
  tirarlos. Sin esto las fotos tomadas en vertical se guardan tumbadas:
  el navegador ya no tiene el EXIF para corregirlas.
- **Unifica formato y tamaño.** Entran JPEG, PNG y WebP de cámaras
  distintas; sale siempre lo mismo.

### Ciclo de vida

Borrar una foto borra el objeto, y reemplazar la foto de una zona borra
la anterior. Sin eso los archivos se acumulan para siempre: nadie los ve,
nadie sabe que están, y se pagan todos los meses.

### Lo que deliberadamente NO se hizo

**No se generan miniaturas.** El sitio público usa `next/image`, que ya
redimensiona y sirve en formatos modernos. Generar tres tamaños por foto
en la subida sería multiplicar por cuatro el almacenamiento y añadir una
cola de trabajos para hacer lo que ya se hace.

**La subida pasa por Laravel**, no del navegador al bucket con una URL
firmada. Cuesta un salto de red más y a cambio la validación del
contenido —que un `.php` renombrado a `.jpg` no entre— ocurre del lado
del servidor. Con una URL firmada, el navegador puede subir dentro de
ella lo que quiera. A este volumen —fotos de casas, subidas por el
personal— el salto extra no se nota.

**No hay CORS en el bucket.** No hace falta: quien sube es Laravel y
quien descarga es una etiqueta `<img>`. Haría falta el día que se pase a
subidas directas desde el navegador.

⚠️ **R2 exige tarjeta registrada para activarse**, incluso para usar solo
el tramo gratuito, y Cloudflare hace preautorizaciones temporales para
comprobarla. No se cobra nada mientras el consumo esté dentro del tramo.

Referenciado desde: `../arquitectura/`, secciones 1, 3 y 8.
