# Configuración


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
