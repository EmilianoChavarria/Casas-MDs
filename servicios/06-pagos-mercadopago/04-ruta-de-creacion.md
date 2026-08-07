# Ruta de creación

1. Crear cuenta en https://www.mercadopago.com.mx (o vincular una existente).
2. Completar activación de cuenta de vendedor: datos fiscales (RFC), cuenta CLABE para depósitos.
3. Ir a **Tus integraciones → Crear aplicación** en https://www.mercadopago.com.mx/developers/panel
4. Tipo de integración: "Pagos online" → Checkout Pro o Checkout API según se decida.
5. Copiar `Public Key` y `Access Token` (modo Test primero, luego Producción).
6. Configurar **Webhooks / Notificaciones IPN**:
   - URL: `https://api.midominio.com/api/v1/webhooks/mercadopago`
   - Eventos: `payment` (creación y actualización de pagos).

