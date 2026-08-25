# 3. Estructura del backend Laravel


```
app/
├── Console/
│   └── Commands/
│       └── ReconcileDepartureSeats.php   # experiences:reconcile-seats (20.3)
├── DTOs/
│   ├── PropertyDTO.php
│   ├── BookingDTO.php
│   ├── AvailabilityRangeDTO.php
│   ├── NightPriceDTO.php
│   ├── QuoteRequestDTO.php
│   └── QuoteDTO.php
├── Events/
│   ├── BookingConfirmed.php
│   ├── BookingCancelled.php
│   ├── PaymentReceived.php
│   ├── SeasonSaved.php               # dispara RecalculatePriceCalendar
│   ├── PriceRuleSaved.php
│   ├── DepartureConfirmed.php        # alcanzó el mínimo para operar
│   ├── DepartureCancelled.php        # dispara reembolsos y avisos
│   ├── GuideAssignedToDeparture.php
│   └── ExperienceReviewSubmitted.php
│   ├── MessageSent.php               # ShouldBroadcast → presence-conversation.{id}
│   ├── MessageRead.php               # ShouldBroadcast
│   └── ConversationAssigned.php      # ShouldBroadcast → admin.inbox
├── Exceptions/
│   ├── PropertyNotAvailableException.php
│   ├── PaymentFailedException.php
│   ├── OverlappingSeasonException.php
│   ├── PromotionNotApplicableException.php
│   ├── PromotionUsageLimitReachedException.php
│   ├── NotEnoughSeatsException.php          # lleva el cupo real disponible (20.3)
│   ├── DepartureNotBookableException.php
│   └── InvalidReviewTokenException.php
├── Http/
│   ├── Controllers/
│   │   └── Api/V1/
│   │       ├── Admin/
│   │       │   ├── PropertyController.php
│   │       │   ├── AmenityController.php
│   │       │   ├── SeasonController.php
│   │       │   ├── PriceRuleController.php
│   │       │   ├── HolidayController.php
│   │       │   ├── PricingController.php          # calendar, preview, recalculate
│   │       │   ├── PromotionController.php
│   │       │   ├── ConversationAdminController.php
│   │       │   ├── BookingAdminController.php
│   │       │   ├── ExperienceController.php           # sección 20
│   │       │   ├── DepartureController.php            # incl. creación en lote y cancelación
│   │       │   ├── GuideController.php                # alta, invitación, baja
│   │       │   ├── ExperienceReviewAdminController.php
│   │       │   ├── UserController.php
│   │       │   ├── CustomerController.php
│   │       │   └── ReportController.php
│   │       ├── Auth/
│   │       │   ├── AuthController.php          # register, login, logout, me
│   │       │   ├── PasswordResetController.php
│   │       │   └── SocialAuthController.php    # redirect, callback, link, unlink
│   │       ├── Chat/
│   │       │   ├── ConversationController.php
│   │       │   └── MessageController.php
│   │       ├── Guide/                                 # panel del guía (sección 20.8.C)
│   │       │   ├── GuideSummaryController.php
│   │       │   ├── GuideDepartureController.php       # SIEMPRE filtrado por guide_id
│   │       │   └── GuideReviewLinkController.php
│   │       └── Public/
│   │           ├── ExperienceSearchController.php     # listado + filtros por categoría
│   │           ├── ExperienceDepartureController.php  # salidas con cupo restante
│   │           ├── ExperienceBookingController.php    # reserva con lockForUpdate (20.3)
│   │           ├── ReviewTokenController.php          # GET/POST /r/{token}, sin sesión
│   │           ├── PropertySearchController.php
│   │           ├── AvailabilityController.php
│   │           ├── QuoteController.php
│   │           ├── PromotionValidationController.php
│   │           ├── BookingController.php
│   │           └── PaymentWebhookController.php
│   ├── Middleware/
│   │   ├── EnsureUserIsAdmin.php
│   │   ├── EnsureUserIsGuide.php               # panel del guía (sección 20.4)
│   │   ├── ResolveGuestConversationToken.php   # acceso de visitante sin cuenta
│   │   └── LogApiRequests.php
│   ├── Requests/
│   │   ├── Property/StorePropertyRequest.php
│   │   ├── Property/UpdatePropertyRequest.php
│   │   ├── Booking/StoreBookingRequest.php
│   │   ├── Booking/QuoteRequest.php
│   │   ├── Season/StoreSeasonRequest.php       # valida traslapes (sección 15.3)
│   │   ├── Promotion/StorePromotionRequest.php
│   │   ├── Experience/StoreDepartureRequest.php    # valida repetición y capacity >= seats_taken
│   │   ├── Experience/StoreExperienceBookingRequest.php
│   │   ├── Experience/StoreExperienceReviewRequest.php
│   │   └── Chat/StoreMessageRequest.php
│   └── Resources/
│       ├── PropertyResource.php
│       ├── BookingResource.php
│       ├── AvailabilityResource.php
│       ├── QuoteResource.php
│       ├── SeasonResource.php
│       ├── PromotionResource.php
│       ├── ConversationResource.php
│       ├── ExperienceResource.php
│       ├── DepartureResource.php          # expone seats_left, nunca capacity interna
│       ├── GuidePublicResource.php        # bio y métricas; sin correo ni teléfono
│       ├── GuideRosterResource.php        # asistentes SIN datos financieros (20.4)
│       └── MessageResource.php
├── Jobs/
│   ├── SendBookingConfirmationEmail.php
│   ├── ProcessPaymentWebhook.php
│   ├── RecalculatePriceCalendar.php       # materializa price_calendar
│   ├── NotifyUnreadMessage.php            # con delay; solo si el destinatario no está presente
│   ├── PurgeOldConversations.php
│   ├── EvaluateDepartureMinimum.php       # confirma o cancela por mínimo (20.5)
│   ├── ExpirePendingExperienceBookings.php
│   ├── IssueDepartureReviewToken.php      # al pasar a completed (20.6)
│   └── GenerateOccupancyReport.php
├── Listeners/
│   ├── NotifyAdminOnNewBooking.php
│   ├── ReleaseHoldOnPaymentFailed.php
│   ├── ReleasePromotionOnBookingCancelled.php   # decrementa used_count
│   ├── PostSystemMessageOnBookingConfirmed.php  # mensaje sender_type=system en el hilo
│   ├── RecalculateExperienceRating.php          # rating_experience → experiences.rating
│   ├── RecalculateGuideRating.php               # rating_guide → guides.rating
│   ├── ReleaseSeatsOnExperienceBookingCancelled.php
│   └── QueuePriceCalendarRecalculation.php
├── Models/
│   ├── Property.php
│   ├── PropertyImage.php
│   ├── Amenity.php
│   ├── Booking.php
│   ├── BookingNight.php
│   ├── Customer.php
│   ├── Season.php
│   ├── PriceRule.php
│   ├── Holiday.php
│   ├── PriceCalendarEntry.php
│   ├── Promotion.php
│   ├── PromotionRedemption.php
│   ├── Conversation.php
│   ├── Message.php
│   ├── Availability.php
│   ├── Payment.php
│   ├── AuditLog.php
│   ├── Role.php
│   ├── SocialAccount.php
│   ├── Experience.php
│   ├── ExperienceImage.php
│   ├── ExperienceItem.php
│   ├── ExperienceDeparture.php
│   ├── ExperienceBooking.php
│   ├── ExperienceAttendee.php          # notes_encrypted con cast 'encrypted' (20.9)
│   ├── ExperienceReview.php
│   ├── Guide.php
│   └── User.php
├── Notifications/
│   ├── BookingConfirmedNotification.php
│   ├── BookingCancelledNotification.php
│   ├── ExperienceBookingConfirmedNotification.php
│   ├── DepartureReminderNotification.php          # huésped, salida − 24 h
│   ├── DepartureCancelledNotification.php
│   ├── GuideAssignedNotification.php
│   ├── GuideRosterNotification.php                # guía, salida − 24 h
│   └── NewChatMessageNotification.php
├── Policies/
│   ├── PropertyPolicy.php
│   ├── BookingPolicy.php
│   ├── PromotionPolicy.php
│   ├── DeparturePolicy.php             # un guía solo ve sus salidas (20.4)
│   ├── ExperienceBookingPolicy.php
│   └── ConversationPolicy.php
├── Repositories/
│   ├── Contracts/
│   │   ├── PropertyRepositoryInterface.php
│   │   ├── BookingRepositoryInterface.php
│   │   ├── SeasonRepositoryInterface.php
│   │   ├── PromotionRepositoryInterface.php
│   │   └── ConversationRepositoryInterface.php
│   └── Eloquent/
│       ├── PropertyRepository.php
│       ├── BookingRepository.php
│       ├── SeasonRepository.php
│       ├── PromotionRepository.php
│       └── ConversationRepository.php
├── Services/
│   ├── PropertyService.php
│   ├── AvailabilityService.php
│   ├── Pricing/
│   │   ├── PricingService.php          # orquesta el pipeline de la sección 15.2
│   │   ├── SeasonResolver.php          # qué temporada cubre una noche (prioridad/traslape)
│   │   ├── DayTypeResolver.php         # weekday | weekend | holiday
│   │   ├── PromotionService.php        # elegibilidad, combinación y canje con lock
│   │   └── PriceCalendarBuilder.php
│   ├── Experiences/
│   │   ├── DepartureService.php        # creación en lote, estados, cancelación
│   │   ├── SeatAllocationService.php   # lockForUpdate + validación de cupo (20.3)
│   │   └── ReviewTokenService.php      # emisión, caducidad, tope por cupo (20.6)
│   ├── ChatService.php                 # persistir → broadcast → notificar
│   ├── BookingService.php
│   ├── PaymentService.php
│   └── ReportService.php
├── Traits/
│   ├── HasAuditColumns.php
│   └── ApiResponder.php
└── Helpers/
    └── DateRangeHelper.php
```

`routes/guide.php` agrupa el panel del guía bajo `auth:sanctum` + `EnsureUserIsGuide`. ⚠️ El middleware **no basta**: cada consulta filtra además por `guide_id` (sección 20.4), porque una policy no protege un listado.

`routes/channels.php` define la autorización de los canales de chat (`conversation.{id}`, `admin.inbox`) — ver sección 16.5.

**Patrón de capas:** Controller → Service → Repository → Model.
El Controller solo valida (FormRequest) y delega. El Service contiene reglas de negocio (ej. "no permitir reservar si hay traslape de fechas"). El Repository abstrae Eloquent para poder testear con mocks o cambiar de ORM si algún día fuera necesario.

Ejemplo de contrato:

```php
interface BookingRepositoryInterface
{
    public function hasOverlap(int $propertyId, Carbon $checkin, Carbon $checkout): bool;
    public function create(BookingDTO $dto): Booking;
}
```

**Separación de responsabilidades en `Pricing/`:** el pipeline de precios se subdivide porque cada paso tiene una razón de cambio distinta — `SeasonResolver` cambia si cambia la regla de desempate de temporadas, `DayTypeResolver` si cambia qué días son fin de semana, `PromotionService` si cambian las reglas de cupones. Meterlos en un solo `PricingService` de 400 líneas los volvería imposibles de testear por separado, y es justo el módulo marcado como de mayor riesgo en la sección 13.

**`ChatService`** encapsula el orden obligatorio `persistir → difundir → notificar` (sección 16.3). Ningún controller debe llamar a `broadcast()` directamente: si lo hace, tarde o temprano habrá un mensaje difundido que no se guardó.

---

