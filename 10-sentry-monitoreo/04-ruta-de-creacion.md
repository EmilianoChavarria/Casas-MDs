# Ruta de creación

1. Crear cuenta en https://sentry.io/signup
2. Crear una **Organization** (ej. `renta-casas`).
3. Crear dos **Projects** dentro de la organización:
   - `renta-casas-backend` (plataforma: Laravel)
   - `renta-casas-frontend` (plataforma: Next.js)
4. Cada proyecto genera un **DSN** (URL única) que se usa para configurar el SDK.
5. Instalar el SDK:
   ```bash
   # Backend
   composer require sentry/sentry-laravel
   php artisan sentry:publish --dsn=<DSN_BACKEND>

   # Frontend
   npx @sentry/wizard@latest -i nextjs
   ```

