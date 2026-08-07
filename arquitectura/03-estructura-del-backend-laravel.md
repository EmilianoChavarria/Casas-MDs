# 3. Estructura del backend Laravel


```
app/
├── Console/
│   └── Commands/
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
│   ├── MessageSent.php               # ShouldBroadcast → presence-conversation.{id}
│   ├── MessageRead.php               # ShouldBroadcast
│   └── ConversationAssigned.php      # ShouldBroadcast → admin.inbox
├── Exceptions/
│   ├── PropertyNotAvailableException.php
│   ├── PaymentFailedException.php
│   ├── OverlappingSeasonException.php
│   ├── PromotionNotApplicableException.php
│   └── PromotionUsageLimitReachedException.php
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
│   │       │   ├── UserController.php
│   │       │   ├── CustomerController.php
│   │       │   └── ReportController.php
│   │       ├── Auth/
│   │       │   └── AuthController.php
│   │       ├── Chat/
│   │       │   ├── ConversationController.php
│   │       │   └── MessageController.php
│   │       └── Public/
│   │           ├── PropertySearchController.php
│   │           ├── AvailabilityController.php
│   │           ├── QuoteController.php
│   │           ├── PromotionValidationController.php
│   │           ├── BookingController.php
│   │           └── PaymentWebhookController.php
│   ├── Middleware/
│   │   ├── EnsureUserIsAdmin.php
│   │   ├── ResolveGuestConversationToken.php   # acceso de visitante sin cuenta
│   │   └── LogApiRequests.php
│   ├── Requests/
│   │   ├── Property/StorePropertyRequest.php
│   │   ├── Property/UpdatePropertyRequest.php
│   │   ├── Booking/StoreBookingRequest.php
│   │   ├── Booking/QuoteRequest.php
│   │   ├── Season/StoreSeasonRequest.php       # valida traslapes (sección 15.3)
│   │   ├── Promotion/StorePromotionRequest.php
│   │   └── Chat/StoreMessageRequest.php
│   └── Resources/
│       ├── PropertyResource.php
│       ├── BookingResource.php
│       ├── AvailabilityResource.php
│       ├── QuoteResource.php
│       ├── SeasonResource.php
│       ├── PromotionResource.php
│       ├── ConversationResource.php
│       └── MessageResource.php
├── Jobs/
│   ├── SendBookingConfirmationEmail.php
│   ├── ProcessPaymentWebhook.php
│   ├── RecalculatePriceCalendar.php       # materializa price_calendar
│   ├── NotifyUnreadMessage.php            # con delay; solo si el destinatario no está presente
│   ├── PurgeOldConversations.php
│   └── GenerateOccupancyReport.php
├── Listeners/
│   ├── NotifyAdminOnNewBooking.php
│   ├── ReleaseHoldOnPaymentFailed.php
│   ├── ReleasePromotionOnBookingCancelled.php   # decrementa used_count
│   ├── PostSystemMessageOnBookingConfirmed.php  # mensaje sender_type=system en el hilo
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
│   └── User.php
├── Notifications/
│   ├── BookingConfirmedNotification.php
│   ├── BookingCancelledNotification.php
│   └── NewChatMessageNotification.php
├── Policies/
│   ├── PropertyPolicy.php
│   ├── BookingPolicy.php
│   ├── PromotionPolicy.php
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

