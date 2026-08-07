# Servicio: Pagos — Mercado Pago

## ¿Para qué se usa?
Método de pago complementario a Stripe (ver `05-pagos-stripe.md`), orientado al mercado mexicano: tarjetas nacionales con mejor tasa de aprobación, OXXO (pago en efectivo) y SPEI (transferencia bancaria).

## Justificación
- Mayor tasa de aceptación con tarjetas emitidas en México frente a procesadores extranjeros.
- **OXXO** es relevante si parte de los clientes no tiene tarjeta de crédito/débito — amplía el mercado direccionable.
- Checkout Pro (alojado, más simple de integrar) o Checkout API (embebido, más control de UX).

**Recomendación de uso combinado:** mostrar ambos métodos en el `BookingForm` del frontend y dejar que el cliente elija; internamente, `PaymentService` decide qué proveedor invocar según el método seleccionado (patrón Strategy).

## Ruta de creación
1. Crear cuenta en https://www.mercadopago.com.mx (o vincular una existente).
2. Completar activación de cuenta de vendedor: datos fiscales (RFC), cuenta CLABE para depósitos.
3. Ir a **Tus integraciones → Crear aplicación** en https://www.mercadopago.com.mx/developers/panel
4. Tipo de integración: "Pagos online" → Checkout Pro o Checkout API según se decida.
5. Copiar `Public Key` y `Access Token` (modo Test primero, luego Producción).
6. Configurar **Webhooks / Notificaciones IPN**:
   - URL: `https://api.midominio.com/api/v1/webhooks/mercadopago`
   - Eventos: `payment` (creación y actualización de pagos).

## Contrato / condiciones
- Sin costo fijo mensual.
- Comisión estándar aproximada: **3.5%–4.99% + IVA** por transacción con tarjeta (varía según modalidad de cobro y si es pago en una sola exhibición o diferido; confirmar tarifa vigente en el panel de "Costos" antes de operar).
- OXXO/efectivo: comisión adicional fija por transacción (verificar en el panel, suele rondar los $10–12 MXN).
- Depósito de fondos: puede ser inmediato (con costo) o en 14 días hábiles (sin costo adicional) — configurable en el dashboard.
- Revisar Términos y Condiciones en https://www.mercadopago.com.mx/ayuda/terminos-y-politicas_194 antes de activar cuenta en producción.

## Configuración

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
