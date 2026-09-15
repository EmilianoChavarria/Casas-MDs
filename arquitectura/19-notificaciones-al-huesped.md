# 19. Notificaciones al huésped

Correos automáticos que cubren todo el recorrido de la reserva. Complementa la sección 7 (correo transaccional) y la 16 (avisos del chat).

> **Las experiencias tienen su propio ciclo** —4 correos al huésped y 3 avisos al guía— documentado en [la sección 20.10](20-experiencias-tours-guiados.md). Comparte el mecanismo, el job diario y el `notification_log` de 19.4.

---

### 19.1 El ciclo completo

| # | Cuándo se envía | Contenido | Notas |
|---|---|---|---|
| 1 | Al crear la reserva | Confirmación de solicitud, desglose de precio, plazo para pagar | **Sin dirección exacta** — ver 19.2 |
| 2 | A mitad del plazo de OXXO/SPEI | Recordatorio de pago + referencia y fecha límite | Solo si `payment_method != card` y sigue `pending` |
| 3 | Al confirmarse el pago | Reserva asegurada, política de cancelación aplicable | Disparado por el webhook, no por el reloj |
| 4 | **Checkin − 2 días** | **Instrucciones de llegada**: dirección exacta, cómo entrar, código de acceso, contacto | El correo más importante del ciclo |
| 5 | Checkin − 1 día | Recordatorio breve, hora de entrada, clima | Reduce el no-show |
| 6 | Checkout + 1 día | Agradecimiento + enlace para dejar reseña | Solo si aún no reseñó; el pico de respuesta está entre 1 y 3 días |
| 7 | Al cancelar | Confirmación, monto reembolsado y plazo | Ver D7 |

Todos son **jobs sobre datos que ya existen**. No requieren tablas nuevas salvo el registro de envíos (19.4).

Los correos 1, 3 y 7 son **reactivos** (los dispara un evento). Los 2, 4, 5 y 6 son **programados** (un job diario los busca por fecha).

---

### 19.2 ⚠️ La dirección y el código de acceso no viajan al reservar

**Es una decisión de seguridad, no de diseño de correos.**

Si la dirección exacta y el código de la cerradura van en el correo de confirmación, quedan **meses** en la bandeja de entrada del huésped — expuestos a cualquier filtración de esa cuenta, reenvío accidental o dispositivo perdido. Y siguen ahí después de que la estancia terminó.

Enviándolos **2 días antes**, la ventana de exposición pasa de meses a días.

El correo de confirmación lleva solo la zona aproximada, que es además lo que muestra el mapa público de la propiedad.

```
properties.access_instructions   TEXT, cifrado en reposo
```

Se guarda con el *cast* `encrypted` de Laravel. **Un volcado de la base de datos no debe entregar los códigos de acceso de las 50 casas.** Es una credencial física: quien la tiene, entra.

⚠️ El código de acceso debería rotarse entre huéspedes. Eso es proceso operativo del cliente, no del sistema — pero conviene decírselo, porque un código fijo compartido por correo durante dos años deja de ser una medida de seguridad.

---

### 19.3 Multi-idioma

Cada plantilla existe en **español, inglés y francés**: 7 correos × 3 idiomas = **21 textos**.

El idioma sale de `customers.locale`, que se fija al reservar o llega del perfil de Google (sección 7.1.1). Laravel lo resuelve con `->locale($customer->locale)` en la notificación; las plantillas viven en `resources/lang/{es,en,fr}`.

⚠️ **Estos textos los tiene que aportar y revisar el cliente.** Es contenido de negocio, no de programación — sobre todo el correo 4, que explica cómo entrar a la casa. Un correo de instrucciones mal traducido al francés genera exactamente la llamada nocturna que se pretendía evitar.

A diferencia de las descripciones de propiedades (D2), aquí **no aplica la traducción automática asistida**: son 21 textos fijos que se escriben una vez y no cambian. Traducirlos bien de entrada es más barato que montar un flujo de revisión.

---

### 19.4 Idempotencia: no enviar dos veces

```
notification_log (id, notifiable_type, notifiable_id, type, locale, sent_at)
                  -- UNIQUE (notifiable_type, notifiable_id, type)
                  -- polimórfico: bookings y experience_bookings (sección 20.10)
```

El `UNIQUE` es la protección real. Si el job diario se ejecuta dos veces —por un reinicio, un despliegue a medias o un solapamiento del scheduler— el segundo intento falla al insertar y no se envía nada.

Sin esto, el huésped recibe el recordatorio de llegada tres veces y el sistema parece roto. Peor: el correo 6 pidiendo reseña, repetido, se percibe como spam.

**Condiciones adicionales antes de enviar:**

| Correo | No enviar si |
|---|---|
| 2 (OXXO) | La reserva ya no está `pending` |
| 4 y 5 | La reserva está `cancelled` o `expired` |
| 6 (reseña) | Ya existe una reseña para esa reserva, o la estancia se canceló |

---

### 19.5 Bajas y rebotes

Los correos 1 a 5 y el 7 son **transaccionales**: el huésped los necesita para completar un servicio que contrató. No llevan opción de baja.

El **correo 6 (solicitud de reseña) sí la lleva**. Es el único con carácter promocional, y agrupar promocional con transaccional bajo el mismo remitente es lo que degrada la reputación del dominio y acaba mandando todo a spam.

```
customers.review_emails_opt_out   BOOLEAN DEFAULT 0
```

**Rebotes:** Resend y SES notifican por webhook los correos rechazados. Un correo que rebota de forma permanente debe marcarse en el `customer` para no seguir enviándole — acumular rebotes daña la reputación del dominio y termina afectando a los correos que sí importan, como la confirmación de reserva.

---

### 19.6 Programación

Añadir a `schedule:run` (sección 8.4):

```php
// Una pasada diaria basta para los correos 2, 4, 5 y 6
Schedule::job(new SendScheduledBookingNotifications)->dailyAt('09:00');
```

**A las 9:00, no a medianoche.** Un correo con instrucciones de llegada que entra a las 3 de la madrugada tiene menos probabilidad de leerse a tiempo, y el huésped puede estar en otra zona horaria.

⚠️ Con huéspedes de México, Estados Unidos y Canadá conviene decidir si la hora de envío es la del negocio o la del huésped. Empezar por la del negocio es lo simple y suficiente; ajustar por zona horaria del huésped es una mejora posterior.

---

### 19.7 Diseño de los correos

Todas las plantillas son Markdown de Laravel (`@component('mail::message')`), así que el aspecto vive en **un tema**, no en cada correo: `resources/views/vendor/mail/html/themes/casa-caribe.css`, activado con `mail.markdown.theme`.

- **Paleta del frontend** (sección 17): fondo gris `sand-50`, tarjeta blanca con borde `sand-200`, radio 18 px y sombra suave; barra superior `lagoon-500`; resumen de la reserva en panel `mist` con borde lagoon.
- **Tipografía:** títulos en Poppins y texto en Inter, cargadas desde Google Fonts. Apple Mail, iOS y Outlook para Mac las muestran; **Gmail y Outlook de escritorio no cargan fuentes web** y usan la pila de respaldo (Segoe UI / Helvetica). El diseño tiene que verse bien con la de respaldo.
- **Botones:** **coral** para la acción del huésped (el mismo criterio que el sitio: coral solo para lo que vende) y **lagoon** (`'color' => 'lagoon'`) para los correos del equipo y de los guías, como el primario del panel.
- **Marca:** encabezado con la "C" y el nombre, pie traducido y firma "El equipo de Casa Caribe". El nombre sale de `mail.brand` (`MAIL_BRAND`), **no de `APP_NAME`**, que nombra a la API y acababa impreso en cada correo como "Casas API".
- ⚠️ El CSS del tema se inyecta en línea al renderizar. Lo que tiene que quedar como hoja de estilos (las media queries del móvil) va en el `<style>` del layout, no en el tema.

**Correo para crear contraseña.** Lo comparten "olvidé mi contraseña" y las invitaciones del personal, guías y co-anfitriones (D6). El del framework salía en inglés y firmado "Casas API"; ahora se arma con `ResetPassword::toMailUsing` y textos propios en `lang/{es,en,fr}/mail.php`:

- Si la cuenta tiene una invitación pendiente (`invited_at` y sin contraseña), dice "Te invitaron a Casa Caribe". A quien nunca tuvo cuenta no se le dice que "pidió cambiar su contraseña".
- ⚠️ Con `toMailUsing`, Laravel **ya no aplica `createUrlUsing`**: la liga a `/establecer-contrasena` se arma dentro con la misma función. Si se toca una, se toca la otra.
- `User` implementa `HasLocalePreference`: toda notificación a una cuenta sale en su `users.locale`, no en el idioma de la petición.

### 19.8 ⚠️ Colas: nada `readonly` en las clases base

Todas las notificaciones van por cola. Al sacar el job, `SerializesModels` vuelve a asignar las propiedades desde la clase **hija**, y PHP no permite inicializar una propiedad `readonly` desde fuera de la clase que la declara:

```
Cannot initialize readonly property App\Notifications\BookingNotification::$booking
from scope App\Notifications\BookingConfirmed
```

Pasó entre el 28-ago y el 15-sep-2026: `BookingNotification`, `ExperienceNotification` y `GuideNotification` declaraban `public readonly` en el constructor, y **ningún correo de reserva, de experiencia ni al guía salió del worker**. Las pruebas no lo vieron porque `Notification::fake()` no serializa nada.

**Regla:** en una clase base de notificación o job encolado, propiedades promovidas sin `readonly`. `QueuedNotificationSerializationTest` pasa un correo de cada base por `serialize`/`unserialize`, lo mismo que hace el worker; una base nueva se agrega ahí.

---
Ver también: [`servicios/07-correo-transaccional.md`](../servicios/07-correo-transaccional.md), secciones 5.4 (reseñas), 5.5 (expiración) y 16 (avisos del chat).

---

