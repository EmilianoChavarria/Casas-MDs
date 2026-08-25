# 4. Estructura del frontend Next.js (App Router)


```
src/
├── app/
│   ├── (public)/
│   │   ├── page.tsx                  # Home
│   │   ├── properties/
│   │   │   ├── page.tsx              # Listado + filtros (Server Component)
│   │   │   └── [slug]/
│   │   │       ├── page.tsx          # Detalle (SSR/ISR)
│   │   │       └── BookingWidget.tsx # Client Component
│   │   ├── experiencias/             # sección 20
│   │   │   ├── page.tsx              # Listado + filtro por categoría (Server Component)
│   │   │   └── [slug]/
│   │   │       ├── page.tsx          # Detalle (SSR) — galería, guía, punto de encuentro
│   │   │       └── DepartureWidget.tsx  # Client: mini calendario + cupo + personas
│   │   ├── r/
│   │   │   └── [token]/page.tsx      # Captura de reseña — sin sesión, móvil, noindex
│   │   ├── legal/
│   │   │   ├── privacidad/page.tsx   # exigido por Google OAuth y LFPDPPP
│   │   │   ├── terminos/page.tsx     # exigido por Stripe
│   │   │   └── cookies/page.tsx
│   │   └── layout.tsx
│   ├── (admin)/
│   │   ├── dashboard/
│   │   │   ├── page.tsx
│   │   │   ├── properties/
│   │   │   ├── bookings/
│   │   │   ├── customers/
│   │   │   ├── seasons-pricing/      # temporadas + reglas por tipo de día
│   │   │   ├── promotions/           # alta/edición de promociones y cupones
│   │   │   ├── chat/
│   │   │   │   ├── page.tsx          # bandeja de conversaciones
│   │   │   │   └── [conversationId]/page.tsx
│   │   │   ├── experiences/          # sección 20
│   │   │   │   ├── page.tsx          # catálogo de experiencias
│   │   │   │   ├── departures/       # calendario de salidas + panel de la salida
│   │   │   │   └── guides/           # equipo de guías + detalle por guía
│   │   │   └── reports/
│   │   └── layout.tsx                # Protegido por middleware
│   ├── (guide)/                      # panel del guía — role = guide (sección 20.8.C)
│   │   ├── guia/
│   │   │   ├── page.tsx              # resumen de carga + próximos tours
│   │   │   ├── [departureId]/page.tsx   # roster + link de reseña
│   │   │   └── encuesta/page.tsx     # vista previa de la encuesta del huésped
│   │   └── layout.tsx                # Protegido por middleware (rol distinto de admin)
│   ├── api/                          # (opcional) BFF routes / proxy
│   ├── layout.tsx
│   └── middleware.ts                 # Protección de rutas admin
├── components/
│   ├── ui/                           # design system propio — tokens y componentes en sección 17
│   │   ├── Input.tsx                 # unificado (el prototipo lo duplica en 4 páginas)
│   │   ├── Field.tsx
│   │   ├── Row.tsx
│   │   ├── SectionHeading.tsx
│   │   ├── EmptyState.tsx
│   │   ├── StatusBadge.tsx
│   │   ├── StarRating.tsx
│   │   └── LanguageSelector.tsx
│   ├── property/
│   │   ├── PropertyCard.tsx
│   │   ├── PropertyGallery.tsx
│   │   └── AvailabilityCalendar.tsx  # muestra precio por noche + mínimo de noches
│   ├── booking/
│   │   ├── BookingForm.tsx
│   │   ├── PriceBreakdown.tsx        # renderiza el QuoteDTO tal cual (no recalcula)
│   │   └── PromoCodeInput.tsx
│   ├── pricing/                      # solo admin
│   │   ├── SeasonCalendar.tsx        # temporadas pintadas por color sobre el año
│   │   ├── SeasonForm.tsx
│   │   └── PricingPreview.tsx        # simulación antes de guardar
│   ├── experience/                   # sección 20
│   │   ├── ExperienceCard.tsx
│   │   ├── ExperienceGallery.tsx     # mosaico
│   │   ├── GuideProfile.tsx          # bio + métricas de confianza
│   │   ├── MeetingPointMap.tsx       # reutiliza SingleLocationMap (Leaflet, 17.9)
│   │   ├── DepartureCalendar.tsx     # mini calendario con cupo restante por fecha
│   │   ├── SeatSelector.tsx          # limitado al cupo de la fecha (20.3)
│   │   └── ReviewForm.tsx            # compartido por /r/[token] y la vista previa del guía
│   └── chat/
│       ├── ChatWidget.tsx            # burbuja flotante del sitio público
│       ├── ChatWindow.tsx
│       ├── MessageList.tsx           # scroll invertido + paginación keyset
│       ├── MessageBubble.tsx
│       ├── MessageComposer.tsx
│       └── TypingIndicator.tsx
├── hooks/
│   ├── useAvailability.ts
│   ├── useBooking.ts
│   ├── useQuote.ts                   # debounced; llama a POST /bookings/quote
│   ├── useEcho.ts                    # suscripción/limpieza de canales
│   ├── useConversation.ts            # historial + envío + optimistic UI
│   └── useUnreadCount.ts
├── services/
│   ├── apiClient.ts                  # fetch/axios con interceptores
│   ├── echoClient.ts                 # instancia ÚNICA de Laravel Echo (singleton)
│   ├── propertyService.ts
│   ├── pricingService.ts
│   ├── chatService.ts
│   └── bookingService.ts
├── store/                            # Zustand
│   ├── authStore.ts
│   ├── bookingStore.ts
│   └── chatStore.ts                  # conversación activa, no leídos, estado de conexión
├── context/
│   └── AuthContext.tsx
├── types/
│   ├── property.ts
│   ├── booking.ts
│   ├── pricing.ts                    # NightPrice, Quote, Season, Promotion
│   └── chat.ts
└── styles/
    └── globals.css                   # Tailwind + body + CSS de Leaflet
```

**Base visual:** los tokens (paleta `lagoon`/`coral`/`sand`, Poppins + Inter, sombras `card`/`pop`), el inventario de componentes y el checklist de migración desde el prototipo están en la **sección 17**. El `tailwind.config.js` del prototipo se adopta como contrato visual sin cambios.


**Decisiones:**
- **Server Components** para listado/detalle público (SEO, menos JS al cliente).
- **Client Components** solo donde hay interactividad (calendario, formulario de reserva, **chat**).
- **Zustand** sobre Redux: menor boilerplate, suficiente para el estado del admin (filtros, carrito de reserva, estado del chat).
- `apiClient.ts` centraliza baseURL, manejo de tokens y refresh, y parseo de errores.
- **El frontend nunca calcula precios.** Muestra el `QuoteDTO` que devuelve el backend. Duplicar la lógica de temporadas en TypeScript garantiza que algún día el total mostrado difiera del cobrado.
- **Fuentes con `next/font/google`**, nunca `@import` de Google Fonts en CSS: autoalojado, sin petición a terceros y sin salto de layout (sección 17.4).
- **Mapas con import dinámico y `ssr: false`** — Leaflet accede a `window` al importarse y rompe el render de servidor.
- **Una sola instancia de Echo** para toda la app. Suscribirse en `useEffect` y **desuscribirse en el cleanup** (`echo.leave(...)`): en dev, StrictMode monta dos veces y sin cleanup los mensajes salen duplicados en pantalla.
- Si el WebSocket cae, `useConversation` degrada a polling cada 10 s y al reconectar pide `?after_id=<último visto>` — el WS no reenvía lo perdido.

Detalle del chat en la sección 16.6; del motor de precios, en la 15.

---

