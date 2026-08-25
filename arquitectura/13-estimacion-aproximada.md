# 13. Estimación aproximada

Estimación para **una persona a tiempo completo**. No incluye el trabajo del cliente (redactar descripciones, revisar traducciones, conseguir fotografías) ni tiempos de espera por decisiones pendientes.

Las filas marcadas ⏳ dependen de una duda abierta en [`dudas-cliente.md`](../dudas-cliente.md) y su rango puede moverse según la respuesta.

| Módulo | Horas | Complejidad | Prioridad | Riesgo principal |
|---|---|---|---|---|
| Setup arquitectura + CI/CD | 16–24 | Media | Alta | Config. de entornos |
| Modelo de datos + migraciones | 20–28 | Media | Alta | Cambios tardíos de esquema |
| CRUD propiedades/amenidades/imágenes | 30–40 | Media | Alta | Manejo de imágenes/S3 |
| Motor de disponibilidad | 16–20 | Media-Alta | Alta | Condiciones de carrera |
| **Motor de precios: temporadas + reglas + quote** | **30–40** | **Alta** | Alta | Traslape de temporadas, rangos que cruzan el año, precio congelado |
| Cargos e impuestos configurables ⏳ **D5** | 12–18 | Media-Alta | Alta | Tres cargas fiscales con bases distintas; el DSA no es porcentaje |
| Huéspedes extra + estancia larga ⏳ **D3 D4** | 6–10 | Baja-Media | Media | — |
| **Promociones y cupones** | **20–28** | Media-Alta | Media | Doble canje concurrente, dos ventanas de fechas (reserva vs. estancia) |
| **Multi-divisa (cobro real MXN/USD/CAD)** ⏳ **D1** | **20–30** | **Alta** | Alta | Congelar tipo de cambio, reembolsos a tasa distinta, conciliar en 3 monedas |
| Reservas (flujo completo) | 24–32 | Alta | Alta | Doble-booking, estados inconsistentes |
| Pagos + webhooks | 24–32 | **Alta** | Alta | Idempotencia de webhooks, reconciliación |
| **Multi-idioma ES/EN/FR + traducción asistida** ⏳ **D2** | **24–36** | **Alta** | Alta | SEO: `hreflang`, sitemap por idioma, riesgo de penalización por traducción sin revisar |
| Frontend público (SEO/SSR) | 40–50 | Media-Alta | Alta | Rendimiento de imágenes, Core Web Vitals |
| Sistema de diseño (tokens, a11y, skeletons, validación) | 8–12 | Baja-Media | Alta | Se paga una vez; omitirlo obliga a retocar cada pantalla después |
| Dashboard admin (incl. calendario de temporadas y promociones) | 48–60 | Media | Media | Volumen de pantallas; la UI de temporadas es más compleja que un CRUD |
| **Chat en tiempo real (Reverb + bandeja admin)** | **40–56** | **Alta** | Media | Infra WebSocket en producción (proxy, timeouts, Supervisor), reconexión y duplicados en el cliente |
| Auth y permisos del panel ⏳ **D6** | 12–16 | Baja-Media | Alta | — |
| **Auth de huéspedes + Google OAuth** | 12–16 | Media | Alta | Vinculación de cuentas por correo (*account takeover* si se omite `email_verified`) |
| **Reseñas y favoritos** (secciones 5.4 y 5.8) | 14–20 | Media | Media | Recálculo de `rating` desnormalizado; datos estructurados de SEO solo con reseñas reales |
| **Notificaciones al huésped** (sección 19) | 12–18 | Media | Alta | 7 plantillas × 3 idiomas; los textos dependen del cliente |
| Páginas legales + aceptación versionada ⏳ **D8** | 4–6 | Baja | **Alta** | Bloquea publicar la app de Google y activar cobros reales |
| Reportes | 12–20 | Media | Baja-Media | Consultas agregadas costosas; normalizar 3 monedas |
| Seguridad/hardening | 12–16 | Media | Alta | — |
| **Experiencias / tours guiados** (sección 20) ⏳ **D9 D10 D11 D12** | **176–240** | **Alta** | Media | Sobreventa de cupo, aislamiento de datos del guía, reseñas por link abierto |
| Despliegue producción | 12–16 | Media | Alta | Primer deploy real, subdominio WebSocket |
| **Total estimado** | **~644–884 h** | | | (**~16–22 semanas** a tiempo completo, una persona) |

> Sin el módulo de experiencias el total sigue siendo **~468–644 h (~12–16 semanas)**. Es el módulo más grande añadido hasta ahora: **+37 %** sobre el proyecto anterior. Su desglose interno está en la sección 20.12.

---

## 13.1 De dónde viene el crecimiento

La estimación original era de **264–352 h**. Casi todo el aumento son **requerimientos que aparecieron después**, no ampliación de alcance por decisión técnica:

| Origen | Horas añadidas | Quién lo pidió |
|---|---|---|
| Chat en tiempo real | +40–56 | Cliente |
| Multi-idioma ES/EN/FR | +24–36 | Cliente (huéspedes de MX, EE.UU. y Canadá) |
| Multi-divisa con cobro real | +20–30 | Cliente |
| Promociones y cupones | +20–28 | Cliente |
| Cargos e impuestos configurables | +12–18 | Obligación fiscal (IVA + ISH + DSA) |
| Auth de huéspedes + Google OAuth | +12–16 | Faltaba en el diseño original |
| Sistema de diseño | +8–12 | Faltaba en el diseño original |
| Huéspedes extra + estancia larga | +6–10 | Pendiente de confirmar |
| Ajustes de modelo de datos | +4–6 | Consecuencia de lo anterior |
| **Experiencias / tours guiados** | **+176–240** | **Cliente (producto nuevo)** |
| **Total añadido** | **+322–452 h** | |

**Experiencias es la partida más grande de todas, y es alcance nuevo declarado**: no es un módulo del sistema de casas, es un segundo producto —con su propio inventario, su propio checkout, su propio personal y sus propias reseñas— dentro de la misma plataforma. Conviene presentarlo así al cliente, porque "añadir tours" suena a una pantalla más y son casi seis semanas.

Dos partidas —auth de huéspedes y sistema de diseño— **no son alcance nuevo: eran huecos**. El diseño original solo contemplaba inicio de sesión de administradores, y el prototipo ya traía pantallas de registro y perfil de huésped que ninguna tabla soportaba.

⚠️ **El total puede moverse según las respuestas del cliente.** El caso más sensible es **D1**: si opta por capturar un precio por moneda a mano en vez de conversión automática, la partida de multi-divisa baja a ~8–12 h, pero el dashboard admin sube un rango similar — porque cada temporada, regla y promoción se captura por triplicado. El total apenas cambia; lo que cambia es dónde cae el trabajo y cuánta captura manual carga el cliente para siempre.

---

## 13.2 Desglose de los módulos grandes

**Chat en tiempo real (40–56 h)** — el más fácil de subestimar:

| Parte | Horas |
|---|---|
| Modelo de datos, API REST de conversaciones/mensajes, policies | 10–14 |
| Instalación y configuración de Reverb, canales, eventos, autorización | 8–10 |
| Frontend: widget público, ventana, composer, optimistic UI, reconexión | 14–20 |
| Bandeja de entrada del admin (asignación, filtros, no leídos) | 6–8 |
| Notificaciones por correo, adjuntos, purga programada | 4–6 |
| Infra de producción: subdominio, Nginx, Supervisor, Cloudflare | *incluido en despliegue* |

**Multi-idioma ES/EN/FR (24–36 h)** — el trabajo está más en el SEO que en la traducción:

| Parte | Horas |
|---|---|
| Esquema `property_translations` + integración DeepL por cola | 6–8 |
| Cola de revisión y aprobación en el panel admin | 6–8 |
| Rutas con prefijo de idioma, `hreflang` recíproco, sitemap por idioma | 8–12 |
| Traducción de la interfaz al francés (el prototipo trae el mecanismo) | 2–4 |
| Localización de correos transaccionales (3 idiomas × plantillas) | 2–4 |

**Multi-divisa (20–30 h)** — asumiendo conversión automática (D1 opción A o C):

| Parte | Horas |
|---|---|
| Tabla `exchange_rates` + job diario + proveedor de tipo de cambio | 4–6 |
| Congelado de la tasa en la reserva, columnas de moneda en `bookings` | 4–6 |
| Stripe multi-divisa + enrutar a Mercado Pago solo en MXN | 6–10 |
| Política de reembolso a tasa distinta | 2–4 |
| Normalizar reportes de ingresos a moneda base | 4–4 |

---

**Experiencias / tours guiados (176–240 h)** — desglose completo en la sección 20.12. Resumen:

| Parte | Horas |
|---|---|
| Modelo de datos, CRUD de experiencias y de guías | 34–46 |
| Motor de salidas (cupo con lock, estados, repetición, mínimo para operar) | 20–26 |
| Reserva + pagos + reembolso automático | 16–22 |
| Frontend público (listado, detalle, tarjeta sticky) | 22–28 |
| Admin (calendario de salidas + sección de guías) | 28–38 |
| Panel del guía | 14–18 |
| Reseñas con doble calificación y link externo | 12–16 |
| Notificaciones, impuestos por producto, reportes, pruebas | 30–46 |

---

## 13.3 Dependencias críticas

- El **motor de precios debe cerrarse antes** que el frontend público (el calendario muestra precio por noche) y antes que reservas (que congelan el desglose).
- **Disponibilidad → reservas → pagos**, en ese orden.
- **Promociones depende de reservas** (el canje ocurre dentro de su transacción).
- **Multi-divisa toca el motor de precios entero.** No es un módulo aparte que se pueda añadir al final: si se construye el motor en una sola moneda y después se convierte, se rehace el cálculo, el congelado y las promociones. Es la dependencia más cara de ignorar.
- **Multi-idioma toca las rutas del frontend público.** Añadir prefijos de idioma después obliga a redirigir URLs ya indexadas por Google, con pérdida temporal de posicionamiento.
- El **sistema de diseño va antes que checkout y formularios admin** — sin tokens de error no hay con qué pintar una validación.
- El **chat no depende de nada ni bloquea nada.**
- **Experiencias depende de pagos y de auth**, y toca `payments`: esa tabla debe nacer **polimórfica** desde las migraciones iniciales aunque el módulo se construya al final (sección 20.7). Es la única parte del módulo que no se puede posponer sin coste.

---

## 13.4 Qué se puede recortar si el calendario aprieta

No todo pesa lo mismo a la hora de posponer:

| Módulo | ¿Se puede diferir? |
|---|---|
| **Chat en tiempo real** (40–56 h) | ✅ **Sí, es el mejor candidato.** No bloquea nada ni lo bloquea nadie. Puede lanzarse después con un formulario de contacto por correo mientras tanto |
| **Promociones y cupones** (20–28 h) | ✅ Sí. El negocio opera sin descuentos; se añaden cuando haga falta |
| **Reportes** (12–20 h) | ✅ Sí, parcialmente. Ocupación e ingresos básicos primero, el resto después |
| **Multi-idioma** (24–36 h) | ⚠️ Solo si se dejan las rutas con prefijo desde el inicio. Diferir el contenido es barato; diferir la estructura de rutas es caro |
| **Multi-divisa** (20–30 h) | ❌ **No.** Retrofitarlo obliga a rehacer el motor de precios |
| **Cargos e impuestos** (12–18 h) | ❌ **No.** Un total sin IVA, ISH y DSA no es un fallo de software, es un problema fiscal |
| **Sistema de diseño** (8–12 h) | ❌ No conviene. Omitirlo obliga a retocar cada pantalla ya construida |
| **Auth de huéspedes** (12–16 h) | ⚠️ Solo si se acepta reservar como invitado, sin cuenta — pero eso elimina `/profile` del prototipo |
| **Experiencias** (176–240 h) | ✅ **Sí, entero.** Es un producto aparte: las casas se venden igual sin él. ⚠️ Con una condición — dejar `payments` polimórfico desde el inicio (20.7). Dentro del módulo hay recortes propios en 20.12 |

Difiriendo chat, promociones y parte de reportes se recortan **~70–100 h**. Difiriendo además el módulo de experiencias completo, el primer lanzamiento vuelve a **340–480 h (~9–12 semanas)** con todo lo indispensable de la renta de casas operando.

---

