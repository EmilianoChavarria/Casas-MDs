# Justificación

- Mayor tasa de aceptación con tarjetas emitidas en México frente a procesadores extranjeros.
- **OXXO** es relevante si parte de los clientes no tiene tarjeta de crédito/débito — amplía el mercado direccionable.
- Checkout Pro (alojado, más simple de integrar) o Checkout API (embebido, más control de UX).

**Recomendación de uso combinado:** mostrar ambos métodos en el `BookingForm` del frontend y dejar que el cliente elija; internamente, `PaymentService` decide qué proveedor invocar según el método seleccionado (patrón Strategy).

