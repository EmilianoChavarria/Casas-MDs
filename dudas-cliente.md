# Dudas pendientes de resolver con el cliente

Decisiones que **no se pueden tomar desde el lado técnico** porque dependen del negocio, del presupuesto o de cómo quiera operar el cliente. Cada una incluye el contexto, las opciones reales y qué se bloquea mientras no se responda.

**Estado:** ⏳ pendiente · ✅ resuelta · ⏸️ aplazada

| # | Tema | Estado | Bloquea |
|---|---|---|---|
| D1 | Moneda y tipo de cambio para huéspedes de Canadá y EE.UU. | ⏳ Pendiente | Motor de precios, checkout, reportes |
| D2 | Traducción del contenido: asistida, no automática | ⏳ Pendiente | Alta de propiedades, SEO del sitio público |
| D3 | ¿Se cobra por huésped adicional? | ⏳ Pendiente | Motor de precios, alta de propiedades |
| D4 | ¿Descuento por estancia larga? | ⏳ Pendiente | Motor de precios, checkout |
| D5 | Base de cálculo de los impuestos (para el contador) | ⏳ Pendiente | Cálculo del total, facturación |
| D6 | ¿Cuántos administradores y cómo entran al panel? | ⏳ Pendiente | Autenticación del panel, alta de usuarios |

---

## D1 — ¿Cómo se fija el precio en dólares canadienses y estadounidenses?

**Estado:** ⏳ Pendiente

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

**Estado:** ⏳ Pendiente

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

**Estado:** ⏳ Pendiente

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

**Estado:** ⏳ Pendiente

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

**Estado:** ⏳ Pendiente

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

**Estado:** ⏳ Pendiente

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

## Cómo usar este documento

- Cada duda es autocontenida: se puede enviar al cliente por separado sin que le falte contexto.
- Las secciones **"La pregunta concreta"** están redactadas para copiarse tal cual en un correo o una reunión.
- Al resolverse una duda, cambiar su estado a ✅ en la tabla del inicio, anotar la respuesta y la fecha, y aplicar la decisión en los documentos de `arquitectura/` que correspondan.
- Las dudas nuevas se agregan aquí conforme aparezcan, con el mismo formato.
