# 12. Roadmap por fases


| Fase | Contenido |
|---|---|
| 1 | Arquitectura y setup de repos/CI/CD |
| 2 | Base de datos (migraciones + seeders) |
| 3 | Backend core (propiedades, amenidades, disponibilidad) |
| 4 | **Motor de precios: temporadas, reglas por tipo de día, `PricingService` + quote** (sección 15) |
| 5 | Frontend público (listado, detalle, calendario con precios por noche) |
| 6 | Autenticación (admin + cliente) |
| 7 | Motor de reservas (anti doble-booking, estados, congelado de precios) |
| 8 | **Promociones y cupones** (canje con lock, reportes de descuento) |
| 9 | Pagos (Stripe/MP + webhooks) |
| 10 | Dashboard admin completo + reportes |
| 11 | **Chat en tiempo real (Reverb, canales, bandeja admin, notificaciones)** (sección 16) |
| 12 | Hardening de seguridad + backups |
| 13 | Producción, monitoreo (Sentry), lanzamiento |

**Por qué en ese orden:**

- El **motor de precios va antes que el frontend público y que reservas**: el calendario muestra precio por noche, y las reservas congelan ese desglose. Construirlo después obliga a rehacer ambos.
- Las **promociones van después de reservas**, no antes: un cupón se canjea dentro de la transacción de reserva; sin motor de reservas cerrado no hay dónde engancharlo.
- El **chat es independiente de todo lo demás** — no bloquea ni es bloqueado por ninguna otra fase. Puede adelantarse si hace falta demo temprana, o posponerse tras el lanzamiento sin afectar el resto. Es la fase con más margen de reprogramación.

---

