# ¿Para qué se usa?

1. **DNS** del dominio principal (`midominio.com`) y subdominios (`api.midominio.com`, `cdn.midominio.com`).
2. **CDN/Proxy** frente al VPS del backend y frente al bucket de R2 (imágenes) — cachea contenido estático cerca del usuario.
3. **WAF (Web Application Firewall)** y protección DDoS básica gratuita frente al backend.
4. Emisión/gestión de certificados SSL de borde (modo Full Strict).

