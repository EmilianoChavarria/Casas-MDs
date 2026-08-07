# Ruta de creación (alternativa: Amazon SES)

1. Crear cuenta AWS (si no existe ya, por el uso de S3/R2 podría no ser necesaria si se usa R2 en vez de S3).
2. Consola SES → **Verified identities → Create identity** → verificar el dominio con los registros DNS que indique.
3. Solicitar salida del modo Sandbox: **Account dashboard → Request production access** (formulario explicando el caso de uso; aprobación de AWS en 24–48h).
4. Crear credenciales IAM específicas con permiso `ses:SendEmail` únicamente (principio de mínimo privilegio).

