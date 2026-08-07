# Configuración adicional


### Page Rules / Cache Rules para imágenes (fase inicial)
```
URL: cdn.midominio.com/*
Configuración: Cache Level = Cache Everything, Edge TTL = 1 mes
```

### Reglas de Firewall recomendadas
- Rate limiting en `api.midominio.com/api/v1/auth/login` (mitiga fuerza bruta a nivel de borde, complementa el `throttle` de Laravel — ver sección 7 del documento principal).
- Bloqueo geográfico opcional si el negocio es 100% doméstico (revisar antes de activar, ya que puede bloquear turistas legítimos reservando desde el extranjero).

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 1, 7, 8 y 10.
