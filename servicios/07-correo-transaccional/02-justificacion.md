# Justificación

- **Resend** tiene API moderna, buena entregabilidad (deliverability) desde el día uno y un tier gratuito generoso para arrancar; integración directa con Laravel vía el driver de Symfony Mailer.
- **Amazon SES** es más barato a gran volumen si ya se usa AWS/S3, pero requiere más configuración inicial (verificación de dominio, salir del "sandbox" solicitando aumento de límite a AWS Support).

**Recomendación:** iniciar con Resend por simplicidad; migrar a SES solo si el volumen de correos crece mucho (miles diarios) y el costo se vuelve relevante.

