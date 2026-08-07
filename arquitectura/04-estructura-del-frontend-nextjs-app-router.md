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
│   │   │   ├── seasons-pricing/
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
│   │   └── AvailabilityCalendar.tsx
│   └── booking/
│       └── BookingForm.tsx
├── hooks/
│   ├── useAvailability.ts
│   └── useBooking.ts
├── services/
│   ├── apiClient.ts                  # fetch/axios con interceptores
│   ├── propertyService.ts
│   └── bookingService.ts
├── store/                            # Zustand
│   ├── authStore.ts
│   └── bookingStore.ts
├── context/
│   └── AuthContext.tsx
├── types/
│   ├── property.ts
│   └── booking.ts
└── styles/
    └── globals.css                   # Tailwind
```

**Decisiones:**
- **Server Components** para listado/detalle público (SEO, menos JS al cliente).
- **Client Components** solo donde hay interactividad (calendario, formulario de reserva).
- **Zustand** sobre Redux: menor boilerplate, suficiente para el estado del admin (filtros, carrito de reserva).
- `apiClient.ts` centraliza baseURL, manejo de tokens y refresh, y parseo de errores.

---

