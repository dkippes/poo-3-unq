# Stateful Traits
Alexandre Bergel, Stéphane Ducasse, Oscar Nierstrasz, Roel Wuyts — Advances in Smalltalk, Proceedings of 14th International Smalltalk Conference (ISC 2006), LNCS vol. 4406, Springer, 2007, pp. 66-90

> Resumen detallado en español, organizado siguiendo la estructura original del paper, para poder seguirlo en paralelo con el PDF. No es traducción literal sino una explicación fiel y completa, en mis propias palabras, de cada sección.

## Abstract

Los traits ofrecen un mecanismo de grano fino para componer clases a partir de componentes reutilizables, evitando los problemas de fragilidad que trae la herencia múltiple y los mixins. Tal como se propusieron originalmente, los traits son *stateless* (sin estado): solo contienen métodos, nunca variables de instancia. El estado solo puede accederse dentro de los traits a través de accessors, que se convierten en *required methods* (métodos requeridos) del trait. Aunque este enfoque funciona razonablemente bien en la práctica, significa que muchos traits, vistos como componentes de software, quedan artificialmente *incompletos*, y que las clases que usan esos traits pueden terminar con grandes cantidades de código pegamento (*glue code*) repetitivo. Aunque herramientas adecuadas pueden mitigar bastante estas limitaciones, los autores buscan una solución más limpia que soporte **stateful traits** (traits con estado). La dificultad clave es manejar los conflictos que surgen cuando los traits compuestos contribuyen variables de instancia cuyos nombres chocan. Se presenta una solución fiel al principio rector de los traits sin estado: *el cliente retiene el control de la composición*. Los stateful traits consisten en una extensión mínima a los traits sin estado, en la que las variables de instancia son puramente locales al scope del trait, a menos que el cliente que compone el trait las haga explícitamente accesibles. Los conflictos de nombres se evitan, y las variables de traits disjuntos pueden fusionarse (*merge*) explícitamente por los clientes. Se discuten y comparan dos estrategias de implementación, y se presenta brevemente un caso de estudio en el que se usaron stateful traits para refactorizar la versión basada en traits de la jerarquía de colecciones de Smalltalk.

## 1 Introduction

Los traits son unidades puras de reuso que consisten únicamente en métodos. Los traits pueden componerse tanto para formar otros traits como para formar clases. Son reconocidos por su potencial para soportar mejor composición y reuso, razón por la cual se integraron en versiones más nuevas de lenguajes como Perl 6, Squeak, Scala, Slate y Fortress. Aunque los traits se diseñaron originalmente para lenguajes dinámicamente tipados, también hubo bastante interés en aplicarlos a lenguajes estáticamente tipados.

Los traits permiten que la herencia se use para reflejar la jerarquía conceptual de un dominio, en lugar de usarse para el reuso de código. El código duplicado puede factorizarse como traits en lugar de tener que encajarlo a la fuerza en lugares incómodos de la jerarquía de clases. Al mismo tiempo, los traits evitan en gran medida los problemas de fragilidad introducidos por los enfoques basados en herencia múltiple y mixins, ya que los traits están completamente divorciados de la jerarquía de herencia.

En su forma original, sin embargo, los traits son **stateless**, es decir, son grupos de métodos sin ninguna variable de instancia. Como los traits no solo proveen métodos sino que también pueden *requerir* métodos, el idioma introducido para manejar el estado fue acceder al estado únicamente a través de accessors. El *cliente* de un trait es ya sea una clase o un trait compuesto que *usa* el trait para construir su implementación. Un principio clave detrás de los traits es que **el cliente retiene el control de la composición**. El cliente, por lo tanto, es responsable de proveer los métodos requeridos y de resolver cualquier conflicto posible. Los accessors requeridos se propagarían a los traits compuestos, y solo la clase cliente que hace la composición final tendría que implementar los accessors faltantes y las variables de instancia a las que dan acceso. En la práctica, los accessors y las variables de instancia podrían generarse fácilmente con una herramienta, de modo que el hecho de que los traits fueran sin estado solo representaba una molestia menor.

Conceptualmente, sin embargo, la falta de estado significa que prácticamente todos los traits son **incompletos**, ya que casi cualquier trait útil va a requerir algunos accessors. Como consecuencia, el mecanismo de métodos requeridos se "abusa" para cubrir la falta de estado. El interfaz requerido de un trait queda entonces saturado de ruido que dificulta entender y, en consecuencia, reutilizar un trait. Aún si los accessors y las variables faltantes pueden generarse automáticamente, muchos clientes terminan siendo "shell classes" (clases cáscara), es decir, clases que no hacen nada más que componer traits con código pegamento repetitivo. Además, si los accessors requeridos se hacen públicos (como ocurre en la implementación de Smalltalk), se viola innecesariamente la encapsulación en las clases cliente. Finalmente, si un trait se modifica para incluir estado adicional, los nuevos accessors requeridos se propagan a todos los traits y clases cliente, introduciendo así una forma de fragilidad que los traits estaban pensados para evitar.

Este paper describe los **stateful traits**, una extensión a los traits sin estado en la cual se introduce un único operador de acceso a variables que le da a los clientes de los traits control sobre la visibilidad de las variables de instancia. El enfoque es fiel al principio rector de los traits sin estado, en el cual el cliente de un trait tiene control total sobre la composición. Es precisamente este principio el que es clave para evitar la fragilidad frente al cambio, ya que no entran en juego reglas implícitas de resolución de conflictos cuando se modifica un trait.

En pocas palabras: las variables de instancia son privadas a un trait. El cliente puede decidir, sin embargo, en el momento de la composición, *acceder* a variables de instancia ofrecidas por un trait usado, o *fusionarlas* (*merge*) con variables ofrecidas por múltiples traits. En este paper se presenta primero un análisis de las limitaciones de los traits sin estado, y luego se presenta el enfoque para lograr stateful traits. Se describen y comparan dos estrategias de implementación, y se describe brevemente la experiencia con un caso de estudio ilustrativo.

**Estructura del paper:** primero se revisan los traits sin estado [SDNB03, DNS+06]. En la Sección 3 se discuten las limitaciones de los traits sin estado. En la Sección 4 se introducen los stateful traits, que soportan la introducción de estado en traits. La Sección 5 esboza algunos detalles de la implementación de los stateful traits. En la Sección 6 se presenta un pequeño caso de estudio en el que se comparan los resultados de refactorizar la jerarquía de colecciones de Smalltalk con traits sin estado y con stateful traits. En la Sección 7 se discuten algunas de las consecuencias más amplias del diseño de los stateful traits. La Sección 8 discute trabajo relacionado. La Sección 9 concluye el paper.

## 2 Stateless traits

### 2.1 Reusable groups of methods

Los traits sin estado son conjuntos de métodos que sirven como el bloque de construcción conductual de las clases y como unidades primitivas de reuso de código. Además de ofrecer comportamiento, los traits también *requieren métodos*, es decir, métodos que son necesarios para que el comportamiento del trait pueda cumplirse.

El paper presenta como ejemplo guía la Figura 1: la clase `SyncStream` se compone de dos traits, `TSyncReadWrite` y `TStream`. El trait `TSyncReadWrite` provee los métodos `syncRead`, `syncWrite` y `hash`. Requiere los métodos `read` y `write`, y los dos métodos accessor `lock` y `lock:`. Los autores usan una extensión de UML para representar traits: la columna derecha de la caja lista los métodos requeridos, mientras que la columna izquierda lista los métodos provistos.

El trait `TSyncReadWrite` define, entre sus métodos provistos, algo como:

```
syncRead
    | value |
    self lock acquire.
    value := self read.
    self lock release.
    ^ value
```

Es decir: el método `syncRead` toma el lock (`self lock acquire`), lee usando el método requerido `read`, libera el lock (`self lock release`), y devuelve el valor leído. De manera análoga, `syncWrite` hace lo mismo pero llamando al método requerido `write`. Como puede verse, `TSyncReadWrite` no puede funcionar por sí solo: necesita que algún cliente le provea `read`, `write`, y los accessors `lock`/`lock:` para acceder al lock.

### 2.2 Composing classes from mixins

El paper resume con la siguiente ecuación cómo se construye una clase a partir de traits:

```
class = superclass + state + trait composition + glue code
```

Es decir, una clase se especifica a partir de una superclase, una definición de estado (sus variables de instancia), un conjunto de traits, y algunos *métodos pegamento* (*glue methods*). Los métodos pegamento se definen en la clase y son los que conectan entre sí a los traits: es decir, implementan los métodos requeridos de los traits (frecuentemente para dar acceso al estado), adaptan los métodos provistos por los traits, y resuelven conflictos entre métodos.

En la Figura 1, la clase `SyncStream` define el campo `lock` y los métodos pegamento `lock`, `lock:`, `isBusy` y `hash`. Los otros métodos requeridos de `TSyncReadWrite`, `read` y `write`, también quedan provistos porque la clase `SyncStream` usa además otro trait, `TStream`, que los provee a ambos.

La composición de traits respeta las siguientes tres reglas:

- **Los métodos definidos en la clase tienen precedencia sobre los métodos del trait.** Esto permite que los métodos pegamento definidos en una clase sobreescriban métodos con el mismo nombre provistos por los traits usados.
- **Propiedad de flattening (aplanado).** Un método no sobreescrito (*non-overridden*) en un trait tiene exactamente la misma semántica que si estuviera implementado directamente en la clase que usa ese trait.
- **El orden de composición es irrelevante.** Todos los traits tienen la misma precedencia, y por lo tanto los métodos en conflicto entre traits deben desambiguarse explícitamente.

Con este enfoque, las clases retienen su rol primario como generadoras de instancias, mientras que los traits son puramente unidades conductuales de reuso. Igual que con los mixins, las clases se organizan en una única jerarquía de herencia simple, evitando así los problemas clave de la herencia múltiple, pero las extensiones incrementales que las clases introducen respecto de sus superclases se especifican usando uno o más traits. A diferencia de los mixins, varios traits pueden aplicarse a una clase en una sola operación: la composición de traits no está ordenada. En lugar de que la composición resulte implícitamente del orden en que se componen los traits (como ocurre con los mixins), la composición queda completamente bajo el control de la clase que compone.

### 2.3 Conflict resolution

Mientras se componen traits pueden surgir conflictos de métodos. Un **conflicto** surge si se combinan dos o más traits que provean métodos con el mismo nombre que no provienen del mismo trait. Los conflictos se resuelven implementando un método a nivel de la clase que *sobreescribe* (*overrides*) los métodos en conflicto, o *excluyendo* (*excluding*) un método de todos los traits salvo uno. Además, los traits permiten el *aliasing* de métodos; esto le permite al programador introducir un nombre adicional para un método provisto por un trait. El nombre nuevo se usa para obtener acceso a un método que de otra forma sería inalcanzable porque fue sobreescrito.

En la Figura 1, los métodos en `TSyncReadWrite` y en `TStream` son usados por `SyncStream`. La composición de traits asociada a `SyncStream` es:

```
TSyncReadWrite@{hashFromSync→hash} + TStream@{hashFromStream→hash}
```

Esto significa que `SyncStream` se compone de (i) el trait `TSyncReadWrite`, para el cual el método `hash` se alias-ea como `hashFromSync`, y (ii) el trait `TStream`, para el cual el método `hash` se alias-ea como `hashFromStream`. Como ambos traits definen un método `hash`, hay un conflicto; mediante el aliasing, la clase `SyncStream` puede seguir accediendo a ambas versiones originales bajo nombres distintos, y luego definir su propio método pegamento `hash` que las combine (en la Figura 1, `hash` en `SyncStream` se calcula como `self hashFromSync bitAnd: self hashFromStream`).

### 2.4 Method composition operators

La semántica de la composición de traits se basa en cuatro operadores: suma, *overriding*, exclusión y aliasing.

El trait **suma** `TSyncReadWrite + TStream` contiene todos los métodos no conflictivos de `TSyncReadWrite` y `TStream`. Si hay un conflicto de métodos, es decir, si `TSyncReadWrite` y `TStream` ambos definen un método con el mismo nombre, entonces en `TSyncReadWrite + TStream` ese nombre queda ligado a un método de conflicto especial (distinguido), que típicamente lanza un error si se invoca sin resolverse. El operador `+` es asociativo y conmutativo.

El operador de ***overriding*** construye un nuevo trait de composición extendiendo una composición de traits existente con algunas definiciones locales explícitas. Por ejemplo, `SyncStream` sobreescribe el método `hash` obtenido de su composición de traits. Esto también puede hacerse con métodos, como se discute más adelante en el paper.

Un trait puede construirse **excluyendo** métodos de un trait existente usando el operador de exclusión `−`. Así, por ejemplo, `TStream − {read, write}` tiene un único método, `hash`. La exclusión se usa para evitar conflictos, o si se necesita reusar un trait que es "demasiado grande" para la aplicación de uno.

El operador de **aliasing de métodos** `@` crea un nuevo trait dando un nombre adicional a un método existente. Por ejemplo, si `TStream` es un trait que define `read`, `write` y `hash`, entonces `TStream @ {hashFromStream → hash}` es un trait que define `read`, `write`, `hash` y `hashFromStream`. El método adicional `hashFromStream` tiene el mismo cuerpo que el método `hash`. Los alias se usan para hacer disponible un método en conflicto bajo otro nombre, quizás para satisfacer los requerimientos de algún otro trait, o para evitar el *overriding*. Nótese que, como el cuerpo del método alias-eado no se cambia de ninguna forma, un alias a un método recursivo no es recursivo.

## 3 Limitations of stateless traits

Los traits soportan el reuso de grupos coherentes de métodos por clases que de otro modo serían independientes. Los traits pueden componerse a partir de otros traits. Como consecuencia, sirven bien como medio para estructurar código. Desgraciadamente, los traits sin estado necesariamente codifican la dependencia respecto del estado en términos de métodos requeridos (es decir, accessors). En esencia, los traits son necesariamente *incompletos*, ya que prácticamente cualquier trait útil se verá forzado a definir accessors requeridos. Esto significa que la clase que compone debe definir las variables de instancia y los accessors faltantes.

La incompletitud de los traits resulta en una serie de limitaciones molestas, a saber: (i) la reusabilidad de los traits se ve impactada porque el interfaz requerido típicamente está saturado de accessors requeridos sin interés, (ii) las clases cliente se ven forzadas a implementar código pegamento repetitivo (*boilerplate*), (iii) la introducción de nuevo estado en un trait propaga accessors requeridos a todas las clases cliente, y (iv) los accessors públicos rompen la encapsulación de la clase cliente.

Aunque estas molestias pueden abordarse en buena medida con herramientas adecuadas, perturban el atractivo de los traits como mecanismo limpio y liviano para componer clases a partir de componentes reutilizables. Un entendimiento adecuado de estas limitaciones es un prerrequisito para considerar cualquier propuesta de un enfoque más general.

### 3.1 Limited reusability

El hecho de que un trait sin estado se vea forzado a codificar el estado en términos de accessors requeridos significa que no puede componerse "de fábrica" (*off-the-shelf*) sin alguna acción adicional. Virtualmente cualquier trait útil es incompleto, aunque la parte faltante pueda completarse trivialmente.

Lo que es peor, sin embargo, es el hecho de que el interfaz requerido de un trait queda saturado con dependencias de accessors requeridos sin interés, en lugar de enfocar la atención en los métodos *hook* no triviales que los clientes deben implementar.

Aunque este problema puede aliviarse parcialmente con soporte de herramientas adecuado que distinga los accessors requeridos sin interés de los demás métodos requeridos, el hecho permanece de que los traits con accessors requeridos nunca pueden reusarse de fábrica sin acción adicional por parte de la clase cliente final.

### 3.2 Boilerplate glue code

La acción adicional necesaria por parte del cliente consiste esencialmente en la generación de código pegamento repetitivo para inyectar las variables de instancia faltantes, los accessors, y el código de inicialización. Claramente este código repetitivo debe generarse para cada y toda clase cliente. En el enfoque más directo, esto va a llevar al tipo de código duplicado que los traits estaban pensados para evitar.

La Figura 2 ilustra una situación así, donde el trait `TSyncReadWrite` necesita acceder a un lock. Esta variable `lock`, el accessor `lock` y el mutador `lock:` deben duplicarse en `SyncFile`, `SyncStream` y `SyncSocket`.

Nuevamente, para evitar esta situación, haría falta soporte de herramientas (i) para generar automáticamente las variables de instancia y los accessors requeridos, y (ii) para generar el código de modo que se evite la duplicación real.

Un efecto secundario desagradable de la necesidad de código pegamento repetitivo es la aparición de "shell classes" (clases cáscara) que no consisten en nada más que código pegamento. En la jerarquía de Smalltalk refactorizada usando traits sin estado [BSD03], se observa que el 24% (7 de 29) de las clases en la jerarquía refactorizada con traits son clases cáscara puras.

### 3.3 Propagation of required accessors

Si la implementación de un trait evoluciona y requiere nuevas variables, eso puede impactar a todas las clases que lo usan, aún si el interfaz permanece intacto. Por ejemplo, si la implementación del trait `TSyncReadWrite` evoluciona y requiere una nueva variable `numberWaiting`, pensada para dar el número de clientes esperando el lock, entonces todas las clases que usan este trait quedan impactadas, aunque el interfaz público no cambie.

Los accessors requeridos se propagan y se acumulan de trait a trait, así que cuando una clase se compone a partir de traits compuestos profundamente, puede ser necesario resolver una gran cantidad de accessors. Cuando se introduce una nueva dependencia de estado en un trait anidado profundamente, los accessors requeridos pueden propagarse a una gran cantidad de clases cliente. De nuevo, herramientas adecuadas pueden mitigar en buena medida las consecuencias de estos cambios, pero sería bienvenida una solución más satisfactoria.

### 3.4 Violation of encapsulation

Los traits sin estado violan la encapsulación de dos formas. Primero, los traits sin estado innecesariamente exponen información sobre su representación interna, ensuciando así su interfaz. Un trait sin estado expone cada parte de su representación necesaria como un método requerido, aún si esta información no es de interés para sus clientes. La encapsulación se vería mejor servida si los traits se asemejaran más a clases abstractas, donde solo los métodos abstractos se declaran explícitamente como responsabilidad de la subclase cliente. De la misma manera, una clase cliente que usa un trait debería ver solo aquellos métodos requeridos que verdaderamente son su responsabilidad implementar, y ningún otro.

La segunda violación tiene que ver con la visibilidad. En Smalltalk, las variables de instancia son siempre privadas. El acceso puede otorgarse a otros objetos provéyendo accessors públicos. Pero si los traits requieren accessors, entonces las clases que usan estos traits *deben* proveer accessors públicos al estado faltante, aún si esto no es deseado.

En principio, este problema podría mitigarse algo en lenguajes al estilo Java incluyendo modificadores de visibilidad para traits sin estado. Un trait podría entonces requerir un accessor *private* o *protected* para el estado faltante. La clase cliente podría entonces suplir estos accessors sin violar la encapsulación (y opcionalmente relajando el modificador requerido). Esta solución, sin embargo, no resolvería el problema para lenguajes al estilo Smalltalk en los cuales todos los métodos son públicos, y solo pueden marcarse como "privados" por convención (es decir, colocando tales métodos en una categoría llamada "private").

## 4 Stateful traits: reconciling traits and state

Ahora se presentan los stateful traits como la solución a las limitaciones de los traits sin estado. Aunque pueda parecer que agregar variables de instancia a los traits representaría una extensión trivial, de hecho hay un número de cuestiones que deben resolverse. Brevemente, la solución aborda las siguientes preocupaciones:

- Los traits sin estado deberían ser un caso especial de los stateful traits. La semántica original de los traits sin estado (y las ventajas de esa solución) no deberían verse impactadas.
- Cualquier extensión debería ser sintáctica y semánticamente mínima. Se busca una solución simple.
- Deberían abordarse las limitaciones listadas en la Sección 3. En particular, debería ser posible expresar traits completos. Solo los métodos que son conceptualmente responsabilidad de las clases cliente deberían listarse como métodos requeridos.
- La solución debería ofrecer una semántica por defecto sensata para el uso de traits, habilitando así el uso de tipo *black-box* (caja negra).
- En consistencia con el principio rector de los traits sin estado, la clase cliente debería retener el control sobre la composición, en particular sobre la política para resolver conflictos. Por lo tanto también se soporta un grado de uso de tipo *white-box* (caja blanca), donde sea necesario.
- Igual que con los traits sin estado, se busca evitar la fragilidad respecto del cambio. Los cambios a la representación de un trait normalmente no deberían afectar a sus clientes.
- La solución debería ser en buena medida independiente del lenguaje. No se depende de características de lenguaje oscuras o exóticas, así que el enfoque debería aplicarse fácilmente a la mayoría de los lenguajes orientados a objetos.

La solución presentada extiende los traits para que posiblemente incluyan variables de instancia. En pocas palabras, hay tres aspectos en el enfoque:

1. Las variables de instancia son, por defecto, *privadas* al scope del trait que las define.
2. El cliente de un trait, es decir, una clase o un trait compuesto, puede *acceder* a variables seleccionadas de ese trait, mapeándolas posiblemente a nombres nuevos. Los nombres nuevos son privados al scope del cliente.
3. El cliente de un trait compuesto puede *fusionar* (*merge*) variables de los traits que usa mapeándolas a un nombre común. El nombre nuevo es privado al scope del cliente.

En las subsecciones siguientes se dan los detalles del modelo de stateful traits.

### 4.1 Stateful trait definition

Un stateful trait extiende a un trait sin estado incluyendo variables de instancia privadas. Un stateful trait por lo tanto consiste en un grupo de métodos públicos y variables de instancia privadas, y posiblemente la especificación de algunos métodos requeridos adicionales que deben implementar los clientes.

**Métodos.** Los métodos definidos en un trait son visibles para cualquier otro trait con el que se componga. Como los métodos son públicos, pueden ocurrir conflictos cuando se componen traits. Los conflictos de métodos para stateful traits se resuelven de la misma forma que con los traits sin estado.

**Variables.** Por defecto, las variables son privadas al trait que las define. Como las variables son privadas, no pueden ocurrir conflictos entre variables cuando se componen traits. Si, por ejemplo, los traits `T1` y `T2` cada uno define una variable `x`, entonces la composición de `T1 + T2` *no* produce un conflicto de variables. Las variables solo son visibles para el trait que las define, a menos que el acceso se amplíe por el trait o clase cliente que hace la composición con el operador de acceso a variables `@@`.

La Figura 3 muestra cómo la situación presentada en la Figura 1 se reimplementa usando stateful traits. La clase `SyncStream` se compone de los traits `TStream` y `TSyncReadWrite`. El trait `TSyncReadWrite` define la variable `lock`, tres métodos `syncRead`, `syncWrite` y `hash`, y requiere los métodos `read` y `write`.

Nótese que, para poder incluir estado en los traits, hay que extender el mecanismo de definición de traits. En la implementación en Smalltalk, esto se logra extendiendo el mensaje enviado a la clase `Trait` con un nuevo argumento de palabra clave que representa las variables de instancia usadas. Por ejemplo, ahora se puede definir el trait `TSyncReadWrite` de la siguiente forma:

```
Trait named: #TSyncReadWrite
    uses: {}
    instVarNames: 'lock'
```

El trait `TSyncReadWrite` no se compone de ningún otro trait y define una variable `lock`. La cláusula `uses:` especifica la composición de traits (vacía en este caso), y `instVarNames:` lista las variables definidas en el trait (es decir, la variable `lock`). El interfaz para definir una clase como composición de traits es el mismo que con traits sin estado. La única diferencia es que la expresión de composición de traits soporta un operador adicional (`@@`) para otorgar acceso a variables de los traits usados. Acá se ve cómo `SyncStream` se compone a partir de los traits `TSyncReadWrite` y `TStream`:

```
Object subclass: #SyncStream
    uses: TSyncReadWrite @ {#hashFromSync→#hash}
            @@ {syncLock→lock}
          + TStream @ {#hashFromStream→#hash}
    instVarNames: ''
    ....
```

En este ejemplo, se otorga acceso a la variable `lock` del trait `TSyncReadWrite` bajo el nombre nuevo `syncLock`. Como se verá a continuación, el operador `@@` provee un grado fino de control sobre la visibilidad de las variables de un trait.

### 4.2 Variable access

Por defecto, una variable es privada al trait que la define. Sin embargo, el operador de acceso a variables (`@@`) permite que las variables sean *accedidas* desde clientes bajo un nombre posiblemente nuevo, y posiblemente *fusionadas* con otras variables.

Si `T` es un trait que define una variable de instancia (privada) `x`, entonces `T@@{y→x}` representa un nuevo trait en el cual la variable `x` puede accederse desde el scope de su cliente bajo el nombre `y`. `x` e `y` representan la misma variable, pero el nombre `x` queda restringido al scope de `T`, mientras que el nombre `y` es visible al scope del cliente que lo encierra (es decir, el scope de la clase que compone). Por ejemplo, en la composición:

```
TSyncReadWrite@{hashFromSync→hash} @@{syncLock→lock}
```

la variable `lock` definida en `TSyncReadWrite` es accesible para la clase `SyncStream` que usa ese trait bajo el nombre `syncLock`. (Nótese que el renombrado a menudo es necesario para distinguir variables con nombres similares que provienen de distintos traits usados.)

En una composición de variables de trait, pueden darse tres situaciones: (i) las variables permanecen privadas (es decir, el operador de acceso a variables no se usa), (ii) se otorga acceso a una variable privada, y (iii) las variables se fusionan.

La Figura 4 ilustra el caso en el que las variables se mantienen privadas: una clase `C` que define una variable `x` y dos métodos `getX` y `setX:`, y dos traits `T1` y `T2` que cada uno define su propia variable `x` (con métodos `getXT1`/`setXT1:` y `getXT2`/`setXT2:` respectivamente). `C`, `T1` y `T2` cada uno tiene su propia variable `x`, sin ningún conflicto.

**Manteniendo las variables privadas.** Por defecto, las variables de instancia son privadas a su trait. Si el scope de las variables no se amplía en el momento de la composición usando el operador de acceso a variables, no ocurren conflictos y los traits no comparten estado. La Figura 4 muestra un caso donde `T1` y `T2` se componen sin que se amplíe el acceso a variables. Cada uno de estos dos traits define una variable `x`. Además, cada uno define métodos accessor. `C` también define una variable `x` y dos métodos `getX` y `setX:`. `T1`, `T2` y `C` cada uno tiene su propia variable `x`.

La composición de traits de `C` es: `T1 + T2`. Nótese que si los métodos entraran en conflicto se usaría la estrategia por defecto de los traits para resolverlos redefiniéndolos localmente en `C`, y que el aliasing de métodos podría usarse para acceder a los métodos sobreescritos.

Esta forma de composición se acerca al enfoque de composición de módulos propuesto en Jigsaw, y soporta un escenario de reuso de tipo caja negra (*black-box*).

Concretamente, si se ejecuta:

```
c := C new.
c setXT1: 1.
c setXT2: 2.
c setX: 3.
```

Entonces, ahora: `c getXT1 = 1`, `c getXT2 = 2`, `c getX = 3` — las tres variables `x` (la de `T1`, la de `T2` y la de `C`) son completamente independientes entre sí.

**Otorgando acceso a variables.** La Figura 5 muestra cómo la clase cliente `C` gana acceso a las variables privadas `x` de los traits `T1` y `T2` usando el operador de acceso a variables `@@`. Como dos variables no pueden tener el mismo nombre dentro de un scope dado, estas variables tienen que renombrarse. La variable `x` de `T1` se vuelve accesible como `xFromT1`, y la `x` de `T2` se vuelve accesible como `xFromT2`. `C` también define un método `sum` que devuelve el valor `xFromT1 + xFromT2`. La composición de traits de `C` es:

```
T1 @@ {xFromT1 → x}
+ T2 @@ {xFromT2 → x}
```

`C` puede entonces construir funcionalidad encima de los traits que usa, sin exponer ningún detalle hacia afuera. Nótese que los métodos en el trait continúan usando el nombre "interno" de la variable tal como está definido en el trait. El nombre dado en el operador de acceso a variables `@@` solo se usa en las clases cliente. Esto es similar al operador de aliasing de métodos `@`.

Con el ejemplo anterior, ejecutando:

```
c := C new.
c setXT1: 1.
c setXT2: 2.
```

ahora `c sum` devuelve `3` (la suma de `xFromT1` y `xFromT2`, accedidas desde `C` bajo esos nombres nuevos, aunque internamente en los traits sigan siendo `x`).

**Fusionando variables (merge).** Variables de varios traits pueden fusionarse cuando se componen, usando el operador de acceso a variables para mapear múltiples variables a un *nombre común* dentro del scope del cliente. Esto se ilustra en la Figura 6.

Tanto `T1` como `T2` dan acceso a sus variables de instancia `x` e `y` (respectivamente) bajo el nombre `w`. Esto significa que `w` queda compartida entre los tres (`T1`, `T2` y el cliente `C`). Esta es la razón por la cual enviar `getX`, `getY`, o `getW` a una instancia de una clase que implementa `C` devuelve el mismo resultado, `3`. La composición de traits de `C` es:

```
T1 @@ {w→x} + T2 @@ {w→y}
```

Concretamente, ejecutando:

```
c := C new.
c setW: 3.
```

Ahora: `c getX = 3`, `c getY = 3`, `c getW = 3` — porque `x` (de `T1`), `y` (de `T2`) y `w` (de `C`) son, de hecho, la misma variable subyacente, fusionada bajo el nombre `w`.

Nótese que la fusión está *completamente* bajo el control de la clase o trait cliente. No puede haber captura accidental de nombres ya que la visibilidad de las variables de instancia nunca se propaga a un scope que la encierre. Los conflictos de nombres de variables no pueden surgir, ya que las variables son privadas a los traits a menos que sean accedidas explícitamente por clientes, y las variables se fusionan cuando se mapean a nombres comunes.

El lector bien podría preguntarse: ¿qué pasa si el cliente también define una variable de instancia cuyo nombre coincide con el nombre bajo el cual se accede a la variable de un trait usado? Supongamos, por ejemplo, que `C` en la Figura 6 intenta además definir una variable de instancia llamada `w`. Los autores consideran que esto es un **error**. Esta situación no puede surgir posiblemente como efecto secundario de cambiar la definición de un trait usado, ya que el cliente tiene control total sobre los nombres de las variables de instancia accesibles dentro de su scope. Como consecuencia, esto no puede ser un caso de captura accidental de nombres, y solo puede interpretarse como un error.

### 4.3 Requirements revisited

Reconsiderando brevemente los requerimientos planteados al principio de la Sección 4: primero, los stateful traits no cambian la semántica de los traits sin estado. Los traits sin estado son puramente un caso especial de los stateful traits. Sintáctica y semánticamente, los stateful traits representan solo una extensión menor a los traits sin estado.

Los stateful traits abordan las cuestiones planteadas en la Sección 3. En particular, (i) ya no hay necesidad de saturar los interfaces de los traits con accessors requeridos, (ii) los clientes ya no necesitan proveer variables de instancia y accessors repetitivos, (iii) la introducción de estado en los traits permanece privada a ese trait, y (iv) no es necesario introducir accessors públicos en las clases cliente. Como consecuencia, es posible definir traits "completos" que no requieran métodos, aunque hagan uso de estado.

La semántica por defecto de los stateful traits habilita el uso de tipo *black-box*, ya que no se expone ninguna representación, y las variables de instancia por defecto no pueden chocar con las de otros traits usados ni con las del cliente. No obstante, el cliente retiene el control de la composición, y puede ganar acceso a las variables de instancia de los traits usados. En particular, el cliente puede fusionar variables de traits, si esto es deseado.

Como el cliente retiene control total de la composición, los cambios a la definición de un trait no pueden propagarse más allá de sus clientes directos. No puede haber efectos secundarios implícitos.

Finalmente, el enfoque es en buena medida independiente del lenguaje. En particular, no hay suposiciones de que el lenguaje anfitrión provea modificadores de acceso para variables de instancia ni mecanismos de scoping exóticos.

## 5 Implementation

Se implementó un prototipo de stateful traits como una extensión de la implementación en Smalltalk de traits sin estado de los mismos autores.

Igual que con los traits sin estado, la composición y el reuso de métodos para stateful traits no incurren en ninguna sobrecarga, ya que los punteros a métodos se comparten entre los diccionarios de métodos de distintos traits y clases. Esto se aprovecha del hecho de que los métodos se buscan por nombre en el diccionario en lugar de accederse por índice y offset, como ocurre con el acceso al estado en la mayoría de los lenguajes orientados a objetos. Sin embargo, al agregar estado a los traits, hay que encontrar una solución al hecho de que el acceso a las variables de instancia no puede ser lineal (es decir, basado en offsets), ya que los mismos métodos de un trait pueden aplicarse a objetos distintos. Una estructura lineal para la representación del estado no siempre puede obtenerse a partir de un grafo de composición. Este es un problema común a los lenguajes que soportan herencia múltiple. Se evaluaron dos implementaciones: *copy-down* y cambio de la representación interna del objeto.

### 5.1 The classical problem of state linearization

Como señaló Bracha, y más recientemente en la Jikes Research Virtual Machine, la noción de funciones virtuales se soporta, en implementaciones de lenguajes de herencia simple como Modula-3, asociando a cada clase una tabla cuyas entradas son las direcciones de los métodos definidos para instancias de esa clase. Cada instancia de una clase contiene una referencia a la tabla de métodos de la clase. Es a través de esta referencia que se ubica el método apropiado a invocar sobre una instancia. Bajo herencia múltiple, esta técnica debe modificarse, ya que las superclases de una clase ya no comparten un prefijo común.

Como un stateful trait puede tener estado privado, y puede usarse en múltiples contextos, no es posible tener una lista de offsets de variables de instancia estática y lineal compartida por todos los métodos del trait y sus usuarios.

La parte superior de la Figura 7 muestra un trait `T3` que usa `T1`, y un trait `T4` que usa `T1` y `T2`. `T1` define 3 variables `x`, `y`, `z`, y `T2` define 2 variables `v`, `x`. La parte inferior muestra una posible representación correspondiente en memoria, que usa offsets. Asumiendo que la indexación arranca en cero, `T2.v` tiene índice cero y `T2.x` tiene índice uno (en el contexto de `T3`, donde solo se usa `T1`). Sin embargo, en `T4` esas mismas dos variables podrían tener índices tres y cuatro (porque ahí se concatenan las variables de `T1` antes que las de `T2`). Entonces, los índices estáticos usados en los métodos de `T1` o `T2` ya no son válidos. Nótese que este problema ocurre independientemente de cómo se componga el trait `T4` a partir de los traits `T1` y `T2` (si necesita o no acceso a las variables, si fusiona o no la variable `x`...). El problema se debe a la representación lineal de las variables en el modelo de objetos subyacente.

### 5.2 Three approaches to state linearization

Hay tres enfoques diferentes disponibles para representar estado no lineal. C++ usa punteros intra-objeto (*intra-object pointers*). Strongtalk usa una técnica de *copy-down* que duplica los métodos que necesitan acceder a variables con offset distinto. Un tercer enfoque, como se hace en Python por ejemplo, es mantener las variables en un diccionario y buscarlas por nombre (*lookup*), de forma similar a lo que se hace con los métodos.

Se implementaron los dos últimos enfoques para Smalltalk, de modo de poder compararlos para el prototipo de los autores. No se implementó la solución de C++ porque hubiera requerido un esfuerzo significativo para cambiar la representación de objetos y hacerla compatible.

### 5.3 Virtual base pointers in C++

En C++, una instancia de una clase `C` se representa concatenando las representaciones de las superclases de `C`. Tal instancia se compone, por lo tanto, de *subobjetos*, donde cada subobjeto corresponde a una *superclase* particular. Cada subobjeto tiene su propio puntero a una tabla de métodos adecuada. En este caso, la representación de una clase no es un prefijo de las representaciones de todas sus subclases.

Cada subobjeto comienza en un offset distinto desde el principio del objeto `C` completo. Estos offsets, llamados *punteros de base virtual* (*virtual base pointers*), pueden calcularse estáticamente. Esta técnica fue iniciada por Krogdahl.

Por ejemplo, consideremos la situación en C++ ilustrada en la Figura 8. La parte superior de la figura muestra un diagrama clásico en forma de diamante usando herencia virtual (es decir, `B` y `C` heredan virtualmente de `A`, por lo tanto la variable `w` se comparte entre `B` y `C`). La parte inferior muestra el layout de memoria de una instancia de `D`. Esta instancia se compone de 4 "subpartes" correspondientes a las superclases `A`, `B`, `C` y `D`. Nótese que la parte de `C`, en lugar de asumir que el estado que hereda de `A` se ubica inmediatamente "arriba" de su propio estado, accede al estado heredado vía el puntero de base virtual. De esta manera, las partes `B` y `C` de la instancia de `D` pueden compartir el mismo estado común de `A`.

Los autores no intentaron implementar esta estrategia en su prototipo de Smalltalk, ya que hubiera requerido una modificación profunda de la VM de Smalltalk. Como Smalltalk soporta solo herencia simple, el layout de objetos es fundamentalmente más simple. Acomodar punteros de base virtual en el layout de un objeto también implicaría cambios al algoritmo de búsqueda de métodos (*method lookup*).

### 5.4 Object state as a dictionary

Un enfoque de implementación alternativo es introducir accesos a variables de instancia basados en nombres y no en offsets. El layout de variables tiene la semántica de una tabla de hash, en lugar de la de un arreglo. Para una variable dada, su offset ya no es constante, como se muestra en la Figura 9. El estado de un objeto se implementa mediante una tabla de hash en la cual varias claves pueden mapear al mismo valor. Por ejemplo, la variable `y` de `T1` y la variable `v` de `T2` se fusionan en `T4`. Por lo tanto, una instancia de `T4` tiene dos variables (claves), `T1.y` y `T2.v`, que en realidad apuntan al mismo valor.

En Python, el estado de un objeto se representa mediante un diccionario. Una expresión como `self.name = value` se traduce a `self.__dict__[name] = value`, donde `__dict__` es una primitiva para acceder al diccionario de un objeto. Una variable se declara y se define simplemente usándola en Python. Por ejemplo, asignarle un valor a una variable no existente tiene el efecto de crear una variable nueva. Representar el estado de un objeto con un diccionario es una forma de lidiar con el problema de linearización de la herencia múltiple.

### 5.5 Copy down methods

Strongtalk es un Smalltalk de alto rendimiento con una máquina virtual consciente de mixins (*mixin-aware*). Un mixin contiene la descripción de sus variables de instancia y variables de clase, y un diccionario de métodos donde se almacena inicialmente todo el código. Uno de los problemas al compartir código entre distintas aplicaciones de un mismo mixin es que el layout físico de las instancias varía entre las distintas aplicaciones del mixin. Este problema se aborda con el mecanismo de **copy down**: (i) los métodos que no acceden a variables de instancia ni a `super` se comparten en el mixin; (ii) los métodos que acceden a variables de instancia pueden tener que copiarse si el layout de variables difiere del de otros usuarios del mixin.

El mecanismo de copy-down favorece la velocidad de ejecución por sobre el consumo de memoria. No hay sobrecarga adicional para acceder a variables. Las variables están ordenadas linealmente, y los métodos que las acceden se duplican y se ajustan con el offset adecuado. Más aún, en Strongtalk, solo los accessors pueden tocar las variables de instancia directamente a nivel de bytecode. La sobrecarga de espacio del copy-down es por lo tanto mínima. El *inlining* efectivo realizado por la VM se encarga del resto, excepto para los accessors, los cuales no imponen ninguna sobrecarga de espacio.

El enfoque basado en diccionario tiene la ventaja de que refleja más directamente la semántica de los stateful traits, y por lo tanto es atractivo para una implementación de prototipo. El desempeño práctico, sin embargo, puede volverse problemático, aún con implementaciones de diccionario optimizadas como las de Python. El enfoque de copy-down, sin embargo, es claramente el mejor enfoque para una implementación rápida. Por lo tanto, los autores decidieron adoptarlo en su implementación de stateful traits en Squeak Smalltalk.

### 5.6 Benchmarks

Como se mencionó en la sección anterior, se adoptó la técnica de copy-down para la implementación de stateful traits. En esta sección se compara el desempeño del prototipo de stateful traits con el de Squeak regular sin traits y con el de la implementación de traits sin estado. Se midió el desempeño de los siguientes dos casos de estudio:

- El ejemplo `SyncStream` introducido al comienzo del paper. El experimento consistió en escribir y leer objetos grandes en un stream 1000 veces. Este ejemplo se elegió para evaluar si el estado se accede de forma eficiente.
- Una aplicación de verificación de links (*link checker*) que parsea páginas HTML para chequear si las URLs en una página web son alcanzables o no. Esto implica parsear archivos HTML grandes en una representación de árbol y correr visitantes (*visitors*) sobre esos árboles. Este caso de estudio se elegió para tener un ejemplo más balanceado que consista tanto en acceso a métodos como en acceso a estado.

Para ambos casos de estudio se comparó la implementación stateful con la implementación de traits sin estado y con Squeak regular. Los resultados se muestran en la Tabla 1:

| | Sin traits | Traits sin estado | Stateful traits |
|---|---|---|---|
| SyncStream | 13912 | 13913 | 13912 |
| LinkChecker | 2564 | 2563 | 2564 |

(Tiempos de ejecución en milisegundos.)

Como puede verse en la tabla, no se introduce sobrecarga por acceder a las variables de instancia definidas en traits y usadas en clientes. Esto era de esperarse: el acceso sigue siendo basado en offsets, y casi no pueden notarse diferencias. En cuanto a la velocidad de ejecución general, se ve que esencialmente no hay diferencia entre las tres implementaciones. Este resultado es consistente con experiencia previa usando traits, y era de esperarse ya que no se cambiaron las partes de la implementación que tratan con métodos.

## 6 Refactoring the Smalltalk collection hierarchy

Se llevó a cabo un caso de estudio en el cual se usaron stateful traits para refactorizar la jerarquía de colecciones de Smalltalk. Previamente se había usado traits sin estado para refactorizar la misma jerarquía [BSD03], y ahora se comparan los resultados de las dos refactorizaciones. La jerarquía de colecciones de Smalltalk basada en traits sin estado consiste en 29 clases, construidas a partir de un total de 52 traits. Entre estas 29 clases hay numerosas clases, que los autores llaman clases *shell* (cáscara), que solo declaran variables y definen sus accessors asociados. Siete de las 29 clases (24%) son clases shell (`SkipList`, `PluggableSet`, `LinkedList`, `OrderedCollection`, `Heap`, `Text` y `Dictionary`).

La refactorización con stateful traits resulta en una redistribución de las variables definidas (en clases) hacia los traits que efectivamente las necesitan y las usan. Otra consecuencia es la disminución del número de métodos requeridos y una mejor encapsulación del comportamiento y la representación interna de los traits.

La Figura 10 muestra un fragmento de la jerarquía de traits sin estado de Smalltalk, donde la clase `Heap` debe definir 3 variables (`array`, `tally`, y `sortBlock`). El comportamiento de esta clase se limita a la inicialización de objetos y a proveer accessors para cada una de estas variables. Usa el trait `THeapImpl`, que requiere todos estos accessors. Estos requerimientos son necesarios para `THeapImpl` ya que se compone de `TArrayBased` y `TSortBlockBased`, que requieren ese estado. Estos dos traits necesitan acceder al estado definido en `Heap`.

La Figura 11 muestra cómo se refactoriza `Heap` para usar stateful traits. Todas las variables se movieron a los lugares donde se necesitaban, llevando al resultado de que `Heap` queda vacía. Las variables previamente definidas en `Heap` ahora se definen en los traits que efectivamente las requieren. `TArrayBased` define dos variables, `array` y `tally`, por lo que ya no necesita especificar ningún accessor como método requerido. Lo mismo ocurre con `TSortBlockBased` y la variable `sortBlock`.

Si se tiene la certeza de que `THeapImpl` no es usado por ninguna otra clase o trait, entonces se puede simplificar aún más esta nueva composición moviendo la implementación del trait `THeapImpl` a `Heap` y eliminando `THeapImpl`. La Figura 12 muestra la jerarquía resultante. La clase `Heap` define métodos como `add:` y `copy`.

Refactorizar la jerarquía de clases de Smalltalk usando stateful traits da múltiples beneficios:

- *Se preserva la encapsulación*: la representación interna no se revela innecesariamente a las clases cliente.
- *Menos definiciones de métodos*: se evitan accessors de variable innecesarios. Los accessors que estaban definidos en `Heap` se eliminan.
- *Menos requerimientos de métodos*: como las variables se definen en los traits que las usan, se evita especificar accessors requeridos. Los accessors de variable para `THeapImpl`, `TArrayBased`, y `TSortBlockBased` ya no son requeridos. No hay propagación de métodos requeridos debido al uso de estado.

## 7 Discussion

### 7.1 Flattening property

En el modelo original de traits sin estado, la composición de traits respeta la *propiedad de flattening*, que establece que un método no sobreescrito en un trait tiene la misma semántica que si estuviera implementado directamente en la clase. Esto implica que los traits pueden "inlinearse" para dar una definición de clase equivalente que no use traits. Es natural preguntarse si una propiedad tan importante se preserva con los stateful traits. En resumen, la respuesta es sí, aunque las variables de los traits pueden tener que alpha-renombrarse para evitar choques de nombres.

Para preservar la propiedad de flattening con stateful traits, hay que asegurarse de que las variables de instancia introducidas por los traits permanezcan privadas al scope de los métodos de ese trait, aún cuando su scope se amplíe al de la clase que compone. Esto puede hacerse de varias formas, dependiendo de los mecanismos de scoping que ofrezca el lenguaje anfitrión. Semánticamente, sin embargo, el enfoque más simple es alpha-renombrar las variables de instancia privadas del trait a nombres que sean únicos en el scope del cliente. Técnicamente, esto podría lograrse con la técnica común de *name-mangling*, es decir, anteponiendo el nombre del trait al nombre de la variable cuando se la inserta en el scope del cliente. El renombrado y la fusión también son consistentes con el flattening, ya que las variables simplemente pueden renombrarse o fusionarse en el scope del cliente.

### 7.2 Limiting change impact

Cualquier enfoque para componer software está destinado a ser frágil respecto de ciertos tipos de cambio: si una característica usada por varios clientes cambia, el cambio afectará a los clientes. Extender un trait de modo que provea métodos adicionales bien puede afectar a los clientes introduciendo nuevos conflictos. Sin embargo, el diseño de la composición de traits, basado en resolución explícita, asegura que tales cambios no puedan llevar a cambios implícitos e inesperados en el comportamiento de clientes directos o indirectos. Un cliente directo generalmente puede resolver un conflicto sin cambiar o introducir ningún otro trait, así que no va a ocurrir efecto dominó (*ripple effect*).

En stateful traits, agregar una variable a un trait no afecta a los clientes porque las variables son privadas. Quitar o renombrar una variable puede requerir que sus clientes directos se adapten, solo si esa variable es accedida explícitamente por esos clientes. Sin embargo, una vez que los clientes directos se han adaptado, no puede ocurrir efecto dominó en clientes indirectos. Al evitar la propagación de métodos requeridos, los stateful traits limitan el efecto de los cambios.

### 7.3 About variable access

Por defecto, una variable de un trait es privada, garantizando así el reuso de tipo caja negra. Al mismo tiempo, se ofrece un operador que le permite al cliente directo acceder a las variables privadas del trait. Esto podría parecer una violación de la encapsulación [Sny86]. Sin embargo, este enfoque es consistente con la visión de que los traits sirven como bloques de construcción para componer clases, sea de forma caja negra o caja blanca. Además, es consistente con el principio de que el cliente de un trait está en control de la composición. Es precisamente este hecho el que asegura que los efectos de los cambios no se propaguen a rincones remotos de la jerarquía de clases.

## 8 Related work

Se revisa brevemente algunas de las numerosas actividades de investigación relevantes a los stateful traits.

**Self.** El lenguaje basado en prototipos Self no tiene noción de clase. Conceptualmente, cada objeto define su propio formato, métodos, y relaciones de delegación. Los objetos se derivan de otros objetos por clonado y modificación. Los objetos pueden tener uno o más objetos padre; los mensajes que no se encuentran en el objeto se buscan y delegan a un objeto padre. Self está basado en la noción de *slots*, que unifica métodos y variables de instancia.

Self usa objetos *trait* para factorizar características comunes. Nada impide que un objeto trait también contenga estado. De forma similar a la noción de traits presentada en este paper, estos objetos trait son esencialmente grupos de métodos. Pero, a diferencia de los traits de los autores, los objetos trait de Self no soportan operadores de composición específicos; en cambio, se usan como objetos padre ordinarios.

**Interfaces con implementación por defecto.** Mohnen propuso una extensión de Java en la cual las interfaces pueden equiparse con un conjunto de implementaciones por defecto de métodos. Así, las clases que implementan tal interfaz pueden declarar explícitamente que quieren usar la implementación por defecto ofrecida por esa interfaz (si la hay). Si más de una interfaz menciona el mismo método, debe proveerse un cuerpo de método. Los conflictos se marcan (*flagged*) automáticamente, pero requieren que el desarrollador los resuelva manualmente. El estado no puede asociarse con las interfaces. Scala también soporta traits, es decir, interfaces parcialmente definidas. Aunque la composición de traits en Scala no sigue exactamente la de los traits sin estado, los traits en Scala no pueden definir estado.

**Mixins.** Los mixins usan el operador ordinario de herencia simple para extender varias clases padre con un conjunto empaquetado de características. Aunque este operador de herencia es adecuado para derivar nuevas clases a partir de clases existentes, no es necesariamente apropiado para componer bloques de construcción reutilizables. Específicamente, porque la composición de mixins se implementa usando herencia simple, los mixins se componen linealmente. Esto da lugar a varios problemas: primero, puede ser difícil encontrar (o puede no existir siquiera) un orden total adecuado de características; segundo, el "código pegamento" que explota o adapta la composición lineal puede quedar dispersado por toda la jerarquía de clases; tercero, las jerarquías de clases resultantes a menudo son frágiles respecto del cambio, de modo que cambios conceptualmente simples pueden impactar muchas partes de la jerarquía.

**Eiffel.** Eiffel es un lenguaje puramente orientado a objetos que soporta herencia múltiple. Las características (*features*, es decir, métodos o variables de instancia) pueden heredarse múltiples veces por distintos caminos. Eiffel provee al programador mecanismos que ofrecen un grado fino de control sobre si tales características se comparten o se replican. En particular, las características pueden ser *renombradas* (*renamed*) por la clase que hereda. También es posible *seleccionar* (*select*) una característica particular en caso de conflictos de nombres. Seleccionar una característica significa que, desde el contexto de la subclase que compone, la característica seleccionada tiene precedencia sobre las posiblemente conflictivas.

A pesar de las similitudes entre el esquema de herencia de Eiffel y la composición de stateful traits, hay algunas diferencias significativas:

- *Renombrado (renaming) vs. aliasing* — En Eiffel, cuando se crea una subclase, las características heredadas pueden renombrarse. Renombrar una característica tiene el mismo efecto que (i) darle un nombre nuevo a esa característica, y (ii) cambiar todas las referencias a esa característica. Esto implica una especie de mapeo a realizar cuando se accede a un método renombrado a través del tipo estático de la superclase.

  Por ejemplo, supongamos que una clase `Component` define un método `update`. Una subclase `GraphicalComponent` renombra `update` a `repaint`, y redefine este `repaint` con una nueva implementación. El siguiente código ilustra esta situación:

  ```eiffel
  class Component
  feature
      update is
          do
              print ('1')
          end
  end

  class GraphicalComponent
  inherit
      Component
          rename
              update as repaint
          redefine
              repaint
          end
      repaint is
          do
              print ('2')
          end
  end
  ```

  En esencia, el método `repaint` actúa como un *override* de `update`. Esto significa que si se le envía `update` a una instancia de `GraphicalComponent`, entonces se llama a `repaint`. Esto se ilustra con el siguiente ejemplo:

  ```eiffel
  f (c: Component) is
      do
          c.update
      end
  f (create {GraphicalComponent})
  ==> 2
  ```

  Es decir: el método `f` recibe un objeto declarado de tipo estático `Component` y le envía `update`; cuando se le pasa una instancia real de `GraphicalComponent`, el envío de `update` termina ejecutando `repaint` (que imprime `2`), gracias al renombrado. Esta es la forma en que Eiffel preserva el polimorfismo mientras soporta el renombrado.

  En los stateful traits, alias-ear un método u otorgar acceso a una variable le asigna un nombre nuevo. El método o la variable, sin embargo, todavía puede invocarse o accederse a través de su nombre original. Esta es una diferencia clave respecto del renombrado de Eiffel, donde el nombre original deja de estar disponible.

- *Fusión de variables (merging variables)* — En contraste con los stateful traits, en Eiffel las variables solo pueden fusionarse si provienen de una superclase común. En los stateful traits, las variables provistas por dos traits pueden fusionarse sin importar cómo se hayan formado esos traits.

**Jigsaw.** Jigsaw tiene un sistema de módulos en el cual un módulo es un scope auto-referencial que liga nombres a valores (es decir, constantes y funciones). Un módulo actúa como una clase (generador de objetos) y como una unidad estructural de software de grano grueso. Los módulos pueden anidarse, por lo tanto un módulo puede definir un conjunto de clases. Se provee un conjunto de operadores para componer módulos. Estos operadores son instanciación, *merge*, *override*, *rename*, *restrict*, y *freeze*.

Aunque hay algunas diferencias entre la definición de un módulo de Jigsaw y los stateful traits, por ejemplo con el operador *rename*, las diferencias más significativas están en la motivación y el contexto. Jigsaw es un framework para definir lenguajes modulares. Jigsaw soporta renombrado completo, y le asigna una interpretación semántica al anidamiento (*nesting*). En Jigsaw, un renombrado es equivalente a un reemplazo textual de todas las ocurrencias del atributo. El operador *rename* distribuye sobre *override*. Esto significa que Jigsaw tiene la siguiente propiedad:

```
(m1 rename a to b) override (m2 rename a to b) = (m1 override m2) rename a to b
```

Los traits están pensados para suplementar lenguajes existentes promoviendo el reuso a pequeña escala (*in the small*), no declaran tipos, no infieren sus requerimientos, y no permiten renombrado. Los traits sin estado no le asignan ningún significado al anidamiento. Los stateful traits son sensibles al anidamiento solo en la medida en que las variables de instancia son privadas a un scope dado. El conjunto de operaciones de Jigsaw también apunta a la completitud, mientras que en el diseño de los traits se sacrifica la completitud en favor de la simplicidad.

Una diferencia notable entre Jigsaw y los stateful traits está en la fusión de variables. En Jigsaw, un módulo puede tener estado, sin embargo las variables no pueden compartirse entre módulos. Con los stateful traits, la misma variable puede ser accedida por los traits que la usan. Un módulo de Jigsaw actúa como una caja negra. El módulo encapsula sus ligaduras (*bindings*) y no puede abrirse. Si bien se valora la composición de tipo caja negra, los stateful traits no adoptan un enfoque tan restrictivo, sino que dejan que el cliente asuma la responsabilidad de la composición, mientras quedan protegidos del impacto de los cambios.

Vale la pena mencionar las cuestiones de tipado que surgen al implementar Jigsaw. Bracha señaló que la dificultad de implementar la herencia en Jigsaw (que está basada en operadores) proviene de la interacción entre el subtipado estructural y las propiedades algebraicas de los operadores de herencia (por ejemplo, *merge* y *override*).

Para ilustrar este problema, consideremos las siguientes clases `A`, `B`, `C`, `D`, `E` y `F`, donde `C` es subclase de `A` y `B`. `E` es subclase de `D` y `C`. `F` es subclase de `D`, `A` y `B`. Tenemos `C = AB`, `E = DC` y `F = DAB`, donde en `C_new = C1 C2 ... Cn`, las superclases de `C_new` se denotan `Ci`. Expandiendo las definiciones de todos los nombres (tal como dicta el tipado estructural), uno encuentra que, por asociatividad, `E = F`. Esta equivalencia dicta que las tres clases tengan el mismo tipo, de modo que puedan usarse de forma intercambiable. Esto a su vez requiere que las tres tengan la misma representación. Sin embargo, usando las técnicas de C++ (Sección 5.3), estas tres clases tienen representaciones distintas. Este problema se evita en los traits, donde un trait no define un tipo.

**Cecil.** Cecil es un lenguaje puramente orientado a objetos que combina un modelo de objetos sin clases (*classless*), un tipo de herencia dinámica, y chequeo de tipos estático opcional. El sistema de tipos estático de Cecil distingue entre subtipado y herencia de código, aún cuando el caso más común es que la jerarquía de subtipado sea paralela a la jerarquía de herencia. Cecil soporta herencia múltiple. Heredar del mismo ancestro más de una vez, sea directa o indirectamente, no tiene ningún efecto más que ubicar al ancestro en relación con los otros ancestros: Cecil no tiene herencia repetida. La herencia en Cecil requiere que un hijo acepte todos los campos y métodos definidos en los padres. Estos campos y métodos pueden sobreescribirse en el hijo, pero facilidades como excluir campos o métodos de los padres, o renombrarlos como parte de la herencia, no están presentes en Cecil. Esta es una diferencia importante respecto de los stateful traits.

## 9 Conclusion

Los traits sin estado ofrecen un enfoque composicional simple para estructurar programas orientados a objetos. Un trait es esencialmente un grupo de métodos puros que sirve como bloque de construcción para clases y como unidad primitiva de reuso de código. Sin embargo, este modelo simple sufre de varias limitaciones, en particular (i) la reusabilidad de los traits se ve impactada porque el interfaz requerido típicamente está saturado de accessors requeridos sin interés, (ii) las clases cliente se ven forzadas a implementar código pegamento repetitivo, (iii) la introducción de nuevo estado en un trait propaga accessors requeridos a todas las clases cliente, y (iv) los accessors públicos rompen la encapsulación de la clase cliente.

Se propuso una forma de hacer **stateful** a los traits de la siguiente manera: primero, los traits pueden tener variables privadas. Segundo, las clases o traits compuestos a partir de traits pueden usar el *operador de acceso a variables* para (i) acceder a variables de los traits usados, (ii) atribuir nombres locales a esas variables, y (iii) fusionar variables de múltiples traits usados, cuando esto se desea. La propiedad de flattening puede preservarse alpha-renombrando los nombres de variable que choquen.

Los stateful traits ofrecen numerosos beneficios: no hay propagación innecesaria de métodos requeridos, los traits pueden encapsular su representación interna, y el cliente puede identificar más claramente cuáles son los métodos requeridos esenciales. Ya no es necesario el código pegamento repetitivo duplicado. Un trait encapsula su propio estado, por lo tanto un trait que evoluciona no rompe a sus clientes si su interfaz público permanece sin modificar.

Los stateful traits representan una extensión relativamente modesta a los lenguajes de herencia simple, que habilita la expresión de clases como composiciones de componentes de software reutilizables y de grano fino. Una pregunta abierta para estudio futuro es si la composición de traits puede subsumir a la herencia basada en clases, llevando a un lenguaje de programación basado en composición —siguiendo el diseño de Jigsaw— en lugar de en herencia, como mecanismo primario para estructurar código.

**Agradecimientos.** Los autores agradecen el apoyo financiero de la Swiss National Science Foundation (proyecto "A Unified Approach to Composition and Extensibility", SNF Project No. 200020-105091/1) y de Science Foundation Ireland y Lero (the Irish Software Engineering Research Centre). También agradecen a Nathanael Schärli, Gilad Bracha, Bernd Schoeller, Dave Thomas y Orla Greevy por sus valiosas discusiones y comentarios, y a Ian Joyner por su ayuda con la implementación de Eiffel en MacOSX.

## References

El paper cierra con una lista de 20 referencias bibliográficas. Entre las más relevantes citadas en el cuerpo del texto: Bracha & Cook sobre mixin-based inheritance [BC90]; Bak et al. sobre mixins en Strongtalk [BGG+02]; la tesis de Bracha sobre Jigsaw [Bra92]; Black, Schärli y Ducasse sobre la refactorización de la jerarquía de colecciones de Smalltalk con traits [BSD03]; Cardelli et al. sobre la definición de Modula-3 [CDG+92]; Chambers sobre Cecil [Cha92]; Ducasse, Nierstrasz, Schärli, Wuyts y Black, el paper original de traits en TOPLAS [DNS+06]; Mohnen sobre interfaces con implementación por defecto en Java [Moh02]; Nierstrasz, Ducasse y Schärli sobre "Flattening Traits" [NDS06]; Sweeney y Gil sobre layout de memoria eficiente para herencia múltiple en C++ [SG99]; Snyder sobre encapsulamiento y herencia [Sny86]; Ungar y Smith sobre el lenguaje Self [US87]; y Ungar, Chambers, Chang y Hölzle sobre "Organizing programs without classes" [UCCH91], entre otras referencias a Eiffel, Scala, Python, Squeak y Slate.
