# Servicio: Hosting / VPS — DigitalOcean

## ¿Para qué se usa?

Servidor donde corre el backend Laravel (Docker: app, Nginx, MySQL, Redis, queue workers). Es la infraestructura base de producción.


## Justificación

- **Cobertura geográfica:** DigitalOcean tiene datacenters en Nueva York (NYC1/NYC3), San Francisco (SFO3) y Toronto (TOR1) —los más cercanos/rápidos para México— y además en Europa (Ámsterdam AMS3, Fráncfort FRA1, Londres LON1) si en algún momento se necesita presencia o residencia de datos en la UE. Es el proveedor con más regiones útiles para este proyecto.
- **Precio:** tras el ajuste de precios de Hetzner del 15-jun-2026, la ventaja de costo/rendimiento que Hetzner tenía sobre DigitalOcean prácticamente desapareció para specs equivalentes. Con precios similares, gana el proveedor con mejor cobertura y ecosistema.
- **Ecosistema integrado:** Cloud Firewalls, Monitoring y alertas sin costo extra; snapshots y backups automáticos desde el mismo panel; y ruta natural de migración a **Managed MySQL** de DigitalOcean cuando el proyecto llegue a la fase de escalado (ver `02-base-datos-mysql.md`) sin cambiar de proveedor ni salir de la red privada (VPC).
- **Crédito de arranque:** cuentas nuevas reciben ~$200 USD en créditos por 60 días, lo que cubre el staging y las primeras pruebas de deploy real sin gasto.

**Alternativas y cuándo usarlas:**
| Proveedor | Cuándo elegirlo |
|---|---|
| DigitalOcean | **Recomendado.** Mejor cobertura de regiones (US + EU), ecosistema integrado (VPC, firewall, DB gestionada), precio hoy equiparable a la competencia |
| Hetzner | Si al comparar en el momento de contratar sigue siendo notablemente más barato para las mismas specs y la latencia/ubicación no es problema |
| Hostinger VPS | Solo si ya tienes cuenta y quieres simplicidad extrema; menos flexible para Docker avanzado |

> **Antes de contratar:** comparar el precio vigente de las specs objetivo (4 vCPU / 8 GB) en https://www.digitalocean.com/pricing/droplets y https://www.hetzner.com/cloud. Ambos proveedores ajustaron precios en 2026; la decisión está tomada sobre paridad de precio, así que vale la pena confirmarla con números del día.


## 💰 Precio y plan gratuito para desarrollo

Un VPS **no es un servicio que tenga plan gratuito permanente** en ningún proveedor serio (DigitalOcean, Hetzner, Hostinger) — siempre se paga desde el primer minuto.

| Opción | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Docker Compose local** (tu propia máquina) | $0 | ✅ Recomendado — así se desarrolla día a día, sin gastar nada; réplica exacta del stack (app, Nginx, MySQL, Redis) que luego corre en el VPS |
| **DigitalOcean con crédito de bienvenida** | ~$200 USD gratis por 60 días en cuentas nuevas | ✅ Ideal para probar el deploy real (Docker, Nginx, SSL) y montar staging sin costo durante los primeros dos meses |
| **DigitalOcean pagando (droplet de prueba)** | Facturación por hora, sin permanencia | ✅ Crear el droplet, probar el deploy, y **destruirlo** al terminar para dejar de pagar (un droplet apagado se sigue cobrando; hay que destruirlo) |

**Recomendación:** desarrollar 100% local con Docker Compose (gratis) y usar el VPS real solo para staging/producción, no para desarrollo diario.


## Ruta de creación

1. Crear cuenta en https://cloud.digitalocean.com — verificar identidad (tarjeta o PayPal) y confirmar que se aplicó el crédito de bienvenida.
2. Crear un **Project** nuevo (ej. `renta-casas-prod`) para separar recursos de prod y staging.
3. Create → **Droplets**:
   - Región: **NYC3** (o SFO3 si el tráfico viene más del occidente de México). Elegir una sola región y mantener ahí todos los recursos.
   - Imagen: Ubuntu 24.04 LTS.
   - Tipo: **Basic (Shared CPU) — 4 vCPU / 8 GB RAM / 160 GB SSD** para arranque; escalar (resize) a 8 vCPU / 16 GB si crece el tráfico.
   - Autenticación: **llave SSH pública** (no uses contraseña).
   - Habilitar **Backups** automáticos (+20% del costo del droplet, vale la pena).
   - Habilitar **Monitoring** (gratis) para métricas de CPU/RAM/disco y alertas.
4. Crear un **Cloud Firewall** (gratis) y asignarlo al droplet: permitir entrante solo 22 (SSH, idealmente solo desde tu IP), 80 y 443.
5. Anotar la IP pública asignada (y considerar reservarla como **Reserved IP** para poder cambiar de droplet sin tocar el DNS).


## Contrato / plan recomendado

- **Basic Droplet 4 vCPU / 8 GB / 160 GB SSD**: suficiente para MySQL + Laravel + Redis en fase inicial (100–1,000 usuarios). Precio de lista de referencia ~$48 USD/mes — **verificar el vigente en el panel antes de contratar**.
- Opción de arranque más económica si el presupuesto aprieta: 2 vCPU / 4 GB (~$24 USD/mes), con resize en caliente cuando MySQL empiece a apretar.
- Backups automáticos: +20% del precio base del droplet.
- Transferencia de datos incluida generosa (varios TB/mes en estos planes); el excedente se cobra por GB — irrelevante para este proyecto porque las imágenes salen por Cloudflare R2, no por el droplet.
- Sin permanencia forzosa, facturación por hora con tope mensual, resize sin perder datos (el resize de disco es irreversible: se puede subir, no bajar).
- SLA de disponibilidad de Droplets: 99.99% según el acuerdo de nivel de servicio de DigitalOcean — revisar el texto vigente antes de comprometerlo con el cliente.


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

# Firewall a nivel de SO (UFW) como capa adicional al Cloud Firewall de DigitalOcean
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

- Droplet 4 vCPU/8GB + backups automáticos (+20%): estimar ~$58 USD/mes tomando el precio de lista de referencia — **verificar el precio actual en https://www.digitalocean.com/pricing/droplets**, que es el número que manda.
- Arranque económico 2 vCPU/4GB + backups: ~$29 USD/mes.
- Escalar a 8 vCPU/16GB cuando el tráfico lo requiera: aproximadamente el doble del plan de 4/8.
- $0/mes durante los primeros 60 días si se aprovecha el crédito de bienvenida, y $0/mes durante el desarrollo si se usa Docker Compose local (ver sección de plan gratuito arriba).

Referenciado desde: `../arquitectura/`, sección 8 y 10.
