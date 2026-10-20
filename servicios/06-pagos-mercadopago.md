# Servicio: Mercado Pago — ❌ DESCARTADO como pasarela general · ✅ activo para el modo manual (D17)

> ⚠️ **Actualización (10-oct-2026): esto NO se revirtió, se acotó.** Mientras
> Stripe no puede operar en producción (alta de empresa pendiente), el
> cliente aceptó Mercado Pago como pasarela de un modo de pago manual por
> link — ver [`dudas-cliente.md`, D17](../dudas-cliente.md). Las dos
> pasarelas conviven en el código (`PaymentGateway`) al mismo tiempo:
> **Stripe sigue siendo la pasarela completa** —tarjeta, OXXO, SPEI— para
> cuando esté lista; **Mercado Pago solo atiende el link de pago manual**,
> nunca los otros tres métodos. La decisión de abajo, de no duplicar
> tarjeta/OXXO/SPEI en dos proveedores, sigue vigente.

> **Decisión del cliente (25-ago-2026): no se contrata como pasarela general.** El sistema usa
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

## ⏳ Producción: estado y pendientes (20-oct-2026)

Revisado en el servidor (`deploy@201.79.51.112`, `/var/www/rivierabnb-backend`)
sin mostrar ninguna clave:

| Qué | Estado |
|---|---|
| `MERCADOPAGO_ACCESS_TOKEN` de **producción** (empieza con `APP_USR-`, no `TEST-`) | ✅ |
| `MERCADOPAGO_SANDBOX=false` | ✅ |
| `MERCADOPAGO_WEBHOOK_BASE_URL=https://api.rivierabnb.com` | ✅ |
| `MERCADOPAGO_WEBHOOK_SECRET` | ⏳ **vacío** |
| Interruptor `payments.manual_mode` en la base | ❌ **apagado**: `GET /api/v1/payment-settings` responde `manual_mode: false` |

⚠️ **Mientras el interruptor esté apagado, el sitio ofrece pago con Stripe**,
que sigue en modo de pruebas: un huésped real no podría pagar de verdad.

### Pendientes (hacerlos con el cliente)

**1. Encender Mercado Pago en producción**

```bash
ssh deploy@201.79.51.112
cd /var/www/rivierabnb-backend
php artisan tinker --execute="App\Models\Configuration::set('payments.manual_mode','true','boolean');"
php artisan cache:clear
```

Comprobar en https://api.rivierabnb.com/api/v1/payment-settings que diga
`"manual_mode":true`.

**2. Poner el secreto del webhook** (no bloquea los cobros: el sistema vuelve a
consultar cada pago a la API de Mercado Pago antes de darlo por bueno; el
secreto agrega la verificación de la firma `x-signature`)

1. Mercado Pago Developers → **Tus integraciones** → la aplicación → **Webhooks**.
2. Confirmar la URL `https://api.rivierabnb.com/api/v1/webhooks/mercadopago`
   con el evento **Pagos** marcado.
3. Copiar la **clave secreta** y pegarla en el `.env` del servidor
   (`nano .env`, línea `MERCADOPAGO_WEBHOOK_SECRET=`). No compartirla por chat.
4. Aplicar:
   ```bash
   php artisan config:cache
   php artisan queue:restart
   ```

**3. Prueba con dinero real**

- **No bajar el precio de una casa real**: alguien podría reservarla barata
  durante la prueba. Dar de alta una **casa de prueba**, publicada solo durante
  la prueba, a **unos $20 MXN por noche**.
- Orden:
  1. Reservar como huésped (cuenta real, correo propio).
  2. Pagar con tarjeta propia por el link de Mercado Pago.
  3. En el panel, **"Confirmar pago de Mercado Pago"** en la reserva.
  4. Revisar que llegue el correo de confirmación y que el cobro salga en
     **Reportes → Caja**.
- **Devolución:** hacerla desde el panel de Mercado Pago. Para registrarla en el
  sistema (y que cuente en los reportes) primero hay que fusionar
  `Casas_back#100` y luego `Casas_front#80` (solicitud de cancelación y
  devoluciones registradas a mano).
- **Comisión:** Mercado Pago cobra la suya y normalmente no la regresa al
  reembolsar. Con $20 son centavos.
- **Al terminar:** archivar la casa de prueba y decidir si la reserva y el pago
  de prueba se quedan en los reportes (anotando que fue prueba) o se marcan
  para que no cuenten.

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
