# 13. Estimación aproximada


| Módulo | Horas | Complejidad | Prioridad | Riesgo principal |
|---|---|---|---|---|
| Setup arquitectura + CI/CD | 16–24 | Media | Alta | Config. de entornos |
| Modelo de datos + migraciones | 12–16 | Media | Alta | Cambios tardíos de esquema |
| CRUD propiedades/amenidades/imágenes | 30–40 | Media | Alta | Manejo de imágenes/S3 |
| Motor de disponibilidad y precios | 30–40 | **Alta** | Alta | Condiciones de carrera, reglas de temporada |
| Reservas (flujo completo) | 24–32 | Alta | Alta | Doble-booking, estados inconsistentes |
| Pagos + webhooks | 24–32 | **Alta** | Alta | Idempotencia de webhooks, reconciliación |
| Frontend público (SEO/SSR) | 40–50 | Media-Alta | Alta | Rendimiento de imágenes, Core Web Vitals |
| Dashboard admin | 40–50 | Media | Media | Volumen de pantallas |
| Auth y permisos | 12–16 | Baja-Media | Alta | — |
| Reportes | 12–20 | Media | Baja-Media | Consultas agregadas costosas |
| Seguridad/hardening | 12–16 | Media | Alta | — |
| Despliegue producción | 12–16 | Media | Alta | Primer deploy real |
| **Total estimado** | **~264–352 h** | | | (~7–9 semanas a tiempo completo, una persona) |

Dependencias críticas: el motor de disponibilidad/precios debe cerrarse antes de construir reservas; pagos depende de reservas; dashboard admin puede avanzar en paralelo al frontend público una vez la API esté estable.

---

