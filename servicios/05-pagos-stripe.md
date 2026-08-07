# Servicio: Pagos — Stripe

## ¿Para qué se usa?

Procesar el pago (total o anticipo) de una reserva con tarjeta, principalmente de huéspedes internacionales. Genera un `PaymentIntent` que se confirma vía webhook y actualiza el estado de la reserva.


## Justificación

- Mejor documentación y SDK del mercado, ampliamente soportado por Laravel (`laravel/cashier` opcional, o SDK directo `stripe/stripe-php`).
- Ideal para tarjetas internacionales (turistas extranjeros reservando casas en México).
- Checkout embebido (Stripe Elements) o alojado (Stripe Checkout) según el nivel de personalización deseado.

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

**Importante:** procesar el webhook de forma **idempotente** (verificar que el `payment_intent.id` no se haya procesado ya) para evitar duplicar confirmaciones si Stripe reintenta la entrega del evento.

Referenciado desde: `../arquitectura/`, secciones 1, 6, 7 y 10.
