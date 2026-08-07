# Servicio: Hosting / VPS — Hetzner Cloud

## ¿Para qué se usa?
Servidor donde corre el backend Laravel (Docker: app, Nginx, MySQL, Redis, queue workers). Es la infraestructura base de producción.

## Justificación
- **Hetzner** ofrece la mejor relación costo/rendimiento frente a DigitalOcean y Hostinger para specs equivalentes (CPU/RAM/SSD).
- Datacenters en EU/US (Ashburn, VA es el más cercano/rápido para México). Si la latencia a México es crítica, evaluar también **DigitalOcean** (datacenter en NYC/SFO) como alternativa con más cercanía geográfica.
- Soporta snapshots, backups automáticos y firewalls a nivel de proveedor (capa extra antes de que el tráfico llegue al servidor).

**Alternativas y cuándo usarlas:**
| Proveedor | Cuándo elegirlo |
|---|---|
| Hetzner | Mejor costo/rendimiento, si la latencia a MX no es crítica |
| DigitalOcean | Si priorizas datacenter más cercano a México (NYC/SFO) o soporte en español |
| Hostinger VPS | Solo si ya tienes cuenta y quieres simplicidad extrema; menos flexible para Docker avanzado |

## Ruta de creación
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

## Contrato / plan recomendado
- **CX32** (~€13.60/mes aprox.): 4 vCPU, 8 GB RAM, 80 GB SSD — suficiente para MySQL + Laravel + Redis en fase inicial (100–1,000 usuarios).
- Backups automáticos diarios: +20% del precio base.
- Sin permanencia forzosa, facturación mensual, se puede escalar (resize) sin perder datos.
- Revisar el SLA de disponibilidad de Hetzner Cloud (~99.9%) en su documentación oficial antes de firmar para producción crítica.

## Configuración inicial del servidor
```bash
# Conexión
ssh root@<IP_SERVIDOR>

# Actualizar sistema
apt update && apt upgrade -y

# Crear usuario no-root con sudo
adduser deploy
usermod -aG sudo deploy

# Instalar Docker y Docker Compose
curl -fsSL https://get.docker.com | sh
usermod -aG docker deploy

# Firewall a nivel de SO (UFW) como capa adicional al firewall de Hetzner
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable

# Fail2ban para proteger SSH de fuerza bruta
apt install fail2ban -y
```

## Variables/credenciales a resguardar en GitHub Secrets
```
VPS_HOST=<ip o dominio>
VPS_USER=deploy
VPS_SSH_KEY=<llave privada del deploy>
```

## Costos aproximados (mensual)
- CX32 + backups: ~€16–17/mes (~$18–19 USD)
- Escalar a CX42 (8 vCPU/16GB) cuando el tráfico lo requiera: ~€27/mes

Referenciado desde: `arquitectura-plataforma-renta.md`, sección 8 y 10.
