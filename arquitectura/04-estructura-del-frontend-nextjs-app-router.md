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
│   │   │   └── reports/
│   │   └── layout.tsx                # Protegido por middleware
│   ├── api/                          # (opcional) BFF routes / proxy
│   ├── layout.tsx
│   └── middleware.ts                 # Protección de rutas admin
├── components/
│   ├── ui/                           # botones, inputs, modal (design system)
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
    └── globals.css                   # Tailwind
```

**Decisiones:**
- **Server Components** para listado/detalle público (SEO, menos JS al cliente).
- **Client Components** solo donde hay interactividad (calendario, formulario de reserva, **chat**).
- **Zustand** sobre Redux: menor boilerplate, suficiente para el estado del admin (filtros, carrito de reserva, estado del chat).
- `apiClient.ts` centraliza baseURL, manejo de tokens y refresh, y parseo de errores.
- **El frontend nunca calcula precios.** Muestra el `QuoteDTO` que devuelve el backend. Duplicar la lógica de temporadas en TypeScript garantiza que algún día el total mostrado difiera del cobrado.
- **Una sola instancia de Echo** para toda la app. Suscribirse en `useEffect` y **desuscribirse en el cleanup** (`echo.leave(...)`): en dev, StrictMode monta dos veces y sin cleanup los mensajes salen duplicados en pantalla.
- Si el WebSocket cae, `useConversation` degrada a polling cada 10 s y al reconectar pide `?after_id=<último visto>` — el WS no reenvía lo perdido.

Detalle del chat en la sección 16.6; del motor de precios, en la 15.

---

