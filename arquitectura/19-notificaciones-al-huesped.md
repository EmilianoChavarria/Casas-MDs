# 19. Notificaciones al huésped

Correos automáticos que cubren todo el recorrido de la reserva. Complementa la sección 7 (correo transaccional) y la 16 (avisos del chat).

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
notification_log (id, booking_id FK, type, locale, sent_at)
                  -- UNIQUE (booking_id, type)
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

Ver también: [`servicios/07-correo-transaccional.md`](../servicios/07-correo-transaccional.md), secciones 5.4 (reseñas), 5.5 (expiración) y 16 (avisos del chat).

---

