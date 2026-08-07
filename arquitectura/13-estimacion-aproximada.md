# 13. Estimación aproximada


| Módulo | Horas | Complejidad | Prioridad | Riesgo principal |
|---|---|---|---|---|
| Setup arquitectura + CI/CD | 16–24 | Media | Alta | Config. de entornos |
| Modelo de datos + migraciones | 16–22 | Media | Alta | Cambios tardíos de esquema |
| CRUD propiedades/amenidades/imágenes | 30–40 | Media | Alta | Manejo de imágenes/S3 |
| Motor de disponibilidad | 16–20 | Media-Alta | Alta | Condiciones de carrera |
| **Motor de precios: temporadas + reglas + quote** | **30–40** | **Alta** | Alta | Traslape de temporadas, rangos que cruzan el año, precio congelado |
| **Promociones y cupones** | **20–28** | Media-Alta | Media | Doble canje concurrente, dos ventanas de fechas (reserva vs. estancia) |
| Reservas (flujo completo) | 24–32 | Alta | Alta | Doble-booking, estados inconsistentes |
| Pagos + webhooks | 24–32 | **Alta** | Alta | Idempotencia de webhooks, reconciliación |
| Frontend público (SEO/SSR) | 40–50 | Media-Alta | Alta | Rendimiento de imágenes, Core Web Vitals |
| Dashboard admin (incl. calendario de temporadas y promociones) | 48–60 | Media | Media | Volumen de pantallas; la UI de temporadas es más compleja que un CRUD |
| **Chat en tiempo real (Reverb + bandeja admin)** | **40–56** | **Alta** | Media | Infra WebSocket en producción (proxy, timeouts, Supervisor), reconexión y duplicados en el cliente |
| Auth y permisos | 12–16 | Baja-Media | Alta | — |
| Reportes | 12–20 | Media | Baja-Media | Consultas agregadas costosas |
| Seguridad/hardening | 12–16 | Media | Alta | — |
| Despliegue producción | 12–16 | Media | Alta | Primer deploy real |
| **Total estimado** | **~352–472 h** | | | (~9–12 semanas a tiempo completo, una persona) |

**Desglose del chat (40–56 h)** — es el módulo nuevo más grande y el más fácil de subestimar:

| Parte | Horas |
|---|---|
| Modelo de datos, API REST de conversaciones/mensajes, policies | 10–14 |
| Instalación y configuración de Reverb, canales, eventos, autorización | 8–10 |
| Frontend: widget público, ventana, composer, optimistic UI, reconexión | 14–20 |
| Bandeja de entrada del admin (asignación, filtros, no leídos) | 6–8 |
| Notificaciones por correo, adjuntos, purga programada | 4–6 |
| Infra de producción: subdominio, Nginx, Supervisor, Cloudflare | *incluido en despliegue* |

**Dependencias críticas:**

- El **motor de precios debe cerrarse antes** que el frontend público (el calendario muestra precio por noche) y antes que reservas (que congelan el desglose).
- **Disponibilidad → reservas → pagos**, en ese orden.
- **Promociones depende de reservas** (el canje ocurre dentro de su transacción).
- El **chat no depende de nada ni bloquea nada**. Es el candidato natural a paralelizar con otra persona, o a recortar/posponer si el calendario aprieta — a diferencia del motor de precios, que no admite recorte.

⚠️ El salto de ~264–352 h a ~352–472 h es de casi un 35%. Chat y promociones no son "extras pequeños": suman entre 60 y 84 horas.

---

