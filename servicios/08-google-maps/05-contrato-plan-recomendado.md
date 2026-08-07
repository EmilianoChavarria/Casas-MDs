# Contrato / plan recomendado

- Google Maps Platform usa el plan **Essentials** (pay-as-you-go) con 10,000 llamadas gratis **por cada API** al mes (Maps JavaScript, Places, Geocoding se cuentan cada una por separado; ya no comparten un crédito único de $200 USD como antes de marzo 2025).
- Costos aproximados una vez agotada la cuota gratis de cada API: Maps JavaScript API ~$7 USD/1,000 cargas; Places Autocomplete ~$2.83 USD/1,000 solicitudes (sesión); Geocoding ~$5 USD/1,000 solicitudes (verificar tarifas vigentes en https://mapsplatform.google.com/pricing/, Google las ajusta con frecuencia).
- Para un catálogo de propiedades de una sola empresa (no marketplace masivo), las 10,000 llamadas gratis mensuales de cada API normalmente cubren el uso completo en desarrollo y en el arranque.
- Configurar **alertas de presupuesto** en Google Cloud (Billing → Budgets & alerts) para evitar sorpresas.

