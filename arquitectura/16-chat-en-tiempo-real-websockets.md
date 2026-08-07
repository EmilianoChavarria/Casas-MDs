# 16. Chat en tiempo real (WebSockets)

Mensajería directa entre el **huésped** (o visitante interesado) y el **administrador** de las casas, dentro de la aplicación.

---

### 16.1 Alcance

**Sí incluye:**
- Chat 1-a-1 huésped ↔ administración, opcionalmente anclado a una propiedad o a una reserva.
- Entrega en tiempo real, indicador de "escribiendo…", acuse de lectura, contador de no leídos.
- Historial persistente y consultable desde el dashboard admin.
- Adjuntar imágenes (ej. comprobante de pago, foto de un desperfecto).
- Notificación por correo si el destinatario está desconectado.

**No incluye (fuera de alcance, YAGNI):**
- Chat entre huéspedes.
- Videollamadas o voz.
- Bots / respuestas automáticas con IA.

---

### 16.2 Decisión técnica: Laravel Reverb

| Opción | Veredicto |
|---|---|
| **Laravel Reverb** (self-hosted, en el mismo VPS) | ✅ **Elegida.** Servidor WebSocket oficial de Laravel, integración nativa con `broadcast()`, sin costo por conexión ni por mensaje. Ya hay experiencia previa con Reverb + Supervisor en este stack |
| Pusher / Ably (gestionado) | Sin infraestructura que mantener, pero se paga por conexión y por mensaje; el chat de soporte genera volumen constante. Reservado como plan B si mantener el proceso resulta costoso |
| Polling HTTP cada N segundos | Descartado para el chat. Sí se conserva como **fallback** cuando el WebSocket no conecta (redes corporativas que bloquean `wss://`) |
| Soketi | Alternativa self-hosted válida, pero Reverb es primera parte de Laravel y ya viene con el framework |

**Regla de oro: el WebSocket es transporte, no almacenamiento.** El mensaje se guarda en MySQL **primero** y se difunde después. Si el socket falla, el mensaje existe igual y aparece al recargar. Nunca depender del canal para la persistencia.

---

### 16.3 Flujo de un mensaje

```
Huésped escribe                      Admin conectado
     │                                      ▲
     │ 1. POST /api/v1/conversations/{id}/messages   (HTTP, no WS)
     ▼                                      │
┌──────────────────┐                        │
│ MessageController │                       │
└────────┬─────────┘                        │
         │ 2. ChatService::send()           │
         ▼                                  │
┌──────────────────┐                        │
│ MySQL: messages   │  ← fuente de verdad   │
└────────┬─────────┘                        │
         │ 3. broadcast(new MessageSent)    │
         ▼                                  │
┌──────────────────┐   4. wss://  private-conversation.{id}
│  Reverb (WS)      │ ───────────────────────┘
└────────┬─────────┘
         │ 5. si el destinatario NO está en el canal (presence)
         ▼
┌──────────────────────────────────┐
│ Job (cola Redis, delay 2 min)     │
│ NotifyUnreadMessage → Resend/SES  │
└──────────────────────────────────┘
```

El paso 1 es HTTP normal, no un evento de cliente por WebSocket. Así el mensaje pasa por validación, FormRequest, policies, rate limiting y auditoría como cualquier otro recurso. El WS solo empuja.

**Optimistic UI:** el frontend pinta el mensaje en gris con estado `sending` en cuanto el usuario presiona enviar, y lo reconcilia por `client_uuid` cuando llega la respuesta HTTP (o el evento WS, lo que llegue primero). Evita la sensación de lentitud sin mentir sobre la entrega.

---

### 16.4 Modelo de datos

```
conversations (
  id,
  customer_id       FK              -- huésped
  property_id       FK NULL         -- si la conversación nace desde una propiedad
  booking_id        FK NULL         -- si nace desde una reserva
  assigned_user_id  FK NULL         -- admin que la atiende
  status            ENUM(open, pending, closed) DEFAULT open
  subject           VARCHAR NULL
  last_message_at   DATETIME        -- desnormalizado, para ordenar la bandeja
  guest_token       CHAR(36) NULL   -- acceso de visitante sin cuenta
  ...auditoría
)

messages (
  id,
  conversation_id FK,
  sender_type     ENUM(customer, admin, system)
  sender_id       BIGINT NULL       -- users.id o customers.id según sender_type
  body            TEXT NULL
  attachment_url  VARCHAR NULL      -- objeto en R2
  attachment_meta JSON NULL         -- {mime, size, width, height}
  client_uuid     CHAR(36)          -- idempotencia + reconciliación optimistic UI
  read_at         DATETIME NULL
  created_at
)
```

**Índices:**
- `conversations(status, last_message_at DESC)` — bandeja de entrada del admin.
- `conversations(customer_id)`, `conversations(booking_id)`.
- `messages(conversation_id, id DESC)` — paginación del historial (keyset, no `OFFSET`).
- `messages(conversation_id, read_at)` — contar no leídos sin escanear todo.
- `messages(client_uuid)` UNIQUE — descarta reenvíos duplicados cuando el cliente reintenta.

**`sender_type = system`** para mensajes automáticos dentro del hilo ("Reserva confirmada", "Pago recibido"). Da contexto al admin sin salir del chat.

**Contador de no leídos:** se calcula con `COUNT(*) WHERE read_at IS NULL AND sender_type != <quien pregunta>` y se cachea en Redis por conversación, invalidando en cada `send` y `markAsRead`. No se guarda como columna en `conversations` — se desincroniza en cuanto haya concurrencia.

**Visitante sin cuenta:** un interesado que aún no reserva no tiene `users.id`. Se crea un `customer` mínimo (nombre + email) y la conversación lleva un `guest_token` UUID que el frontend guarda en cookie httpOnly. Ese token autoriza el canal privado. Al registrarse o reservar, la conversación se vincula al `customer_id` definitivo y el token se anula.

---

### 16.5 Canales de broadcasting

```php
// routes/channels.php

// Hilo de una conversación: solo su dueño y los administradores
Broadcast::channel('conversation.{conversationId}', function ($user, int $conversationId) {
    $conversation = Conversation::findOrFail($conversationId);

    if ($user->isAdmin()) {
        return ['id' => $user->id, 'name' => $user->name, 'role' => 'admin'];
    }

    return $conversation->customer_id === $user->customer_id
        ? ['id' => $user->id, 'name' => $user->name, 'role' => 'customer']
        : false;
});

// Bandeja global del admin: notifica conversaciones nuevas sin estar dentro del hilo
Broadcast::channel('admin.inbox', fn ($user) => $user->isAdmin());
```

Se usa **presence channel** (`presence-conversation.{id}`) y no `private`, porque el estado de presencia es justo lo que permite decidir si mandar el correo de "tienes un mensaje sin leer" y mostrar "en línea".

La autorización pasa por `/broadcasting/auth`, protegido por Sanctum — la misma sesión que el resto del dashboard. **Nunca** un canal público: los canales públicos son legibles por cualquiera que adivine el ID.

**Eventos:**

| Evento | Canal | Payload |
|---|---|---|
| `MessageSent` | `presence-conversation.{id}` + `admin.inbox` | mensaje completo serializado |
| `MessageRead` | `presence-conversation.{id}` | `message_ids[]`, `read_at` |
| `ConversationAssigned` | `admin.inbox` | `conversation_id`, `user_id` |
| "escribiendo…" | `presence-conversation.{id}` | **whisper** (client event), no toca el backend |

El indicador de escritura va por `whisper()` a propósito: dispara decenas de eventos por minuto y no vale la pena que cada tecla llegue a PHP.

---

### 16.6 Frontend (Next.js)

```
src/
├── components/chat/
│   ├── ChatWidget.tsx        # burbuja flotante en el sitio público (Client Component)
│   ├── ChatWindow.tsx
│   ├── MessageList.tsx       # scroll invertido + paginación keyset hacia arriba
│   ├── MessageBubble.tsx
│   ├── MessageComposer.tsx   # textarea + adjuntar + whisper de typing
│   └── TypingIndicator.tsx
├── hooks/
│   ├── useEcho.ts            # instancia única de Laravel Echo (singleton)
│   ├── useConversation.ts    # historial + envío + optimistic UI
│   └── useUnreadCount.ts
└── app/(admin)/dashboard/chat/
    ├── page.tsx              # bandeja de entrada
    └── [conversationId]/page.tsx
```

```ts
// services/echoClient.ts
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';   // Reverb habla el protocolo Pusher

export const echo = new Echo({
  broadcaster: 'reverb',
  key:      process.env.NEXT_PUBLIC_REVERB_APP_KEY,
  wsHost:   process.env.NEXT_PUBLIC_REVERB_HOST,
  wsPort:   443,
  wssPort:  443,
  forceTLS: true,
  enabledTransports: ['ws', 'wss'],
  authEndpoint: `${process.env.NEXT_PUBLIC_API_URL}/broadcasting/auth`,
  withCredentials: true,   // cookie de Sanctum
});
```

**Puntos de cuidado en React:**
- El chat es **siempre Client Component**. Nada de WebSockets en Server Components.
- Suscribirse en `useEffect` y **desuscribirse en el cleanup** (`echo.leave(channel)`). En dev, el StrictMode monta dos veces: sin cleanup se duplican los mensajes en pantalla.
- Una sola instancia de `Echo` para toda la app (singleton en módulo), no una por componente.
- Reconexión con backoff exponencial; mientras esté caído, degradar a polling cada 10 s del endpoint REST para no perder mensajes.
- Al reconectar, pedir `GET /messages?after_id=<último visto>` — el WS no reenvía lo perdido durante la desconexión.

---

### 16.7 Infraestructura

Servicio adicional en `docker-compose.yml` (complementa la sección 8.2):

```yaml
  reverb:
    build: ./docker/php
    command: php artisan reverb:start --host=0.0.0.0 --port=8080
    depends_on: [app, redis]
    expose: ["8080"]
```

Proxy inverso en Nginx (subdominio `ws.midominio.com`):

```nginx
location / {
    proxy_pass          http://reverb:8080;
    proxy_http_version  1.1;
    proxy_set_header    Upgrade $http_upgrade;
    proxy_set_header    Connection "upgrade";
    proxy_set_header    Host $host;
    proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto $scheme;
    proxy_read_timeout  60s;
}
```

Supervisor para mantener el proceso vivo:

```ini
[program:rentas-reverb]
command=php /var/www/html/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
numprocs=1
user=deploy
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/reverb.log
```

⚠️ **En Cloudflare**, el proxy naranja soporta WebSockets, pero conviene revisar el timeout de conexión inactiva (100 s en plan Free). Mantener `ping/pong` activo desde el cliente para que la conexión no se corte sola.

Variables de entorno:

```
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=<generado>
REVERB_APP_KEY=<generado>
REVERB_APP_SECRET=<generado>
REVERB_HOST=ws.midominio.com
REVERB_PORT=443
REVERB_SCHEME=https
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080
```

---

### 16.8 Seguridad del chat

| Riesgo | Mitigación |
|---|---|
| Escuchar conversaciones ajenas | Canales privados/presence + autorización en `routes/channels.php` contra `customer_id`; jamás canales públicos |
| XSS vía mensaje | Guardar texto plano; renderizar como texto en React (nunca `dangerouslySetInnerHTML`); si se quiere markdown, sanitizar en servidor con lista blanca |
| Spam / flood | `throttle:30,1` en `POST /messages`; `body` máx. 2,000 caracteres |
| Adjuntos maliciosos | Mismo pipeline de imágenes de la sección 7: `mimes:jpg,png,webp,pdf`, `max:5120`, reprocesado antes de subir a R2, URL firmada con expiración |
| Fuga de datos personales | No permitir que el chat cambie estados de reserva ni pagos; es solo mensajería |
| `guest_token` filtrado | UUID v4, cookie `httpOnly` + `Secure` + `SameSite=Lax`, expira a los 30 días, se invalida al vincular la cuenta |
| Retención | Purga de conversaciones `closed` con más de 24 meses vía `schedule:run` (política de datos personales) |

---

### 16.9 Escalabilidad

Complementa la tabla de la sección 11:

| Etapa | Acción sobre el chat |
|---|---|
| 100–1,000 usuarios | Un proceso Reverb en el mismo VPS. Sobra de largo (soporta miles de conexiones concurrentes) |
| 1,000–10,000 | Subir `ulimit -n`, monitorear memoria del proceso; separar Reverb a su propio contenedor con límites de recursos |
| 10,000–100,000 | Varias instancias de Reverb con **escalado horizontal vía Redis pub/sub** (`REVERB_SCALING_ENABLED=true`) detrás de un balanceador con sticky sessions; o migrar a Pusher/Ably si el costo operativo supera al de la licencia |

Métricas a vigilar en Sentry/logs: conexiones concurrentes, mensajes/segundo, memoria del proceso Reverb y tasa de reconexión del cliente (una tasa alta delata un problema de red o de timeout en el proxy).

---

Ver también: [`servicios/12-websockets-reverb.md`](../servicios/12-websockets-reverb.md), secciones 7, 8 y 11.

---

