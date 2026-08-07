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
| 9 | Autenticación: panel admin + **huéspedes con Google OAuth** (secciones 5.6 y 7.1.1) |
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

**Fases bloqueadas por dudas del cliente:** la 6 depende de **D1** y **D5**, la 8 de **D2**, la 9 de **D6**, y parte de la 5 de **D3** y **D4**. La fase 16 está bloqueada por **D8** (textos legales), que impide activar cobros reales y publicar la app de Google. Ver [`dudas-cliente.md`](../dudas-cliente.md).

---

### Puesta en marcha: arranque en limpio

**Decisión: el sistema arranca vacío.** No se construyen importadores. El cliente captura sus propiedades desde el panel de administración: descripciones, fotos, amenidades, coordenadas y precios.

⚠️ **La captura de contenido no está incluida en las horas de la sección 13.** Esa estimación cubre construir el sistema, no llenarlo. Cincuenta casas con descripciones y fotografías son varios días de trabajo de alguien — conviene planificarlo aparte y decir quién lo hace y cuándo.

**Reservas ya cobradas al momento de lanzar.** Si existen reservas futuras vendidas antes de la puesta en marcha y nadie las carga, **el sistema venderá esas noches como disponibles** — el calendario arrancaría mintiendo. Es el mismo tipo de fallo que evita la decisión de canal único (sección 5.3), por otra puerta.

**El mecanismo para cargarlas ya está previsto y no requiere trabajo adicional:**

- El alta manual de reservas desde el panel ya forma parte del alcance — es la razón por la que `customers.user_id` es nullable (sección 5.6).
- `bookings.source` distingue el origen.
- `payments.provider = 'externo'` registra un cobro hecho fuera del sistema.

Esa misma funcionalidad sirve después del lanzamiento, porque el cliente seguirá recibiendo reservas por teléfono y transferencia. No es una herramienta de migración de un solo uso.

**Antes de abrir el sitio al público:** cargar las reservas en curso y verificar que el calendario de cada casa refleja la ocupación real. Es un paso de la lista de verificación del lanzamiento, no algo que se descubra después.

---

