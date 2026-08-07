# Configuración


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
