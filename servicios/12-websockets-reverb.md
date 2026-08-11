# Servicio: WebSockets / Tiempo Real — Laravel Reverb

## ¿Para qué se usa?

1. **Chat huésped ↔ administrador** dentro de la aplicación (sección 16 del documento de arquitectura): entrega instantánea de mensajes, indicador de "escribiendo…", acuses de lectura.
2. **Notificaciones en vivo del dashboard admin**: aviso de nueva reserva o de pago recibido sin recargar la página.
3. (Opcional, a futuro) **Calendario colaborativo**: si dos administradores editan disponibilidad a la vez, ver los cambios del otro en tiempo real.

## Justificación

Reverb es el servidor WebSocket oficial de Laravel: se instala como paquete del propio framework, se levanta con `php artisan reverb:start` y funciona directo con `broadcast()`, `Broadcast::channel()` y Laravel Echo. No hay costo por conexión ni por mensaje, y corre en el mismo VPS que ya se paga.

Habla el **protocolo Pusher**, así que el cliente es `laravel-echo` + `pusher-js` sin cambios. Si algún día conviene dejar de mantener el proceso, migrar a Pusher o Ably es cambiar variables de entorno, no reescribir el frontend.

Contra las alternativas gestionadas: un chat de soporte genera tráfico constante y conexiones de larga duración, que es exactamente lo que los servicios por consumo cobran caro. Con volumen bajo Reverb es gratis; con volumen alto sigue siendo el costo de un VPS.

## 💰 Precio y plan gratuito para desarrollo

| Opción | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Laravel Reverb (self-hosted)** | **$0** — paquete open source (MIT), corre en el VPS ya contratado | ✅ Sí, gratis e ilimitado. Recomendado |
| **Pusher Channels** | Plan Sandbox gratuito permanente: 200,000 mensajes/día, 100 conexiones concurrentes, sin tarjeta. Plan Startup desde ~$49 USD/mes (500 conexiones, 1M mensajes/día) | ✅ Sí, el sandbox alcanza de sobra para desarrollo |
| **Ably** | Free tier permanente: 6M mensajes/mes, 200 canales y 200 conexiones concurrentes. Planes pagos desde ~$29 USD/mes | ✅ Sí, free tier generoso |
| **Soketi (self-hosted)** | $0, open source | ✅ Sí, alternativa a Reverb |

**Recomendación:** Reverb self-hosted desde el día uno. No requiere cuenta externa ni tarjeta, y en desarrollo se levanta con un `docker compose up`. Mantener Pusher identificado como plan B documentado, no contratado.

## Ruta de creación

**No hay cuenta que crear** — es un paquete, no un SaaS.

1. Instalar en el backend:
   ```bash
   php artisan install:broadcasting
   # elegir "reverb" cuando lo pregunte; publica config/reverb.php y routes/channels.php
   ```
2. El comando genera automáticamente `REVERB_APP_ID`, `REVERB_APP_KEY` y `REVERB_APP_SECRET` en el `.env`. **Regenerar credenciales distintas para producción** — nunca reutilizar las de desarrollo.
3. Añadir el servicio `reverb` al `docker-compose.yml` (ver sección 16.7).
4. Para producción: apuntar un subdominio (`ws.midominio.com`) al VPS en Cloudflare, y configurar el bloque `location` de Nginx con los headers `Upgrade`/`Connection` del proxy WebSocket.
5. Registrar el proceso en Supervisor para que sobreviva a reinicios.

**Si en el futuro se migra a Pusher:** crear cuenta en https://pusher.com → Channels → nueva app → copiar `app_id`, `key`, `secret`, `cluster` → cambiar `BROADCAST_CONNECTION=pusher` y las variables. El código de aplicación no cambia.

## Contrato / plan recomendado

- **Reverb:** sin contrato ni costo recurrente propio. El "costo" es la RAM que consume el proceso en el VPS (unos ~50–100 MB en reposo) y el tiempo de mantenerlo vivo.
- Considerar migrar a un gestionado solo si se cumple alguna de estas condiciones: se necesita alta disponibilidad multi-región, las conexiones concurrentes superan lo que aguanta el VPS, o el mantenimiento del proceso está costando más que los ~$49 USD/mes de un plan pago.

## Configuración

### Variables de entorno (`.env` de Laravel)
```
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=<generado por install:broadcasting>
REVERB_APP_KEY=<generado>
REVERB_APP_SECRET=<generado>

# Cómo lo ve el navegador (público)
REVERB_HOST=ws.midominio.com
REVERB_PORT=443
REVERB_SCHEME=https

# Dónde escucha el proceso (interno)
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

# Mantener viva la conexión detrás de Cloudflare (ver "Timeouts" abajo)
REVERB_APP_ACTIVITY_TIMEOUT=30
REVERB_APP_PING_INTERVAL=45

# Escalado horizontal (solo con varias instancias, ver sección 11)
REVERB_SCALING_ENABLED=false
```

### Timeouts: mantener la conexión viva detrás de Cloudflare

Cloudflare con proxy naranja soporta WebSockets, pero **corta las conexiones inactivas a los 100 s en plan Free** (no es configurable en ese plan). Entre el navegador y Reverb hay tres intermediarios con timeout propio, y deben ordenarse de menor a mayor:

| Orden | Quién | Valor | Dónde se configura |
|---|---|---|---|
| 1 | Cliente (`pusher-js`) hace ping | **30 s** sin tráfico | `REVERB_APP_ACTIVITY_TIMEOUT` |
| 2 | Servidor Reverb pinguea inactivos | **45 s** | `REVERB_APP_PING_INTERVAL` |
| 3 | Cloudflare corta inactivas | **100 s** (fijo en Free) | — |
| 4 | Nginx corta inactivas | **3600 s** | `proxy_read_timeout` |

**No hay que programar un heartbeat.** El protocolo Pusher ya lo trae: el servidor manda `activity_timeout` en el `pusher:connection_established` del handshake, y `pusher-js` (debajo de Laravel Echo) emite `pusher:ping` solo tras ese tiempo sin tráfico. Solo se ajustan los números.

Los defaults de Reverb (`activity_timeout` 30, `ping_interval` 60) ya quedan por debajo de los 100 s, así que funciona sin tocar nada — se declaran explícitos para que nadie los suba después sin entender la consecuencia. Confirmar los nombres exactos en el `config/reverb.php` que genere `install:broadcasting` en la versión instalada.

⚠️ **Los mensajes emitidos mientras un cliente está desconectado se pierden**: Reverb no tiene buffer ni reenvío, y la desconexión ocurre igual (pestaña en segundo plano en iOS, cambio de red, `deploy` que reinicia el proceso). El frontend debe refetchear por REST al reconectar — ver sección **16.9** del documento de arquitectura.

### Variables del frontend (`.env.local` de Next.js)
```
NEXT_PUBLIC_REVERB_APP_KEY=<mismo REVERB_APP_KEY>
NEXT_PUBLIC_REVERB_HOST=ws.midominio.com
```

⚠️ Solo `REVERB_APP_KEY` va al frontend. `REVERB_APP_SECRET` **jamás** — todo lo prefijado con `NEXT_PUBLIC_` queda visible en el bundle del navegador.

### Docker Compose
```yaml
  reverb:
    build: ./docker/php
    command: php artisan reverb:start --host=0.0.0.0 --port=8080
    depends_on: [app, redis]
    expose: ["8080"]
    restart: unless-stopped
```

### Nginx (subdominio WebSocket)
```nginx
server {
    listen 443 ssl http2;
    server_name ws.midominio.com;

    location / {
        proxy_pass          http://reverb:8080;
        proxy_http_version  1.1;
        proxy_set_header    Upgrade $http_upgrade;
        proxy_set_header    Connection "upgrade";
        proxy_set_header    Host $host;
        proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
```

⚠️ **`proxy_read_timeout` debe ser el timeout más laxo de la cadena.** Con el valor típico de 60 s, Nginx cierra la conexión justo cuando toca el ping del servidor y el chat se cae aproximadamente cada minuto. Quien detecta conexiones muertas es el `ping/pong` del protocolo Pusher, no el proxy.

### Supervisor
```ini
[program:rentas-reverb]
process_name=%(program_name)s
command=php /var/www/html/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
numprocs=1
user=deploy
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/reverb.log
```

### Emitir un evento
```php
// app/Events/MessageSent.php
class MessageSent implements ShouldBroadcast
{
    public function __construct(public Message $message) {}

    public function broadcastOn(): array
    {
        return [new PresenceChannel("conversation.{$this->message->conversation_id}")];
    }

    public function broadcastWith(): array
    {
        return ['message' => new MessageResource($this->message)];
    }
}
```

### Verificar que funciona
```bash
php artisan reverb:start --debug        # log de conexiones y eventos en consola
php artisan tinker
>>> broadcast(new App\Events\MessageSent(App\Models\Message::first()));
```

Referenciado desde: `../arquitectura/`, secciones 8, 11 y **16**.
