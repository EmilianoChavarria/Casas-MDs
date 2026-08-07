# Ruta de creación

1. Crear/usar cuenta de Cloudflare (la misma que se usará para DNS, ver `../09-cloudflare-dns-cdn/`).
2. Panel → **R2 Object Storage** → Create bucket (ej. `rentas-casas-media`).
3. Ir a **Manage R2 API Tokens** → Create API Token con permisos "Object Read & Write" limitado a ese bucket.
4. Copiar `Access Key ID`, `Secret Access Key` y el `Account ID` (forman el endpoint).
5. (Opcional pero recomendado) Conectar el bucket a un dominio propio vía **R2 → bucket → Settings → Public Access → Connect Domain** (ej. `cdn.midominio.com`) para servir imágenes con tu propio dominio en vez de la URL genérica de R2.

