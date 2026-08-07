# Configuración


### Variables de entorno (`.env`)
```
MERCADOPAGO_PUBLIC_KEY=APP_USR-xxxxx
MERCADOPAGO_ACCESS_TOKEN=APP_USR-xxxxx
MERCADOPAGO_WEBHOOK_SECRET=<clave de validación de firma, si aplica>
```

### Flujo en el backend (resumen, SDK oficial `mercadopago/dx-php`)
```php
// PaymentService.php
$client = new PreferenceClient();
$preference = $client->create([
    "items" => [[
        "title" => "Reserva Casa Palmar",
        "quantity" => 1,
        "unit_price" => (float) $booking->total_price,
        "currency_id" => "MXN",
    ]],
    "external_reference" => (string) $booking->id,
    "notification_url" => config('app.url') . '/api/v1/webhooks/mercadopago',
]);
```

```php
// PaymentWebhookController.php — validar el origen y consultar el pago real por su ID
// (Mercado Pago envía solo el ID en la notificación; siempre se debe re-consultar
// el estado del pago vía API, nunca confiar en el payload de la notificación por sí solo)
$payment = (new PaymentClient())->get($request->input('data.id'));

if ($payment->status === 'approved') {
    ProcessPaymentWebhook::dispatch($payment);
}
```

**Importante:** al igual que con Stripe, procesar de forma idempotente usando `external_reference` (el `booking_id`) para evitar doble confirmación.

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 1, 6, 7 y 10.
