# 12. Roadmap por fases


| Fase | Contenido |
|---|---|
| 1 | Arquitectura y setup de repos/CI/CD |
| 2 | Base de datos (migraciones + seeders) |
| 3 | **Sistema de diseño**: tokens, `focus-visible`, z-index, skeletons, validación (sección 17.7) |
| 4 | Backend core (propiedades, amenidades, disponibilidad) + Places Autocomplete |
| 5 | **Motor de precios: temporadas, reglas por tipo de día, `PricingService` + quote** (sección 15) |
| 6 | **Cargos e impuestos + multi-divisa** — ambos dentro del motor de precios, no después |
| 7 | Frontend público (listado, detalle, calendario con precios por noche) + tiles de mapa |
| 8 | **Multi-idioma: rutas con prefijo, `hreflang`, traducción asistida** (secciones 4 y 17) |
| 9 | Autenticación: panel admin + **huéspedes con Google OAuth** (secciones 5.4 y 7.1.1) |
| 10 | Motor de reservas (anti doble-booking, estados, congelado de precios) |
| 11 | **Promociones y cupones** (canje con lock, reportes de descuento) |
| 12 | Pagos (Stripe/MP + webhooks, enrutado por moneda) |
| 13 | Dashboard admin completo + reportes |
| 14 | **Chat en tiempo real (Reverb, canales, bandeja admin, notificaciones)** (sección 16) |
| 15 | Hardening de seguridad + backups |
| 16 | Producción, monitoreo (Sentry), lanzamiento |

**Por qué en ese orden:**

- El **sistema de diseño va en la fase 3**, antes de construir pantallas. Sin tokens de error no hay con qué pintar una validación, y añadirlos después obliga a retocar cada formulario ya hecho.
- El **motor de precios va antes que el frontend público y que reservas**: el calendario muestra precio por noche, y las reservas congelan ese desglose. Construirlo después obliga a rehacer ambos.
- **Cargos, impuestos y multi-divisa van pegados al motor de precios (fase 6), no al final.** Es la dependencia más cara de ignorar: construir el motor en una sola moneda y sin impuestos, para añadirlos después, implica rehacer el cálculo, el congelado de reservas y las promociones.
- El **multi-idioma va justo tras el frontend público (fase 8)**, porque cambia la estructura de rutas. Añadir prefijos de idioma más tarde obliga a redirigir URLs ya indexadas por Google.
- Las **promociones van después de reservas**, no antes: un cupón se canjea dentro de la transacción de reserva; sin motor de reservas cerrado no hay dónde engancharlo.
- El **chat es independiente de todo lo demás** — no bloquea ni es bloqueado por ninguna otra fase. Puede adelantarse si hace falta demo temprana, o posponerse tras el lanzamiento sin afectar el resto. Es la fase con más margen de reprogramación (ver 13.4).

**Fases bloqueadas por dudas del cliente:** la 6 depende de **D1** y **D5**, la 8 de **D2**, la 9 de **D6**, y parte de la 5 de **D3** y **D4**. Ver [`dudas-cliente.md`](../dudas-cliente.md).

---

