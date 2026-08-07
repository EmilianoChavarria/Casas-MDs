# Configuración


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

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 1, 6, 7 y 10.
