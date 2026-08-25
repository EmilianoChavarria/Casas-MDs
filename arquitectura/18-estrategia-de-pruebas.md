# 18. Estrategia de pruebas

## 18.1 Principio: cobertura dirigida, no porcentaje global

**No se persigue un umbral de cobertura para todo el proyecto.** Un mínimo global empuja a escribir pruebas donde es fácil, no donde importa: se llega al 70% cubriendo *getters* y controladores CRUD mientras el motor de precios sigue con dos pruebas. La métrica queda verde y el riesgo intacto.

El criterio es otro: **se prueba a fondo donde el fallo es silencioso y caro.**

| Tipo de fallo | Cómo se descubre | ¿Lo detecta una prueba manual? |
|---|---|---|
| Precio mal calculado | Semanas después, cuadrando ingresos | ❌ No — el sistema no lanza ningún error |
| Doble reserva | Cuando el huésped llega a la casa | ❌ No — requiere concurrencia real |
| Webhook procesado dos veces | Cuadrando pagos contra el proveedor | ❌ No |
| Botón mal alineado | De inmediato, al abrir la página | ✅ Sí |
| Subida de imágenes rota | De inmediato | ✅ Sí |

Los tres primeros son exactamente lo que una prueba automatizada atrapa y una persona probando a mano no. Los dos últimos son lo contrario. La estrategia sigue esa división.

## 18.2 Qué bloquea un merge

**Obligatorio — sin estas pruebas en verde no se integra a `develop` ni a `main`:**

| Módulo | Qué se prueba |
|---|---|
| **`PricingService`** | Los siete casos de la tabla 15.9: estancia que cruza dos temporadas, temporada anual que envuelve el fin de año, traslape con la misma prioridad, precio congelado tras cambiar la temporada, redondeo que cuadra, noche vs. día de calendario, coherencia del `+20%` en las tres monedas |
| **Anti doble-booking** | Dos reservas **concurrentes** del mismo rango: exactamente una debe confirmarse. Se prueba con transacciones paralelas reales, no secuenciales |
| **Canje de cupón** | Dos reservas concurrentes con `usage_limit = 1`: solo una consume el cupón |
| **`ExpirePendingBookings`** | La carrera del webhook de pago llegando en el mismo instante que vence el plazo (sección 5.5), en ambas direcciones |
| **Webhooks de pago** | Idempotencia: el mismo evento entregado tres veces produce un solo pago. Los proveedores reintentan por diseño |
| **Vinculación de cuentas OAuth** | Las cinco reglas de la sección 7.1.1, en especial que un `email_verified = false` **no** vincule |
| **Cancelaciones** | Que el reembolso y la liberación de `availability` ocurran en la misma transacción, y que un fallo del reembolso no libere fechas |
| **Cupo de experiencias** (sección 20.3) | Dos reservas **concurrentes** por la última plaza: una confirma, la otra recibe `not_enough_seats` con el cupo real. Y que `seats_taken` quede correcto |
| **Aislamiento del guía** (20.4) | Un guía autenticado **no** obtiene salidas ni asistentes de otro guía, ni llamando la API directamente. Y que el roster no incluya totales ni método de pago |
| **Mínimo para operar** (20.5) | Salida bajo el mínimo → `cancelled`, reembolso del 100% a todos y plazas liberadas, todo en la misma transacción |
| **Token de reseña** (20.6) | Token caducado, token inexistente y token que ya alcanzó el tope por cupo: los tres rechazados. Y que un token no se emita antes de `completed` |

**Pruebas de humo — se escriben, pero no bloquean:** CRUD de propiedades y amenidades, listados con filtros, subida de imágenes, login con contraseña, endpoints de lectura del chat.

⚠️ **Riesgo aceptado a conciencia:** una regresión en el CRUD puede llegar a producción. Se asume porque se ve de inmediato y se corrige rápido — a diferencia de un precio mal calculado, que se descubre tarde y ya cobró mal a decenas de huéspedes.

## 18.3 Herramientas

| Qué | Con qué |
|---|---|
| Framework | **Pest** — viene por defecto en Laravel 12 |
| Base de datos de prueba | **MySQL 8 real**, no SQLite |
| Datos | Factories y seeders de Laravel |
| Concurrencia | Transacciones paralelas con `pcntl` o dos conexiones simultáneas |
| Frontend | Sin pruebas automatizadas en la versión inicial |

⚠️ **MySQL real y no SQLite, a propósito.** Toda la estrategia anti doble-booking depende de `SELECT ... FOR UPDATE` y de restricciones `UNIQUE` bajo concurrencia. SQLite no implementa el bloqueo a nivel de fila igual que InnoDB: una prueba de doble-booking pasaría en SQLite y el sistema fallaría en producción. Es el escenario más peligroso posible — una prueba verde que da confianza falsa. El `docker-compose` de la sección 8.2 ya levanta MySQL, así que no hay infraestructura nueva que montar.

**Frontend sin pruebas automatizadas** en esta fase: el riesgo ahí es visible y barato de corregir, y montar Playwright o Testing Library añadiría tiempo que rinde más en el backend. Se revisa si aparecen regresiones repetidas en el checkout.

## 18.4 Integración con CI

Se amplía el job `test` de la sección 9.3, que hoy solo ejecuta `php artisan test` sin base de datos:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8
        env:
          MYSQL_DATABASE: testing
          MYSQL_ROOT_PASSWORD: secret
        ports: ['3306:3306']
        options: >-
          --health-cmd="mysqladmin ping" --health-interval=10s
          --health-timeout=5s --health-retries=5
    steps:
      - uses: actions/checkout@v4
      - run: composer install --prefer-dist --no-interaction
      - run: cp .env.testing .env && php artisan key:generate
      - run: php artisan migrate --force
      - run: php artisan test --parallel
```

`main` y `develop` quedan protegidas: sin CI en verde no hay merge, tal como ya define la sección 9.1.

## 18.5 Qué NO se prueba

Declararlo evita discusiones después:

- **Código de terceros.** No se prueba que Stripe cobre ni que Reverb entregue mensajes. Sí se prueba **la integración propia**: que el webhook se procese bien, que el evento se emita al canal correcto.
- **Llamadas reales a servicios externos.** Stripe, Mercado Pago, DeepL y Google se simulan con *fakes*. Una suite que depende de la red es lenta e inestable, y gasta cuota.
- **La interfaz de administración**, más allá de humo. Es la superficie con más pantallas y menos riesgo silencioso.
- **Rendimiento y carga.** Queda fuera hasta que existan datos reales de tráfico (sección 11).

## 18.6 Costo

**~15–25 h repartidas por módulo**, no como partida aparte. Escribir las pruebas junto al código cuesta una fracción de escribirlas después: retroajustar pruebas a código ya hecho suele costar el doble o más, porque hay que entender de nuevo qué se pretendía y desenredar dependencias que nadie pensó para ser sustituidas.

Es la única partida del proyecto que se abarata haciéndola a tiempo y se encarece mucho posponiéndola. Por eso está repartida y no al final del cronograma.

---

Ver también: sección 9 (DevOps), sección 15.9 (casos del motor de precios), sección 5.5 (expiración).

---

