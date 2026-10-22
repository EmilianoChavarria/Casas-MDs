# Servicio: Hosting / VPS — DigitalOcean

## ¿Para qué se usa?

Servidor donde corre el backend Laravel (Nginx, PHP-FPM, MySQL, worker de colas, scheduler) directo sobre el sistema operativo — **sin Docker**. Es la infraestructura base de producción.

> ⚠️ **Decisión (13-oct-2026): sin Docker, instalación nativa.** La versión anterior de este documento planteaba Docker Compose. Se cambió porque en este proyecto nunca se ha usado Docker ni en desarrollo local (todo corre nativo), hay prisa por lanzar, y a esta escala —un servidor, un operador— la ganancia de Docker no compensa la curva de aprendizaje ni el consumo extra de memoria en un droplet de 4 GB. Sigue siendo una decisión reversible más adelante: es un cambio de infraestructura, no de código.


## Justificación

- **Cobertura geográfica:** DigitalOcean tiene datacenters en Nueva York (NYC1/2/3), San Francisco (SFO3) y Toronto (TOR1) —los más cercanos/rápidos para México— y además en Europa si en algún momento se necesita presencia ahí. Es el proveedor con más regiones útiles para este proyecto.
- **Ecosistema integrado:** Cloud Firewalls, Monitoring y alertas sin costo extra; snapshots y backups automáticos desde el mismo panel.
- **Lo que de verdad pasó al contratarlo (13-oct-2026):** NYC1 y NYC2 tenían los planes Basic más baratos bloqueados por falta de capacidad en el datacenter (no es un problema de cuenta ni de método de pago). **Toronto (TOR1) sí los tenía disponibles** — se usó esa región en su lugar. Sigue siendo Norteamérica, buena latencia hacia México.


## 💰 Precio y plan gratuito para desarrollo

Un VPS **no es un servicio que tenga plan gratuito permanente** — siempre se paga desde el primer minuto.

**Recomendación:** desarrollar 100% local (como ya se hace) y usar el VPS real solo para producción.


## Droplet contratado (13-oct-2026)

| | |
|---|---|
| Nombre | `rivierabnb-backend` |
| Región | Toronto — TOR1 |
| Imagen | Ubuntu 24.04 (LTS) x64 |
| Plan | Basic, CPU Regular, **2 vCPU / 4 GB RAM / 80 GB SSD** — **$24 USD/mes** |
| Backups automáticos | ❌ No activados (se puede activar después; +20% del costo) |
| Managed Database | ❌ No — MySQL corre en el mismo droplet |
| Monitoring (gratis) | ✅ Activado |
| IPv6 | ❌ No activado |
| Autenticación | Llave SSH (`aldrick-pc`), sin contraseña |
| IP pública | `201.79.51.112` |
| IP privada | `10.118.0.2` |

Se eligió el plan de $24 en vez del de $48 (4 vCPU/8 GB) que sugería la versión anterior de este documento: para un sitio que todavía no tiene tráfico real, es gasto que no se estaba usando. El droplet se puede redimensionar hacia arriba después, sin perder datos, cuando haga falta.

**Estado (14-oct-2026): backend y frontend en producción, funcionando.** `https://api.rivierabnb.com` responde con HTTPS, base de datos migrada y con catálogos sembrados, worker de colas y scheduler corriendo como servicios del sistema, usuario administrador creado. Frontend en `https://rivierabnb.com` (Vercel, ver `15-hosting-frontend-vercel.md`), conectado al backend real. Google Maps y Google OAuth con proyecto de Google Cloud nuevo, configurados y probados (ver `08-google-maps.md` y `14-auth-google-oauth.md`).

**Pendiente:**
- Mergear los PRs de documentación (backend/VPS y frontend).
- Mercado Pago: `MERCADOPAGO_WEBHOOK_SECRET` sigue vacío (opcional — sin ella el webhook se procesa igual, solo sin verificar la firma de origen).
- Stripe: sigue en credenciales de prueba, pendiente del alta de la empresa del cliente.


## Bitácora de configuración

Cada comando, en el orden real en que se corrió, con qué hace y por qué. Todo se ejecuta conectado por SSH como `root`, salvo que se indique otra cosa.

### Llave SSH (en la máquina local, antes de crear el droplet)

```bash
ssh-keygen
```
Genera el par de llaves (`id_ed25519` / `id_ed25519.pub`) que después se sube a DigitalOcean al crear el droplet, para entrar sin contraseña. Sin contraseña en la llave: es la máquina propia del desarrollador, no un servidor compartido.

### Conexión inicial

```bash
ssh root@201.79.51.112
```
Primera conexión: pide confirmar la huella digital del servidor (`yes`). Entra directo porque la llave pública ya quedó asociada al droplet desde su creación.

### Actualizar el sistema

```bash
apt update && apt upgrade -y
```
`update` refresca la lista de paquetes disponibles; `upgrade` instala las versiones más nuevas de lo que ya viene en Ubuntu. Un droplet recién creado casi siempre tiene paquetes desactualizados — hacerlo primero evita instalar software nuevo sobre una base vieja.

⚠️ Si avisa "Pending kernel upgrade", hay que reiniciar (ver abajo) antes de seguir — si no, el servidor sigue corriendo el kernel viejo hasta el próximo reinicio, que podría pasar en un mal momento (ej. en medio de un despliegue).

### Instalar el stack (Nginx, PHP, MySQL)

```bash
apt install -y nginx mysql-server php8.3-fpm php8.3-cli php8.3-mysql php8.3-mbstring php8.3-xml php8.3-curl php8.3-zip php8.3-bcmath php8.3-gd php8.3-exif php8.3-intl php8.3-redis unzip git curl software-properties-common
```

| Paquete | Para qué |
|---|---|
| `nginx` | Servidor web: recibe las peticiones HTTP/HTTPS y se las pasa a PHP-FPM |
| `mysql-server` | Base de datos |
| `php8.3-fpm` | El intérprete de PHP corriendo como servicio, el que de verdad ejecuta el Laravel |
| `php8.3-cli` | PHP por línea de comandos — para `artisan`, `composer`, el worker de colas |
| `php8.3-mysql` | Conexión de PHP a MySQL |
| `php8.3-mbstring`, `php8.3-xml`, `php8.3-curl`, `php8.3-bcmath`, `php8.3-intl` | Extensiones que Laravel y sus dependencias dan por hechas |
| `php8.3-zip`, `php8.3-gd`, `php8.3-exif` | Las tres que ya habían dado problemas antes en desarrollo local (procesar imágenes y archivos comprimidos) — se instalan desde el principio para no repetir ese error |
| `php8.3-redis` | Por si se usa Redis para colas/caché más adelante; no cuesta nada tenerlo listo |
| `unzip`, `git`, `curl` | Herramientas básicas para desplegar el código |
| `software-properties-common` | Para poder agregar repositorios de paquetes adicionales si hiciera falta |

Ubuntu 24.04 ya trae PHP 8.3 en sus repositorios por defecto — no hizo falta agregar el PPA de Ondřej Surý que usan versiones más viejas de Ubuntu.

### Reiniciar para aplicar el kernel nuevo

```bash
reboot
```
Aplica la actualización de kernel detectada en el `apt upgrade`. Corta la conexión SSH — hay que esperar ~30 segundos y volver a conectarse con el mismo `ssh root@201.79.51.112`.

### Composer

```bash
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
```
Lo instala como `composer`, disponible en cualquier carpeta.

### Usuario `deploy` (sin privilegios de root)

```bash
adduser deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```
El código y los procesos de la aplicación corren como `deploy`, nunca como `root` — si algo se compromete, el daño queda limitado a lo que ese usuario puede tocar. Se le copia la misma llave SSH del operador para poder entrar como `deploy` sin generar una llave aparte para eso.

### Asegurar MySQL

```bash
mysql_secure_installation
```
Quita usuarios anónimos, desactiva el login remoto de `root`, borra la base de datos `test`. En Ubuntu, `root` de MySQL usa autenticación por socket (entra por ser `root` del sistema operativo, no por contraseña) — es el comportamiento esperado, no hace falta ponerle contraseña.

### Firewall (UFW)

```bash
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable
```
Solo deja pasar SSH, HTTP y HTTPS — todo lo demás queda bloqueado por defecto.

### Base de datos y usuario de la aplicación

```bash
mysql -e "CREATE DATABASE rivierabnb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -e "CREATE USER 'rivierabnb'@'localhost' IDENTIFIED BY '<contraseña>';"
mysql -e "GRANT ALL PRIVILEGES ON rivierabnb.* TO 'rivierabnb'@'localhost';"
mysql -e "FLUSH PRIVILEGES;"
```
Un usuario de MySQL dedicado a la aplicación, nunca `root` de MySQL para esto.

### Clonar el código (como `deploy`, con Deploy Key de solo lectura)

```bash
mkdir -p /var/www/rivierabnb-backend        # como root
chown deploy:www-data /var/www/rivierabnb-backend

su - deploy
ssh-keygen -t ed25519 -C "rivierabnb-backend-deploy" -f ~/.ssh/deploy_key -N ""
cat ~/.ssh/deploy_key.pub   # se agrega en GitHub → repo → Settings → Deploy keys (solo lectura)

cat >> ~/.ssh/config << 'EOF'
Host github.com
  IdentityFile ~/.ssh/deploy_key
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config

git clone git@github.com:EmilianoChavarria/Casas_back.git /var/www/rivierabnb-backend
```
Una Deploy Key en vez de las credenciales personales del desarrollador: de solo lectura, específica de este repo, y se puede revocar sin afectar ninguna otra cosa.

### Dependencias de la aplicación

```bash
cd /var/www/rivierabnb-backend
composer install --no-dev --optimize-autoloader
```

### `.env` de producción

Se escribió a mano con los valores reales (ver la plantilla completa en el historial de configuración — no se repite aquí por longitud). Diferencias clave frente al `.env` de desarrollo:

- `APP_ENV=production`, `APP_DEBUG=false` — nunca exponer detalles internos en un error público.
- `APP_KEY` generada de nuevo en el servidor (`php artisan key:generate`), nunca reutilizada de desarrollo.
- `SESSION_DOMAIN=.rivierabnb.com` y `SANCTUM_STATEFUL_DOMAINS` con los dominios reales — necesario para que la cookie de sesión funcione entre el frontend (Vercel) y este backend.
- `SESSION_SECURE_COOKIE=true` — exige HTTPS.
- Credenciales **reales** de Mercado Pago (cuenta verificada de Omar Carrillo Cen), `MERCADOPAGO_SANDBOX=false`, `MERCADOPAGO_WEBHOOK_BASE_URL=https://api.rivierabnb.com`.
- Ya no incluye variables de Clip — ese proveedor se quitó del código.

### Permisos, migraciones y datos base

```bash
sudo chown -R deploy:www-data /var/www/rivierabnb-backend
sudo chmod -R 775 storage bootstrap/cache

php artisan migrate --force
php artisan db:seed --force
```
`storage/` y `bootstrap/cache/` necesitan que PHP-FPM (que corre como `www-data`) pueda escribir ahí — logs, caché de vistas, sesiones. `--force` es obligatorio porque en producción Laravel pide confirmación interactiva, que se queda esperando para siempre en un script o una terminal no interactiva.

El sistema arranca sin casas ni reservas a propósito — eso se captura desde el panel.

### Primer usuario administrador

Sin comando dedicado — se crea a mano:

```bash
php artisan tinker
```
```php
$role = App\Models\Role::where('slug', 'admin')->first();
App\Models\User::create([
    'role_id' => $role->id,
    'name' => '...',
    'email' => '...',
    'password' => bcrypt('...'),
    'is_active' => true,
    'email_verified_at' => now(),
]);
```

### Nginx

```bash
sudo tee /etc/nginx/sites-available/rivierabnb-backend > /dev/null << 'EOF'
server {
    listen 80;
    server_name api.rivierabnb.com;
    root /var/www/rivierabnb-backend/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    client_max_body_size 20M;
}
EOF

sudo ln -s /etc/nginx/sites-available/rivierabnb-backend /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

### SSL (Let's Encrypt / Certbot)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d api.rivierabnb.com
```
Certbot edita la configuración de Nginx solo, agrega el certificado, redirige HTTP → HTTPS, y programa la renovación automática — no hay que hacer nada más para que el certificado se mantenga vigente.

### Worker de colas (systemd)

```bash
sudo tee /etc/systemd/system/rivierabnb-queue.service > /dev/null << 'EOF'
[Unit]
Description=RivieraBnb queue worker
After=network.target mysql.service

[Service]
User=deploy
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/rivierabnb-backend/artisan queue:work --sleep=3 --tries=3 --max-time=3600

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable rivierabnb-queue
sudo systemctl start rivierabnb-queue
```
`Restart=always`: si el proceso muere por cualquier motivo, systemd lo vuelve a levantar solo, y arranca automático si se reinicia el servidor. `--max-time=3600`: el worker se reinicia solo cada hora, para no acumular memoria indefinidamente.

### Scheduler (cron)

```bash
(crontab -l 2>/dev/null; echo "* * * * * cd /var/www/rivierabnb-backend && php artisan schedule:run >> /dev/null 2>&1") | crontab -
```
Corre cada minuto; Laravel decide internamente qué tarea programada le toca ejecutar en ese momento (expirar reservas sin pagar, recordatorios, tasas de cambio, etc.).


## Despliegue automático (16-oct-2026)

> ✅ **Implementado.** GitHub Actions despliega solo al VPS cuando el CI de `Casas_back` termina bien en `main` — ver el workflow `deploy.yml` del repo y el PR [#89](https://github.com/EmilianoChavarria/Casas_back/pull/89). Probado de punta a punta: primer despliegue automático exitoso el 16-oct-2026.

Secrets del repo `Casas_back` (GitHub → Settings → Secrets and variables → Actions):
```
DEPLOY_HOST=201.79.51.112
DEPLOY_USER=deploy
DEPLOY_SSH_KEY=<llave privada dedicada, generada solo para esto — no es la llave personal del operador ni la Deploy Key de solo lectura que clona el repo>
```

La llave pública correspondiente está en `authorized_keys` del usuario `deploy` en el servidor. El `deploy` también tiene un permiso de `sudo` acotado a un solo comando (nada más):
```
# /etc/sudoers.d/deploy-php-reload
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl reload php8.3-fpm
```
El worker de colas se reinicia sin sudo: `php artisan queue:restart` solo le avisa al proceso que termine su job actual y salga — `Restart=always` de systemd lo vuelve a levantar con el código nuevo.

### Límites de subida de archivos (16-oct-2026)

> ⚠️ **Ajuste necesario, no viene así por defecto.** Ubuntu instala PHP con `upload_max_filesize=2M`, muy por debajo de los 5MB que el código permite para fotos de casas/zonas — cualquier foto de celular normal lo superaba y fallaba con el mensaje crudo `validation.uploaded` (PHP rechaza el archivo antes de que Laravel llegue a revisarlo).

Editar **el `php.ini` de FPM, no el de CLI** (son archivos distintos):
```bash
sed -i 's/^upload_max_filesize = .*/upload_max_filesize = 10M/' /etc/php/8.3/fpm/php.ini
sed -i 's/^post_max_size = .*/post_max_size = 12M/' /etc/php/8.3/fpm/php.ini
systemctl reload php8.3-fpm
```
`post_max_size` queda arriba de `upload_max_filesize` a propósito (el POST completo incluye el archivo más el resto del formulario). Combina con el límite de 20MB que ya tiene Nginx (`client_max_body_size`, ver la sección de Nginx arriba).


## Seguridad: parches de PHP

### PHP2Shell — CVE-2026-109618 y CVE-2026-109619 (21-oct-2026)

Dos fallas críticas de PHP (CVSS 9.2) que permitían ejecutar código en el
servidor **sin iniciar sesión**. Afectaban a todo PHP desde la 5.4; los
detalles se filtraron el 20-oct y hubo ataques en curso. El front (Vercel,
Next.js) no usa PHP: solo estaba expuesto este droplet.

**Qué se hizo** (como `root`, el usuario `deploy` no tiene permisos para esto):

1. **Mitigación** recomendada por los investigadores mientras se parchaba,
   con respaldo previo de los dos `php.ini`
   (`php.ini.bak-2026-10-21`):
   ```bash
   sed -i 's/^;\?session.upload_progress.enabled.*/session.upload_progress.enabled = Off/' \
     /etc/php/8.3/fpm/php.ini /etc/php/8.3/cli/php.ini
   ```
   Laravel no usa esa función; se deja apagada.
2. **Parche** de Ubuntu (canal `noble-security`). Ubuntu no sube el número a
   8.3.36: aplica la corrección sobre su versión, que pasó de
   `8.3.6-0ubuntu0.24.04.12` a **`8.3.6-0ubuntu0.24.04.13`**.
   ```bash
   apt update
   apt install -y --only-upgrade 'php8.3*'
   systemctl restart php8.3-fpm
   ```
   `needrestart` reinició solo `rivierabnb-queue` y `rivierabnb-reverb`.
3. **Verificado:** el changelog del paquete
   (`zcat /usr/share/doc/php8.3-fpm/changelog.Debian.gz | head`) trae
   `CVE-2026-109618_109619.patch`; `php -i` muestra
   `session.upload_progress.enabled => Off`; PHP-FPM, Nginx, cola y Reverb
   `active`; `https://api.rivierabnb.com/up` → 200.

**Protección adicional:** `api.rivierabnb.com` pasa por el proxy de Cloudflare
(nube naranja), que publicó reglas de WAF contra estos ataques. Ayuda, pero no
reemplaza el parche.

### Parches automáticos

`unattended-upgrades` está **encendido**
(`/etc/apt/apt.conf.d/20auto-upgrades` con `Unattended-Upgrade "1"`): Ubuntu
instala solo los parches del canal de seguridad una vez al día. Por eso, ante
una falla urgente, **conviene aplicarlo a mano el mismo día** en vez de esperar
la corrida diaria.

Pendiente: quedaron **13 actualizaciones que no son de PHP** sin aplicar
(`apt list --upgradable`). Aplicarlas en un rato tranquilo con `apt upgrade`; si
alguna pide reiniciar el droplet, la API se cae alrededor de un minuto.

### Si vuelve a pasar

```bash
apt update && apt list --upgradable 2>/dev/null | grep -i php
apt install -y --only-upgrade 'php8.3*'
systemctl restart php8.3-fpm
zcat /usr/share/doc/php8.3-fpm/changelog.Debian.gz | head -15   # confirmar la CVE
systemctl is-active php8.3-fpm nginx rivierabnb-queue rivierabnb-reverb
curl -s -o /dev/null -w "%{http_code}\n" https://api.rivierabnb.com/up
```

Acceso como `root`: desde la PC con la llave `aldrick-pc`, o desde
DigitalOcean → droplet → **Access → Launch Droplet Console**.

## Costos aproximados (mensual)

- Droplet 2 vCPU / 4 GB: **$24 USD/mes**, sin backups automáticos.
- Si más adelante se activan los backups automáticos: +20%, ≈ $29 USD/mes.
- Transferencia de datos incluida generosa; irrelevante aquí porque las imágenes salen por Cloudflare R2, no por el droplet.

Referenciado desde: `../arquitectura/`, sección 8 y 10.
