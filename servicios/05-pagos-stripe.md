# Servicio: Pagos — Stripe

## ¿Para qué se usa?

Procesar el pago (total o anticipo) de una reserva con tarjeta, principalmente de huéspedes internacionales. Genera un `PaymentIntent` que se confirma vía webhook y actualiza el estado de la reserva.


## Justificación

- Mejor documentación y SDK del mercado, ampliamente soportado por Laravel (`laravel/cashier` opcional, o SDK directo `stripe/stripe-php`).
- Ideal para tarjetas internacionales (turistas extranjeros reservando casas en México).
- Checkout **alojado** (Stripe Checkout). ✅ **Decidido (28-ago-2026).**

**Por qué alojado y no Elements:** el número de tarjeta nunca toca este dominio, así que el alcance de PCI se queda en SAQ-A, el más bajo. Y resuelve los tres métodos que el sistema ofrece —tarjeta, OXXO y SPEI— en una sola integración: con Elements, la ficha de OXXO y la CLABE de SPEI necesitan pantalla propia, porque no caben en un formulario de tarjeta. Se paga con menos control del aspecto: la página de pago es de Stripe, personalizable con logo y colores pero no con el diseño del sitio.

⚠️ **Implica una Checkout Session, no un PaymentIntent suelto.** Un PaymentIntent solo devuelve un `client_secret` para montar el formulario uno mismo; no tiene página propia a la que mandar al huésped. Durante un tiempo el backend creó PaymentIntents mientras el frontend esperaba una `checkout_url` que nunca llegaba, y el botón de pagar no hacía nada.

⚠️ **`payments.provider_ref` es la Checkout Session (`cs_…`), no el PaymentIntent.** Es lo que existe al abrir el cobro y lo que viaja en los webhooks `checkout.session.*`. Pero **un reembolso va contra el PaymentIntent**, que solo se conoce cuando el webhook confirma el pago: por eso se guarda aparte en `provider_payment_ref`. Reembolsar contra la sesión falla.

**Complementar con Mercado Pago** (ver `06-pagos-mercadopago.md`) para métodos locales mexicanos (OXXO, SPEI, tarjetas de débito nacionales con mejores tasas de aprobación).


## 💰 Precio y plan gratuito para desarrollo

Stripe **no cobra nada por mes ni por usar el modo de pruebas**:

| Modo | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Test mode / Sandbox** | $0, sin límite de tiempo ni de transacciones | ✅ Sí — tarjetas de prueba, webhooks de prueba, todo el flujo completo sin mover dinero real; es el modo recomendado para todo el desarrollo |
| **Live mode** | Sin costo fijo mensual; solo comisión por transacción real (ver abajo) | Solo se activa cuando el proyecto pasa a producción |

**Recomendación:** desarrollar y probar 100% en modo Test (gratis e indefinido) y activar el modo Live solo al lanzar a producción.


## Ruta de creación

1. Crear cuenta en https://dashboard.stripe.com/register
2. Completar el proceso de activación de cuenta (KYC): datos fiscales de la empresa, cuenta bancaria de destino, tipo de negocio ("Bienes raíces / alquiler vacacional" o similar).
3. Ir a **Developers → API keys**: copiar `Publishable key` y `Secret key` (usar primero en modo Test).
4. Ir a **Developers → Webhooks → Add endpoint**:
   - URL: `https://api.midominio.com/api/v1/webhooks/stripe`
   - Eventos a escuchar: `payment_intent.succeeded`, `payment_intent.payment_failed`, `charge.refunded`.
   - Copiar el `Signing secret` (`whsec_...`).
5. Activar cuenta en modo Live una vez probado en Test (requiere aprobación de Stripe, puede tomar 1–3 días hábiles).


## Contrato / condiciones

- Sin costo fijo mensual, sin permanencia.
- Comisión estándar México: **3.6% + $3 MXN por transacción con tarjeta** (verificar tarifa vigente en el dashboard, puede variar).
- Fondos depositados en 2–7 días hábiles según el tipo de cuenta y antigüedad.
- Revisar los **Términos de Servicio de Stripe** en https://stripe.com/mx/legal antes de operar, en particular las políticas de contracargos (chargebacks) — el negocio (no Stripe) asume el riesgo de contracargo salvo con Radar/protecciones adicionales contratadas.


## Configuración


### Variables de entorno (`.env`)
```
STRIPE_KEY=pk_live_xxxxx
STRIPE_SECRET=sk_live_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx
```

### Flujo en el backend (resumen)
```php
// PaymentService.php
public function createIntent(Booking $booking): PaymentIntent
{
    return \Stripe\PaymentIntent::create([
        'amount' => $booking->total_price * 100, // centavos
        'currency' => 'mxn',
        'metadata' => ['booking_id' => $booking->id],
    ]);
}
```

```php
// PaymentWebhookController.php — SIEMPRE verificar la firma del webhook
$event = \Stripe\Webhook::constructEvent(
    $request->getContent(),
    $request->header('Stripe-Signature'),
    config('services.stripe.webhook_secret')
);

match ($event->type) {
    'payment_intent.succeeded' => ProcessPaymentWebhook::dispatch($event->data->object),
    'payment_intent.payment_failed' => ReleaseHoldOnPaymentFailed::dispatch($event->data->object),
    default => null,
};
```

**Importante:** procesar el webhook de forma **idempotente**. Stripe reintenta la entrega durante días si tu servidor tarda o devuelve un 500, así que el mismo evento llega varias veces.

La implementación usa una tabla `webhook_events` con `UNIQUE (provider, event_id)`. ⚠️ **La protección real es el índice al insertar, no un `exists()` previo**: dos entregas simultáneas del mismo evento pasarían la comprobación a la vez.

### Eventos que se escuchan

| Evento de Stripe | Significa |
|---|---|
| `checkout.session.completed` | Pagado con tarjeta → confirma la reserva |
| `checkout.session.async_payment_succeeded` | El voucher de OXXO/SPEI se pagó → confirma la reserva |
| `checkout.session.async_payment_failed` | El voucher de OXXO/SPEI caducó sin pagarse |
| `payment_intent.payment_failed` | Rechazado → **no** libera fechas: puede reintentar con otra tarjeta |
| `payment_intent.succeeded` | Solo por si un cobro se abrió sin sesión (por ejemplo desde el panel de Stripe). No casa con ningún pago del sistema; se registra para conciliar |
| `charge.refunded` | Reembolso confirmado |

⚠️ **`checkout.session.async_payment_succeeded` no es opcional.** Con OXXO la sesión se completa al emitir la ficha, no al cobrar: sin este evento, una reserva pagada en la tienda no se confirmaría nunca.

⚠️ **El objeto del evento es la sesión, no el intent:** trae `amount_total`, no `amount`. Leer solo `amount` deja el importe en null justo en el evento que confirma la reserva.

### ⚠️ Vencimiento de OXXO y SPEI

Si el voucher vive 3 días y el sistema expira la reserva a las 48 h, alguien puede pagar en la tienda una reserva que ya se liberó y quizá se revendió.

⚠️ **Con Checkout alojado esto se invierte: el apartado de un voucher NO se toca.** `session.expires_at` es cuándo deja de abrirse la página de pago, no cuándo caduca el voucher — y Stripe no admite sesiones de más de 24 h. Aplicarlo al apartado recortaría a un día el plazo de 72 h de la reserva, que es exactamente el error que este apartado advierte, al revés. El huésped entra a la página una vez, saca la ficha y la paga tres días después. Para tarjeta sí se usa, porque ahí la sesión *es* el plazo.

### ⚠️ Reembolsar un pago en efectivo

Un pago hecho con OXXO **no se devuelve en efectivo**: Stripe pide los datos bancarios del huésped para transferir. Ese flujo necesita una pantalla propia cuando se active el método — no es el mismo botón que el reembolso de tarjeta.

Referenciado desde: `../arquitectura/`, secciones 1, 6, 7 y 10.
