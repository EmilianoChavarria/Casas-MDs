# Servicio: Correo Transaccional — Resend (primario) / Amazon SES (alternativa)

## ¿Para qué se usa?
Envío de correos transaccionales: confirmación de reserva, cancelación, recordatorios, notificaciones al admin de nuevas reservas.

## Justificación
- **Resend** tiene API moderna, buena entregabilidad (deliverability) desde el día uno y un tier gratuito generoso para arrancar; integración directa con Laravel vía el driver de Symfony Mailer.
- **Amazon SES** es más barato a gran volumen si ya se usa AWS/S3, pero requiere más configuración inicial (verificación de dominio, salir del "sandbox" solicitando aumento de límite a AWS Support).

**Recomendación:** iniciar con Resend por simplicidad; migrar a SES solo si el volumen de correos crece mucho (miles diarios) y el costo se vuelve relevante.

## 💰 Precio y plan gratuito para desarrollo
| Proveedor | Plan gratuito | ¿Sirve para pruebas de desarrollo? |
|---|---|---|
| **Resend** | Free tier **permanente**: 3,000 correos/mes (100/día), 1 dominio, sin tarjeta de crédito | ✅ Sí — de sobra para todo el desarrollo y hasta para el arranque en producción |
| **Amazon SES** | 3,000 mensajes/mes gratis, pero **solo los primeros 12 meses** en cuentas AWS creadas antes de jul-2025; cuentas nuevas reciben en su lugar $200 USD de crédito general de AWS | ⚠️ Parcial — no es gratis a largo plazo, requiere salir del modo Sandbox y verificar dominio incluso para pruebas |

**Recomendación:** usar Resend en desarrollo (free tier permanente, cero fricción) y evaluar SES solo si el volumen de producción lo justifica económicamente.

## Ruta de creación (Resend)
1. Crear cuenta en https://resend.com
2. **Domains → Add Domain** (ej. `midominio.com`).
3. Agregar los registros DNS que Resend indique (SPF, DKIM, DMARC recomendado) — esto se hace en Cloudflare si el DNS está ahí (ver `09-cloudflare-dns-cdn.md`).
4. Esperar verificación del dominio (usualmente minutos, hasta 24h).
5. **API Keys → Create API Key** con permiso de solo envío (`Sending access`).

## Ruta de creación (alternativa: Amazon SES)
1. Crear cuenta AWS (si no existe ya, por el uso de S3/R2 podría no ser necesaria si se usa R2 en vez de S3).
2. Consola SES → **Verified identities → Create identity** → verificar el dominio con los registros DNS que indique.
3. Solicitar salida del modo Sandbox: **Account dashboard → Request production access** (formulario explicando el caso de uso; aprobación de AWS en 24–48h).
4. Crear credenciales IAM específicas con permiso `ses:SendEmail` únicamente (principio de mínimo privilegio).

## Contrato / plan recomendado
- **Resend:** tier gratuito permanente 3,000 correos/mes (100/día); plan Pro desde $20 USD/mes por 50,000 correos; sin descuento por pago anual.
- **Amazon SES:** $0.10 USD por 1,000 correos enviados tras agotar el tier gratuito de los primeros 12 meses (o el crédito de bienvenida en cuentas nuevas); extremadamente barato a volumen alto, pero requiere salir del Sandbox de AWS.

## Configuración

### Variables de entorno (`.env`, con Resend)
```
MAIL_MAILER=resend
RESEND_KEY=re_xxxxxxxxxxxx
MAIL_FROM_ADDRESS=reservas@midominio.com
MAIL_FROM_NAME="Renta Casas"
```

### Registros DNS requeridos (agregar en Cloudflare)
```
TXT   @             "v=spf1 include:resend.io ~all"
TXT   resend._domainkey   <valor DKIM proporcionado por Resend>
TXT   _dmarc        "v=DMARC1; p=quarantine; rua=mailto:dmarc@midominio.com"
```

### Notification de Laravel (resumen)
```php
class BookingConfirmedNotification extends Notification
{
    public function via($notifiable) { return ['mail']; }

    public function toMail($notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Confirmación de tu reserva')
            ->view('emails.booking-confirmed', ['booking' => $this->booking]);
    }
}
```

Se despacha desde un Job en cola (`SendBookingConfirmationEmail`, ver `03-cache-colas-redis.md`) para no bloquear el request del pago.

Referenciado desde: `arquitectura-plataforma-renta.md`, secciones 1, 3 y 10.
