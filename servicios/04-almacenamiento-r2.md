# Servicio: Almacenamiento de Imágenes — Cloudflare R2

## ¿Para qué se usa?
Almacenar las imágenes de las propiedades (`property_images`) y los backups de MySQL. Se sirven a través de CDN hacia el frontend.

## Justificación
- **R2 sobre S3:** compatible con la API de S3 (Laravel lo soporta nativamente con el driver `s3`), pero **sin costos de egress** (transferencia de salida gratuita), lo cual es clave porque las imágenes de propiedades generan mucho tráfico de lectura.
- Integración directa con Cloudflare CDN (ver `09-cloudflare-dns-cdn.md`) para servir las imágenes con caché en el borde, cerca del usuario final.

**Cuándo usar S3 en su lugar:** si el equipo ya opera fuertemente en AWS (ej. usa Amazon SES para correo, Lambda, etc.) puede convenir mantener todo en un solo proveedor por simplicidad operativa. Para este proyecto, R2 es la opción con mejor costo-beneficio.

## Ruta de creación
1. Crear/usar cuenta de Cloudflare (la misma que se usará para DNS, ver `09-cloudflare-dns-cdn.md`).
2. Panel → **R2 Object Storage** → Create bucket (ej. `rentas-casas-media`).
3. Ir a **Manage R2 API Tokens** → Create API Token con permisos "Object Read & Write" limitado a ese bucket.
4. Copiar `Access Key ID`, `Secret Access Key` y el `Account ID` (forman el endpoint).
5. (Opcional pero recomendado) Conectar el bucket a un dominio propio vía **R2 → bucket → Settings → Public Access → Connect Domain** (ej. `cdn.midominio.com`) para servir imágenes con tu propio dominio en vez de la URL genérica de R2.

## Contrato / plan recomendado
- **Storage:** $0.015 USD/GB-mes.
- **Class A operations** (escrituras, ej. subir imagen): $4.50 USD por millón.
- **Class B operations** (lecturas): $0.36 USD por millón.
- **Egress: $0** (esta es la diferencia clave frente a S3, que cobra ~$0.09/GB de salida).
- Para un catálogo de propiedades con cientos de imágenes y tráfico moderado, el costo mensual esperado es de unos pocos dólares.

## Configuración

### Variables de entorno (`.env` de Laravel)
```
FILESYSTEM_DISK=r2

R2_ACCESS_KEY_ID=<access key>
R2_SECRET_ACCESS_KEY=<secret key>
R2_BUCKET=rentas-casas-media
R2_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com
R2_URL=https://cdn.midominio.com   # dominio público conectado al bucket
```

### `config/filesystems.php`
```php
'disks' => [
    'r2' => [
        'driver' => 's3',
        'key' => env('R2_ACCESS_KEY_ID'),
        'secret' => env('R2_SECRET_ACCESS_KEY'),
        'region' => 'auto',
        'bucket' => env('R2_BUCKET'),
        'endpoint' => env('R2_ENDPOINT'),
        'url' => env('R2_URL'),
        'use_path_style_endpoint' => false,
    ],
],
```

### Flujo de subida (Service, resumen)
```php
public function storePropertyImage(Property $property, UploadedFile $file): PropertyImage
{
    // Validar y re-procesar antes de subir (ver 07-seguridad en doc principal)
    $optimized = Image::make($file)->encode('webp', 80);

    $path = "properties/{$property->id}/" . Str::uuid() . '.webp';
    Storage::disk('r2')->put($path, $optimized);

    return $property->images()->create([
        'url' => Storage::disk('r2')->url($path),
    ]);
}
```

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 1, 3 y 8.
