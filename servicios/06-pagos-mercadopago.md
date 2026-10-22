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

## ✅ Producción: probado con dinero real (22-oct-2026)

Revisado en el servidor (`deploy@201.79.51.112`, `/var/www/rivierabnb-backend`)
sin mostrar ninguna clave:

| Qué | Estado |
|---|---|
| `MERCADOPAGO_ACCESS_TOKEN` de **producción** (empieza con `APP_USR-`, no `TEST-`) | ✅ |
| `MERCADOPAGO_SANDBOX=false` | ✅ |
| `MERCADOPAGO_WEBHOOK_BASE_URL=https://api.rivierabnb.com` | ✅ |
| `MERCADOPAGO_WEBHOOK_SECRET` | ✅ **vacío a propósito** — ver el aviso en "Configuración" más abajo, nunca ponerle nada |
| Interruptor `payments.manual_mode` en la base | ✅ **encendido** |

### Prueba con dinero real — ya hecha

Un pago real de $48.10 MXN sobre una "Casa Prueba", pagado por el propio
operador con su tarjeta. Sirvió además para encontrar y corregir un bug real
(ver el aviso de `MERCADOPAGO_WEBHOOK_SECRET` abajo): los avisos de pago real
se rechazaban todos con 400, así que la reserva se hubiera quedado
"pendiente" para siempre sin que nadie se enterara. Ya corregido y
confirmado: el aviso llega, se reprocesa y `reported_paid_at` queda puesto.

⚠️ **Un detalle al repetir esta prueba:** si quien paga usa una tarjeta o
cuenta de Mercado Pago ligada a la MISMA cuenta que está configurada para
*cobrar* (la del `.env`), el botón "Pagar" se queda bloqueado sin ningún
mensaje de error — Mercado Pago no deja que una cuenta se pague a sí misma.
No es un bug del sitio; hace falta una tarjeta/cuenta de otra persona para
probar, o usar ya las credenciales reales del negocio en vez de unas de
prueba personales.

**Pendientes (22-oct-2026):**
- La devolución de esta prueba se registra desde el panel de Mercado Pago,
  y en el sistema con la solicitud de cancelación (`Casas_back#100` /
  `Casas_front#80`, ya mergeados).
- ✅ **Credenciales del negocio (Omar Carrillo Cen)** puestas en el `.env`
  de producción el 22-oct-2026, en lugar de la cuenta personal del
  operador usada para la prueba. Los pagos de huéspedes ya llegan a la
  cuenta del negocio.
- ✅ **Reportes por proveedor:** tabla "Por proveedor" en Reportes → Caja,
  con cobrado, comisión, neto, devuelto y lo que se quedó cada uno
  (`Casas_back#103` / `Casas_front#84`).
- ⏳ **Comisión real de Mercado Pago** (`Casas_back#104` /
  `Casas_front#89`): hasta ahora se guardaba en cero. Al desplegar, correr
  una vez en el servidor como `deploy`:
  ```bash
  cd /var/www/rivierabnb-backend
  php artisan payments:reconcile --dry-run   # solo enseña
  php artisan payments:reconcile             # guarda
  ```

### Cómo cambiar las credenciales de Mercado Pago

1. Mercado Pago Developers (cuenta del negocio) → **Tus integraciones** → la
   aplicación → **Credenciales de producción**: Access Token y Public Key,
   los dos empiezan con `APP_USR-`.
2. **Antes:** revisar que no haya reservas pendientes de pago. Su link es de
   la cuenta anterior; si se paga después del cambio, el dinero llega a esa
   cuenta y hay que confirmarla a mano.
3. En el servidor:
   ```bash
   ssh deploy@201.79.51.112
   cd /var/www/rivierabnb-backend
   cp .env .env.bak-$(date +%F)
   nano .env        # MERCADOPAGO_ACCESS_TOKEN y MERCADOPAGO_PUBLIC_KEY
   ```
   No tocar `MERCADOPAGO_WEBHOOK_SECRET` (vacío) ni `MERCADOPAGO_SANDBOX=false`.
4. Comprobar sin mostrar las claves y aplicar:
   ```bash
   grep -cE '^MERCADOPAGO_ACCESS_TOKEN=APP_USR-' .env   # 1
   grep -cE '^MERCADOPAGO_PUBLIC_KEY=APP_USR-' .env     # 1
   php artisan config:cache
   php artisan queue:restart
   sudo systemctl reload php8.3-fpm
   ```
5. Probar con una reserva de pocos pesos y una tarjeta que **no** sea de la
   cuenta que cobra (Mercado Pago no deja que una cuenta se pague a sí
   misma).
6. Si algo falla: `cp .env.bak-AAAA-MM-DD .env` y repetir el paso 4.

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
MERCADOPAGO_WEBHOOK_SECRET=
```

> ⚠️ **Debe quedarse vacío — no es la clave del panel de "Webhooks".** Descubierto el 22-oct-2026, con el primer pago real: el aviso le llega al backend por el `notification_url` que se manda al CREAR la preferencia de pago, no por la suscripción de "Webhooks" del panel. Ese canal entrega siempre en formato IPN clásico (parámetros en la URL, sin `x-signature`), sin importar qué se configure en el panel. Poner ahí la clave secreta que Mercado Pago muestra en **Webhooks → Configurar notificaciones** hace que el backend rechace **todos** los avisos reales con 400 — un pago real se queda en "pendiente" para siempre, sin ningún error visible salvo en el log del servidor. No hace falta esa firma para estar seguros: el backend siempre vuelve a consultar el pago directo con la API de Mercado Pago antes de dar nada por bueno.

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
