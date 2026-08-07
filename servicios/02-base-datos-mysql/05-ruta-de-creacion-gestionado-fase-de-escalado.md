# Ruta de creación (gestionado, fase de escalado)

1. Crear cuenta en el proveedor elegido (ej. DigitalOcean: https://cloud.digitalocean.com).
2. Databases → Create Database Cluster → MySQL 8.
3. Elegir región cercana al VPS de la app (para minimizar latencia).
4. Configurar "Trusted Sources" para que solo el VPS del backend pueda conectarse (firewall a nivel de base de datos).
5. Copiar cadena de conexión y credenciales.

