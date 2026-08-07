# Ruta de creación

1. Crear cuenta en https://www.hetzner.com/cloud
2. Verificar identidad (tarjeta o PayPal).
3. Crear proyecto nuevo (ej. `renta-casas-prod`).
4. Crear servidor:
   - Ubicación: Ashburn, VA (US East) — menor latencia a México desde la costa este.
   - Imagen: Ubuntu 24.04 LTS.
   - Tipo: CX32 (4 vCPU / 8GB RAM) para arranque; escalar a CX42 si crece tráfico.
   - Añadir tu llave SSH pública (no uses contraseña).
   - Habilitar backups automáticos (+20% del costo del servidor, vale la pena).
5. Crear un **Firewall** de Hetzner: permitir solo 22 (SSH, idealmente solo desde tu IP), 80, 443.
6. Anotar la IP pública asignada.

