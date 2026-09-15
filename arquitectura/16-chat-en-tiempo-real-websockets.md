# 16. Chat en tiempo real (WebSockets)

Mensajería directa entre el **huésped** (o visitante interesado) y el **administrador** de las casas, dentro de la aplicación.

---

### 16.1 Alcance

**Sí incluye:**
- Chat 1-a-1 huésped ↔ administración, opcionalmente anclado a una propiedad, una reserva, una experiencia o una **solicitud de salida privada** (20.5.1: la solicitud abre la conversación).
- **Puntos de entrada construidos:** "Pregúntale al anfitrión" en la ficha de cada casa (con o sin cuenta), "Escribir sobre esta reserva" en el detalle de la reserva del huésped, la solicitud de salida privada y "Mis mensajes". Con sesión, escribir otra vez sobre la misma casa o la misma reserva **continúa** la conversación abierta; sin sesión nunca se reutiliza (bastaría con escribir el correo de otra persona para entrar a su hilo).
- En la bandeja, el equipo ve **si el cliente escribe desde una reserva** (código, fechas, huéspedes, estado) o si no, y cuántas reservas tiene (en esa casa y en total).
- Chat de experiencias con el guía: **no incluido**, pendiente de **D15**.
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
  experience_id     FK NULL         -- si nace desde una experiencia
  private_request_id FK NULL UNIQUE -- la solicitud de salida privada que la abrió (20.5.1)
  assigned_user_id  FK NULL         -- admin que la atiende
  status            ENUM(open, pending, closed) DEFAULT open
  subject           VARCHAR NULL
  last_message_at   DATETIME        -- desnormalizado, para ordenar la bandeja
  locale            CHAR(2)         -- idioma de los avisos y correos al cliente
  guest_token_hash  CHAR(64) NULL UNIQUE -- acceso de visitante: se busca por hash
  guest_token       TEXT NULL       -- el mismo token cifrado (APP_KEY), para reenviar la liga por correo
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

**Visitante sin cuenta:** un interesado que aún no reserva no tiene `users.id`. Se resuelve su `customer` por correo (misma regla que las reservas, 5.6) y la conversación lleva un token aleatorio de 40 caracteres que viaja **en la liga** `/mensajes/t/{token}`: se le enseña al enviar la solicitud y se le manda por correo.

✅ **Cambio respecto al diseño original (15-sep-2026):** el token **no autoriza un canal privado** ni vive en cookie. Autorizar un canal sin sesión obligaba a un segundo mecanismo de autenticación en `/broadcasting/auth`, y un canal público con el token en el nombre lo filtraría. El visitante **consulta por HTTP cada 5 s** (`after_id`); quien tiene cuenta usa WebSocket. En la base se guarda el **hash** para buscar y una copia **cifrada** para reenviar la liga en el correo de "mensaje sin leer". La página de la liga va con `noindex` y `referrer: no-referrer`.

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

✅ **Cómo quedó construido (15-sep-2026):**

| Diseño | Construido | Por qué |
|---|---|---|
| `presence-conversation.{id}` | `private-conversation.{id}` | La presencia solo servía para decidir el correo de no leído y el "en línea". El correo se resolvió con un job con retraso que mira `read_at` (ver abajo); presencia e "escribiendo…" quedan pendientes |
| `admin.inbox` | `staff.inbox` | Lo atienden administradores **y personal**. Los guías no: `isStaff()` los excluye |
| `/broadcasting/auth` | `/api/broadcasting/auth` con `auth:sanctum` | Bajo `/api` lo cubre el CORS de la API y usa la cookie del SPA, sin token en JavaScript |
| `MessageRead` con `message_ids[]` | `MessagesRead` con `reader` | Se marca leído todo lo que le llegó a un lado; el otro pinta la doble palomita |

**Correo de "mensaje sin leer":** cada mensaje encola `NotifyUnreadMessage` con 2 minutos de retraso. Si al correr ya está leído, no hace nada. Para no mandar un correo por mensaje, `notification_log` reclama una ventana de 30 minutos por conversación y por lado.

**Eventos:** `MessageSent` es `ShouldBroadcast` (encolado) y `ShouldDispatchAfterCommit`: si Reverb no responde, el envío HTTP del mensaje no falla.

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

✅ **Cómo quedó construido (15-sep-2026):** en vez de widget flotante, el cliente tiene **Mis mensajes** (`/mensajes`, `/mensajes/{id}`, `/mensajes/t/{token}`) y el equipo una bandeja en `/dashboard/messages`. Un solo componente de hilo (`components/chat/ChatThread.tsx` + `hooks/useChatThread.ts`) sirve a los tres, contra una interfaz `ChatEndpoint` que abstrae si habla con `/me`, con la liga o con `/admin`. `lib/echo.ts` es el singleton de Echo y autoriza canales con `fetch` + cookie de sesión. El widget flotante, "escribiendo…" y el contador global de no leídos quedan pendientes.

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
    proxy_read_timeout  3600s;
    proxy_send_timeout  3600s;
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

# Mantener viva la conexión detrás de Cloudflare (ver cadena de timeouts abajo)
REVERB_APP_ACTIVITY_TIMEOUT=30
REVERB_APP_PING_INTERVAL=45
```

---

### 16.8 Cadena de timeouts (mantener la conexión viva)

Entre el navegador y el proceso Reverb hay tres intermediarios, cada uno con su propio timeout de inactividad. **Deben ordenarse de menor a mayor**, o el eslabón más estricto tira la conexión antes de que el `ping` alcance a demostrar que sigue viva:

| Orden | Quién | Valor | Dónde se configura |
|---|---|---|---|
| 1 | Cliente (`pusher-js`) hace ping | **30 s** sin tráfico | `REVERB_APP_ACTIVITY_TIMEOUT` (el servidor se lo dicta al cliente en el handshake) |
| 2 | Servidor Reverb pinguea inactivos | **45 s** | `REVERB_APP_PING_INTERVAL` |
| 3 | Cloudflare corta inactivas | **100 s** (plan Free, no configurable) | — |
| 4 | Nginx corta inactivas | **3600 s** | `proxy_read_timeout` |

**No hay que escribir un heartbeat a mano.** Reverb habla el protocolo Pusher, que ya trae `ping/pong`: en el handshake el servidor manda `activity_timeout` dentro de `pusher:connection_established`, y `pusher-js` (debajo de Laravel Echo) emite `pusher:ping` solo tras ese tiempo sin tráfico. Lo único que se configura son los números de arriba.

Los valores por defecto de Reverb (`activity_timeout` 30, `ping_interval` 60) ya caen debajo de los 100 s de Cloudflare, así que funciona sin tocar nada. Se documentan explícitamente para que nadie los suba "para ahorrar tráfico" y rompa el chat sin entender por qué. Confirmar los nombres exactos en el `config/reverb.php` que genere `install:broadcasting` en la versión instalada.

**El error clásico:** dejar `proxy_read_timeout 60s` en Nginx. Empata con el ping del servidor y la conexión se cae aproximadamente cada minuto, con Cloudflare o sin él. Nginx debe ser siempre el más laxo de la cadena.

---

### 16.9 Resincronización al reconectar

Más importante que los timeouts: **un broadcast que se emite mientras el cliente está desconectado se pierde para siempre.** Reverb no tiene buffer ni reenvío. Y la desconexión es inevitable aunque toda la cadena de timeouts esté perfecta:

- Pestaña en segundo plano en iOS Safari — el sistema suspende el socket.
- Cambio de WiFi a datos móviles, o túnel/elevador.
- Cada `deploy` que reinicia el proceso Reverb.

`pusher-js` reconecta solo, así que el síntoma es traicionero: el chat *se ve* funcionando, pero le faltan mensajes en medio. Nadie reporta el bug porque nadie sabe que faltó algo.

**Regla:** el WebSocket entrega mensajes *nuevos*; la fuente de verdad del historial siempre es REST. Al reconectar hay que traer el hueco.

```ts
// hooks/useEcho.ts — al recuperar la conexión, traer lo que se perdió
echo.connector.pusher.connection.bind('connected', () => {
  // lastMessageId = el id más alto que el cliente ya tiene en pantalla
  refetchMessagesSince(conversationId, lastMessageId);
});
```

Del lado del backend, esto solo necesita que el endpoint de listado ya soporte el filtro — que es la misma paginación keyset de la sección 16.4, en sentido inverso:

```
GET /api/conversations/{id}/messages?after_id=<lastMessageId>
```

Al aplicar el resultado, deduplicar por `id`: si un mensaje llegó por WebSocket *y* por el refetch, debe aparecer una sola vez. Un `Map` por `id` al hacer merge lo resuelve.

Conviene además mostrar el estado de conexión en la UI (un indicador discreto de "reconectando…" enganchado a los eventos `connecting`/`unavailable` de `pusher.connection`): si la reconexión falla del todo, el usuario debe enterarse en lugar de quedarse mirando un chat mudo.

---

### 16.10 Seguridad del chat

| Riesgo | Mitigación |
|---|---|
| Escuchar conversaciones ajenas | Canales privados/presence + autorización en `routes/channels.php` contra `customer_id`; jamás canales públicos |
| XSS vía mensaje | Guardar texto plano; renderizar como texto en React (nunca `dangerouslySetInnerHTML`); si se quiere markdown, sanitizar en servidor con lista blanca |
| Spam / flood | `throttle:30,1` en `POST /messages`; `body` máx. 2,000 caracteres |
| Adjuntos maliciosos | Mismo pipeline de imágenes de la sección 7: `mimes:jpg,png,webp,pdf`, `max:5120`, reprocesado antes de subir a R2, URL firmada con expiración |
| Fuga de datos personales | No permitir que el chat cambie estados de reserva ni pagos; es solo mensajería |
| Liga de visitante filtrada | Token aleatorio de 40 caracteres, buscado por hash; token equivocado = 404 igual que inexistente; `throttle` en lectura y escritura; página `noindex` + `no-referrer`. ⚠️ Quien tenga la liga lee la conversación: por eso solo va al correo del cliente y a la pantalla de quien envió la solicitud |

#### 16.10.1 Retención de conversaciones — ✅ DECIDIDO: diferenciada por tipo

No todas las conversaciones valen lo mismo. Una consulta de alguien que nunca reservó es dato personal sin contrapartida operativa; el hilo de una reserva real es evidencia.

| Tipo | Criterio | Retención | Por qué |
|---|---|---|---|
| **Sin reserva** (`booking_id IS NULL`) | Interesado que preguntó y nunca reservó | **12 meses** desde el último mensaje | Sin valor operativo pasado un ciclo anual completo de temporadas. Conservarlo solo acumula dato personal expuesto |
| **Con reserva** (`booking_id NOT NULL`) | Hilo vinculado a una estancia | **5 años** desde el `checkout` | Alineado con la conservación de comprobantes fiscales. Es la evidencia de lo acordado ante una disputa, un contracargo o una reclamación tardía |

```php
// Job: PurgeOldConversations (schedule:run diario)
Conversation::query()
    ->where('status', 'closed')
    ->where(function ($q) {
        $q->whereNull('booking_id')
          ->where('last_message_at', '<', now()->subMonths(12));
    })
    ->orWhere(function ($q) {
        $q->whereNotNull('booking_id')
          ->whereHas('booking', fn ($b) => $b->where('checkout', '<', now()->subYears(5)));
    })
    ->chunkById(500, fn ($rows) => $rows->each->forceDelete());
```

Ambos plazos viven en `configurations` (`chat.retention_months_orphan`, `chat.retention_years_booked`), no en el código: es una política que puede cambiar sin desplegar.

⚠️ **La purga es borrado real (`forceDelete`), no *soft delete***. Un `deleted_at` no cumple una política de retención de datos personales — el dato sigue ahí. Los adjuntos en R2 se borran en el mismo job; si no, quedan huérfanos y accesibles por URL.

⚠️ **Anonimizar en vez de borrar no es alternativa aquí.** Sustituir nombre, correo y teléfono en las columnas deja intacto el cuerpo de los mensajes, donde el huésped escribe su propio teléfono, su número de vuelo o su dirección. Daría cumplimiento aparente sin cumplimiento real.

⚠️ **Consistencia con el aviso de privacidad.** Estos plazos deben coincidir con lo que declare el aviso de privacidad del cliente. Si el aviso dice otra cosa, manda el aviso — ajustar la configuración, no al revés.
| Retención | Política diferenciada por tipo de conversación — ver 16.10.1 |

---

### 16.11 Escalabilidad

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

