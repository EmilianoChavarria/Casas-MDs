# Servicio: Mercado Pago — ❌ DESCARTADO

> **Decisión del cliente (25-ago-2026): no se contrata.** El sistema usa
> **Stripe como única pasarela** ([`05-pagos-stripe.md`](05-pagos-stripe.md)),
> que cubre tarjeta en cualquier moneda y, con cuenta de Stripe México,
> también **OXXO y SPEI**.
>
> **Por qué una sola:** un panel para conciliar, un modelo de webhooks que
> mantener, un juego de credenciales que rotar. La integración de pagos es
> la parte más delicada del sistema y duplicarla duplica el riesgo, no solo
> el trabajo.
>
> **Lo que se pierde, y conviene tenerlo presente:**
>
> - ⚠️ **Comisión algo mayor en tarjeta nacional**: Stripe México ~3.6% +
>   $3 MXN frente a ~3.49% + $4 aquí. Decenas de pesos por reserva.
> - ⚠️ **Tasa de aprobación**: Mercado Pago suele aprobar algo más en
>   tarjetas mexicanas, por su relación con los bancos locales. Si aparecen
>   rechazos inexplicables, es lo primero que hay que medir.
>
> **Este documento se conserva** por si el cliente quiere reconsiderarlo:
> tiene las tarifas, la ruta de alta y la configuración. El código habla
> con una interfaz `PaymentGateway`, así que añadir esta pasarela sería una
> clase nueva y una línea en el contenedor de servicios — nada del flujo de
> reservas cambiaría.

---

## ¿Para qué se usa?

Método de pago complementario a Stripe (ver `05-pagos-stripe.md`), orientado al mercado mexicano: tarjetas nacionales con mejor tasa de aprobación, OXXO (pago en efectivo) y SPEI (transferencia bancaria).


## Justificación

- Mayor tasa de aceptación con tarjetas emitidas en México frente a procesadores extranjeros.
- **OXXO** es relevante si parte de los clientes no tiene tarjeta de crédito/débito — amplía el mercado direccionable.
- Checkout Pro (alojado, más simple de integrar) o Checkout API (embebido, más control de UX).

**Recomendación de uso combinado:** mostrar ambos métodos en el `BookingForm` del frontend y dejar que el cliente elija; internamente, `PaymentService` decide qué proveedor invocar según el método seleccionado (patrón Strategy).


## 💰 Precio y plan gratuito para desarrollo

Mercado Pago tampoco cobra por mes ni por probar:

| Modo | Costo | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Credenciales de Test + Usuarios de prueba** (panel de developers) | $0, sin límite de tiempo | ✅ Sí — se crean "usuarios de prueba" (comprador y vendedor ficticios) para simular el flujo completo de pago, incluyendo webhooks, sin mover dinero real |
| **Producción** | Sin costo fijo mensual; solo comisión por transacción real (ver abajo) | Solo al activar cuenta en producción |

**Recomendación:** usar credenciales de Test (`TEST-...`) durante todo el desarrollo; cambiar a credenciales de Producción (`APP_USR-...`) solo al lanzar.


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
- **Checkout Pro / Link de pago** (tarjeta o SPEI): ~**3.49% + $4 MXN** por transacción con acreditación instantánea, + IVA (16%) sobre la comisión (tarifas ago-2026, confirmar vigentes en el panel de "Costos" antes de operar, ya que Mercado Pago las ajusta con frecuencia).
- **Código QR** (cobro en persona): 0.99%, sin cargo fijo — la opción más económica si aplica al negocio.
- **Transferencias SPEI**: gratis.
- OXXO/efectivo: comisión adicional fija por transacción (verificar en el panel).
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

Referenciado desde: `../arquitectura/`, secciones 1, 6, 7 y 10.
