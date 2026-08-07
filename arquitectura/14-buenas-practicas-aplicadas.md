# 14. Buenas prácticas aplicadas


- **SOLID** en Services (una responsabilidad por servicio: `PricingService` no toca disponibilidad, `AvailabilityService` no calcula precios).
- **Repository Pattern** para desacoplar Eloquent de la lógica de negocio (facilita tests unitarios con mocks).
- **Service Layer** entre Controllers y Repositories — Controllers delgados.
- **DTOs** para pasar datos entre capas sin exponer el Model de Eloquent directamente a la lógica de negocio.
- **RESTful** estricto (recursos en plural, verbos HTTP correctos, códigos de estado consistentes).
- **PSR-12** vía `laravel/pint` en el pipeline de CI.
- **DRY/KISS/YAGNI:** no construir multi-tenant ni multi-idioma de propiedades hasta que exista un requerimiento real; empezar simple y extraer abstracciones solo cuando se repitan 3+ veces.
- **Clean Architecture parcial:** no es necesario un desacople extremo (hexagonal completo) para este tamaño de proyecto; el patrón Controller→Service→Repository ya da suficiente testabilidad y mantenibilidad sin sobre-ingeniería.
