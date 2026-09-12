# Dudas pendientes de resolver con el cliente

Decisiones que **no se pueden tomar desde el lado técnico** porque dependen del negocio, del presupuesto o de cómo quiera operar el cliente. Cada una incluye el contexto, las opciones reales y qué se bloquea mientras no se responda.

**Estado:** ⏳ pendiente · ✅ resuelta · ⏸️ aplazada

| # | Tema | Estado | Bloquea |
|---|---|---|---|
| D1 | Moneda y tipo de cambio para huéspedes de Canadá y EE.UU. | ✅ Resuelta | Motor de precios, checkout, reportes |
| D2 | Traducción del contenido: asistida, no automática | ✅ Resuelta | Alta de propiedades, SEO del sitio público |
| D3 | ¿Se cobra por huésped adicional? | ✅ Resuelta | Motor de precios, alta de propiedades |
| D4 | ¿Descuento por estancia larga? | ✅ Resuelta | Motor de precios, checkout |
| D5 | Base de cálculo de los impuestos (para el contador) | ✅ Resuelta | Cálculo del total, facturación |
| D6 | ¿Cuántos administradores y cómo entran al panel? | ✅ Resuelta | Autenticación del panel, alta de usuarios |
| D7 | Política de cancelación y reembolsos | ✅ Resuelta | Reservas, pagos, disponibilidad |
| D8 | Textos legales: aviso de privacidad y términos | ⏳ En curso | **Publicar app de Google, activar Stripe en producción** |
| D9 | ¿Los guías entran al sistema, y cómo? | ✅ Resuelta | Panel del guía, rol y autenticación (sección 20) |
| D10 | Impuestos de las experiencias: ¿IVA solo, o también ISH? | ⏳ Pendiente | Total de la experiencia, facturación |
| D11 | Link de reseña: ¿uno por grupo o uno por reserva? | ✅ Resuelta | Reseñas de experiencias, estrellas en Google |
| D12 | Cobro y cancelación de experiencias | ✅ Resuelta | Checkout de experiencias, reembolsos |
| D14 | Base de cálculo de la comisión del co-anfitrión | ⏳ Pendiente | Liquidación al dueño, reporte de su panel |

---

## Respuestas del cliente — 25 de agosto de 2026

Diez dudas resueltas, una en curso y una pendiente. Cada sección conserva
su contexto y sus opciones originales; aquí queda lo que se decidió y lo
que se asumió al aplicarlo.

| # | Decisión |
|---|---|
| **D1** | **El administrador captura el precio en la moneda que elija** (MXN, USD o CAD) y puede cambiar entre ellas. **El huésped siempre ve conversión automática.** |
| **D2** | **Traducción automática sin revisión.** ⚠️ Ver la advertencia de SEO más abajo. |
| **D3** | **Sí: cargo por huésped adicional.** |
| **D4** | **Sí, con umbral configurable por el administrador** — él decide a partir de cuántas noches aplica y cuánto. |
| **D5** | **IVA e ISH sobre el subtotal; DSA como cuota por noche.** |
| **D6** | **El administrador da de alta al personal con su correo.** Si esa persona entra con Google y el correo coincide, se vincula a la cuenta que ya existe. Sin correo dado de alta, no hay acceso. |
| **D7** | **Configuración general + política individual por casa** que la sobrescribe. |
| **D8** | ⏳ Un abogado los está preparando. Se avanza con textos de relleno. |
| **D9** | **Sí: los guías entran con cuenta propia y panel propio.** |
| **D10** | ⏳ Pendiente. Se avanza con **solo IVA** y perfil fiscal separado del de las casas. |
| **D11** | **Los dos:** link por reserva (verificado) y link de grupo como respaldo. |
| **D12** | **Cobro completo al reservar.** |

### Supuestos aplicados

Lo que las respuestas no precisaban y se resolvió con el criterio más
conservador. **Si alguno no es lo que el cliente quiso, decirlo ahora sale
mucho más barato que después de escribir el motor de precios.**

| Sobre | Supuesto |
|---|---|
| D1 | La moneda base para reportes y conciliación es **MXN**. Un precio capturado en USD se guarda en USD con su moneda, y se convierte para mostrar y para reportar. La tasa se congela al confirmar la reserva. |
| D3 | El cargo por persona extra es **por noche**, y se define por propiedad: número de huéspedes incluidos + monto por persona adicional. |
| D4 | Los descuentos por duración son **varios escalones** (por ejemplo 7 y 28 noches), configurables globalmente y sobrescribibles por propiedad — mismo patrón que D7. |
| D7 | Las políticas se guardan como **plantillas reutilizables**; la propiedad apunta a una y, si no apunta a ninguna, hereda la general. |
| D11 | El `AggregateRating` de Google se emite **solo a partir de las reseñas verificadas** (las que llegaron por link de reserva). Las del link de grupo se muestran en el sitio pero no alimentan los datos estructurados. |
| D12 | Se aceptan OXXO y SPEI **solo si la salida es posterior al vencimiento de la referencia**; si no, se ofrece únicamente tarjeta. La ventana para decidir si una salida opera es de **24 h** salvo indicación contraria. |

### ⚠️ D2 — Riesgo asumido a conciencia

Se eligió traducción **automática sin revisión**. El cliente decide, y así
queda documentado, pero conviene que sepa exactamente qué se está
aceptando:

Google trata el contenido traducido a máquina y publicado sin revisión
como **spam** en sus políticas. La sanción no es sutil: puede afectar al
posicionamiento de las páginas de propiedades en inglés y francés, que son
justamente las que se construyen para captar a los huéspedes de EE.UU. y
Canadá. Es el mismo motivo por el que las reseñas no se traducen (5.4).

**Mitigación que no cuesta horas de desarrollo:** el esquema guarda
`status` por traducción (`machine` / `reviewed`). Si más adelante alguien
revisa aunque sea las diez casas más visitadas, se marcan como revisadas
sin tocar código. Recomendado antes de invertir en anuncios o SEO.

> **D9 a D12 pertenecen al módulo de experiencias** ([`arquitectura/20-experiencias-tours-guiados.md`](arquitectura/20-experiencias-tours-guiados.md)).

---

## D1 — ¿Cómo se fija el precio en dólares canadienses y estadounidenses?

**Estado:** ✅ Resuelta (25-ago-2026) — captura en la moneda que elija el admin; el huésped ve conversión automática

### Contexto

El cliente pidió cobrar en varias monedas (MXN, CAD, USD), no solo mostrar un precio estimado. Eso significa que al huésped canadiense se le cobra realmente en dólares canadienses, no en pesos.

Para que eso funcione hay que definir **de dónde sale el precio en cada moneda**. Hay dos formas y el cliente tiene que elegir, porque cambian su trabajo diario.

### La pregunta concreta

> ¿Prefiere capturar **un solo precio en pesos** y que el sistema calcule automáticamente el equivalente en dólares, o prefiere **capturar los tres precios a mano** para tener control exacto de cada uno?

### Opciones

**Opción A — Un solo precio en pesos, conversión automática**

El administrador captura $2,500 MXN por noche. El sistema consulta el tipo de cambio una vez al día y muestra el equivalente en CAD y USD. Al confirmar una reserva, la tasa se congela para que el total no cambie después.

- ✅ El administrador captura una sola vez, igual que con las descripciones.
- ✅ Funciona igual para temporadas altas, fines de semana y promociones, sin trabajo extra.
- ⚠️ El precio en dólares **cambia solo, día a día**. Un huésped que vio $338 CAD ayer puede ver $344 hoy sin que nadie haya tocado nada. No es un error, es el tipo de cambio moviéndose.

**Opción B — Un precio por moneda, capturado a mano**

El administrador captura $2,500 MXN, $185 CAD y $135 USD por separado.

- ✅ Control total. Precios redondos y estables en cada moneda.
- ✅ Sin riesgo de que una fluctuación cambiaria deje un precio mal.
- ⚠️ Triplica la captura. Y no es solo el precio base: **cada temporada, cada regla de fin de semana y cada promoción** habría que capturarla tres veces. Con varias temporadas activas, el volumen de captura hace muy probable que algún precio quede mal puesto.

**Opción C — Precio en pesos con conversión y un margen de holgura**

Como la Opción A, pero aplicando un pequeño colchón sobre el tipo de cambio del mercado y redondeando a la decena, para absorber la fluctuación diaria y los cargos de conversión del procesador de pagos.

- ✅ Precios más estables, sin sorpresas por movimientos del tipo de cambio.
- ✅ Cubre los costos de procesar un pago en moneda extranjera.
- ⚠️ Un huésped que compare con una calculadora de divisas verá que el tipo de cambio aplicado no es exactamente el del mercado. Es práctica normal en la industria, pero **no conviene anunciar "tipo de cambio del día"** si se aplica un margen.

### ⚠️ Aclaración importante sobre las comisiones

En la conversación se mencionó un 3%. **Ese número era un ejemplo de colchón, no una comisión real de Stripe.** Los datos que sí están documentados:

- Comisión estándar de Stripe en México: **3.6% + $3 MXN por transacción** con tarjeta.
- Los pagos con **tarjetas internacionales** y las **conversiones de moneda** suelen llevar cargos adicionales sobre esa tarifa base.

**Antes de comprometer un número con el cliente hay que confirmar las tarifas vigentes de tarjeta internacional y conversión de divisa directamente en el panel de Stripe**, porque varían por país y cambian con el tiempo. Ver [`servicios/05-pagos-stripe.md`](servicios/05-pagos-stripe.md).

### Dato adicional que el cliente debe conocer

**Si el huésped paga en dólares, el pago se procesa obligatoriamente por Stripe.** Mercado Pago opera esencialmente en pesos mexicanos. Es decir: la moneda que elija el huésped determina qué procesador cobra, y por tanto qué comisión se paga. Los pagos en pesos pueden seguir yendo por Mercado Pago, que suele ser más conveniente en México.

### Otras dos cosas que hay que definir con el cliente

1. **Reembolsos.** Si un huésped pagó $340 CAD y cancela dos meses después, el tipo de cambio ya se movió. ¿Se le devuelven los $340 CAD que pagó, o el equivalente en pesos al día de la cancelación? Son cantidades distintas y hay que fijar la política por escrito.
2. **Reportes.** Los ingresos llegarán en tres monedas. Los reportes de la sección 13 los convertirán todos a pesos para poder compararlos. Conviene confirmar que es lo que el cliente espera ver.

### Qué se bloquea mientras no se responda

El motor de precios (sección 15), el checkout y los reportes de ingresos. Se puede avanzar en el resto del sistema, pero **el motor de precios no se debe cerrar sin esta decisión** — retrofitar multi-moneda después obliga a rehacer el cálculo de precios, el congelado de reservas y las promociones.

---

## D2 — La traducción será asistida, no automática. ¿Lo aprueba el cliente?

**Estado:** ✅ Resuelta (25-ago-2026) — ⚠️ traducción automática **sin** revisión, riesgo de SEO asumido por el cliente

### Contexto

El cliente pidió que el administrador **no** tenga que escribir el nombre y la descripción de cada casa tres veces (español, inglés, francés). Debe capturar una sola vez y que el sistema traduzca.

Eso es viable y prácticamente gratis: el servicio elegido es **DeepL API**, cuyo plan gratuito permanente cubre 500,000 caracteres al mes. Con 50 propiedades, todo el catálogo consume unos 160,000 caracteres **una sola vez**, y solo se vuelve a traducir cuando se edita una casa. Costo estimado: **$0**.

**Pero hay una condición que el cliente tiene que aceptar, y no es técnica: es de posicionamiento en Google.**

### El problema

Google clasifica como **spam** el texto *"traducido por una herramienta automática sin revisión humana antes de publicarlo"*. Está escrito literalmente en sus políticas de spam para búsqueda web. Las consecuencias van desde pérdida de posicionamiento hasta la desindexación del sitio.

Esto importa especialmente aquí porque **el posicionamiento en Google es la razón principal por la que el sitio público se construye con renderizado en servidor**. Es el objetivo central del frontend. Publicar traducción automática sin revisar atacaría justo lo que se está intentando conseguir.

### La solución propuesta: traducción **asistida**

La diferencia es importante y conviene explicarla bien al cliente, porque suena a más trabajo del que realmente es:

> **Google no exige que un humano *escriba* la traducción. Exige que un humano la *revise*.**

Flujo propuesto:

1. El administrador **escribe una sola vez, en español**. ✅ El requerimiento del cliente se respeta íntegro.
2. Al guardar, el sistema traduce automáticamente a inglés y francés en segundo plano.
3. Aparece en el panel una **cola de revisión**: "3 traducciones pendientes de aprobar".
4. El administrador abre la traducción, la lee, corrige si algo suena raro y pulsa **Aprobar**. Alrededor de **un minuto por casa y por idioma**.
5. **Solo las traducciones aprobadas se publican e indexan** en Google. Las no aprobadas muestran el texto en español mientras tanto.

### Ejemplo concreto

El administrador escribe en español:

> *"Casa frente al mar con alberca privada. A 5 minutos caminando de la playa. Cuenta con 4 recámaras, aire acondicionado y estacionamiento techado para dos autos."*

El sistema genera automáticamente:

> **EN:** *"Beachfront house with a private pool. A 5-minute walk from the beach. Features 4 bedrooms, air conditioning and covered parking for two cars."*
>
> **FR:** *"Maison en bord de mer avec piscine privée. À 5 minutes à pied de la plage. Elle dispose de 4 chambres, de la climatisation et d'un parking couvert pour deux voitures."*

El administrador las lee, confirma que dicen lo correcto y aprueba. **No escribió nada en inglés ni en francés.**

### Por qué el riesgo real es bajo en este caso

Las descripciones de casas de renta son mayormente **factuales**: número de recámaras, distancia a la playa, si hay alberca, si aceptan mascotas. Ese tipo de texto se traduce muy bien de forma automática y tiene mucho menos riesgo que contenido editorial o de marketing elaborado. DeepL además es especialmente sólido traduciendo al francés, que es el idioma más delicado de los tres.

### La pregunta concreta

> ¿Acepta el cliente que el administrador dedique **aproximadamente un minuto por casa y por idioma** a revisar y aprobar las traducciones generadas por el sistema, en lugar de publicarlas directamente sin revisar?

Y una segunda, derivada:

> Si el cliente **no** quiere ni siquiera revisar: ¿prefiere que las casas sin traducción aprobada se muestren **en español** dentro del sitio en inglés/francés, o que **no aparezcan** en esos idiomas?
>
> - Mostrar en español: el inventario siempre es visible, pero la página mezcla idiomas.
> - No aparecer: cada versión del sitio se ve impecable, pero **hay inventario que ciertos visitantes nunca verán** — y es fácil que nadie se dé cuenta de que está pasando.

### Nota sobre las amenidades

Las amenidades (alberca, WiFi, aire acondicionado…) son un catálogo fijo de unos 30 términos. **No se traducen automáticamente**: se traducen a mano una sola vez al configurar el sistema y no vuelven a tocarse. No suponen trabajo para el administrador.

### Qué se bloquea mientras no se responda

El formulario de alta de propiedades y la estrategia de indexación del sitio público. El desarrollo puede avanzar asumiendo el flujo de revisión propuesto, pero **si el cliente lo rechaza hay que rediseñar** cómo se decide qué contenido se indexa en cada idioma.

---

## D3 — ¿El precio cambia según cuántas personas se hospeden?

**Estado:** ✅ Resuelta (25-ago-2026) — sí, cargo por huésped adicional por noche

### Contexto

Cada casa tiene una capacidad máxima registrada (por ejemplo, 8 personas). Lo que **no** está definido es si el precio es el mismo para 2 huéspedes que para 8.

Esto no se había contemplado y hay que preguntarlo, porque cambia cómo se captura el precio de cada casa y cómo se cotiza una reserva.

### La pregunta concreta

> Si una casa admite hasta 8 personas y la reservan solo 2, ¿pagan lo mismo que si fueran 8? ¿O quiere cobrar un extra por cada persona adicional a partir de cierto número?

### Opciones

**Opción A — Precio plano hasta la capacidad**

La casa cuesta lo mismo la ocupen 2 o 10 personas, hasta el máximo permitido.

- ✅ Más simple de comunicar. El huésped ve un precio y es ese.
- ✅ Menos captura para el administrador.
- ⚠️ Los grupos grandes salen subsidiados por los pequeños. Ocho personas consumen bastante más agua, luz y gas, desgastan más la casa y dejan más trabajo de limpieza. Con precio plano, todo eso lo absorbe el negocio.
- ⚠️ Se renuncia a una fuente de ingreso que la mayoría de la competencia sí usa.

**Opción B — Precio base hasta N personas, más un cargo por persona adicional**

Por ejemplo: *"hasta 6 personas $3,000 por noche; cada persona adicional $300 por noche"*. Es el estándar de la industria y el huésped ya está acostumbrado a verlo.

- ✅ El precio refleja el consumo real.
- ✅ Palanca de ingreso adicional sin subir la tarifa base, que es lo que el huésped compara al buscar.
- ⚠️ El número de huéspedes lo declara el propio cliente al reservar. Si llegan más de los declarados, cobrar la diferencia depende de que alguien lo verifique en la casa. **Es control operativo, no del sistema.** Conviene definir qué se hace en ese caso.

**Opción C — Precios escalonados por rangos**

Por ejemplo: 1–4 personas $2,500; 5–6 personas $3,000; 7–8 personas $3,400.

- ✅ Control fino, permite precios redondos en cada tramo.
- ⚠️ Multiplica la captura del administrador — y no solo una vez: cada temporada podría querer tramos distintos. Es el mismo problema de volumen de captura que ya se descartó al decidir la moneda.

### Recomendación técnica

**Opción B.** Es el estándar del sector, se implementa con dos datos por casa (*hasta cuántas personas entra en el precio* y *cuánto cuesta cada persona extra por noche*) y se integra en el motor de precios sin complicarlo.

### Qué se bloquea mientras no se responda

El formulario de alta de propiedades y el cálculo de la cotización. Se puede empezar asumiendo la Opción B y desactivarla poniendo el cargo extra en cero si el cliente prefiere la A — pero **la Opción C sí requiere rediseño**, así que conviene descartarla o confirmarla pronto.

---

## D4 — ¿Habrá descuento por estancias largas?

**Estado:** ✅ Resuelta (25-ago-2026) — sí, con umbral y porcentaje configurables por el admin

### Contexto

Es práctica habitual en renta vacacional ofrecer un descuento cuando el huésped se queda una semana o un mes. Airbnb lo tiene estandarizado y los huéspedes lo buscan activamente al comparar.

No está definido si el cliente lo quiere.

### La pregunta concreta

> ¿Quiere ofrecer un descuento automático a quien se quede una semana o más? ¿Y otro mayor para estancias de un mes o más? Si es así, ¿de qué porcentaje en cada caso?

### Por qué conviene considerarlo

Las estancias largas son **más rentables por unidad de esfuerzo**, aunque el precio por noche sea menor:

- Una sola limpieza en vez de tres o cuatro.
- Una sola gestión de entrada y salida.
- **Cero días vacíos entre reservas**, que es donde de verdad se pierde dinero.

Una casa ocupada 21 noches seguidas con 10% de descuento suele dejar más que tres estancias de 5 noches con dos huecos de por medio.

Además, en destinos como Tulum el segmento de estancia prolongada (trabajo remoto, temporadas de invierno) crece de forma sostenida.

### Opciones

**Opción A — Dos descuentos fijos: semanal y mensual**

A partir de 7 noches un porcentaje, a partir de 28 noches otro mayor. Es el modelo que usa Airbnb.

- ✅ El huésped ya lo conoce y puede comparar directamente.
- ✅ Simple de configurar: dos porcentajes por casa.
- ⚠️ Los umbrales quedan en 7 y 28. Si más adelante se quiere "a partir de 5 noches", hay que modificar el sistema (cambio menor, pero cambio).

**Opción B — Tramos configurables a medida**

El administrador define los umbrales que quiera: 3 noches −5%, 7 noches −12%, 30 noches −25%.

- ✅ Flexibilidad total, ajustable por casa.
- ⚠️ Añade una pantalla de administración más y reglas para resolver qué tramo aplica cuando dos coinciden. Flexibilidad que puede no usarse nunca.

**Opción C — Sin descuento por estancia larga**

El precio por noche no cambia con la duración.

- ✅ Nada que configurar ni explicar.
- ⚠️ Se queda fuera del segmento de estancia prolongada, y el sitio se ve peor al comparar contra alojamientos que sí lo ofrecen.

### Dato importante para el cliente

**El descuento por estancia larga no es lo mismo que un cupón de promoción, y se pueden acumular.** El descuento largo se aplica primero, sobre el precio de las noches; un cupón promocional se aplica después, sobre el resultado. Si el cliente **no** quiere que se acumulen, hay que decirlo — es una regla que se configura, pero debe decidirse.

### Recomendación técnica

**Opción A**, con los porcentajes que decida el cliente (referencia habitual del sector: 10% semanal, 20% mensual). Si más adelante hace falta más flexibilidad, migrar a la Opción B es un cambio acotado.

### Qué se bloquea mientras no se responda

El cálculo de la cotización y el desglose que ve el huésped en el checkout. Se puede construir dejando los porcentajes en cero (equivalente a la Opción C) y activarlos cuando el cliente responda.

---

## D5 — Base de cálculo de los impuestos (pregunta para el contador)

**Estado:** ✅ Resuelta (25-ago-2026) — IVA e ISH sobre el subtotal; DSA como cuota por noche

### Contexto

En Quintana Roo aplican tres cargas fiscales sobre el hospedaje, y **cada una se calcula de forma distinta**:

| Carga | Tipo | Referencia |
|---|---|---|
| IVA | 16%, federal | Porcentaje |
| ISH (Impuesto Sobre Hospedaje) | 3% en Q. Roo, estatal | Porcentaje |
| DSA (Derecho de Saneamiento Ambiental) | Monto fijo por noche | ~$83 MXN en Cancún, ~$36 MXN en el resto del estado |

⚠️ **Estas cifras deben confirmarse: cambian por ejercicio fiscal y varían por estado y municipio.** No deben tomarse como definitivas desde el lado técnico.

El sistema está diseñado para soportar impuestos configurables de ambos tipos (porcentaje y monto fijo por noche), así que las tasas se pueden ajustar sin tocar código. Lo que **sí** necesita definirse antes de programar el cálculo es **sobre qué base se aplica cada uno**.

### Las preguntas concretas (para el contador del cliente)

> 1. **¿El ISH se calcula solo sobre el importe de las noches, o también sobre el cargo de limpieza?** El total cambia según la respuesta.
> 2. **¿El IVA se aplica sobre la misma base que el ISH, o sobre el importe más el ISH ya sumado?** Es decir, ¿hay impuesto sobre impuesto?
> 3. **El DSA se cobra "por habitación por noche". En una casa completa, ¿cuenta como una unidad o como el número de recámaras?** En una casa de 4 recámaras y 4 noches, la diferencia es entre $144 y $576.
> 4. **¿Aplica algún impuesto o derecho municipal adicional** en los municipios donde están las casas?
> 5. **¿El cliente factura como persona física o moral, y bajo qué régimen?** Determina cómo se emiten los CFDI de las reservas.
> 6. La retención de impuestos por plataformas digitales aplica a intermediarios tipo Airbnb. **Como este es el sitio propio del cliente y no un marketplace, se entiende que él es el contribuyente directo — ¿es correcto?**
>
> 7. **¿Cuántas facturas (CFDI) se emiten al mes hoy, y quién las emite?** El sistema **no** timbrará facturas en su versión inicial (ver sección 5.7): el huésped que la necesite lo marca al reservar y se emite por fuera. Esa decisión se tomó asumiendo que la mayoría de huéspedes serán turistas extranjeros sin RFC. **Si el volumen real de facturas es alto, hay que revisarla antes del lanzamiento**, porque emitirlas a mano es trabajo que crece con las ventas.

### Ejemplo para ilustrar la duda

Casa en Tulum, 4 noches, precio de noches $15,990 y limpieza $1,200:

```
Noches                                        $ 15,990.00
Limpieza                                      $  1,200.00
                                              ───────────
Base gravable  (¿incluye limpieza? ← duda 1)  $ 17,190.00

IVA 16%                                       $  2,750.40
ISH 3%                                        $    515.70
DSA ($36 × 4 noches ← ¿o × recámaras? duda 3) $    144.00
                                              ───────────
TOTAL                                         $ 20,600.10

Ingreso real del negocio:  $17,190.00
En tránsito al fisco:       $3,410.10
```

### Qué se bloquea mientras no se responda

El cálculo del total en el checkout y la emisión de facturas. **El sistema se puede construir con las tasas configurables y una base por omisión** (impuestos sobre noches + limpieza, DSA por casa), y corregirla cuando el contador responda — es cambiar configuración, no código. Pero **debe confirmarse antes de salir a producción**, porque un total mal calculado es un problema fiscal, no un error de software.

---

## D6 — ¿Cuántas personas van a administrar el sistema, y cómo entran?

**Estado:** ✅ Resuelta (25-ago-2026) — alta por correo desde el panel; Google solo vincula si el correo ya existe

### Contexto

El diseño actual **asume que habrá más de un administrador**: contempla una pantalla para dar de alta y gestionar usuarios del panel, y un sistema de roles. Pero eso nunca se confirmó con el cliente.

Si en realidad solo va a haber una persona administrando, sobra maquinaria. Y si van a ser varias, hay que definir quién puede hacer qué — no es lo mismo alguien que solo consulta reservas que alguien que puede cambiar precios o dar de baja propiedades.

Por otro lado, se va a integrar "Continuar con Google" para que los huéspedes se registren sin llenar formularios. Queda pendiente si el personal usará también ese método o mantendrá usuario y contraseña.

### Las preguntas concretas

> 1. **¿Cuántas personas van a usar el panel de administración?** ¿Solo el dueño, o también recepción, personal de limpieza, un contador?
> 2. **¿Todas hacen lo mismo, o hay que limitar qué puede hacer cada una?** Por ejemplo: que recepción vea y confirme reservas pero no pueda modificar precios ni dar de baja casas.
> 3. **¿Cómo prefiere que entren al panel: con usuario y contraseña, o con su cuenta de Google?**
> 4. Si es con Google: **¿usarán correos personales de Gmail, o el negocio tiene (o quiere tener) correos con dominio propio?**

### Por qué importa la pregunta 3

**Opción A — Usuario y contraseña**

- ✅ No depende de ningún servicio externo. Funciona aunque Google tenga una caída.
- ✅ No hace falta que el personal tenga cuenta de Google.
- ⚠️ Hay que gestionar contraseñas del personal, que son **el activo más sensible del sistema**: quien entra al panel ve todas las reservas, los datos personales de todos los huéspedes y puede modificar precios. Una contraseña débil o reutilizada del personal es bastante más peligrosa que la de cualquier huésped.

**Opción B — Entrar con Google**

- ✅ Se hereda la seguridad de Google sin construir nada: verificación en dos pasos, detección de accesos desde ubicaciones raras, alertas.
- ✅ El sistema deja de almacenar contraseñas del personal.
- ⚠️ **Si el personal usa cuentas personales de Gmail**, el acceso al panel queda atado a una cuenta que el negocio no controla. Cuando alguien deje de trabajar ahí, hay que acordarse de revocar su acceso manualmente — y si nadie se acuerda, sigue entrando.
- ✅ Con correos de dominio propio (Google Workspace) ese problema desaparece: se desactiva la cuenta y el acceso muere con ella. Pero Google Workspace tiene un costo mensual por usuario.

### Recomendación técnica

Si el negocio ya tiene o piensa tener correos con dominio propio, **Opción B restringida a ese dominio** es la más segura y la que menos mantenimiento pide.

Si el personal va a usar Gmail personal, es preferible la **Opción A**, acompañada de una regla operativa clara de dar de baja al usuario cuando alguien sale del equipo.

### Qué se bloquea mientras no se responda

La autenticación del panel de administración. **El sistema de huéspedes con Google no depende de esto** y puede construirse ya. Si más adelante el panel también usa Google, se reaprovecha casi todo el trabajo — el rehacer sería acotado, pero conviene evitarlo.

---

## D7 — Política de cancelación y reembolsos

**Estado:** ✅ Resuelta (25-ago-2026) — política general + individual por casa que la sobrescribe

### Contexto

El diseño del sitio ya contempla una sección de "política de cancelación" en la página de cada casa. **Pero no está definido qué dice esa política**, y sin eso no se puede programar qué ocurre cuando alguien cancela.

Cancelar no es solo marcar una reserva como cancelada. Implica devolver dinero (total o parcialmente), liberar las fechas para que otro pueda reservarlas, y decidir qué pasa con el cargo de limpieza, los impuestos y el cupón de descuento si lo usó. Cada una de esas piezas necesita una respuesta.

### Las preguntas concretas

> 1. **¿Todas las casas tendrán la misma política, o quiere poder ser más estricto en las más solicitadas?** Por ejemplo, condiciones más duras para cancelar una casa en Navidad que en temporada baja.
>
> 2. **¿Cuántos días antes de la llegada se puede cancelar, y qué porcentaje se devuelve en cada caso?** Ejemplo de referencia del sector: 100% si cancela con 7 días o más de anticipación, 50% entre 2 y 6 días, 0% con menos de 2 días.
>
> 3. **¿El huésped puede cancelar por sí mismo desde su perfil, o tiene que solicitarlo y usted lo autoriza?**
>
> 4. **¿Se devuelve el cargo de limpieza?** (No hubo limpieza que pagar, así que lo normal es devolverlo íntegro aunque el resto sea parcial.)
>
> 5. **¿Qué pasa si es usted quien cancela** porque la casa tuvo un desperfecto o se inundó? Lo habitual es devolver el 100% sin importar la fecha, y a veces compensar de alguna forma.
>
> 6. **¿Y si el huésped simplemente no llega** y nunca avisó? Suele tratarse distinto de una cancelación.

### ⚠️ Un costo que probablemente no está contemplado

**La comisión del procesador de pagos no se recupera al reembolsar.**

Si un huésped pagó $20,600 MXN y se le devuelve el 100%, Stripe retiene su comisión (aprox. **3.6% + $3 MXN ≈ $745 MXN**). Ese dinero **no vuelve al negocio**: se perdió en el momento del cobro.

Es decir: **cada cancelación con reembolso total cuesta dinero real**, aunque parezca una operación neutra.

> 7. **¿Prefiere absorber esa comisión, o descontarla del reembolso al huésped?**

Si decide descontarla, **tiene que estar escrito en la política antes de que nadie reserve**. Reembolsar menos de lo que el huésped espera, sin haberlo advertido, es una disputa de cargo casi segura — y una disputa cuesta más que la comisión.

### Sobre los impuestos

Los impuestos cobrados (IVA, ISH, DSA) corresponden a un hospedaje que no ocurrió. Lo razonable es devolverlos siempre e íntegros, incluso cuando el reembolso del alojamiento sea parcial — pero conviene **confirmarlo con el contador** junto con las preguntas de D5.

### Un punto importante sobre cambiar la política después

La política se **congela en el momento de reservar**. Si un huésped reservó bajo "cancelación gratuita hasta 7 días antes" y meses después la política se endurece, **ese huésped conserva las condiciones que aceptó**.

Es deliberado y no es negociable desde lo técnico: cambiar la política retroactivamente equivaldría a modificar un acuerdo ya cerrado, y es fuente directa de disputas. La política nueva aplica solo a las reservas hechas a partir de ese momento.

### Recomendación técnica

**Políticas predefinidas asignables por casa** (modelo tipo Airbnb): se definen dos o tres políticas nombradas —flexible, moderada, estricta— y a cada casa se le asigna una. Si el cliente solo quiere una, se asigna la misma a todas y no estorba; si más adelante quiere diferenciar, ya está listo.

**Valor por omisión para no bloquear el desarrollo:** política única "moderada" (100% con 7+ días, 50% entre 2 y 6, 0% con menos de 2), limpieza e impuestos reembolsables siempre, comisión absorbida por el negocio. Se ajusta con configuración cuando el cliente responda.

### Qué se bloquea mientras no se responda

El flujo de cancelación completo: pantalla del huésped, ejecución del reembolso contra Stripe o Mercado Pago, y liberación de fechas. **También obliga a una tabla nueva de reembolsos** — la de pagos actual no puede registrar devoluciones parciales, porque un reembolso no es "el pago cambió de estado", es un movimiento aparte con su propia referencia en el procesador.

El desarrollo puede avanzar con el valor por omisión de arriba, pero **no debe salir a producción sin la política confirmada por escrito**: es lo que el huésped acepta al reservar.

---

## D8 — Textos legales: aviso de privacidad, términos y condiciones

**Estado:** ⏳ En curso (un abogado los prepara) · ⚠️ **Bloquea el lanzamiento**

### Por qué esto no puede dejarse para el final

No es un trámite opcional. **Tres cosas que ya están decididas no se pueden activar sin estos documentos:**

| Bloqueo | Consecuencia si falta |
|---|---|
| **Publicar la app de Google** (para "Continuar con Google") | Google exige una política de privacidad accesible. Sin publicarla, la app queda con un **tope de 100 usuarios** |
| **Activar Stripe y Mercado Pago en producción** | Ambos exigen términos y condiciones y política de reembolso visibles antes de permitir cobros reales |
| **Cumplimiento de la LFPDPPP** | El aviso de privacidad es obligatorio para tratar datos personales de huéspedes |

El desarrollo construye las páginas, las traduce a los tres idiomas y registra qué versión aceptó cada huésped. **El contenido tiene que aportarlo el cliente o su abogado** — es responsabilidad legal de quien opera el negocio.

### La pregunta concreta

> ¿Tiene ya un aviso de privacidad y unos términos y condiciones? Si no, ¿cuenta con un abogado que los prepare? **Conviene pedírselos ahora, no la semana del lanzamiento**, porque sin ellos no se pueden activar los cobros reales ni el acceso con Google.

### Qué deben cubrir (checklist derivada de lo que el sistema hace)

Esta lista sale de las decisiones ya tomadas. Entregársela al abogado le ahorra tiempo y evita omisiones:

**Aviso de privacidad — datos que se recogen y por qué:**

- Nombre, correo, teléfono, documento de identidad y país del huésped.
- **Inicio de sesión con Google:** se recibe nombre, correo verificado, foto e idioma. Implica que Google sabe que la persona usa este sitio.
- **Chat con la administración:** los mensajes se conservan **12 meses** si no hubo reserva, y **5 años** si la conversación está vinculada a una reserva.
- **Traducción automática:** las descripciones de las propiedades se procesan con DeepL, un servicio externo. (Las reseñas y los mensajes de chat **no** se envían a traducir.)
- **Procesadores de pago:** Stripe y Mercado Pago reciben los datos necesarios para cobrar. El sistema **no almacena números de tarjeta**.
- **Imágenes y archivos** almacenados en Cloudflare R2.
- **Correos automáticos** que se envían y cómo darse de baja de la solicitud de reseña.
- ⚠️ **Datos de salud de los tours** (sección 20.9): restricciones alimentarias, alergias, "no sabe nadar", movilidad. Son **datos personales sensibles** bajo la LFPDPPP y requieren consentimiento expreso. Hay que decir para qué se usan (seguridad del tour), quién los ve (el guía asignado) y cuánto se conservan.
- Derechos ARCO: cómo ejercerlos y a qué correo.

**Términos y condiciones:**

- **La política de cancelación completa** (depende de **D7**). Si se decide descontar la comisión del procesador del reembolso, **tiene que estar escrito aquí antes de que nadie reserve** — reembolsar menos de lo esperado sin advertirlo es una disputa asegurada.
- Qué ocurre si el huésped no se presenta.
- Qué ocurre si cancela el anfitrión.
- Número máximo de huéspedes y consecuencias de excederlo (relacionado con **D3**).
- Horarios de entrada y salida.
- Reglas de la casa: mascotas, fiestas, fumar.
- Moneda de cobro y, si aplica, tipo de cambio (depende de **D1**).
- Impuestos incluidos en el precio (depende de **D5** y, para los tours, de **D10**).
- **Experiencias / tours** (sección 20): política de cancelación propia (**D12**), qué pasa si la salida se cancela por no alcanzar el mínimo de personas —reembolso íntegro— y a qué hora y dónde es el punto de encuentro.

### Detalle importante sobre el versionado

Los documentos se **versionan por fecha** y cada reserva guarda **qué versión aceptó el huésped**. Es el mismo principio que congelar el precio y la política de cancelación: ante una disputa hay que poder demostrar qué condiciones estaban vigentes cuando esa persona reservó, no las de hoy.

Esto significa que **cambiar los términos más adelante es seguro** — no afecta retroactivamente a quien ya reservó.

### Qué se bloquea mientras no se responda

El lanzamiento a producción, en tres frentes a la vez: cobros reales, acceso con Google más allá de 100 usuarios, y cumplimiento de protección de datos. El desarrollo puede avanzar por completo con textos de relleno, pero **no se puede salir a producción sin los definitivos**.

---

## D9 — ¿Los guías van a entrar al sistema, y cómo inician sesión?

**Estado:** ✅ Resuelta (25-ago-2026) — sí, con panel propio · Amplía **D6**

### Contexto

El módulo de experiencias (sección 20) contempla un **panel propio para el guía**: ve sus tours asignados, la lista de asistentes con sus notas y el link de reseña para compartir con el grupo.

Eso implica que el guía es **un usuario más del sistema**, con rol propio. Pero no todos los clientes quieren eso: si los guías son externos y rotan, dar de alta cuentas puede ser más carga administrativa que beneficio.

El diseño ya contempla las dos formas — `guides.user_id` es opcional — pero hay que saber cuál se usa desde el principio, porque cambia si el panel del guía se construye o no.

### La pregunta concreta

> ¿Quiere que cada guía entre al sistema con su propia cuenta y vea sus tours desde el celular, o prefiere que el administrador le mande la lista de asistentes por WhatsApp o correo antes de cada salida?
>
> Y si entran: ¿con contraseña, con un enlace mágico al correo, o con su cuenta de Google?

### Opciones

**Opción A — Los guías entran al panel (lo que muestra el prototipo)**

- ✅ El guía tiene la lista actualizada siempre, sin depender de que alguien se la mande.
- ✅ Copia y comparte el link de reseña él mismo al terminar el tour, que es cuando más responde la gente.
- ⚠️ Hay que dar de alta, dar de baja y soportar cuentas de personal que quizá rote.
- ⚠️ Aumenta la superficie de seguridad: un guía es un usuario con acceso a datos de huéspedes.

**Opción B — Sin panel: el admin manda el roster**

- ✅ Cero gestión de cuentas. Ahorra **14–18 h** de desarrollo.
- ⚠️ Alguien tiene que acordarse de mandar la lista antes de cada salida, cada día.
- ⚠️ El link de reseña lo comparte el admin, no el guía — y llega más tarde y peor.

**Recomendación:** con 2–3 guías fijos, la opción B es defendible al arrancar y el panel se añade después. Con guías que rotan o más de cinco, la opción A se paga sola en el primer mes.

### Qué se bloquea mientras no se responda

El sprint S1 del módulo (rol, autenticación e invitación de guías) y la decisión de construir o no el panel del guía completo.

---

## D10 — Impuestos de las experiencias: ¿solo IVA, o también ISH?

**Estado:** ⏳ Pendiente · ⚠️ **Pregunta para el contador** · Amplía **D5**

### Contexto

Las casas llevan tres cargas: **IVA**, **ISH** (Impuesto Sobre Hospedaje) y **DSA**. Un tour guiado **no es hospedaje**, así que en principio:

| Cargo | Casas | Experiencias |
|---|---|---|
| IVA 16% | Aplica | **Aplica** |
| ISH | Aplica | ❌ En principio **no** |
| DSA / cuota por noche | Aplica | ❌ No |

Pero la tasa de ISH y su base son **estatales**, y algunas entidades gravan servicios turísticos conexos. Esto no se puede decidir desde el lado técnico.

### La pregunta concreta

> Un tour guiado que se vende aparte de la casa, ¿lleva solo IVA, o el estado también grava ese servicio? ¿Y se factura con la misma clave que el hospedaje o con una distinta?

### Por qué importa más de lo que parece

Si se aplica ISH a un tour por copiar la configuración de las casas, se está **cobrando de más al huésped y declarando mal**. Si se omite un impuesto que sí aplica, el faltante lo pone el negocio.

Técnicamente obliga a que el motor de cargos tenga **perfiles fiscales por tipo de producto**, no una configuración global. Es un cambio pequeño si se hace al construir el motor (fase 6) y una refactorización si se hace después.

### Qué se bloquea mientras no se responda

El desglose del total de la experiencia (sprint S3) y su facturación. El desarrollo puede avanzar con "solo IVA" como valor por omisión, pero **no debe salir a producción sin confirmarlo**.

---

## D11 — El link de reseña: ¿uno por grupo o uno por persona?

**Estado:** ✅ Resuelta (25-ago-2026) — los dos: link por reserva (verificado) y link de grupo como respaldo

### Contexto

El diseño pide que el guía comparta **un link con todo el grupo** al terminar el tour. Es cómodo y funciona.

El problema: **ese link no verifica nada**. Cualquiera con la URL puede dejar una reseña — un competidor, un conocido, o el propio guía queriendo subir su calificación. Las reseñas de las casas no tienen ese hueco: solo puede reseñar quien completó una estancia pagada (sección 5.4).

### La pregunta concreta

> Las reseñas de los tours, ¿pueden dejarlas todas las personas del grupo con un mismo link que comparte el guía, o prefiere que a cada quien le llegue su propio link a su correo, para garantizar que solo reseñe quien realmente fue?

### Opciones

**Opción A — Un link por salida (lo que pide el prototipo)**

- ✅ El guía lo comparte en el grupo de WhatsApp en dos segundos, delante de todos.
- ✅ Máxima tasa de respuesta: se pide en caliente.
- ⚠️ **No son reseñas verificadas.** Se defiende con caducidad, tope por cupo y límite por IP, pero el hueco existe.
- ⚠️ **No se pueden emitir estrellas en los resultados de Google** (`AggregateRating`). Publicar datos estructurados de calificaciones sin respaldo verificable es motivo de acción manual de Google — el mismo aviso de la sección 5.4.

**Opción B — Un link por reserva, enviado por correo**

- ✅ Reseña verificada, igual que las casas.
- ✅ Habilita las estrellas en Google.
- ⚠️ Llega por correo, no en caliente: responde menos gente.
- ⚠️ +4–6 h de desarrollo.

**Recomendación:** empezar con **A** (el diseño ya lo pide, y al arrancar el volumen de reseñas es bajo) y dejar **B** preparado en el esquema para migrar sin romper nada. Lo que **no** hay que hacer es usar A y emitir estrellas en Google.

### Qué se bloquea mientras no se responda

El sprint S5 y la decisión de emitir o no datos estructurados de calificación en las páginas de experiencias.

---

## D12 — Cobro y cancelación de las experiencias

**Estado:** ✅ Resuelta (25-ago-2026) — cobro completo al reservar · Amplía **D7**

### Contexto

Una experiencia se cobra por adelantado, igual que una casa, pero tiene dos diferencias que cambian las reglas:

1. **Puede cancelarla el operador**, no solo el huésped: si una salida no alcanza el mínimo de personas para operar, se cancela.
2. **El ticket es mucho más pequeño.** Un tour de $800 MXN paga proporcionalmente **más comisión de procesador** que una reserva de $10,000, porque la parte fija (~$3 MXN + porcentaje) pesa mucho más.

### Las preguntas concretas

> **1.** ¿Se cobra el tour completo al reservar, o solo un anticipo y el resto en el punto de encuentro?
>
> **2.** ¿Con cuánta anticipación puede cancelar un huésped y recibir su dinero de vuelta? (Práctica estándar del sector: reembolso íntegro hasta 24 h antes; sin reembolso después.)
>
> **3.** ¿Cuántas horas antes se decide si una salida opera o se cancela por falta de gente? (Sugerido: 24 h.)
>
> **4.** ¿Se aceptan pagos en OXXO/SPEI para tours? Una referencia de OXXO tarda hasta ~3 días en pagarse; para un tour que sale el sábado, eso puede no llegar a tiempo.

### Lo que ya está decidido y no se pregunta

⚠️ **Si cancela el operador, el reembolso es del 100 %, sin descontar comisiones.** No es negociable: cancela el negocio, no el cliente. Descontar la comisión de Stripe de un reembolso que el huésped no provocó es una disputa de tarjeta asegurada — y en una disputa, quien cancela pierde.

### Qué se bloquea mientras no se responda

El sprint S3 (checkout de experiencias) y el texto de la política de cancelación que se muestra en la tarjeta de reserva. La respuesta a la pregunta 2 debe entrar en los **términos y condiciones** de **D8**.

---

## D14 — ¿Sobre qué importe se calcula la comisión del co-anfitrión?

**Estado:** ⏳ Pendiente — se avanza con **solo alojamiento** · Relacionada con **D5** y **D7**

### Contexto

Un dueño externo publica su casa en el sistema y queda como **co-anfitrión**: el administrador mantiene el control de qué se publica y a qué precio, y el dueño ve sus reservas, su calendario y lo que le corresponde. De cada reserva, un porcentaje se queda el administrador como comisión.

El porcentaje se pacta con cada dueño y ya se captura al asignarle la casa. Lo que nadie ha decidido es **sobre qué importe se aplica**, y no es un detalle de redacción: una reserva típica reparte su total en cuatro cosas distintas —alojamiento, huésped adicional, cargos de servicio e impuestos— y no todas son dinero del mismo dueño.

⚠️ **Esto se pregunta ahora y no después por una razón concreta.** La comisión **se congela en cada reserva**, igual que el desglose de precios y la política de cancelación: si mañana se renegocia el porcentaje, las reservas ya cobradas siguen diciendo lo que decían. Eso es deliberado —lo contrario reescribiría liquidaciones ya pagadas— pero tiene una consecuencia: **las reservas creadas antes de esta respuesta se quedan para siempre con el supuesto que está aplicado hoy**. Cambiar la regla después no arregla el pasado, solo cambia el futuro.

### La pregunta concreta

> **1.** De una reserva, ¿sobre qué parte se cobra la comisión: solo el alojamiento, el alojamiento más los cargos de servicio, o el total incluyendo impuestos?
>
> **2.** Cuando una reserva se cancela y por la política de cancelación se retiene una parte del dinero, ¿el dueño cobra su parte de lo retenido, o se lo queda todo el administrador?

### Opciones para la pregunta 1

Sobre una reserva de ejemplo: alojamiento **$10,000**, huésped adicional **$1,000**, cargo de limpieza **$800**, impuestos **$1,900** (IVA e ISH sobre el subtotal, según D5). Total al huésped: **$13,700**. Comisión pactada: **15%**.

| Opción | Base | Comisión | Le queda al dueño |
|---|---|---|---|
| **A. Solo alojamiento** *(la aplicada hoy)* | $11,000 | $1,650 | $9,350 |
| **B. Alojamiento + cargos** | $11,800 | $1,770 | $9,230 |
| **C. Total, impuestos incluidos** | $13,700 | $2,055 | $8,945 |

**A — solo alojamiento.** Es lo que está aplicado y lo más habitual en el sector. El dueño paga comisión sobre lo que le renta su casa, que es lo que el administrador le está vendiendo.

**B — alojamiento más cargos.** Defendible si el cargo de limpieza es un servicio que el dueño presta y cobra. ⚠️ **Si la limpieza la opera el administrador —que es lo normal aquí— esta opción le cobra comisión al dueño sobre un dinero que el administrador ya se está quedando entero.** Cobraría dos veces por lo mismo.

**C — total con impuestos.** ⚠️ **No se recomienda.** El IVA y el ISH son dinero en tránsito al SAT: no se lo queda el dueño ni el administrador. Cobrar comisión sobre ellos significa que el dueño paga por recaudar un impuesto que entrega íntegro. Además ata la comisión a la tasa fiscal: si el ISH sube, la comisión sube sola sin que nadie lo haya pactado.

**Lo que no cambia entre las tres opciones:** la base va **neta de descuentos**. Si una promoción rebaja la reserva un 20%, la comisión se calcula sobre lo que realmente entró, no sobre el precio de tarifa. Lo contrario haría que la promoción la pagara entera el administrador de su bolsillo, sin que nadie lo decidiera.

**Lo que ya está decidido y no se pregunta:** la **comisión del procesador de pagos la absorbe el administrador**. El dueño ve una cuenta limpia, sin descuentos de Stripe que no entiende ni puede verificar.

### Opciones para la pregunta 2 — la cancelación

Hoy **no hay nada implementado**: si una reserva se cancela, sus importes de comisión se quedan como estaban al reservar y nadie los ajusta. Es un pendiente conocido, no una decisión.

Sobre el ejemplo anterior cancelado con un 50% de retención según la política (D7) — se retienen **$6,850**:

| Opción | Qué pasa | Comisión | Le queda al dueño |
|---|---|---|---|
| **A. Proporcional** *(recomendada)* | La comisión se recalcula sobre lo retenido | $825 | $4,675 |
| **B. Se anula** | Nadie cobra comisión de una reserva cancelada | $0 | $5,500 |
| **C. Se conserva entera** | El administrador cobra la comisión original | $1,650 | $3,850 |

**A** reparte la pérdida igual que se reparte la ganancia, y es la que no genera discusión. **C** puede llegar a dejar al dueño en negativo si la retención es pequeña, y ese es el escenario en el que un socio se va.

### Qué se bloquea mientras no se responda

**Nada del desarrollo:** el módulo funciona y la base es configurable, se cambia sin tocar código.

Lo que bloquea es la **primera liquidación real a un co-anfitrión**. Por el congelado que se explica arriba, conviene responder **antes de dar de alta al primer dueño**: después, esas reservas ya no se pueden recalcular sin reescribir su histórico.

La pregunta 2 bloquea, además, lo que diga el **contrato con el co-anfitrión** sobre cancelaciones, que debería decir lo mismo que hace el sistema.

---

## Cómo usar este documento

- Cada duda es autocontenida: se puede enviar al cliente por separado sin que le falte contexto.
- Las secciones **"La pregunta concreta"** están redactadas para copiarse tal cual en un correo o una reunión.
- Al resolverse una duda, cambiar su estado a ✅ en la tabla del inicio, anotar la respuesta y la fecha, y aplicar la decisión en los documentos de `arquitectura/` que correspondan.
- Las dudas nuevas se agregan aquí conforme aparezcan, con el mismo formato.
