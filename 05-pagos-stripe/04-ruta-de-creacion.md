# Ruta de creación

1. Crear cuenta en https://dashboard.stripe.com/register
2. Completar el proceso de activación de cuenta (KYC): datos fiscales de la empresa, cuenta bancaria de destino, tipo de negocio ("Bienes raíces / alquiler vacacional" o similar).
3. Ir a **Developers → API keys**: copiar `Publishable key` y `Secret key` (usar primero en modo Test).
4. Ir a **Developers → Webhooks → Add endpoint**:
   - URL: `https://api.midominio.com/api/v1/webhooks/stripe`
   - Eventos a escuchar: `payment_intent.succeeded`, `payment_intent.payment_failed`, `charge.refunded`.
   - Copiar el `Signing secret` (`whsec_...`).
5. Activar cuenta en modo Live una vez probado en Test (requiere aprobación de Stripe, puede tomar 1–3 días hábiles).

