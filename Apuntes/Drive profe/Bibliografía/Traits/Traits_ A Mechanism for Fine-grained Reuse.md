# Traits: A Mechanism for Fine-grained Reuse
Stéphane Ducasse, Oscar Nierstrasz, Nathanael Schärli, Roel Wuyts, Andrew P. Black — ACM TOPLAS (versión extendida del paper ECOOP 2003)

> Resumen detallado en español, organizado siguiendo la estructura original del paper, para poder seguirlo en paralelo con el PDF. No es traducción literal sino una explicación fiel y completa, en mis propias palabras, de cada sección.

## Abstract

La herencia es un mecanismo de reuso bien conocido y aceptado en los lenguajes orientados a objetos. Lamentablemente, debido a la granularidad gruesa de la herencia, puede ser difícil descomponer una aplicación en una jerarquía de clases óptima que maximice el reuso de software. Los esquemas existentes basados en herencia simple, herencia múltiple o mixins plantean numerosos problemas para el reuso. Para superar estos problemas, los autores proponen los **traits**, unidades de reuso puras que consisten únicamente en métodos. Desarrollan un modelo formal de traits que establece cómo pueden componerse, ya sea para formar otros traits o para formar clases. También describen una validación experimental en la que aplican traits para refactorizar una aplicación no trivial en unidades componibles.

## 1 Introduction

La herencia en los lenguajes orientados a objetos está bien establecida como un mecanismo de modificación incremental que puede ser muy efectivo para habilitar el reuso de código entre clases similares. Lamentablemente, la herencia simple es inadecuada para expresar clases que comparten características no heredadas de su (único) padre común. Las características compartidas deben o bien forzarse dentro del padre común (donde no pertenecen), o bien duplicarse en las clases que deberían compartirlas.

Para superar esta limitación, los diseñadores de lenguajes propusieron varias formas de herencia múltiple, así como otros mecanismos, como los **mixins**, que permiten componer clases incrementalmente a partir de conjuntos de características.

A pesar de que pasaron casi veinte años, ni la herencia múltiple ni los mixins lograron una aceptación amplia. Resumiendo la contribución de Alan Snyder al panel sobre herencia de OOPSLA '87, Steve Cook escribió: *"Multiple inheritance is good, but there is no good way to do it"* ("la herencia múltiple es buena, pero no hay una buena forma de hacerla").

No solo la herencia múltiple plantea serios problemas de implementación: a menudo también es inapropiada como mecanismo de reuso. Aunque la herencia múltiple permite reutilizar una clase (o un conjunto de clases), una clase frecuentemente **no** es el elemento que uno desea reutilizar. Esto es porque las clases cumplen dos roles que compiten entre sí. Por un lado, una clase es primariamente un **generador de instancias**; por eso, la mayoría de los lenguajes orientados a objetos recientes, como Java y C#, hacen que cada clase agrupe un conjunto completo de características básicas al requerir que sea (directa o indirectamente) subclase de la clase dedicada `Object`. Por otro lado, una clase tiene un rol secundario como **unidad de reuso**: debería agrupar un conjunto mínimo de características que puedan reutilizarse sensatamente juntas. Desgraciadamente, estos dos roles entran en conflicto. Como las clases deben adoptar una posición fija en la jerarquía de clases, (i) puede ser difícil o imposible factorizar métodos "wrapper" (que extienden otros métodos con funcionalidad adicional) como clases reutilizables, (ii) las características en conflicto heredadas por distintos caminos pueden ser difíciles de resolver, y (iii) las características sobreescritas pueden ser difíciles de acceder o componer. Quizás por estas razones, los diseñadores de lenguajes recientes como Java y C# decidieron que la complejidad introducida por la herencia múltiple supera su utilidad.

Los **Flavors** fueron un primer intento de abordar estos problemas: son implementaciones pequeñas e incompletas de clases que pueden "mezclarse" (*mixed in*) en lugares arbitrarios de la jerarquía de clases. Más tarde se desarrollaron nociones más sofisticadas de mixins (Bracha & Cook; Mens & van Limberghen; Flatt, Krishnamurthi & Felleisen; Ancona, Lagorio & Zucca).

Los mixins usan el operador ordinario de herencia simple para extender varias clases padre con el mismo conjunto de características. Aunque este operador de herencia es adecuado para derivar nuevas clases a partir de clases existentes, no necesariamente es apropiado para componer bloques de construcción reutilizables. Específicamente, porque la composición de mixins se implementa usando herencia, los mixins se componen **linealmente**. Esto da lugar a varios problemas: primero, puede ser difícil, o incluso imposible, encontrar un orden total adecuado de las características. Segundo, el "código pegamento" (*glue code*) que explota o adapta la composición lineal puede quedar dispersado por toda la jerarquía de clases. Tercero, las jerarquías de clases resultantes suelen ser frágiles respecto al cambio, de modo que cambios conceptualmente simples pueden impactar muchas partes de la jerarquía. Por estas razones, los autores creen que los mixins nunca lograron un éxito amplio en los lenguajes orientados a objetos predominantes.

Los **traits** representan una solución simple a estos diversos dilemas. En pocas palabras, un trait es un conjunto de métodos, divorciado de cualquier jerarquía de clases. Los traits pueden componerse en cualquier orden. La entidad compuesta tiene control completo sobre la composición y puede resolver conflictos explícitamente, sin recurrir a la linearización. Las clases se organizan en una única jerarquía de herencia simple, y pueden usar traits puramente para especificar la diferencia incremental de comportamiento respecto de sus superclases.

Este modelo simple tiene las siguientes consecuencias:

- Los dos roles quedan claramente separados: los traits son puramente unidades de reuso, y las clases son generadores de instancias.
- Los traits son componentes de software simples que tanto proveen como requieren métodos (los métodos requeridos son aquellos usados por, pero no implementados en, un trait).
- Las clases se componen a partir de traits, resolviendo en el proceso cualquier conflicto y posiblemente proveyendo los métodos requeridos.
- Los traits no especifican estado, así que el único conflicto que puede surgir al combinar traits es un conflicto de métodos. Tal conflicto puede resolverse sobreescribiendo o excluyendo.
- Los traits pueden "inlinearse" (*inlined*), un proceso que los autores llaman **"flattening"** (aplanado): el hecho de que un método se origine en un trait en lugar de en una clase no afecta la semántica de la clase.
- Las dificultades experimentadas con la herencia múltiple desaparecen con los traits, porque los traits están divorciados de la jerarquía de herencia.
- Las dificultades experimentadas con los mixins también desaparecen, porque los traits no imponen ningún orden de composición.

Este paper extiende un trabajo anterior de los mismos autores presentando una explicación precisa y formal de los traits, un análisis más detallado de los problemas asociados a los distintos mecanismos de herencia múltiple y mixin, y una discusión más extensa del trabajo relacionado.

**Estructura del paper:** la sección 2 da una visión general de los problemas que surgen con la herencia múltiple y los mixins; la sección 3 muestra en detalle cómo estos problemas afectan a lenguajes de programación existentes, ilustrando su uso con numerosos ejemplos. La sección 4 presenta los traits formalmente y demuestra cómo resuelven estos problemas al especificar clases. La sección 5 discute las decisiones de diseño más importantes y evalúa los traits respecto a los problemas identificados en la sección 2, mientras que la sección 6 presenta la implementación de traits. La sección 7 resume los resultados de aplicaciones realistas de traits: una refactorización de la jerarquía de colecciones de Smalltalk, una refactorización del núcleo del lenguaje Smalltalk bootstrapeándolo con traits, y una aplicación de traits para abordar el problema de la composición segura de metaclases. La sección 8 discute trabajo relacionado. El paper concluye e indica trabajo futuro en la sección 9.

## 2 Problems with Inheritance and Composability

Aunque la herencia es ampliamente considerada la característica clave de la programación orientada a objetos, también está cargada de definiciones e interpretaciones contradictorias y en competencia. A lo largo de los años, los investigadores desarrollaron varias formas de herencia: simple, múltiple y de mixin. Cada una de estas formas da respuestas distintas a los problemas de **descomposición** (cómo descomponer una base de software en unidades de reuso adecuadas) y **composición** (cómo componer esas unidades para obtener una jerarquía de clases adecuada para el dominio de la aplicación).

### 2.1 Decomposition Problems

La programación orientada a objetos ofrece una herramienta para modelar dominios arbitrarios como jerarquías de clases. Pero la forma en que descomponemos los conceptos de un dominio en clases no es necesariamente la forma correcta de descomponer las implementaciones de esas clases en conjuntos de características. Los autores consideran brevemente tres problemas de descomposición:

**Características duplicadas.** La herencia simple es la forma más simple de herencia: permite que una clase herede de (a lo sumo) una superclase. Aunque está bien aceptada, la herencia simple no es suficientemente expresiva para factorizar todas las características comunes compartidas por clases en una jerarquía compleja. Como ejemplo, las clases de stream de Smalltalk `ReadStream`, `WriteStream` y `ReadWriteStream`: como sugieren sus nombres, la clase `ReadWriteStream` contiene características provistas tanto por `ReadStream` como por `WriteStream`. Sin embargo, la herencia simple permite que `ReadWriteStream` herede de solo una de estas clases. En Smalltalk, `ReadWriteStream` hereda de `WriteStream` y luego duplica algunos métodos de `ReadStream`. (Nótese que la extensión de la herencia simple con interfaces, como la promueven Java y C#, aborda los problemas de subtipado y modelado conceptual, pero no ayuda en nada con el problema del código duplicado.)

**Jerarquías inapropiadas.** Una forma común de evitar esta duplicación de código es implementar ciertos métodos "demasiado arriba" en la jerarquía. La idea es que, en lugar de duplicar un método, se lo mueve a una superclase hasta que esté disponible en todas las clases donde realmente se lo necesita. En el ejemplo, esto significa que el programador podría implementar todos los métodos de lectura en la clase `PositionableStream`, que es la superclase común más baja de `ReadStream` y `WriteStream`. Como consecuencia, estos métodos se heredarían en la clase `ReadWriteStream`, y por lo tanto no necesitarían duplicarse. La táctica funciona, pero el precio es alto: `PositionableStream` queda contaminada con muchos métodos que nada tienen que ver con el posicionamiento, y `WriteStream` aparenta implementar muchos métodos de lectura, aunque estos métodos fallarán o producirán un comportamiento inconsistente si alguna vez se los usa.

Tanto la herencia múltiple como los mixins intentan aliviar estos problemas permitiendo que una clase obtenga características de múltiples fuentes, pero, como veremos, cada uno da lugar a otros problemas.

**Wrappers duplicados.** La herencia múltiple, tal como la proveen lenguajes como C++ y Eiffel, permite que una clase reutilice características de múltiples clases padre, pero no permite escribir una entidad reutilizable que extienda métodos implementados en clases todavía desconocidas.

Esta limitación se ilustra con un ejemplo: supongamos que una clase `A` contiene métodos `read` y `write:` que dan acceso no sincronizado a algunos datos. Si se vuelve necesario sincronizar el acceso, se puede crear una clase `SyncA` que hereda de `A` y envuelve los métodos `read` y `write:`: es decir, `SyncA` define nuevos métodos `read` y `write:` que llaman a los métodos heredados bajo el control de un lock.

Supongamos ahora que la clase `A` es parte de un framework que también contiene otra clase `B` con métodos `read` y `write:`, y que se quiere usar la misma técnica para crear una versión sincronizada de `B`. Naturalmente, sería deseable factorizar el código de sincronización para reutilizarlo tanto en `SyncA` como en `SyncB`.

Con herencia múltiple, la forma natural de compartir código entre clases distintas es heredar de una superclase común. Esto significa que habría que mover el wrapper de sincronización a una clase `SyncReadWrite` que sería la superclase de tanto `SyncA` como `SyncB`. Lamentablemente, esto no funciona porque los envíos a `super` se resuelven estáticamente en la mayoría de las formas de herencia múltiple, como las de C++ y Eiffel. Por lo tanto, los envíos a `super` en los métodos de `SyncReadWrite` se referirían estáticamente a métodos de **su propia** superclase, y no a métodos de `A` o `B`.

Las soluciones alternativas son torpes y solo implican más código duplicado: por ejemplo, las llamadas a super en `SyncReadWrite` podrían reemplazarse por llamadas a métodos abstractos `directRead` y `directWrite:`, que luego se implementarían tanto en `SyncA` como en `SyncB` para llamar, respectivamente, a los métodos `read` y `write:` de `A` y de `B`.

Los **mixins** resuelven este problema particular mediante un `super` de ligadura tardía (*late-binding*). Un mixin es una especificación de subclase abstracta que puede aplicarse a varias clases padre para extenderlas con el mismo conjunto de características. En lugar de definir `SyncReadWrite` como una clase, se la define como un mixin. Entonces `SyncA` y `SyncB` aplicarán cada uno el mixin a una superclase distinta, obteniendo el comportamiento de wrapper deseado.

### 2.2 Composition Problems

Aunque hay una progresión clara en poder expresivo desde la herencia simple, pasando por la herencia múltiple, hasta los mixins, esta expresividad no viene sin costo. Tanto la herencia múltiple como los mixins plantean numerosos problemas cuando se considera cómo se componen las clases a partir de características compartidas.

**Características en conflicto.** Uno de los problemas de la herencia múltiple es la ambigüedad que surge cuando características en conflicto se heredan por caminos distintos. Una situación particularmente problemática es el "problema del diamante" (también llamado "herencia fork-join"), que ocurre cuando una clase hereda de la misma clase padre por múltiples caminos. La raíz de una jerarquía de clases (o de una sub-jerarquía) a menudo consiste en una clase que provee algún comportamiento por defecto común que puede ser sobreescrito por subclases (por ejemplo, métodos `=`, `hash`, y `asString`): esto es precisamente la causa de los conflictos que surgen cuando se reutilizan varias de estas clases.

Las características que entran en conflicto pueden ser métodos o atributos (variables de instancia). Mientras que los conflictos de métodos pueden resolverse relativamente fácil (por ejemplo, sobreescribiendo), los atributos en conflicto son más problemáticos. Aún si las declaraciones son consistentes, no queda claro si los atributos en conflicto deberían heredarse una vez o múltiples veces, ni cómo deberían inicializarse esos atributos.

La herencia simple no sufre este problema; tampoco lo sufren los mixins basados en herencia simple. Con la composición de mixins, los mixins se aplican a las clases de una en una, generando nuevas subclases en una jerarquía de herencia simple. No surgen conflictos porque las características de cada mixin simplemente extienden o sobreescriben las de la clase a la que se aplica. Sin embargo, el hecho de que los mixins deban aplicarse en un orden particular lleva a otros problemas, como veremos a continuación.

**Falta de control y dispersión del código pegamento.** La composición de mixins es **lineal**: todos los mixins usados por una clase deben heredarse uno a la vez. Las características definidas en mixins que aparecen más tarde en el orden sobreescriben todas las características con el mismo nombre de mixins anteriores. Cuando los conflictos deberían resolverse seleccionando y combinando características de distintos mixins, puede no existir un orden total adecuado. Así, aunque se evita el problema de los conflictos ambiguos, los mixins meten la composición de características en una camisa de fuerza de la que puede ser difícil escapar.

Como consecuencia, con los mixins, la entidad compuesta **no tiene control total** sobre la forma en que se componen los mixins: en cambio, la forma en que las características individuales se sobreescriben y extienden mutuamente está dictada por el orden total impuesto sobre los mixins. Obtener la combinación deseada de características puede requerir introducir código pegamento en nuevos mixins intermedios, o incluso modificar los mixins componentes. Ninguna de las dos alternativas es satisfactoria. Modificar un mixin es problemático porque puede romper otras clases que usan ese mixin; introducir mixins intermedios hace que el código pegamento quede disperso por toda la jerarquía de herencia, lo que hace que la composición sea difícil de entender y adaptar.

Como ejemplo, los autores consideran una clase `MyRectangle` que usa dos mixins, `MColor` y `MBorder`, que cada uno provee métodos `asString` y `serializeOn:`. Las implementaciones de `asString` en los mixins primero llaman a la implementación heredada de la superclase (usando la palabra clave `super`) y luego extienden el string resultante con un carácter separador seguido de información específica sobre su propio estado.

Supongamos ahora que, por razones de compatibilidad, se necesita serializar la clase `MyRectangle` de modo que el valor `rgb` aparezca antes que `borderWidth`. Porque la composición de mixins es lineal, esto significa que el mixin `MColor` debe aplicarse antes que el mixin `MBorder`. Desgraciadamente, esto también significa que el orden de los métodos `asString` también cambia, así que los atributos de color se imprimirán antes que los atributos de borde, lo que puede no ser lo que se desea.

El núcleo del problema es que, desde dentro de la entidad compuesta `MyRectangle`, no es posible controlar separadamente cómo se componen las distintas características. Esto es porque en `MyRectangle` solo se puede acceder al comportamiento mezclado de `Rectangle + MColor + MBorder`, pero no al comportamiento original de `MColor` y `Rectangle` por separado.

Así, si se necesita personalizar cómo se componen las características —sea porque se necesita un orden distinto de serialización, o un carácter separador diferente entre los dos strings— hay que modificar los mixins involucrados, lo cual es problemático porque potencialmente rompe a todos los demás clientes de esos mixins.

Vale notar que los **mixins compuestos** (propuestos por Bracha) no dan más control sobre la composición que los mixins ordinarios. Igual que los mixins, los mixins compuestos solo proveen una composición lineal en la que todas las características de los mixins involucrados quedan totalmente ordenadas. Esto significa que, en el ejemplo anterior, ocurriría el mismo problema si se combinaran los dos mixins `MColor` y `MBorder` para crear un mixin compuesto `MColorAndBorder` y luego se usara ese mixin compuesto para definir la nueva clase `MyRectangle`. La única diferencia es que el problema se manifestaría durante la definición del mixin compuesto en lugar de durante la definición de la clase `MyRectangle`.

**Jerarquías frágiles.** El orden total de los mixins puede llevar a un problema adicional: las jerarquías de herencia resultantes a menudo son frágiles respecto al cambio. Agregar un método nuevo a uno de los mixins sobreescribe implícitamente todos los métodos con el mismo nombre de mixins que aparecen antes en la cadena. Además, puede ser imposible reestablecer el comportamiento original del compuesto sin agregar o cambiar varios mixins en la cadena de herencia. Esto es especialmente crítico si se modifica un mixin que se usa en muchos lugares de la jerarquía de clases.

Como ilustración, supongamos que en el ejemplo anterior el mixin `MBorder` no define inicialmente un método `asString`. Esto significa que la implementación de `asString` en `MyRectangle` será la especificada por `MColor`. Supongamos que, más tarde, se agrega el método `asString` al mixin `MBorder`. Debido al orden total de los mixins, esto sobreescribe implícitamente la implementación provista por `MColor`. Peor aún, el comportamiento original de la clase compuesta `MyRectangle` no se puede reestablecer sin cambiar más de los mixins involucrados en la composición. Si se quiere evitar el efecto dominó causado por cambios futuros a los mixins existentes, hay que introducir un nuevo "mixin pegamento" entre los mixins `MColor` y `MBorder`, que haga disponible el método `asString` provisto por `MColor` bajo un nuevo nombre como `colorAsString`, y luego agregar otro método pegamento `asString` a la clase `MyRectangle`.

Con muchas formas de herencia múltiple también se observa un problema de fragilidad similar respecto a los cambios. Como las características con el mismo nombre pueden heredarse desde distintas clases padre, una sola palabra clave (por ejemplo, `super`) no es suficiente para acceder a los métodos heredados sin ambigüedad. Por ejemplo, C++ obliga a nombrar la superclase apropiada para acceder a un método sobreescrito; versiones recientes de Eiffel adoptan una técnica análoga. Aunque nombrar explícitamente la fuente de las características sobreescritas hace posible componer características de múltiples clases padre, este "embebido" de referencias explícitas a clases en el código fuente hace que el código sea frágil respecto a cambios en la jerarquía de clases.

## 3. Detailed Discussion of the Problems

La sección anterior dio una visión general de los problemas. Esta sección los ilustra con mayor detalle en lenguajes de programación concretos.

### 3.1 Mixins in Strongtalk and Jam

Strongtalk y Jam son extensiones de Smalltalk y Java respectivamente que incorporan mixins. Ambos sufren las limitaciones del orden total impuesto por la composición de mixins.

Tomando el ejemplo de `MyRectangle` con los mixins `MColor` y `MBorder` (ver sección 2.2), si se necesita serializar en un orden particular (`rgb` antes que `borderWidth`), hay que aplicar `MColor` antes que `MBorder`. Pero esto también afecta el orden de `asString`, algo que puede no ser lo deseado: las dos preocupaciones están acopladas.

El problema de raíz es que los métodos `asString` de los mixins hacen dos cosas a la vez: imprimen su estado propio **y** llaman a la implementación heredada (`super`). Separar estas dos responsabilidades requeriría modificar los mixins, lo que puede romper otros clientes.

En Strongtalk se podría separar la parte de impresión de estado propia:

```smalltalk
MColor>>asString
    ↑ self color asString

MBorder>>asString
    ↑ self borderWidth asString
```

Pero luego, para que `MyRectangle` tenga acceso a los tres métodos `asString` (el de `Rectangle`, el de `MColor`, y el de `MBorder`), es necesario interponer dos "glue mixins" adicionales (`MGlue1` y `MGlue2`) en la cadena de herencia. `MGlue1` captura `Rectangle>>asString` bajo el nombre `rectAsString`, y `MGlue2` captura `MColor>>asString` bajo el nombre `colorAsString`. Finalmente, `MyRectangle` define su propio `asString` combinando los tres:

```smalltalk
MyRectangle>>asString
    ↑ self rectAsString, ' ', super asString, ' ', self colorAsString.
```

El resultado es que la jerarquía de herencia involucra seis entidades distintas para entender la composición de `MyRectangle`. El código pegamento queda disperso en varios niveles, dificultando la comprensión del programa. JAM sufre exactamente los mismos problemas.

### 3.2 Multiple Inheritance and Mixins in C++

C++ es único en permitir tanto herencia múltiple nativa como la simulación de mixins mediante templates.

**Herencia múltiple en C++.** Una característica distintiva es que si una clase base se declara virtual, el diamante se comparte y los atributos se heredan una sola vez. Esto ayuda con los conflictos de atributos, pero no resuelve el problema de los wrappers genéricos.

Retomando el ejemplo de `SyncA`: en C++ se escribe directamente con referencias calificadas al scope, como `A::read()`, en lugar de `super`:

```cpp
class SyncA : public A {
public:
    virtual int read() {
        acquireLock();
        result = A::read();
        releaseLock();
        return result;
    }
    virtual void write(int n) {
        acquireLock();
        A::write(n);
        releaseLock();
    }
    // ...
};
```

Si se quiere compartir el código de sincronización en una clase `SyncReadWrite`, los `super`-sends se resuelven estáticamente y solo pueden referir a la superclase propia de `SyncReadWrite`, no a `A` ni `B`. La solución con métodos abstractos `directRead` y `directWrite` (Fig. 5 del paper) termina duplicando código en `SyncA` y `SyncB`.

**Mixins basados en templates en C++.** El mecanismo de templates permite expresar una clase con superclase genérica, lo que equivale a un mixin con late-binding de `super`:

```cpp
template <class Super>
class MSyncReadWrite : public Super {
public:
    virtual int read() {
        acquireLock();
        result = Super::read();
        releaseLock();
        return result;
    }
    // ...
};

class SyncA : public MSyncReadWrite<A> {};
class SyncB : public MSyncReadWrite<B> {};
```

Esto resuelve el problema de los wrappers duplicados, pero cuando se componen múltiples mixins (`MSyncReadWrite` con `MLogOpenClose`), vuelven a aparecer los problemas de linealización: el programador debe elegir un orden, y ese orden puede no ser el correcto para todas las características a la vez.

C++ ofrece una tercera opción al permitir acceder a superclases indirectas con calificadores de scope anidados (por ejemplo `MSyncReadWrite::MLogOpenClose::reset()`), pero esto hace el código extremadamente frágil y difícil de mantener, ya que la clase depende de los detalles internos de toda la jerarquía.

### 3.3 CLOS

A diferencia de C++ y Eiffel, la herencia múltiple de CLOS impone un orden lineal sobre las superclases. Esto tiene la ventaja de que la palabra clave `call-next-method` alcanza para llamar al método del siguiente en la cadena sin ambigüedad, y los super-sends se resuelven dinámicamente, lo que permite wrappers genéricos.

Sin embargo, la linearización de CLOS genera problemas similares a los de los mixins: a menudo no es claro cómo debería linearizarse una jerarquía de herencia múltiple compleja, y el comportamiento resultante puede ser sorpresivo. Otras linearizaciones usadas en Lisp y sus derivados (Loops, Dylan, C3) no proveen soluciones fundamentales: siguen favoreciendo la resolución automática de conflictos, lo que puede ocultar errores.

## 4. Traits — Composable Units of Behavior

Los traits ofrecen una solución simple a los problemas planteados. Las clases retienen su rol primario de generadoras de instancias, organizadas en una única jerarquía de herencia simple. Los traits son puramente unidades de reuso.

Los traits están implementados en Squeak (un dialecto Smalltalk) y fueron adoptados como característica estándar de Scala. También hay implementaciones para Perl 5 y trabajo en curso para C# y Perl 6 (bajo el nombre "roles").

La composición de traits respeta tres reglas fundamentales:
- Los métodos definidos directamente en la clase tienen precedencia sobre los provistos por un trait.
- Propiedad de flattening: un método no sobreescrito en un trait tiene la misma semántica que si estuviera implementado directamente en la clase.
- El orden de composición es irrelevante: todos los traits tienen la misma precedencia, y los conflictos deben resolverse explícitamente.

Un conflicto surge cuando dos o más traits proveen métodos con el mismo nombre que no provienen del mismo trait original. Se resuelve implementando un glue method en la clase (que sobreescribe los conflictivos) o excluyendo uno de ellos.

### 4.1 Classes and Methods

El modelo formal parte de tres conjuntos disjuntos primitivos:
- `N`: nombres de métodos,
- `B`: cuerpos de métodos,
- `A`: nombres de atributos (variables de instancia).

Para representar conflictos y métodos requeridos, se extiende `B` a un retículo plano `B*` agregando dos elementos especiales:
- `⊥` (bottom): método requerido (definido pero no implementado en el trait).
- `⊤` (top): método en conflicto.

El operador join `⊔` combina dos cuerpos: si son el mismo, da ese cuerpo; si son distintos, da `⊤`.

**Definición 1 — Método:** una función parcial que mapea un nombre `a ∈ N` a un cuerpo `m ∈ B`. Notación: `a ↦ m`.

**Definición 2 — Diccionario de métodos:** función total `d: N → B*` que mapea solo un subconjunto finito de nombres a cuerpos en `B` (sin mapear nada a `⊤`). Por ejemplo:

```
d = {a ↦ m₁, b ↦ m₂}
```

Para modelar el comportamiento de `self` y `super`, se definen:
- `selfSends(m)`: conjunto de nombres de métodos usados en self-sends del cuerpo `m`.
- `superSends(m)`: conjunto de nombres de métodos usados en super-sends del cuerpo `m`.

**Definición 3 — Clase:** una clase `c ∈ C` es la clase vacía `nil`, o una secuencia `⟨α, d⟩·c'` con atributos `α ⊂ A`, diccionario de métodos `d ∈ D`, y superclase `c' ∈ C`.

Ejemplo: la clase `c = ⟨{i}, {a ↦ m₂, b ↦ m₃}⟩·⟨∅, {a ↦ m₁}⟩·nil` tiene atributo `i` y método `a ↦ m₂` que sobreescribe `a ↦ m₁` en la superclase. El método `b` usa `super`, así que la semántica de `a ↦ m₂` depende de la cadena de herencia.

### 4.2 Traits

Un trait modela la extensión de un diccionario de métodos donde algunos métodos pueden estar en conflicto.

**Definición 4 — Trait:** una función `t: N → B*` donde `t⁻¹(B ∪ {⊤})` es finito.

Un trait provee métodos (mapeados a cuerpos en `B`) y puede tener conflictos (mapeados a `⊤`) o métodos requeridos (mapeados a `⊥`). Todo diccionario de métodos es un trait libre de conflictos; `D ⊂ T`.

**Definición 5 — Conflictos de un trait:**
```
conflicts(t) = {l | t(l) = ⊤}
```

**Definición 6 — Métodos provistos:**
```
provided(t) = t⁻¹(B)
```
Es el conjunto de nombres que `t` mapea a cuerpos reales (ni `⊥` ni `⊤`).

**Definición 7 — Métodos requeridos:**
```
required(t) = selfSends(t) \ provided(t)
```
Los requeridos son los que el trait usa con `self` pero no implementa él mismo. Los super-sends no se cuentan aquí; se verifican al ensamblar la clase.

Los traits no pueden especificar estado. Acceden a estado de forma indirecta a través de required methods que funcionan como accessors provistos por la clase que usa el trait.

### 4.3 Composing Classes from Traits

La relación entre clases y traits se resume en la ecuación:

```
Clase = Superclase + Estado + Traits + Glue methods
```

Una clase derivada de una superclase agrega atributos (estado), usa traits para obtener comportamiento reutilizable, e implementa glue methods que:
- proveen los required methods de los traits (generalmente como accessors de atributos),
- resuelven conflictos entre traits,
- adaptan el comportamiento según sea necesario.

Formalmente, una clase compuesta desde traits tiene la forma:

```
⟨α, d ▷ t⟩·c'
```

donde `t` es un trait (o composición de traits) y `d` es un diccionario de métodos que puede sobreescribir y extender a `t`. El operador `▷` (override) define cómo `d` reemplaza métodos de `t`.

**Definición 8 — Suma de traits (`+`):** la unión de métodos no conflictivos y la marcación de conflictos para los que se superponen con distintos cuerpos:

```
(t₁ + t₂)(l) = t₁(l) ⊔ t₂(l)
```

**Proposición 1:** La suma de traits es asociativa y conmutativa.

Ejemplo:
```
{a ↦ m₁, b ↦ m₂, c ↦ m₃} + {a ↦ m₁, b ↦ m₄} = {a ↦ m₁, b ↦ ⊤, c ↦ m₃}
```
El método `a` no conflictúa porque proviene del mismo cuerpo `m₁`. El método `b` sí conflictúa porque tiene cuerpos distintos (`m₂` vs `m₄`).

**Definición 9 — Override (`▷`):** un diccionario `d` sobreescribe un trait `t`:

```
(d ▷ t)(l) = t(l)   si d(l) = ⊥
             d(l)   en otro caso
```

El override es el mecanismo principal para resolver conflictos: `d` puede proveer implementaciones para los métodos en conflicto o requeridos de `t`.

**Propiedad de flattening:** Si `c = ⟨α, d ▷ t⟩·c'` está bien definida y `d' = d ▷ t` es libre de conflictos, entonces `c = ⟨α, d'⟩·c'` es una definición equivalente que no usa traits. Es decir, los traits pueden "inlinearse" sin cambiar la semántica de la clase.

El flattening implica además:
- Los métodos de clase tienen precedencia sobre los del trait (por la definición de `▷`).
- Los métodos del trait tienen precedencia sobre los de la superclase (por el flattening, el trait se "incorpora" a la clase antes de aplicar herencia).
- `super` en un trait tiene la misma semántica que si el método estuviera directamente en la clase: busca en la superclase de la clase que usa el trait.

**Ejemplo de running example — streams:** Se construye una librería de streams usando los traits `TReadStream`, `TWriteStream` y `TSynchronize`. En Squeak (los nombres de traits empiezan con `T`, los de clases no):

```smalltalk
Trait named: #TReadStream uses: {}

    atStart
        ↑ self position = self minPosition.

    atEnd
        ↑ self position >= self maxPosition.

    setToEnd
        self position: self maxPosition.

    setToStart
        self position: self minPosition.

    maxPosition
        ↑ self collection size.

    on: aCollection
        self collection: aCollection.
        self setToStart.

    next
        ↑ self atEnd
            ifTrue: [nil]
            ifFalse: [self collection at: self nextPosition].

    minPosition
        ↑ 0.

    nextPosition
        self position: self position + 1.
        ↑ self position.

    "required methods (en itálica en el paper):"
    "collection, collection:, position, position:"
```

La clase `ReadStream` usa ese trait y provee los required methods como accessors de sus variables de instancia:

```smalltalk
Object subclass: #ReadStream
    instanceVariableNames: 'position collection'
    uses: TReadStream

    initialize
        self collection: String new

    position       ↑ position.
    position: aNumber   position := aNumber.
    collection     ↑ collection.
    collection: aCollection  collection := aCollection.
```

### 4.4 Composite Traits

Los traits pueden componerse de otros traits para factorizar comportamiento compartido. A diferencia de las clases, la mayoría de los traits no están completos: no satisfacen todos sus propios required methods. Los requerimientos no satisfechos de subtraits simplemente se propagan como requerimientos del trait compuesto.

**Definición 10 — Trait compuesto:** expresión de la forma `d ▷ t` donde `d ∈ D` y `t` es una cláusula de composición usando `+` (suma), `→` (alias) y `−` (exclusión).

Ejemplo: los traits `TReadStream` y `TWriteStream` comparten muchos métodos (posicionamiento). Se factoriza el comportamiento común en `TPositionableStream`:

```
TReadStream  usa TPositionableStream y agrega: on:, next
TWriteStream usa TPositionableStream y agrega: on:, nextPut:
```

```smalltalk
Trait named: #TReadStream uses: TPositionableStream

    on: aCollection
        self collection: aCollection.
        self setToStart.

    next
        ↑ self atEnd
            ifTrue: [nil]
            ifFalse: [self collection at: self nextPosition].
```

La propiedad de flattening sigue siendo válida incluso con múltiples niveles de composición: la semántica de un método no depende de si está definido en un trait o en entidades que usan ese trait.

### 4.5 Conflict Resolution

Un conflicto surge si y solo si se componen dos traits que proveen métodos con el mismo nombre pero con **distintos cuerpos**. Si el mismo cuerpo llega por múltiples caminos, no hay conflicto (excepción del mismo método).

Los conflictos se resuelven **explícitamente** en la cláusula de composición de la clase o del trait compuesto, mediante:

1. **Override:** definir un glue method en la clase que sobreescribe los métodos conflictivos.
2. **Exclusion:** eliminar uno de los métodos conflictivos.
3. **Aliasing:** crear un nombre alternativo para un método antes de que sea sobreescrito, para que pueda seguir siendo accedido.

**Definición 11 — Alias (`t[a→b]`):** introduce un nombre adicional `b` para el método `a` del trait `t`:

```
t[a→b](l) = t(l)         si l ≠ a
             t(b)         si l = a y t(a) = ⊥
             ⊤            en otro caso
```

Ejemplo:
```
{a ↦ m₁, b ↦ m₂}[c→b] = {a ↦ m₁, b ↦ m₂, c ↦ m₂}
```

Nota: `{a ↦ m₁, b ↦ m₂}[a→b] = {a ↦ ⊤, b ↦ m₂}` — intentar crear un alias con un nombre ya ocupado introduce un conflicto.

**Definición 12 — Exclusion (`t − a`):** elimina el método `a` del trait `t`:

```
(t − a)(l) = ⊥    si a = l
              t(l)  en otro caso
```

Ejemplo:
```
{a ↦ m₁, b ↦ ⊤} − b = {a ↦ m₁}
```

**Ejemplo concreto — TSyncReadStream:** El trait `TSyncReadStream` sincroniza el acceso a `next`. Se construye como la composición de `TSynchronize` y `TReadStream`, con el alias `readNext` para el método `next` original de `TReadStream`:

```smalltalk
Trait named: #TSynchronize uses: {}

    acquireLock
        self semaphore wait.

    releaseLock
        self semaphore signal.

    initialize
        self semaphore: Semaphore new.
        self releaseLock.

    "required: semaphore, semaphore:"
```

```smalltalk
Trait named: #TSyncReadStream
    uses: TSynchronize + (TReadStream @ {#readNext -> #next})

    next
        | read |
        self acquireLock.
        read := self readNext.
        self releaseLock.
        ↑ read.
```

El operador `@` crea el alias `readNext` para el método `next` de `TReadStream`. Luego `next` se sobreescribe en `TSyncReadStream` para que adquiera el lock, llame al método original por el alias, y libere el lock.

**Ejemplo de conflicto — ReadWriteStream:** La clase `ReadWriteStream` usa `TReadStream` y `TWriteStream`, que ambos proveen su versión del método `on:`. Se resuelve excluyendo uno:

```smalltalk
Stream subclass: #ReadWriteStream
    uses: (TReadStream − {#on:}) + TWriteStream
```

Como `TReadStream` y `TWriteStream` se componen ambos desde `TPositionableStream`, todos los métodos originarios de este último llegan idénticos por ambos caminos y no crean conflicto.

### 4.6 Well-definedness

**Definición 13 — Diccionario de una clase `c` (`dict(c)`):** la composición `▷` de los diccionarios (aplanados) en la cadena de herencia:

```
dict(nil) = {}
dict(⟨α, d⟩·c') = d ▷ dict(c')
```

**Definición 14 — Clase válida:** `c` es válida si `conflicts(dict(c)) = ∅`.

**Definición 15 — Method lookup (`c ≫ a`):** `dict(c)(a)`.

**Definición 19 — Clase bien fundamentada (well-founded):** `c` es well-founded si todos los super-sends en su incremento (`delta(c)`) están ligados, es decir, si `superSends(delta(c)) ⊆ provided(super(c))` y `super(c)` es well-founded.

**Definición 20 — Clase bien definida:** `c` es bien definida si es válida y well-founded.

**Proposición 2 (Flattening property, formal):** Si `c = ⟨α, d ▷ t⟩·c'` está bien definida y `d' = d ▷ t`, entonces `c = ⟨α, d'⟩·c'` es una definición equivalente y aplanada de `c`.

### 4.7 Refactoring, Reachability and Equivalence

La propiedad de flattening garantiza que la semántica de una clase no cambia cuando se reescribe como composición de traits. Pero no alcanza para razonar sobre la equivalencia de clases cuando se refactoriza una jerarquía entera.

Para eso se introduce el concepto de **alcanzabilidad**: qué cuerpos de métodos pueden alcanzarse mediante cadenas de self-sends y super-sends desde una clase.

**Definición 21 — `c↑ā`:** la secuencia de resoluciones de mensajes desde la clase `c` siguiendo la cadena `ā = a₁a₂...aₙ`. Si `aₙ` puede resolverse, da `⟨m, c'⟩` donde `m` es el cuerpo y `c'` la clase en la que se encontró.

**Definición 22 — Reachability:** un cuerpo `m ∈ B` es alcanzable desde `c` si existe alguna secuencia `ā` tal que `c|ā = m`.

**Definición 23 — Conjunto de alcanzabilidad:**
```
reachable(c) = {⟨ā, c|ā⟩ | ā ∈ N⁺, c|ā ≠ ⊥}
```

**Definición 24 — Equivalencia de clases:** `c ≡ c'` si y solo si `reachable(c) = reachable(c')`.

**Proposición 4:** `c ≡ c' ⇒ provided(c) = provided(c')`.

Con esta definición se puede mostrar que la jerarquía refactorizada con traits (Fig. 18 del paper) es equivalente a la original (Fig. 17): `ReadStream ≡ ReadStream'`, `WriteStream ≡ WriteStream'`, etc.

Una clase es **concreta** si `required(c) = ∅` (todos sus self-sends están provistos); abstracta en caso contrario.

## 5. Discussion and Evaluation

Esta sección evalúa los traits respecto a los problemas de la sección 2 y discute las decisiones de diseño más importantes.

### 5.1 Evaluation Against the Identified Problems

**Características duplicadas.** El código duplicado puede factorizarse fácilmente en traits únicos y luego componerse en clases arbitrarias, independientemente de su posición en la jerarquía. El ejemplo de la jerarquía de streams (Fig. 17 vs Fig. 18) muestra cómo la duplicación del método `next` en `ReadStream` y `ReadWriteStream` desaparece al introducir el trait `TReadStream`.

**Jerarquías inapropiadas.** Con trait composition como mecanismo primario de reuso de grano fino, la jerarquía de herencia queda libre para capturar conformancia y relaciones conceptuales. El programador puede mover métodos reutilizables a traits y aplicarlos solo a las clases donde son apropiados.

**Wrappers duplicados.** Los wrappers genéricos como el de sincronización se expresan directamente con traits: `super` en un trait refiere a la superclase de la clase que **usa** el trait, no a la superclase del trait en sí. Si `SyncA` y `SyncB` usan el trait `TSyncReadWrite`, los super-sends del trait se resolverán correctamente contra `A` y `B` respectivamente al momento de componer.

**Características en conflicto.** Los traits evitan completamente los conflictos de estado, porque los traits no pueden definir estado. Los conflictos de métodos se resuelven explícitamente: excluyendo uno de los métodos conflictivos, o sobreescribiéndolo con un glue method. En general surgen menos conflictos con traits que con herencia múltiple, porque en la práctica los traits tienden a ser pequeños y enfocados.

**Falta de control y dispersión del código pegamento.** La suma de traits es asociativa y conmutativa, por lo que el orden de composición es irrelevante. La entidad compuesta tiene control total sobre la composición: puede decidir independientemente cómo resolver cada conflicto. El glue code siempre reside en la entidad compuesta, no disperso en la jerarquía. Retomando el ejemplo de `MyRectangle` con los traits `TColor` y `TBorder`:

```smalltalk
Rectangle subclass: #MyRectangle
    uses: TColor + TBorder

    asString
        ↑ self rectAsString, ' ', self colorAsString, ' ', self borderAsString.
```

Todo el glue code está en un solo lugar, `MyRectangle`, y puede controlarse de forma independiente.

**Jerarquías frágiles.** Si un trait T agrega un nuevo método, el único efecto visible para sus clientes directos es que puede introducir un nuevo conflicto. Gracias a la conmutatividad y la resolución explícita de conflictos, este nuevo conflicto no puede propagarse implícitamente: el cliente directo lo detecta y puede resolverlo exclusivamente a nivel local (excluyendo el nuevo método) sin necesidad de cambiar otros traits ni de introducir glue mixins adicionales.

### 5.2 Design Decisions

**Separar reutilización de clases.** Aunque los traits están inspirados en mixins, son un concepto nuevo: se componen con operadores distintos a la herencia simple, y no pueden definir estado. Como los mixins, son unidades de reuso más finas que las clases y no están atadas a un lugar fijo en la jerarquía. Esta separación de roles (traits = reuso, clases = generadoras de instancias) mejora el reuso y el modelado conceptual.

**Herencia simple y flattening.** Se eligió conservar la herencia simple en lugar de reemplazarla. Herencia simple y trait composition son complementarias. La herencia simple permite reutilizar todas las características de una clase (métodos y atributos). Con herencia simple, no hay conflictos de atributos y `super` es suficiente para acceder a métodos sobreescritos sin ambigüedad. La propiedad de flattening además provee una ruta de migración suave: incluso con cientos de traits compuestos anidados, las herramientas pueden mostrar y editar el código en una vista aplanada como si fuera herencia simple pura.

**Aliasing en lugar de renaming.** Para acceder a métodos sobreescritos o en conflicto, los traits usan **aliasing** en lugar de renaming explícito de clases (como hace C++ con el scope operator `::` o Eiffel con `Precursor`). Las razones:
- Las referencias a nombres de traits contradicen el flattening, porque previenen la creación de una vista aplanada semánticamente consistente.
- Las referencias a nombres de traits acoplan el código a la estructura interna de los traits: mover un método de un trait a otro invalida las referencias.
- Las referencias nominales a traits requerirían extender la sintaxis del lenguaje base.

El aliasing en cambio es compatible con el flattening: durante el proceso de aplanado se introduce simplemente un nuevo nombre para el método aliasado.

A diferencia del renaming de Eiffel, el alias **no elimina** el nombre original; simplemente agrega un nombre alternativo. El nombre original permanece accesible en el trait que usa ese alias.

**Colisiones de nombres no intencionales.** Cuando dos traits requieren métodos semánticamente distintos con el mismo nombre, los traits no pueden componerse fácilmente. Los aliases alivian el problema solo parcialmente. Una solución completa requeriría herramientas de refactorización sofisticadas o un mecanismo de encapsulación flexible (trabajo futuro señalado por los autores).

**Estrategias de resolución de conflictos.** El paper defiende la "same-operation exception": si el mismo cuerpo de método es obtenido múltiples veces por distintos caminos (porque ambos traits usan un subtrait común), no hay conflicto. Esto evita conflictos espurios en jerarquías con diamantes de traits, y tiene semántica intuitiva. Una alternativa (Snyder) sería marcar todo método obtenido por múltiples caminos como conflicto, incluso si el cuerpo es idéntico: los autores argumentan que esto sería más peligroso porque el programador resolvería un conflicto arbitrariamente, y si luego uno de los traits cambia su implementación del método, el conflicto ya no sería detectado.

### 5.3 C++ Revisited

C++ es el único lenguaje conocido que soporta tanto herencia múltiple como templates. Usando templates con clases base virtuales, es posible simular trait composition en C++. La idea es expresar cada trait como una clase template con una clase base virtual genérica, e instanciar todos los traits con la misma clase base concreta, componiendo luego las instancias mediante herencia múltiple:

```cpp
template <class Super>
class TLogOpenClose : virtual public Super {
public:
    virtual void open() { ... };
    virtual void close() { ... };
    virtual void reset() { ... };
protected:
    virtual void log(String s) { ... };
};

template <class Super>
class TSyncReadWrite : virtual public Super {
public:
    virtual int read() { ... };
    virtual void write(int n) { ... };
protected:
    virtual void acquireLock() { ... };
    virtual void releaseLock() { ... };
};

class MyDocument : public TLogOpenClose<Document>,
                   public TReadWriteSync<Document> {
    // glue methods
};
```

La declaración de la base como `virtual` es crucial: permite que `Document` se herede una sola vez y que los métodos de los traits puedan sobreescribirse en `MyDocument`. Esta aproximación equivale en runtime al comportamiento de los traits, pero solo soporta el operador suma (`+`); aliasing (`→`) y exclusión (`−`) no se pueden expresar directamente en C++. Esto significa que todos los conflictos deben resolverse sobreescribiéndolos (en lugar de excluirlos), lo que puede cambiar la semántica de futuros conflictos.

La complejidad intrínseca de esta combinación de mecanismos (templates anidados, clases base virtuales) explica por qué no fue identificada antes como un idioma general de composición en C++.

### 5.4 Traits as a General Composition Mechanism

CLOS y C++ permiten simular traits, pero a un costo significativo en complejidad. El programador de C++ necesita un profundo conocimiento de templates anidados y clases base virtuales para obtener las propiedades de robustez que los traits garantizan por diseño. El programador de CLOS necesita metacomputación explícita y conocimiento profundo de la linearización.

En contraste, los traits proveen un único mecanismo de composición con propiedades bien definidas (conmutatividad, flattening, resolución explícita de conflictos). Lenguajes modernos como Java, C#, Python y Ruby no pueden expresar directamente esta forma de composición, lo que confirma que los traits son una contribución genuina como mecanismo de composición general.

## 6. Implementation

Los traits están completamente implementados en Squeak, un dialecto open-source de Smalltalk. La implementación tiene dos partes: una extensión del lenguaje y una extensión de las herramientas de programación.

### 6.1 Language Extension

Para agregar traits a Squeak, se extendió la implementación de las clases para incluir una variable de instancia adicional que contiene la información de la cláusula de composición (qué traits usa la clase, con qué exclusiones y aliases).

Cuando una clase `C` usa un trait `T`, el diccionario de métodos de `C` se extiende con todos los métodos de `T` que no son sobreescritos por `C`. Para un alias, se agrega una segunda entrada en el diccionario con el nuevo nombre. Como los métodos compilados no dependen del lugar donde son usados, el bytecode puede compartirse entre el trait y todas las clases que lo usan. La excepción son los métodos con `super`: estos almacenan en su tabla literal una referencia explícita a la superclase, por lo que cuando un trait con tales métodos se aplica a una clase, esos métodos se copian en la clase y la referencia a la superclase se modifica apropiadamente.

En Smalltalk, las clases son objetos de primera clase (instancias de una metaclase). La implementación introduce el concepto de **classtrait**: un classtrait puede asociarse con cada trait. Para preservar la compatibilidad de metaclases, cuando un trait se usa en una clase, el classtrait asociado se usa automáticamente en la metaclase. Un trait sin classtrait puede usarse tanto en clases como en metaclases; uno con classtrait solo en clases.

La implementación es completamente compatible hacia atrás con herencia simple en Squeak. No duplica código fuente y solo duplica bytecode cuando hay super-sends. Un programa con traits tiene el mismo rendimiento que el programa equivalente implementado con herencia simple pura; la única penalización de rendimiento proviene del uso de accessor methods, que de todos modos son inline por los compiladores JIT modernos.

### 6.2 Programming Tools

Además de la extensión del lenguaje, la implementación incluye una extensión del browser de Smalltalk.

Para cada clase (y cada trait), el browser muestra los traits que la componen. Gracias a la propiedad de flattening, el browser puede aplanar esta estructura jerárquica en cualquier nivel, preservando la semántica. El browser también muestra:
- los provided methods y required methods de cada trait,
- los métodos sobreescritos,
- los glue methods que conectan los traits.

Esto permite trabajar en dos vistas:
- **Vista aplanada:** la clase aparece como un conjunto no estructurado de métodos, igual que en herencia simple pura, como si los traits no existieran.
- **Vista de composición:** muestra cómo las responsabilidades de la clase se descomponen en traits y cómo los traits están pegados entre sí.

El browser soporta compilación incremental: cuando un método de un trait se agrega, cambia o elimina, todos los usuarios de ese trait se actualizan inmediatamente. Si una modificación introduce un nuevo conflicto o un nuevo requerimiento insatisfecho, las clases y traits afectados se agregan a una lista de "to do" automáticamente. Las herramientas también asisten al programador en la generación del código pegamento necesario y en la eliminación de conflictos.

## 7. Experience

La usabilidad de los traits se validó experimentalmente en dos aplicaciones principales sobre Squeak.

### 7.1 Refactoring the Smalltalk Collection Hierarchy

La jerarquía de colecciones de Smalltalk, refinada durante más de 20 años, es un ejemplo paradigmático de programación orientada a objetos. El problema es que la herencia simple no es suficientemente expresiva para modelar un conjunto tan diverso de clases que comparten muchas propiedades en distintas combinaciones: ordenamiento explícito vs implícito, extensibilidad, acceso por clave vs por índice, comparación por identidad vs igualdad, etc.

En la implementación original, esto obliga a duplicar código o a subir métodos demasiado alto en la jerarquía (y luego deshabilitarlos en subclases donde no aplican). Cerca del 9% de los métodos originales estaban implementados "demasiado alto" para habilitar el code sharing.

La refactorización con traits consistió en:
- Crear traits para las distintas propiedades de colecciones (extensibilidad, secuenciamiento, comparación, enumeración, etc.).
- Separar la interfaz de las colecciones de su implementación (por ejemplo, "sorted-extensible interface" combinable libremente con "linked-list implementation" o "array-based implementation").
- Introducir subtraits finos y reutilizables incluso fuera de la jerarquía de colecciones (por ejemplo, un trait para "emptiness" que require `size` y provee `isEmpty`, `notEmpty`, `ifEmpty:`, etc.).

**Resultados cuantitativos:**
- Se refactorizaron 29 clases originalmente con 635 métodos.
- La versión con traits usa 60 traits distintos y reduce el total de métodos a 567 (más del 10% menos).
- El código fuente de la implementación basada en traits tiene un 12% menos de código que el original.
- Todos los métodos canceladores de herencia inapropiada fueron eliminados.

**Comparación con mixins:** Con mixins se necesitarían cadenas de herencia de hasta 35 niveles (vs 22 traits en paralelo). Los mixins no tienen propiedad de flattening, así que la semántica de los super-calls depende del lugar exacto en la cadena; esto hace que jerarquías tan profundas sean prácticamente incomprensibles. Además, en la refactorización hubo múltiples situaciones donde no existe un orden total adecuado para los mixins (por ejemplo, el trait `TSortedImpl` presentaba dos conflictos simultáneos —`at:ifAbsent:` y `collect:`— que requerían resoluciones en direcciones opuestas). Con traits se usó exclusión para resolverlos directamente.

**Comparación con herencia múltiple:** La herencia múltiple tampoco habría sido suficiente: para los "adaptors" (como `TIdentityAdaptor`, que convierte colecciones de comparación por igualdad en colecciones por identidad) se necesitan wrappers genéricos que no pueden expresarse sin duplicación con herencia múltiple estática (ver sección 2.1).

### 7.2 Applying Traits to Metaclasses and the Smalltalk Kernel

En lenguajes puros orientados a objetos como Smalltalk, las clases son objetos de primera clase, instancias de metaclases. Muchas propiedades de clase (singleton, final, abstract) aplican a muchas clases distintas, y sería natural compartir el código que implementa estas propiedades entre las metaclases correspondientes.

El problema es que dar al programador control explícito sobre las metaclases genera problemas de compatibilidad entre el nivel de clases y el nivel de metaclases: código que funciona en una clase puede fallar cuando se usa en otra clase con metaclase distinta.

La solución propuesta es representar las propiedades de clase como traits. Las metaclases se construyen como la composición de los traits correspondientes a las propiedades que tienen (singleton, abstract, final, etc.). La jerarquía de herencia se usa para garantizar la compatibilidad entre el nivel de clases y el de metaclases (manteniendo las jerarquías paralelas de Smalltalk: si `A` es superclase de `B`, entonces `A class` es superclase de `B class`).

Este enfoque usa los mismos mecanismos de herencia simple y trait composition para resolver problemas tanto en el nivel base como en el meta-nivel, sin requerir que el programador aprenda un mecanismo adicional exclusivo para metaclases.

**Bootstrapping del kernel de Smalltalk con traits.** Una vez que los traits podían representar propiedades de clase, fue natural refactorizar el propio kernel de Squeak con traits. El nuevo kernel contiene las clases tradicionales (`Behavior`, `ClassDescription`, `Metaclass`, `Class`) pero las construye desde traits como `TInstantiator`, `TInstanceEnumerator`, `TFamilyAccess` y `TMethodDictionaryManagement`. El nuevo kernel también incluye las clases `TraitBehavior`, `TraitDescription`, `Trait` y `ClassTrait` para representar los traits en sí, construidas también desde traits compartidos con las clases del kernel.

Esto tuvo la ventaja adicional de que los experimentos con el lenguaje (por ejemplo, cambiar la gestión del diccionario de métodos) son más fáciles de realizar, porque los distintos aspectos del lenguaje están encapsulados en traits que pueden recomponerse de formas distintas.

## 8. Related Work

**Otros constructos llamados "Traits".** El lenguaje Self usa "trait objects" para factorizar características comunes (grupos de métodos), pero los Self trait objects no tienen operadores de composición específicos: se usan como objetos padre ordinarios, con búsqueda por profundidad-primera y sin exclusión ni aliasing. El software del Xerox Star usaba "traits" como entidades primitivas para construir objetos más complejos, pero estos tenían más en común con herencia múltiple que con el modelo de este paper. Larch traits son fragmentos reutilizables de especificaciones formales, relacionados en nombre pero no en propósito.

**PIE.** El Personal Information Environment (Goldstein & Bobrow) introduce "perspectivas": un nodo puede tener múltiples perspectivas con distintas superclases independientes y proveer solo métodos (sin estado). Se asemeja a los traits, pero a diferencia de estos, los métodos de una perspectiva no se fusionan en el nodo; el nodo no entiende directamente los mensajes implementados por sus perspectivas, sino que hay que enviar el mensaje especificando la perspectiva. Esto evita conflictos de nombres pero impide el modelado de grano fino y acopla clientes con la estructura de perspectivas.

**Enfoques basados en templates (C++).** La Standard Template Library y la Boost Lambda Library proveen estructuras de datos parametrizadas, pero esto es genericidad por tipo, no composición de características. VanHilst y Notkin, y Smaragdakis y Batory, muestran que los templates de C++ pueden usarse para composición de características (mixin layers), pero sufren los mismos problemas de linealización que los mixins.

**RESOLVE.** El framework RESOLVE define un mecanismo de composición de componentes de software similar en espíritu a los traits, con parámetros de realización que permiten componer componentes flexiblemente. La principal diferencia es que RESOLVE es más pesado: distingue componentes abstractos de concretos, usa instanciación en lugar de herencia, y es "rich and fairly complex". Los traits son una extensión ligera de la herencia simple que garantiza ciertas propiedades (flattening) con un mecanismo de composición mínimo.

**Enfoques relacionados con mixins.** GenVoca (Batory) modela refinamientos de clase como funciones, lo que permite composición flexible pero también puede introducir nuevos datos miembro y constructores (va más allá del comportamiento puro). Mixin layers (Smaragdakis & Batory) escalan los mixins a la granularidad de múltiples clases, pero siguen sufriendo fragilidad por linealización. Las dynamic libraries de Unix, vistas como componentes en capas, también sufren el mismo problema.

**Aspect-Oriented Programming (AOP).** Los aspectos pueden agregar métodos a clases existentes y entretejer código antes o después de la ejecución de un método. La diferencia fundamental es que los aspectos son concerns que no pueden encapsularse limpiamente en un procedimiento/objeto/mixin ordinario; por definición, cruzan límites de clase y modifican el comportamiento de muchas clases a la vez de formas sistémicas. Los traits en cambio se usan para construir clases desde cero, no para modificar clases existentes. Un aspecto puede modificar el comportamiento de muchas clases a la vez; un trait no puede hacer esto.

**Delegation y Jigsaw.** La delegación (object-based inheritance) es una forma de composición dinámica, diseñada para adaptación de componentes en tiempo de ejecución. El framework Jigsaw (Bracha) define operadores de composición de módulos (merge, rename, restrict) notablemente similares a los de los traits: `merge` de Bracha es conmutativo como la suma de traits. Las diferencias principales son de motivación y escala: Jigsaw apunta a composición "en lo grande" (frameworks completos), con namespaces, tipos declarados y renaming semántico completo, mientras que los traits apuntan a composición "en lo pequeño" (dentro de una definición de clase), manteniendo la simplicidad.

**Logtalk.** Extensión OO de Prolog que soporta categorías para compartir código entre clases, pero sin aliasing ni exclusión, y con lookup depth-first que resuelve conflictos implícitamente, lo que lleva a problemas de escala.

**Mohnen, Caesar, Mezini.** Mohnen propone interfaces Java con implementaciones por defecto (precursor de Java 8 default methods), con detección automática de conflictos pero sin exclusión ni aliasing. Caesar (Mezini & Ostermann) usa "collaboration interfaces" similares a los traits en cuanto a required methods, pero confía en mecanismos de resolución de conflictos de herencia múltiple (C++) con sus problemas asociados. Mezini propone "adjustments" (módulos software similares a mixins) como una capa de combinación explícita entre objetos y clases, pero el modelo es más dinámico y complejo.

## 9. Conclusions and Future Work

Este paper introdujo los traits como un mecanismo para construir y estructurar clases en programas orientados a objetos. Un mismo trait puede reutilizarse en muchas clases, independientemente de su posición en la jerarquía de herencia. Los traits se manipulan con cuatro operadores — suma, override, exclusión y aliasing — diseñados para permitir un grado adecuado de flexibilidad de composición sin quedar sujetos a los problemas de los mixins y la herencia múltiple.

Las propiedades favorables de composición hacen de los traits una extensión ideal a los lenguajes de herencia simple. Los traits son completamente compatibles hacia atrás con Smalltalk y no requieren modificar la sintaxis de métodos del lenguaje subyacente. La propiedad de flattening garantiza que el código resultante no es menos comprensible que el original: siempre es posible ver y editar el código como si estuviera escrito con herencia simple pura, sin traits.

Contar con las herramientas correctas resultó crucial para obtener el máximo beneficio de los traits. En la implementación en Squeak, se extendió el browser para que permita al programador alternar entre la vista aplanada y la vista de composición, y enfatice los glue methods que definen cómo se conectan los traits.

Los traits se usaron exitosamente en varios casos de estudio: la refactorización de la jerarquía de colecciones de Smalltalk (29 clases, 60 traits, 10% menos de métodos, 12% menos de código fuente), y el bootstrapping del kernel de Squeak con traits para representar propiedades de metaclases de forma uniforme.

**Trabajo futuro señalado por los autores:**
1. Evaluar el impacto de namespaces y encapsulación sobre la propiedad de flattening.
2. Considerar los efectos de permitir que los traits especifiquen variables de estado.
3. Extender la composición de traits para que pueda reemplazar a la herencia.
4. Evaluar la posibilidad de usar traits para modificar el comportamiento de instancias individuales en tiempo de ejecución.
5. Explorar más aplicaciones de traits al refactorizado de jerarquías de clases complejas.
6. Resolver los problemas de tipos en lenguajes con tipos estáticos (trabajo en progreso con Java y .NET; traits ya son característica estándar de Scala).

---

> Las referencias bibliográficas del paper incluyen ~60 entradas (Bracha, Cook, Stroustrup, Meyer, Kiczales, Ungar, Goldberg & Robson, Ingalls, etc.). No se listan en detalle aquí; remitirse directamente al PDF para citas específicas.
