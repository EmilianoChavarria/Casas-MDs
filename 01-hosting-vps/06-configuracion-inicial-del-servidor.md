# Configuración inicial del servidor

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

