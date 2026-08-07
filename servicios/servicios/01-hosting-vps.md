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


## 💰 Precio y plan gratuito para desarrollo

Un VPS **no es un servicio que tenga plan gratuito permanente** en ningún proveedor serio (Hetzner, DigitalOcean, Hostinger) — siempre se paga desde el primer minuto.

| Opción | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Docker Compose local** (tu propia máquina) | $0 | ✅ Recomendado — así se desarrolla día a día, sin gastar nada; réplica exacta del stack (app, Nginx, MySQL, Redis) que luego corre en el VPS |
| **Hetzner Cloud (servidor real de prueba)** | Facturación por hora, sin permanencia (el plan más pequeño ronda unos pocos €/mes si se deja prendido, o céntimos si se crea y se destruye en minutos) | ✅ Útil para probar el deploy real (Docker, Nginx, SSL) antes de producción — crear el servidor, probar, y destruirlo cuando termines para no seguir pagando |
| **DigitalOcean** | Ofrece ~$200 USD en créditos por 60 días a cuentas nuevas | ✅ Alternativa si quieres probar sin poner tarjeta "en serio" desde el día uno |

**Nota (ago-2026):** Hetzner ajustó precios el 15-jun-2026 y el plan CX32 mencionado abajo quedó descontinuado para nuevas contrataciones (reemplazado por specs equivalentes en la línea CX/CPX vigente). Verificar el precio y nombre de plan actual directamente en https://www.hetzner.com/cloud antes de contratar, ya que puede haber cambiado desde que se escribió este documento.

**Recomendación:** desarrollar 100% local con Docker Compose (gratis) y usar el VPS real solo para staging/producción, no para desarrollo diario.


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

- Servidor 4 vCPU/8GB + backups automáticos (+20%): estimar entre €15–20/mes (~$16–22 USD) según el plan vigente al momento de contratar — **verificar precio actual en el dashboard de Hetzner**, ya que el plan CX32 usado como referencia original fue descontinuado en el ajuste de precios de junio 2026.
- Escalar a 8 vCPU/16GB cuando el tráfico lo requiera: rango esperado ~€25–30/mes.
- $0/mes durante el desarrollo si se usa Docker Compose local (ver sección de plan gratuito arriba).

Referenciado desde: `../../arquitectura/`, sección 8 y 10.
