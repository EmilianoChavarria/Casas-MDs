# 3. Estructura del backend Laravel


```
app/
├── Console/
│   └── Commands/
├── DTOs/
│   ├── PropertyDTO.php
│   ├── BookingDTO.php
│   └── AvailabilityRangeDTO.php
├── Events/
│   ├── BookingConfirmed.php
│   ├── BookingCancelled.php
│   └── PaymentReceived.php
├── Exceptions/
│   ├── PropertyNotAvailableException.php
│   └── PaymentFailedException.php
├── Http/
│   ├── Controllers/
│   │   └── Api/V1/
│   │       ├── Admin/
│   │       │   ├── PropertyController.php
│   │       │   ├── AmenityController.php
│   │       │   ├── SeasonController.php
│   │       │   ├── PricingController.php
│   │       │   ├── BookingAdminController.php
│   │       │   ├── UserController.php
│   │       │   ├── CustomerController.php
│   │       │   └── ReportController.php
│   │       ├── Auth/
│   │       │   └── AuthController.php
│   │       └── Public/
│   │           ├── PropertySearchController.php
│   │           ├── AvailabilityController.php
│   │           ├── BookingController.php
│   │           └── PaymentWebhookController.php
│   ├── Middleware/
│   │   ├── EnsureUserIsAdmin.php
│   │   └── LogApiRequests.php
│   ├── Requests/
│   │   ├── Property/StorePropertyRequest.php
│   │   ├── Property/UpdatePropertyRequest.php
│   │   └── Booking/StoreBookingRequest.php
│   └── Resources/
│       ├── PropertyResource.php
│       ├── BookingResource.php
│       └── AvailabilityResource.php
├── Jobs/
│   ├── SendBookingConfirmationEmail.php
│   ├── ProcessPaymentWebhook.php
│   └── GenerateOccupancyReport.php
├── Listeners/
│   ├── NotifyAdminOnNewBooking.php
│   └── ReleaseHoldOnPaymentFailed.php
├── Models/
│   ├── Property.php
│   ├── PropertyImage.php
│   ├── Amenity.php
│   ├── Booking.php
│   ├── Customer.php
│   ├── Season.php
│   ├── PriceRule.php
│   ├── Availability.php
│   ├── Payment.php
│   ├── AuditLog.php
│   └── User.php
├── Notifications/
│   ├── BookingConfirmedNotification.php
│   └── BookingCancelledNotification.php
├── Policies/
│   ├── PropertyPolicy.php
│   └── BookingPolicy.php
├── Repositories/
│   ├── Contracts/
│   │   ├── PropertyRepositoryInterface.php
│   │   └── BookingRepositoryInterface.php
│   └── Eloquent/
│       ├── PropertyRepository.php
│       └── BookingRepository.php
├── Services/
│   ├── PropertyService.php
│   ├── AvailabilityService.php
│   ├── PricingService.php
│   ├── BookingService.php
│   ├── PaymentService.php
│   └── ReportService.php
├── Traits/
│   ├── HasAuditColumns.php
│   └── ApiResponder.php
└── Helpers/
    └── DateRangeHelper.php
```

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

---

