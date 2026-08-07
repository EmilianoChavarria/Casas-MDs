# ¿Para qué se usa?

1. **Cache** de catálogos poco cambiantes (propiedades, amenidades, temporadas) para reducir carga a MySQL.
2. **Colas (queues)** de Laravel: envío de correos, procesamiento de webhooks de pago, generación de reportes — todo lo que no debe bloquear el request HTTP.
3. Backend de **sesiones** (si se usa Sanctum SPA) para que el estado no dependa del disco local (requisito para escalado horizontal, sección 11).

