# Contrato / plan recomendado

- Self-hosted: sin costo adicional.
- Upstash: free tier permanente de 256 MB y 500,000 comandos/mes (sin tarjeta); plan pago pay-as-you-go desde $0.20 USD por 100,000 comandos + $0.25 USD/GB-mes de storage por encima de 1GB, ancho de banda gratis hasta 200GB/mes — conveniente cuando el tráfico aún es variable. Planes fijos desde $10 USD/mes.
- Redis Cloud: plan gratuito 30MB, planes pagos desde ~$5 USD/mes.

**Recomendación:** mantener self-hosted hasta que el volumen de colas/cache justifique un servicio gestionado con alta disponibilidad.

