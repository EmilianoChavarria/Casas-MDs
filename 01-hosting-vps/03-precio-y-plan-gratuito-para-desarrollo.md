# 💰 Precio y plan gratuito para desarrollo

Un VPS **no es un servicio que tenga plan gratuito permanente** en ningún proveedor serio (Hetzner, DigitalOcean, Hostinger) — siempre se paga desde el primer minuto.

| Opción | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Docker Compose local** (tu propia máquina) | $0 | ✅ Recomendado — así se desarrolla día a día, sin gastar nada; réplica exacta del stack (app, Nginx, MySQL, Redis) que luego corre en el VPS |
| **Hetzner Cloud (servidor real de prueba)** | Facturación por hora, sin permanencia (el plan más pequeño ronda unos pocos €/mes si se deja prendido, o céntimos si se crea y se destruye en minutos) | ✅ Útil para probar el deploy real (Docker, Nginx, SSL) antes de producción — crear el servidor, probar, y destruirlo cuando termines para no seguir pagando |
| **DigitalOcean** | Ofrece ~$200 USD en créditos por 60 días a cuentas nuevas | ✅ Alternativa si quieres probar sin poner tarjeta "en serio" desde el día uno |

**Nota (ago-2026):** Hetzner ajustó precios el 15-jun-2026 y el plan CX32 mencionado abajo quedó descontinuado para nuevas contrataciones (reemplazado por specs equivalentes en la línea CX/CPX vigente). Verificar el precio y nombre de plan actual directamente en https://www.hetzner.com/cloud antes de contratar, ya que puede haber cambiado desde que se escribió este documento.

**Recomendación:** desarrollar 100% local con Docker Compose (gratis) y usar el VPS real solo para staging/producción, no para desarrollo diario.

