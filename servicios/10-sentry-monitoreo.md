# Servicio: Monitoreo de Errores — Sentry

## ¿Para qué se usa?

Captura y alerta en tiempo real de excepciones/errores no manejados tanto en el backend (Laravel) como en el frontend (Next.js), con stack trace, contexto de request/usuario y agrupación de errores repetidos.


## Justificación

Sin esto, los errores en producción solo se detectan si un usuario los reporta o revisando logs manualmente. Sentry permite detectar y priorizar bugs antes de que se conviertan en un problema de soporte, y correlaciona errores de frontend y backend en un mismo proyecto.


## 💰 Precio y plan gratuito para desarrollo

| Plan | Costo | ¿Sirve para dev/arranque en solitario? |
|---|---|---|
| **Developer** | $0/mes, **permanente**, sin tarjeta de crédito | ✅ Sí — 5,000 errores/mes, 10,000 unidades de performance/mes, 1 usuario, retención de 30 días; cubre perfectamente el desarrollo y el arranque mientras el proyecto lo mantenga una sola persona |
| **Team** | $26 USD/mes (anual) | Necesario cuando se suma más de una persona al equipo o se supera el volumen de eventos gratis |

**Recomendación:** usar el plan Developer desde el día uno del proyecto; no hace falta pagar nada hasta que haya más de un desarrollador dando mantenimiento o el volumen de errores lo justifique.


## Ruta de creación

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


## Contrato / plan recomendado

- **Plan Developer (gratuito, permanente):** 5,000 errores/mes, 10,000 unidades de performance/mes, 1 usuario, retención 30 días — suficiente para el arranque del proyecto.
- **Plan Team ($26 USD/mes, facturación anual):** más eventos, más usuarios del equipo, alertas avanzadas — considerar cuando haya más de una persona dando mantenimiento o el volumen de errores supere el tier gratuito.


## Configuración


### Variables de entorno

**Backend (`.env`)**
```
SENTRY_LARAVEL_DSN=https://xxxx@oXXXX.ingest.sentry.io/XXXX
SENTRY_TRACES_SAMPLE_RATE=0.2
```

**Frontend (`.env.local`)**
```
NEXT_PUBLIC_SENTRY_DSN=https://xxxx@oXXXX.ingest.sentry.io/YYYY
SENTRY_AUTH_TOKEN=<token para subir source maps en el build de CI/CD>
```

### Alertas recomendadas (Sentry → Alerts → Create Alert Rule)
- Notificar por email/Slack cuando ocurra un error nuevo (no visto antes) en producción.
- Notificar si un mismo error supera 10 ocurrencias en 1 hora (indica un problema sistémico, ej. caída de un proveedor de pago).
- Configurar un canal de Slack o correo del equipo como destino de las alertas.

### Integración con el pipeline de CI/CD
Subir source maps automáticamente en el deploy del frontend para que los stack traces en Sentry muestren código legible en vez de minificado:
```yaml
# En el workflow de GitHub Actions del frontend, tras el build
- name: Upload source maps to Sentry
  run: npx sentry-cli releases files "${{ github.sha }}" upload-sourcemaps ./.next
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
```

Referenciado desde: `../arquitectura/`, secciones 7 y 10.
