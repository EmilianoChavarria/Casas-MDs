# Configuración


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
