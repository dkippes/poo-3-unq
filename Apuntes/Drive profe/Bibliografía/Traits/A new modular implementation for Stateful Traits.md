# A new modular implementation for Stateful Traits
Pablo Tesone, Stéphane Ducasse, Guillermo Polito, Luc Fabresse, Noury Bouraqadi — Science of Computer Programming, preprint del 8 de abril de 2020

> Resumen detallado en español, organizado siguiendo la estructura original del paper, para poder seguirlo en paralelo con el PDF. No es traducción literal sino una explicación fiel y completa, en mis propias palabras, de cada sección.

## Abstract

Aclaración inicial importante: el término "traits" está sobrecargado en la literatura, así que en este trabajo "traits" se refiere específicamente al modelo y a la implementación *sin estado* (stateless) descritos en los artículos de Schärli et al. (el paper "padre" de Traits que ya conocemos).

Los traits ofrecen una forma flexible de soportar reuso de código tipo herencia múltiple en el contexto de un lenguaje de herencia simple. Pharo incluye la segunda implementación de traits stateless, basada en la versión original de Schärli. Pero aun siendo la segunda iteración, esa implementación presenta varias limitaciones:

1. **No soporta estado en los traits** (no se pueden definir variables de instancia dentro de un trait).
2. Su implementación es **monolítica**, es decir, está profundamente acoplada con el resto del kernel del lenguaje: no se puede cargar ni descargar como un módulo aparte. Además, el soporte de traits impacta a *todas* las clases, incluso a las que no usan traits. Y aunque las herramientas de desarrollo tienen soporte completo para trabajar con clases, el soporte para traits es más limitado porque las clases y los traits no exponen el mismo Metaobject Protocol (MOP). Finalmente, por ser monolítica y estar integrada en el kernel, es difícil extender esta implementación actual.

Este artículo describe una nueva implementación **modular** y **extensible** de traits: se puede cargar y descargar como cualquier otro paquete. Además, las clases que no usan traits no se ven impactadas. Finalmente, esta nueva implementación incluye un MOP nuevo y cuidadosamente diseñado que es compatible tanto con clases como con traits. Esto permite reutilizar las herramientas existentes sin que necesiten soporte especial para traits. Luego, siguiendo la semántica propuesta para los traits *stateful* (con estado) en el paper de Bergel, Ducasse, Nierstrasz y Wuyts (BDNW07, "Stateful Traits"), se presenta una nueva implementación de traits con estado, que es una extensión de esta nueva implementación modular.

Los traits modulares se implementaron usando metaclases especializadas como mecanismo principal de extensión del lenguaje. Al reemplazar la implementación, se redujo el tamaño del kernel del lenguaje Pharo en un 15%. Este modelo y esta implementación se usan en producción desde Pharo 7.0 (enero de 2019).

**Palabras clave:** traits, lenguajes dinámicos, metaobject protocol, extensión de lenguaje, lenguajes modulares, especialización de metaclases.

## 1. Introduction

La literatura contiene múltiples conceptos e implementaciones llamados "traits". Varios lenguajes ofrecen traits: Self, Scala, SEDEL, Groovy, Fortress y Scheme. También se han hecho extensiones de lenguaje para Java y Javascript. Pero cada una de estas implementaciones tiene su propia semántica y consideraciones, y a veces introducen definiciones que entran en conflicto entre sí.

En este paper, los autores usan el término *traits* exactamente como en los artículos de Schärli et al. y en la implementación de Lienhard (la tesis de bootstrapping de traits). Citan textualmente la definición clásica:

> "A trait is a set of methods, divorced from any class hierarchy. Traits can be composed in arbitrary order. The composite entity (class or trait) has complete control over the composition and can resolve conflicts explicitly, without resorting to linearization."

Es decir: un trait es un conjunto de métodos, divorciado de cualquier jerarquía de clases. Los traits pueden componerse en cualquier orden. La entidad compuesta (clase o trait) tiene control total sobre la composición y puede resolver conflictos explícitamente, sin recurrir a la linearización (la técnica que usan los mixins/CLOS para ordenar la herencia múltiple).

Además, ya que se han hecho varias variaciones, como los **stateful traits** (traits con estado) y los **freezable traits**, el paper se refiere a este modelo original (sin estado) como *stateless traits*. También hay variaciones de esta implementación en Perl y PHP. Además, SEDEL presenta una implementación "de juguete" relacionada que agrega traits como ciudadanos de primera clase a un lenguaje con tipado estático.

Los traits dan soporte a reuso de código tipo herencia múltiple en el contexto de un lenguaje de herencia simple. Resuelven el problema del diamante (el problema clásico de la herencia múltiple). Los métodos duplicados se extraen como traits, que a su vez son compartidos por muchas clases. Los traits también son componibles entre sí, para formar otros traits. Reppy y Turon proponen una forma de reducir el código repetitivo (boilerplate) mediante el uso de traits, sin necesidad de metaprogramación.

El lenguaje Pharo incluye una implementación de traits *stateless*. Es la implementación de referencia descripta en la literatura (los papers de Schärli, Ducasse-Nierstrasz-Schärli-Wuyts-Black, y Nierstrasz-Ducasse-Schärli). La implementación de Pharo 6.0 de traits stateless reimplementó completamente la versión original; sin embargo, esta segunda versión todavía tiene varias limitaciones tanto a nivel de implementación como a nivel de modelo:

**A nivel de implementación.** La implementación es *monolítica*: está profundamente acoplada con el resto del kernel del lenguaje (esto se detalla en la Sección 2). Esto impide que los desarrolladores de Pharo puedan cargar, descargar o actualizar fácilmente el soporte de traits. Restringe la flexibilidad del sistema y hace más complejo el proceso de bootstrap. En particular, las clases que no usan traits tienen una dependencia inútil con la implementación de Trait (por ejemplo, variables de instancia sin uso, código condicional sin sentido para ellas).

**A nivel de modelo.** El modelo implementado presenta limitaciones: los traits stateless no soportan la definición de estado dentro de los traits. El programador se ve entonces forzado a definir accessors e inicialización en todas las clases que usan un trait que requiere ese estado de instancia. Los *stateful traits* fueron propuestos para extender los traits con soporte de estado. En esa alternativa, las variables de instancia, los accessors y la inicialización que necesita un trait se definen en el trait mismo, pero a costa de un cambio en el layout de instancia (usando acceso basado en diccionario en lugar de acceso indexado). Sin embargo, los stateful traits nunca llegaron a implementarse completamente ni a adoptarse, debido a la complejidad de modificar la implementación monolítica existente y de actualizar todas las herramientas existentes.

El impacto de introducir un concepto nuevo en un modelo y sistema existente a menudo no es tenido en cuenta explícitamente por los diseñadores de lenguajes. Basándose en más de 10 años de experiencia trabajando con traits, este paper reflexiona sobre la introducción de traits en un sistema existente y su impacto en las herramientas existentes. Al hacerlo, se analiza el espacio de diseño de la API de clases y traits en términos del **Metaobject Protocol** expuesto al desarrollador.

> Nota al pie sobre qué es un MOP, citando a Kiczales et al.: "Primero, los elementos básicos del lenguaje de programación —clases, métodos y funciones genéricas— se hacen accesibles como objetos. Como estos objetos representan fragmentos de un programa, reciben el nombre especial de Metaobjects. Segundo, las decisiones individuales sobre el comportamiento del lenguaje se codifican en un protocolo que opera sobre estos Metaobjects: un Metaobject Protocol. Tercero, para cada tipo de Metaobjects se crea una clase por defecto, que establece el comportamiento por defecto del lenguaje en forma de métodos del protocolo."

Este paper describe una implementación modular para introducir traits en un lenguaje de programación, así como una nueva implementación de traits stateful basada en la semántica definida en BDNW07. Las contribuciones de este trabajo son:

- Un análisis de las limitaciones de la implementación actual de traits *stateless* de Pharo. Este análisis va más allá de la falta de soporte de estado (Sección 2). Muestra que los traits deberían ser **polimórficos** respecto de las clases y exponer una API revisada para dar soporte a quienes construyen herramientas.

  > Nota al pie: "polimórfico" significa que un objeto expone la misma API que otro objeto con el que es polimórfico. En términos de Java, dos objetos implementan la misma interfaz, así que son polimórficos.

- Una nueva implementación de traits *stateless* compatible con clases por diseño, que provee una integración fluida con las herramientas de programación existentes (Sección 3). Esta implementación resuelve los problemas clave de compatibilidad entre clases y traits desde la perspectiva de la API. Se rediseñó un MOP estructural para clases y traits que toma en cuenta el origen de los métodos.

- Un nuevo Metaobject Protocol que soporta un sistema con y sin traits, dando soporte tanto a traits stateless como stateful (Sección 4).

- Una nueva implementación de **traits stateful**. Se extiende la nueva implementación con traits con estado, siguiendo la semántica descripta en la literatura (BDNW07). Esta nueva implementación es el fundamento del modelo de traits de Pharo 7.0 y se usa en un entorno industrial (Sección 5).

- Una implementación **modular**. La nueva implementación es modular porque es una biblioteca cargable y extensible. Esta nueva biblioteca también asegura que las clases que no usan traits no se vean impactadas, porque se apoya en metaclases especializadas que encapsulan la integración con traits. Esta implementación modular les da a los desarrolladores la libertad de cargar o descargar esta característica del lenguaje, dándole flexibilidad al equipo de desarrollo. Incluso es posible modificar la biblioteca sin afectar al resto del entorno (Sección 6).

Comparado con el modelo e implementación actual de traits stateless en Pharo 6.0 (nota: el paper siempre llama "implementación actual" a la de Pharo 6.0, y "nueva implementación" a la descripta en este paper, que corresponde a Pharo 7.0), la solución propuesta mejora la modularización de la base de código existente: la simplifica, reduciendo el tamaño del kernel del lenguaje Pharo en un 15%. Acelera el proceso de bootstrap general en un 30% para construir kernels de lenguaje reducidos (Sección 7). La Sección 8 presenta las consideraciones detrás de las distintas decisiones de diseño, y la Sección 9 compara esta solución con trabajos relacionados. Finalmente, la Sección 10 concluye el paper.

## 2. Limitations of Stateless Traits and its Monolithic Implementation

La implementación de traits stateless presenta una serie de limitaciones. El análisis de las limitaciones de los traits stateless presentado por Bergel et al. en BDNW07 todavía aplica a la implementación de traits en Pharo 6.0. El paper no lo repite, sino que se enfoca en los **problemas prácticos** que enfrentó el equipo de desarrollo de Pharo a lo largo de los años.

### 2.1. Vocabulary

Antes de seguir, el paper fija un vocabulario común que se usa en todo el artículo:

- Una **clase** define o tiene métodos y variables de instancia. Cada clase tiene una única superclase de la que hereda. Una clase crea instancias.
- Los **traits stateless** definen métodos. Los **traits stateful** además definen variables de instancia. Los traits no pueden crear instancias (no son instanciables).
- Cuando una clase se compone a partir de un trait, decimos que la clase **usa** (*uses*) el trait. Una clase que usa un trait tiene acceso a los métodos (y variables de instancia, si aplica) definidos en el trait.
- Se usa el término **traited class** ("clase traiteada") para referirse a clases que usan traits, y **normal classes** ("clases normales") para clases que no usan traits.
- Por convención de lectura, todos los nombres de traits se prefijan con una T (por ejemplo, `TNamed`).
- Finalmente, el **origin** (origen) de un método es la clase o el trait que lo define.

### 2.2. Overloaded Metaobject Protocol

En las clases normales, los métodos se originan desde dos posibles lugares: o bien están definidos en la clase misma, o bien están definidos en alguna de sus superclases (es decir, son heredados). El MOP de las clases fue diseñado originalmente para manejar esta diferencia: existen dos familias de mensajes para acceder a los métodos:

- Una familia de mensajes se usa para acceder a los métodos definidos en la clase (es decir, `selectors`, `methods`).
- La otra familia, con mensajes prefijados con "all", se usa para acceder tanto a los métodos definidos como a los heredados (es decir, `allSelectors`, `allMethods`).

Esta separación de mensajes provee una interfaz clara que muestra dónde están definidos los métodos.

Para hacer que los traits stateless y las clases traiteadas fueran compatibles con las herramientas existentes, el MOP cuidadosamente diseñado no fue rediseñado, sino simplemente **sobrecargado**. Por ejemplo, el método `methods` se modificó para devolver no solo los métodos definidos en la clase misma, sino también los aportados por los traits usados. Esto es un problema porque ciertas operaciones de navegación, como editar el método original, de repente se volvieron mucho más complejas de definir. Para identificar el origen de un método, el MOP de la clase traiteada se modificó para proveer una nueva familia de mensajes prefijados con "local" (es decir, `localSelectors`, `localMethods`). De forma similar a las versiones sin prefijo, los métodos prefijados con "local" devuelven los elementos definidos en la clase misma. Sin embargo, mientras la intención era buena, la situación resultante no es satisfactoria, ya que tener dos MOPs distintos fuerza a las herramientas a usar código condicional para detectar si están manipulando una clase normal o una clase traiteada, para así usar la API correspondiente correcta.

### 2.3. Trait and Class Coupled Implementation

En la versión actual (Pharo 6.0), la implementación de traits está fuertemente acoplada dentro del modelo de metaclases por defecto de Pharo. No hay forma de tener una clase sin soporte de traits. Esto es un problema porque la gran mayoría de las clases no usan traits. En Pharo 6.1, solo 127 clases usan traits sobre un total de 6486 clases del sistema: esto significa que menos del 1.95% de las clases están usando traits.

> Nota al pie: este cómputo no toma en cuenta el hecho de que cuando una clase raíz de una jerarquía de herencia usa un trait, sus subclases también deberían contarse como usuarias de traits.

El objetivo original de tener una implementación acoplada era aprovechar las capacidades de reuso de los traits. Sin embargo, el resultado, aunque funcional, hace que la lógica del código sea más compleja. El código que efectivamente integra traits con clases está embebido dentro de las metaclases estándar de Pharo: `Behavior`, `ClassDescription`, `Class` y `Metaclass`. Internamente, estas clases manejan tanto el caso en que una clase usa traits como el caso en que no los usa, principalmente usando condicionales. El Listado 1 muestra un ejemplo de código condicional disperso por todo el kernel del lenguaje:

```smalltalk
Behavior >> isComposedBy: aTrait
  "Answers if this object includes trait aTrait into its composition"
  aTrait isTrait ifFalse: [ ^ false ].
  ^ self hasTraitComposition
    ifTrue: [ self includesTrait: aTrait ]
    ifFalse: [ false ]
```

Este método responde si el objeto incluye al trait `aTrait` en su composición. Primero verifica si `aTrait` realmente es un trait (si no lo es, devuelve `false` directamente). Luego, pregunta si `self` (la clase consultada) tiene composición de traits; si la tiene, delega en `includesTrait:`; si no la tiene, simplemente devuelve `false`. Es un ejemplo de "mixed up implementation in Kernel" (implementación mezclada en el kernel): nótese cómo este método vive en `Behavior`, la raíz común de clases y metaclases, y tiene que manejar con condicionales el caso de clases que ni siquiera usan traits.

Esta implementación dispersa impide que los desarrolladores de Pharo puedan descargar la implementación de traits, o tener un kernel más chico. Esto impacta el tiempo de bootstrap y las posibilidades futuras de evolución del kernel.

**Requisitos para la nueva implementación.** Como la modularidad puede interpretarse de distintas formas, el paper define con precisión cuáles son los requisitos para una implementación de traits modular. La implementación debería ser modular en el sentido de que:

- Las clases que no usan traits no deberían depender de la implementación de traits.
- El soporte de traits debería poder cargarse a demanda.
- Para soportar traits, la jerarquía existente de una clase no debería modificarse. Para beneficiarse del soporte de traits, una clase no debería verse forzada a heredar de una clase especial. Esto es particularmente importante dado que Pharo solo soporta herencia simple.

De estos requisitos se desprende que la granularidad del soporte de traits es una clase y sus subclases. Es decir: usar un trait en una clase solo afecta a esa clase y a sus subclases, pero no a ninguna de sus superclases.

### 2.4. Wrong Unified API between Traits and Classes

En el segundo intento de proponer una mejor API para traits y clases, la comunidad hizo cambios para proponer una API más unificada entre las clases `Trait` y `Class`. La intención inicial era que una clase pudiera entender los mismos mensajes que un trait, en particular para evitar que quienes construyen herramientas tuvieran que escribir código condicional complejo. En la implementación de Pharo 6.0, `Trait` y `Class` eran ambos usuarios del trait `TBehavior`. El trait `TBehavior` definía métodos comunes que permitían usar traits y clases de forma polimórfica en algunas situaciones, por ejemplo, en el compilador.

Sin embargo, traits y clases no proveían una API totalmente compatible. De hecho, algunos comportamientos de clase, como la creación de instancias, no tenían sentido en los traits. Varios métodos en la clase `Trait` fueron cancelados (es decir, lanzaban un error) o tenían una implementación incompatible. Como los traits no pueden instanciarse, su método `basicNew` lanzaba una excepción para evitar la creación de instancias.

Otros mensajes sin sentido contenían un comportamiento válido pero cuestionable (por ejemplo, `superclass` devolviendo `nil`). Este comportamiento inconsistente y posiblemente accidental produjo mucho código condicional en las herramientas existentes para manejar estos casos especiales y evitar mal comportamientos.

El Listado 2 muestra un ejemplo de código condicional. Esta implementación es necesaria porque, por diseño, los traits devuelven `nil` al mensaje `instanceSide` (mensaje con el que las clases devuelven la clase del lado de instancia); y, también por diseño, los traits no soportaban variables de clase.

> Nota al pie: en Smalltalk, cuando un programador define una clase, el sistema crea internamente una instancia de otra clase que se crea automáticamente: su metaclase, llamada por ejemplo `Person class`. El mensaje `instanceSide` se asegura de que, al enviarse a una clase o a su metaclase, se devuelva la clase (del lado de instancia).

Por eso, el desarrollador tenía primero que protegerse contra el caso de que el argumento fuera un trait y que el mensaje `instanceSide` devolviera `nil`; luego, que el argumento pudiera ser un "class trait" (un trait aplicado al lado de clase de una clase), para finalmente poder estar seguro de que el argumento es una clase. El Listado 2 muestra que la situación es todavía más confusa, ya que usa `cls` en lugar de `classOrTrait` como nombre de la variable temporal:

```smalltalk
AbstractTool >> browseClassVarRefsOf: aClass
  | cls |
  cls := aClass instanceSide.
  cls isTrait
    ifFalse: [ self systemNavigation browseClassVarRefs: cls ]
```

Como se ve, tener una API mal pensada introduce más complejidad en los métodos que tienen que tratar con clases y traits.

## 3. A Modular Stateless Trait Implementation

Implementar una implementación de traits modular y extensible presenta una serie de desafíos que se resuelven en la nueva implementación presentada en este artículo.

Se empieza con un ejemplo de traits stateless como los definió Schärli et al. (Sección 3.1). Se recuerda al lector el álgebra de composición de traits stateless (Sección 3.2). Luego, dado que el kernel de Pharo deriva del kernel de Smalltalk, para facilitar la comprensión se explica y bosqueja una versión simplificada del kernel Smalltalk original que no incluye traits (Sección 3.3). Usando el primer ejemplo, se presenta la nueva implementación modular de traits stateless (Sección 3.4).

### 3.1. Studying a Trait Usage

Para hacer más clara la descripción de la implementación, la Figura 1 del paper describe una implementación basada en traits de una biblioteca de Stream, usando los mecanismos descriptos por Schärli et al. Esta biblioteca da la capacidad de leer líneas. Lee desde un stream hasta encontrar un carácter de fin de línea.

La clase `ReadLineStream` usa dos traits: `TStream` y `TReadLine`. El primer trait provee el comportamiento requerido para leer desde el stream, y el segundo, la capacidad de manejar la lectura de líneas de texto.

El trait `TReadLine` define solamente los métodos `readLine` y `hash`, y requiere el método `read`. Se define el trait `TReadLine` así:

```smalltalk
Trait named: #TReadLine
  uses: {}
```

El trait `TReadLine` no usa ningún otro trait. La cláusula `uses:` especifica la composición de traits (vacía en este caso).

La interfaz para definir una clase como composición de traits es una extensión de la interfaz usada para definir clases sin usar traits. La interfaz extendida incluye la cláusula `uses:` para expresar la composición de traits. Acá vemos cómo `ReadLineStream` usa los traits `TReadLine` y `TStream`:

```smalltalk
Object subclass: #ReadLineStream
  uses: TReadLine @ {#hashFromReadLine -> #hash}
      + TStream @ {#hashFromStream -> #hash}
  instVarNames: ''
  ....
```

En este ejemplo, ambos traits proveen una implementación de `hash`, lo cual produce un **conflicto**, ya que la clase resultante no puede tener dos veces el mismo método. En este ejemplo se decidió tener una implementación propia de `hash` en `ReadLineStream` que combina ambas implementaciones existentes en los traits. Esta implementación reemplaza las existentes pero requiere tener acceso a ellas. Entonces, se aplica una operación de **aliasing** a los métodos aportados por los traits. El `hash` de `TReadLine` se renombra como `hashFromReadLine`, y el de `TStream` se renombra como `hashFromStream`. El aliasing de los métodos se hace mediante el operando alias (`@`) en la composición de traits de la clase `ReadLineStream`.

### 3.2. Stateless Trait Composition Algebra

La composición de traits implementada sigue la semántica definida en la implementación original de traits (Schärli et al.). El álgebra de composición de traits permite combinar distintos traits para formar uno más complejo. Por lo tanto, las clases pueden usar distintos traits junto con un álgebra de composición de traits para expresar la resolución de conflictos. Esta sección puede saltearse si el lector ya conoce la composición de traits (como ya la conocemos del paper "padre"). Es una presentación breve de los tres operadores de composición de traits que se cargan cuando se carga la biblioteca de traits.

**Sequence (secuencia).** La operación de secuencia permite combinar dos o más traits existentes. Los traits usados en la secuencia no tienen ninguna prioridad entre sí. La combinación resultante tiene todos los métodos definidos en los traits originales. Las secuencias se crean usando el operador `+`:

```smalltalk
Object subclass: #AnExampleClass
  uses: T1 + T2 + T3
  instanceVariableNames: ''
  ...
```

**Alias Method (método alias).** Cuando dos o más traits definen el mismo método, ocurre un conflicto. Una alternativa para resolver el conflicto es alias-ear (asignarle un alias a) uno de los métodos en conflicto con otro nombre, usando el operador `@`. Este operador recibe un array de asociaciones (`->`) con el nuevo nombre y el nombre original. La composición resultante tiene todos los métodos definidos en el trait, pero con el método aliaseado renombrado:

```smalltalk
Object subclass: #AnExampleClass
  uses: T1 @ { #new -> #original }
  instanceVariableNames: ''
  ...
```

**Remove Method (eliminar método).** Cuando existe un conflicto entre dos métodos, otra alternativa para resolverlo es eliminar uno de los métodos en conflicto. El operador `-` permite generar una nueva composición con un método eliminado. Esta composición incluye todos los métodos originales excepto el eliminado:

```smalltalk
Object subclass: #AnExampleClass
  uses: T1 - #aMethodToRemove
  instanceVariableNames: ''
  ...
```

### 3.3. Starting with a Implicit Metaclass Kernel

Esta subsección describe el kernel original de Smalltalk y Pharo que **no incluye traits**. La Figura 2 del paper presenta el kernel de metaclases original de Pharo (y de su ancestro, Smalltalk), como un kernel de metaclases implícito tradicional. Cada clase tiene su propia metaclase, por ejemplo, la clase `Object` es instancia de la metaclase `Object class`. La metaclase `Object class` es instancia de la clase `Metaclass`. La clase `Metaclass` es instancia de la metaclase `Metaclass class`, y la metaclase `Metaclass class` es instancia de la clase `Metaclass`. `Object class` hereda de `Class`.

El kernel se descompone en `Behavior` (la raíz de todas las metaclases y clases), `ClassDescription`, y `Class` y `Metaclass`. `Behavior` define la esencia de una clase: una superclase, un diccionario de métodos y una forma de describir el estado de instancia (esto es lo que se llamaría en la materia el "method dictionary" + "instanceFormat").

La Figura 3 muestra una vista simplificada de este kernel original, a efectos de presentación. Se compactan `Behavior`, `ClassDescription`, `Class` y `Metaclass` en la clase `Class`, como se hace en CLOS. Cada clase en el sistema es un ciudadano de primera clase con una superclase, un diccionario de métodos, y una descripción del formato de instancia. El sistema usa metaclases para darle un comportamiento común a las clases.

Este kernel simplificado no tiene ningún soporte para traits. Usando este kernel simplificado, se presenta la implementación modular de stateful traits que resuelve los problemas mencionados en la Sección 2.

### 3.4. Extending the Kernel with a Modular Stateless Trait Implementation

Esta sección presenta la extensión del kernel (Figura 3) para soportar traits, así como una instanciación de este meta-modelo para representar la biblioteca de streams basada en traits (Figura 1).

Para soportar traits stateless de forma modular, la implementación aprovecha el soporte de metaclases existente mostrado en la Figura 2. Como se muestra en la Figura 4, se define una metaclase especializada llamada **`TraitedMetaclass`**, que es usada por las clases traiteadas y por la clase `Trait`. Con la nueva implementación:

- Las clases que usan traits **no** son instancias de `Metaclass`, sino de `TraitedMetaclass`.
- Las clases normales quedan sin modificar, siguen siendo instancias de la metaclase `Metaclass class`.

Siguiendo el modelo de traits de Lienhard, una clase traiteada usa un trait y su metaclase traiteada usa el correspondiente trait del lado de clase ("class-side trait"). En efecto, cuando un trait define un método que envía un mensaje a su lado de clase, el método correspondiente debería estar disponible en la clase que lo compone. Por lo tanto, los traits se definen y se aplican como un **par de traits** (un trait y su trait del lado de clase) a una clase dada.

La Figura 4 ilustra esto con `TStream` y `TReadLine` siendo instancias de `Trait`, y sus contrapartes del lado de clase, `TStream classSide` y `TReadLine classSide`, siendo instancias de la clase `Trait class`. La clase `ReadLineStream` es instancia de la clase `ReadLineStream class`, que a su vez es instancia de la nueva metaclase `TraitedMetaclass`, que es la que maneja las clases que usan traits.

Todo el comportamiento relacionado con traits se elimina de las clases del kernel y se hace disponible en la implementación de traits. Como se muestra en la Figura 4, todo el comportamiento de traits queda definido en `TraitedMetaclass` y en las clases traiteadas. Agregar el comportamiento necesario a `TraitedMetaclass` es una operación simple porque `TraitedMetaclass` es una clase regular del entorno: extiende el comportamiento de `Metaclass` mediante herencia y *overriding* (sobreescritura). Sin embargo, integrar este comportamiento de soporte de traits al sistema no es tan simple. Hace falta integrar esta nueva implementación con soporte de traits dentro de las clases, y agregar nuevos métodos y variables de instancia para manejar la composición de traits.

En la implementación presentada en este artículo, el comportamiento adicional necesario en las clases traiteadas está factorizado en un trait llamado **`TTraitedClass`**. Este trait se incluye automáticamente en todas las metaclases de las clases traiteadas, como se muestra en la Figura 5. Al hacerlo, las clases del kernel de Pharo no tienen ninguna referencia a la implementación de traits, y se preserva el paralelismo entre las jerarquías de clases y metaclases.

Usando este modelo, se logra la modularidad empaquetando `Trait`, `TraitedMetaclass` y `TTraitedClass` en una biblioteca de traits separada que es importada explícitamente por los usuarios.

## 4. Designing a Modular Metaobject Protocol

Para tener una buena integración entre las herramientas y el modelo subyacente, el Metaobject Protocol tiene que ser claro y consistente. Para permitir la extensión del lenguaje kernel con una implementación modular de traits, el MOP también debería soportar nuevas extensiones de forma coherente y compatible.

Se propone un MOP con dos conjuntos de mensajes: un conjunto para manejar **elementos definidos localmente** (*locally defined elements*) y otro conjunto para manejar **elementos contribuidos** (*contributed elements*). Un elemento es o bien un método o bien una variable de instancia.

- Los **elementos definidos localmente** son los elementos definidos directamente en la clase o trait.
- Los **elementos contribuidos** son elementos definidos en otro lugar pero que de todas formas son accesibles para la clase o trait.

Considerando el kernel del lenguaje sin traits, los elementos contribuidos de una clase son el conjunto de elementos heredados. Cuando la biblioteca de traits está cargada, los elementos contribuidos de una clase traiteada son la lista de elementos heredados *y* los elementos definidos en los traits usados. Notar que esta lista de elementos contribuidos debería tomar en cuenta la semántica de traits: el hecho de que los elementos locales (de la clase o trait que compone) tienen precedencia sobre los contribuidos (heredados o usados de traits).

En el ejemplo de `ReadLineStream` (Figura 1), la clase `ReadLineStream` tiene el método definido localmente `hash`, los métodos contribuidos heredados de `Object` (por ejemplo `equals:` y `printString`) y los métodos contribuidos por el uso de los traits `TReadLine` y `TStream`.

El MOP de Pharo se extiende con nuevos mensajes para soportar traits. Por claridad, el paper solo presenta los mensajes relacionados con métodos; sin embargo, la misma estructura es aplicable a otros elementos de una clase, como sus variables de instancia. Notar que este Metaobject Protocol es **compartido por clases y traits**:

- **`methods`.** Devuelve todos los métodos definidos localmente en la clase dada.
- **`selectors`.** Devuelve todos los selectores de métodos definidos localmente en la clase dada.
- **`contributedMethods`.** Devuelve todos los métodos contribuidos a esta clase. En el caso de clases normales, incluye los métodos en la jerarquía de clases. En el caso de clases traiteadas, incluye los métodos en la jerarquía de clases y los métodos contribuidos por los traits.
- **`contributedSelectors`.** Devuelve los selectores de los métodos devueltos por `contributedMethods`.
- **`allMethods`.** Devuelve todos los métodos, incluyendo los métodos definidos localmente y los métodos contribuidos.
- **`allSelectors`.** Devuelve los selectores de los métodos devueltos por `allMethods`.
- **`originOfMethod: aMethod`.** Este mensaje devuelve la clase o el trait que define a `aMethod`.

Usando este Metaobject Protocol, las herramientas pueden acceder a los elementos de una clase y a los elementos contribuidos a una clase sin preocuparse por su origen. El origen de un elemento se usa cuando se modifica el elemento original (es decir, el método definido en un trait, en el caso de un trait). Además, este MOP delega la resolución de los elementos a la clase correspondiente explotando el polimorfismo, para tener una extensión de lenguaje modular. Los elementos contribuidos de una clase normal (sin usar traits) se calculan con el método definido en `Class` (Listado 3), y los elementos contribuidos de una clase traiteada se calculan con el método definido en `TTraitedClass` (Listado 4). Entonces, la implementación en las clases del kernel (como `Class`) no sabe cómo incluir las contribuciones de los traits; esto se resuelve en el código implementado en `TTraitedClass`.

```smalltalk
Class >> contributedMethods
  "Returns the methods in the superclass chain"
  ^ self superclass allMethods
```

```smalltalk
TTraitedClass >> contributedMethods
  "Concatenate the methods from superclass and from the used traits"
  ^ self superclass allMethods, self
      traitComposition allMethods
```

Notar la elegancia de la solución: la versión de `Class` simplemente delega en la cadena de superclases (`superclass allMethods`). La versión de `TTraitedClass` extiende esa misma idea concatenando además los métodos que aporta la composición de traits (`self traitComposition allMethods`). Como `TTraitedClass` es un trait que se mezcla en la metaclase de las clases traiteadas, *sobreescribe* el método `contributedMethods` heredado de `Class`, sin que `Class` necesite saber nada sobre traits.

### 4.1. Traited Class Definition

Para tener una implementación de traits realmente modular, la sintaxis para definir clases traiteadas también se carga junto con la biblioteca de traits. Los Listados 5 y 6 muestran la definición de una clase traiteada (`ReadLineStream`) y de una clase normal (`Color`). Como se ve, la definición no es la misma para clases traiteadas y clases normales; la definición de clase tiene que extenderse para introducir la composición de traits.

```smalltalk
"Defining the class Account without traits"
Object subclass: #Color
  instanceVariableNames:
    'red blue green alpha'
  classVariableNames: ''
  package: 'Colors'
```

```smalltalk
"Defining the class Employee using a trait composition"
Object subclass: #ReadLineStream
  uses:  TReadLine @ {#hashFromReadLine -> #hash}
      + TStream @ {#hashFromStream -> #hash}
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'Streams'
```

Dado que la definición de clases en Pharo se hace mediante envíos de mensajes a la superclase, se agrega una nueva definición como un **método de extensión** a `Class`. Este método de extensión permite que todas las clases del sistema generen nuevas subclases usando traits. Un módulo en Pharo es capaz de contribuir a clases existentes a través de métodos de extensión. Cuando se carga un módulo que contiene métodos de extensión, los métodos de extensión se instalan en la clase objetivo.

Cada clase del sistema es responsable de regenerar su propia definición. Entonces, a cada clase se le pregunta su definición a través del mensaje `definition`; la nueva sintaxis de definición de clases se especializa adecuadamente en `TraitedMetaclass` y en `TTraitedClass`.

## 5. Supporting Stateful Traits

Además de ofrecer una implementación modular y extensible de traits stateless, las discusiones de los autores con los diseñadores e ingenieros de la plataforma de análisis Moose mostraron que existía una necesidad real de traits con estado. Moose es una plataforma de análisis de software y de datos. Los meta-modelos de Moose representan una gran cantidad de datos sobre los elementos del lenguaje, y los traits stateless no podían capturar este aspecto importante de la representación.

Siguiendo la semántica propuesta para traits *stateful* en BDNW07, se extendió esta nueva implementación (presentada en las secciones anteriores) para soportar estado en los traits. Esta nueva implementación demuestra la extensibilidad del diseño de la implementación, al introducir el soporte de variables de instancia y de clase en los traits, y sus inicializaciones.

### 5.1. Studying a Stateful Trait Example

Antes de presentar esta extensión, primero se presenta un ejemplo extraído del artículo original de stateful traits. La Figura 6 del paper muestra este ejemplo sobre una implementación de un `SyncStream`. Esta clase extiende el comportamiento de un stream agregando un lock para sincronizar sus accesos.

La clase `SyncStream` usa los traits `TStream` y `TSyncReadWrite`. El trait `TSyncReadWrite` define la variable `lock`, tres métodos `syncRead`, `syncWrite` y `hash`, y requiere los métodos `read` y `write`.

Siguiendo las especificaciones del modelo de stateful traits, la nueva implementación extiende el mecanismo de definición de traits. Se define el trait `TSyncReadWrite` así:

```smalltalk
Trait named: #TSyncReadWrite
  uses: {}
  instVarNames: 'lock'
```

El trait `TSyncReadWrite` no usa ningún otro trait, y define una variable de instancia llamada `lock`. La cláusula `uses:` especifica la composición de traits (vacía en este caso), e `instVarNames:` lista las variables definidas en el trait (es decir, la variable `lock`). La interfaz para definir una clase como composición de traits es la misma que con traits stateless. Acá la definición de `SyncStream` usando los traits `TSyncReadWrite` y `TStream`:

```smalltalk
Object subclass: #SyncStream
  uses:  TSyncReadWrite @ {#hashFromSync -> #hash}
      + TStream @ {#hashFromStream -> #hash}
  instVarNames: ''
  ....
```

En esta implementación, todas las variables de instancia son accesibles para los métodos definidos en la clase y en todos los traits usados. En el ejemplo, el método `isBusy` implementado en `SyncStream` accede a la variable de instancia definida en `TSyncReadWrite`. El conjunto final de variables de una clase traiteada contiene las variables definidas en su jerarquía y las variables definidas en los traits usados.

Cuando se define una clase traiteada, sus variables de clase y de instancia se calculan mediante un proceso simple de **flattening** (aplanado). Luego, los métodos contribuidos por sus traits usados son recompilados dentro de esta clase traiteada. Durante este proceso de compilación, el índice de las variables de instancia se recalcula para integrar todas las variables definidas en la clase y las contribuidas por herencia y por los traits usados.

Notar que en el futuro podría evaluarse una implementación basada en **copy-down** ("copia hacia abajo"), como la usada en lenguajes que soportan mixins.

> Nota al pie sobre copy-down: es una forma de minimizar la repetición de código en una implementación de mixins. "Los métodos que no acceden a variables de instancia ni a `super` se comparten en el mixin. Los métodos que acceden a variables de instancia pueden tener que especializarse para la invocación, donde el acceso a la variable de instancia se personaliza según la estructura de la instancia de la invocación." La idea es que, como el único método impactado por cambios a la representación interna del objeto (debido a la aplicación de un mixin) son los métodos que acceden a campos, basta con copiar hacia abajo, en las subclases (clases que representan la aplicación del mixin a otra clase), los métodos que acceden a campos. Todos los demás métodos quedan aplicables a todas las clases de aplicación del mixin.

El álgebra de composición de traits stateful extiende la de traits stateless (ver Sección 3.2). Incluye operaciones requeridas para resolver conflictos que surgen con variables de instancia, siguiendo la semántica descripta en BDNW07. Las operaciones extendidas son: **renaming** (`@@`, renombrado) y **removing** (`--`, eliminación) de variables de instancia.

### 5.2. Conflict Resolution

Cuando existen dos o más variables de instancia con el mismo nombre en la composición de traits, hay un conflicto. Un conflicto ocurre en dos escenarios distintos: (1) las variables de instancia tienen el mismo nombre pero sus valores no pueden compartirse o su uso es incompatible, y (2) las variables de instancia deberían fusionarse porque contienen la misma información y se requiere que sea compartida por ambos traits.

El Listado 7 ilustra el primer escenario:

```smalltalk
Trait named: #TNamed
  uses: {}
  instanceVariableNames: 'name'.

Trait named: #TWithRole
  uses: {}
  instanceVariableNames: 'name'.

Object subclass: #Person
  uses: TNamed + (TWithRole @@ {#name -> #roleName})
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'APackage'
```

La clase `Person` usa dos traits (`TNamed` y `TWithRole`); ambos traits definen y usan la variable de instancia `name`. Sin embargo, estas variables de instancia se usan de formas incompatibles: el trait `TNamed` la usa para almacenar el nombre de la persona, mientras que el trait `TWithRole` la usa para almacenar el nombre del rol. En este escenario las variables de instancia deberían renombrarse. Durante el proceso de flattening se crean ambas variables de instancia (`name` y `roleName`), y los métodos en `TWithRole` que usan la variable conflictiva original se reescriben para usar el nuevo nombre (`roleName`). El renombrado se logra usando el operador `@@` sobre `TWithRole`. Este operador crea una nueva composición con la variable de instancia renombrada según la definición dada.

El Listado 8 muestra un conflicto resuelto mediante la fusión de dos variables de instancia:

```smalltalk
Trait named: #TNamed
  uses: {}
  instanceVariableNames: 'name'.

Trait named: #TFormalName
  uses: {}
  instanceVariableNames: 'name prefix'.

Object subclass: #Person
  uses: TNamed + (TFormalName -- #name)
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'APackage'
```

En este escenario, ambos traits (`TNamed` y `TFormalName`) usan una variable de instancia `name` para almacenar el nombre de la persona. Su uso es compatible. Tener una variable duplicada con la misma información no es deseable. Para resolver este problema, se usa el operador `--` para eliminar la variable de instancia de `TFormalName`, dejando la definida en `TNamed`. Durante el proceso de flattening solo se crea una variable de instancia, y todos los métodos la usan.

**Un escenario sin conflicto.** Este tercer escenario es sobre dos traits con distintas variables de instancia, pero que almacenan la misma información. Esto no es un conflicto, pero fusionar las variables de instancia es deseable para no duplicar el mismo estado. Para resolver este escenario, primero se renombra una de las variables de instancia y luego el resultado se fusiona con la del otro trait. Para fusionarlas, se elimina la variable de instancia `name` de la composición resultante.

El Listado 9 muestra este escenario. El trait `TDisplayName` almacena el nombre de la persona en la variable de instancia `personName`, mientras que `TNamed` lo almacena en una variable de instancia llamada `name`. Como se dijo, no es un conflicto, pero la información no debería duplicarse en dos variables de instancia distintas. Para fusionarlas, primero se renombra la variable de instancia en `TDisplayName` de `personName` a `name`. Esto reemplaza a todos los usuarios de `personName` por `name`. Luego, como ambos traits tienen la variable de instancia `name`, se la elimina de uno de los traits. La clase resultante tendrá solamente una variable de instancia `name`:

```smalltalk
Trait named: #TNamed
  uses: {}
  instanceVariableNames: 'name'.

Trait named: #TDisplayName
  uses: {}
  instanceVariableNames: 'personName'.

Object subclass: #Person
  uses: TNamed + ((TDisplayName @@ {#personName -> #name}) -- #name)
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'APackage'
```

### 5.3. Trait Initialization

La inicialización de instancias de clases traiteadas es un punto importante a abordar. La pregunta es cómo componer e invocar la inicialización de los traits usados desde la entidad compuesta (sea un trait o una clase). En particular, no se desea que la inicialización de la clase tenga que cambiarse cada vez que cambia algún trait usado (de los traits que la clase usa). Se propone una estrategia simple para soportar esto.

La inicialización de las variables de instancia definidas en los traits se realiza mediante la redefinición del método **`initializeTrait`**. Cada trait que requiera inicialización de variables de instancia sobreescribe este método. Si un trait no incluye este método, se genera un método vacío durante el proceso de flattening. Cuando se instancia un objeto de una clase traiteada, se ejecuta tanto el código de inicialización de la clase como el código de inicialización del trait. El Listado 10 muestra cómo el trait `TTraitedClass`, que se aplica al lado de clase de todas las clases traiteadas, redefine el método `new`.

```smalltalk
TTraitedClass >> new
  ^ self new
      initialize;
      initializeTrait;
      yourself.
```

Es decir: cuando se crea una nueva instancia de una clase traiteada, además de `initialize` (la inicialización habitual de la clase) se invoca también `initializeTrait` (la inicialización aportada por los traits), y luego se devuelve la instancia (`yourself`).

Un escenario que debe tratarse correctamente es cuando una clase traiteada usa una composición de traits que incluye varios traits con código de inicialización. En este escenario, todo el código de inicialización debería instalarse en la clase traiteada. Para evitar choques de nombres, los distintos métodos `initializeTrait` se aliasean, y se genera un nuevo método `initializeMethod` que invoca a los métodos aliaseados. En el Listado 11, el método `initializeTrait` invoca a los métodos `initializeTSyncReadWrite` e `initializeTStream`, que fueron generados por el aliasing:

```smalltalk
SyncStream >> initializeTrait
  self initializeTSyncReadWrite.
  self initializeTStream.
```

### 5.4. Metaobject protocol support

El MOP propuesto soporta traits stateful siguiendo la misma idea expresada para traits stateless. Las mismas operaciones descriptas para métodos son aplicables a las variables de instancia. Como se dijo en la Sección 4, las operaciones relacionadas con variables de instancia tienen la siguiente semántica:

- **`slots`.** Devuelve todas las variables de instancia definidas localmente en la clase dada.

  > Nota al pie: "Slots" es el nuevo nombre para las variables de instancia de primera clase en Pharo 7.0.

- **`contributedSlots`.** Devuelve todas las variables de instancia contribuidas a esta clase. En el caso de clases normales, incluye las variables de instancia en la jerarquía de clases. En el caso de clases traiteadas, incluye las variables de instancia en la jerarquía de clases y las variables de instancia contribuidas por los traits.
- **`allSlots`.** Devuelve todas las variables de instancia, incluyendo las localmente definidas y las contribuidas.
- **`originOfSlot: anInstanceVariable`.** Devuelve la clase o el trait que define la variable de instancia.

Usando este MOP, las herramientas se implementan sin tener que diferenciar si están trabajando con clases normales, traits o clases traiteadas.

## 6. Implementation Details

La solución propuesta fue validada reemplazando la implementación de traits en Pharo 7.0. Esta implementación es parte de la versión de Pharo 7.0 desplegada y publicada. Incluye la implementación descripta en la Sección 3, el MOP propuesto en la Sección 4, y la extensión propuesta en la Sección 5.

Esta implementación requirió resolver una serie de cuestiones técnicas. Las soluciones se presentan en esta sección.

### 6.1. Polymorphic Classes and Traits

Para simplificar la integración de traits con las herramientas existentes, los traits se representan como **clases regulares**: son subclases de la clase `Trait`. La única restricción impuesta por diseño es que los traits **no pueden extenderse mediante herencia**. La superclase común `Trait` permite especializar fácilmente métodos específicos de traits e implementar una interfaz limpia y polimórfica con clases regulares. Por ejemplo, la clase `Trait` sobreescribe `compile:` para compilar e instalar un método localmente en un trait, y luego propagar dicho cambio a todos sus usuarios.

### 6.2. Implementing an extensible algebra

La Figura 7 del paper ilustra cómo se implementa el álgebra de composición de traits de forma extensible. Se implementaron las operaciones del álgebra de composición de traits como un conjunto de objetos de primera clase. Estos objetos de primera clase se representan como clases subclases de **`TraitComposition`**. `TraitComposition` define métodos para manejar la composición. Primero se representaron las operaciones stateless, como se muestra en la Figura 7(a): una jerarquía abstracta `TraitComposition` con subclases concretas como `EmptyComposition`, `Sequence`, ..., `AliasMethod`. Luego, ese conjunto de operaciones se extiende para soportar las operaciones de traits stateful y la resolución de conflictos (Sección 5), agregando una extensión `SlotsOperations` con subclases como `RemoveSlot`, `RenameSlot`, ..., `MergeSlot`, como se muestra en la Figura 7(b).

### 6.3. Flattening and Method Dictionaries

La solución maneja la construcción de clases de la misma forma que la implementación de Pharo 6.0: los métodos contribuidos por un trait se aplanan (*flatten*) dentro del **method dictionary** (diccionario de métodos) de la clase. Entonces, la **máquina virtual (VM)** ejecuta los métodos definidos en la clase, los aportados por los traits usados y los heredados, usando el lookup de métodos por defecto de la VM, sin penalizaciones en la ejecución. Sin embargo, las clases traiteadas tienen **dos diccionarios de métodos**.

- Un primer diccionario de métodos se usa para almacenar todos los métodos visibles para la VM. Este diccionario de métodos es interno a la implementación, y no es accesible a través del MOP.
- Para ser totalmente compatibles con las herramientas existentes, las clases traiteadas y sus metaclases tienen un diccionario de métodos adicional, con solo los métodos definidos en la clase. Este segundo diccionario de métodos es el que es accesible a través del MOP. De esta forma, el diccionario de métodos que es visible desde afuera tiene la misma semántica que el diccionario de métodos de una clase normal. Cualquier cambio a los métodos disponibles de la clase se refleja en el diccionario de métodos oculto.

Cada método instalado en la clase traiteada conoce el trait original donde fue definido. Esta información se almacena como propiedades adicionales en el método. El acceso a esta información se realiza a través del Metaobject Protocol de la clase, y la existencia de esta propiedad adicional es un detalle de implementación que, nuevamente, no es parte del MOP y por lo tanto no se expone al usuario.

En la implementación de Pharo se observa que el impacto en memoria de tener un segundo diccionario de métodos es mínimo. Este segundo diccionario de métodos comparte los símbolos y métodos con el diccionario por defecto. La implementación solo requiere memoria extra para la colección misma. Midiendo la imagen de Pharo 7.0, el número promedio de métodos locales en las 545 clases que usan traits es 2. Entonces, el impacto promedio por clase es de 32 bytes en imágenes de 32 bits, y de 64 bytes en una imagen de 64 bits (4 variables de instancia para el diccionario mismo, y 2 variables de instancia por cada una de las 2 asociaciones usadas).

### 6.4. Supporting Live Programming

Pharo es un entorno de **live programming** (programación en vivo). Permite al desarrollador modificar su código mientras se está ejecutando. Entonces, cualquier cambio hecho a un trait debería propagarse a los usuarios de ese trait. Para hacerlo, un trait conoce a todos sus usuarios, y cuando se modifica (es decir, hay una modificación de método o de variable de instancia) notifica a sus usuarios. También los usuarios de un trait están sujetos a cambio (es decir, una clase se define para usar el trait, o una clase que usa el trait se modifica para no usarlo más).

Este mecanismo de propagación se implementa usando dos estrategias. Primero, el mecanismo de notificación se implementa a través de un sistema de eventos de **announcement** (anuncio). El sistema de eventos de anuncio genera eventos cada vez que el sistema se modifica. Todas las herramientas que necesitan reaccionar a estos cambios están suscriptas a estos eventos. Segundo, un trait intercepta sus propias modificaciones sobreescribiendo los métodos correspondientes en la clase `Trait`.

El sistema de eventos de anuncio se usa para actualizar a los usuarios de un trait cuando una clase traiteada se crea, se modifica o se elimina del sistema. La sobreescritura de modificación se usa cuando se modifica un trait (es decir, sus métodos o variables de instancia), y los usuarios deben actualizarse.

### 6.5. Class Builder and Installer Replacement

La creación e instalación de clases en Pharo la realizan un **Class Builder** y un **Class Installer**. Estos componentes del sistema realizan todas las operaciones necesarias para crear, modificar e instalar clases en el entorno. Para poder tener una implementación modular de traits, hicieron falta algunos cambios en ellos.

La solución reemplaza el class builder y el class installer con variantes que soportan extensiones. Se implementa un nuevo conjunto de class builder y class installer, llamados **`ShiftClassBuilder`** y **`ShiftClassInstaller`**. Por defecto, esta nueva implementación no tiene soporte para traits, ya que está pensada para incluirse en el kernel de Pharo sin traits. Sin embargo, este class builder permite la extensión del proceso de construcción mediante plugins, llamados **enhancers** (potenciadores/realzadores).

El class builder y el class installer son configurables usando distintos **builder enhancers**. Los builder enhancers contribuyen al proceso de construcción e instalación (Figura 8). También permite configurar qué clase de metaclase usar y cómo se crea. La implementación de traits se maneja con un enhancer específico: **`TraitBuildEnhancer`**. Este configura la clase y la metaclase para que tengan las variables de instancia agregadas por los traits, y también instala todos los métodos provistos por la composición de traits. Esto se logra sobreescribiendo las acciones por defecto del build enhancer.

La Figura 8 muestra que `ShiftClassBuilder` se relaciona con `DefaultBuilderEnhancer`, y `TraitBuildEnhancer` es una subclase de `DefaultBuilderEnhancer` que implementa los métodos `compileMethodsFor: aBuilder`, `configureClass: newClass superclass: superclass withLayoutType: layoutType slots: slots`, y `afterMethodCompiled: aBuilder`.

## 7. Validation

Como se dijo antes, la solución se aplicó a Pharo 7.0. Esto permite minimizar el kernel del lenguaje que necesita ser bootstrapeado en la creación de una imagen de Pharo 7.0.

> Nota al pie: una imagen de Pharo es un archivo binario agnóstico de la plataforma que contiene todos los objetos que representan al sistema: las clases de la distribución completa de Pharo y los métodos. Una imagen por defecto consiste en alrededor de 6000 clases y 120000 métodos.

La imagen de Pharo 7.0 se crea, después de cada commit, desde cero, generando una imagen de Pharo a partir del código fuente. En el proceso de generación de la imagen, primero se bootstrapea una imagen pequeña. Esta imagen contiene el kernel de lenguaje mínimo necesario para poder cargar código. Después de crear esta imagen mínima, se cargan todas las bibliotecas y herramientas, incluyendo la nueva implementación de traits. Esta es la imagen que se distribuye.

**Removing Traits from Kernel (eliminar traits del kernel).** Uno de los objetivos de implementar el soporte de traits como una biblioteca era eliminar el soporte de traits del kernel del lenguaje Pharo. El kernel del lenguaje incluye todos los elementos necesarios para cargar código nuevo y ejecutar código Pharo. La eliminación representa una disminución de 22.606 líneas de código (15.36%) y 2.897 métodos (20.79%).

> Nota al pie: esta información se extrajo del repositorio de GitHub de Pharo, calculando la diferencia antes y después de aplicar los cambios.

La Tabla 1 muestra el detalle del código eliminado:

| | Old Impl. | New Impl | Diminution |
|---|---|---|---|
| Lines | 147,248 | 124,642 | 22,606 (15.35%) |
| Packages | 49 | 44 | 5 (10.20%) |
| Classes | 694 | 587 | 107 (15.42%) |
| Methods | 13,937 | 11,040 | 2,897 (20.79%) |

Tener un kernel más chico permite crear imágenes más chicas con solo la funcionalidad requerida. Eliminar la implementación de traits del kernel del lenguaje acelera el proceso de bootstrap en un 30%, pasando de un proceso de bootstrap inicial de 16 minutos a un proceso de bootstrap de 11 minutos. Por el contrario, el tiempo de carga de las bibliotecas aumentó solo 1 minuto. Esto produce una mejora general de velocidad del 20% para el proceso completo.

> Nota al pie: información recuperada de las estadísticas del CI Server de Pharo.

**About traits and classes polymorphism (sobre el polimorfismo de traits y clases).** La implementación anterior intentaba definir una API polimórfica para clases y traits. Esto llevaba a situaciones ad-hoc donde, aunque ambas entidades ofrecían la misma API, los desarrolladores todavía tenían que ser conscientes de si estaban manipulando una clase o un trait (por ejemplo, no tiene sentido mostrar las superclases de un trait). Esto producía código condicional innecesario para diferenciar traits y clases. Este problema no solo está presente en el kernel, sino también en todas las herramientas y bibliotecas que manipulan traits y clases en la nueva implementación.

Algunas de las herramientas afectadas son el compilador (*Opal*), gestores de versiones (*Epicea* y *Monticello*), navegación de código (*GT-Spotter*), framework de modelado (*Ring*) y el navegador y editor de clases (*Nautilus* y *Calypso*). Las nuevas implementaciones representan una reducción de 6.557 líneas de código distribuidas en 89 paquetes. Este impacto es visible en el kernel del lenguaje, pero es más importante cuando se analiza el sistema completo.

**Moose.** Moose es una plataforma de análisis de software y de datos. Desde su versión 7.0 usa traits stateful como un ladrillo básico clave para modelar lenguajes (Java, Pharo, PostgreSQL, PowerBuilder, Ada, VisualBasic...) y realizar análisis de evolución de software. El núcleo de Moose define 123 traits stateful que representan preocupaciones elementales de programas. Esos traits luego se reutilizan para representar distintos lenguajes y construcciones de lenguaje. Esto no es una validación formal, pero la última versión de Moose es una usuaria intensiva de traits stateful.

## 8. Discussion

Se discute ahora el impacto de la nueva implementación y se revisitan algunas decisiones de diseño.

### 8.1. Tool Support

Los IDEs modernos están compuestos de muchas herramientas dedicadas. Pharo ofrece de fábrica: navegadores de mensajes, un cross-referencer, herramientas de navegación, navegador de paquetes/clases, navegador de cambios, inspectores de objetos avanzados, debuggers, autocompletado, soporte de VCS, y motores de reglas de calidad estáticos, por nombrar algunos.

Una implementación de traits cargable debería incluir todos los recursos y modificaciones necesarias para que las herramientas le den soporte completo. Debido a que los traits son polimórficos respecto de las clases, se puede eliminar la mayor parte del código condicional en las herramientas. Algunas herramientas deberían adaptarse para usar el nuevo Metaobject Protocol para clases normales y clases traiteadas. Sin embargo, este tema sigue siendo un esfuerzo en curso, ya que la integración definitiva requiere modificar herramientas existentes y la mayoría de las herramientas no están preparadas para ser extendidas o para permitir la inclusión de componentes conectables (*pluggable*). La misma consideración es aplicable a la nueva definición de clases necesaria para los traits. Existen herramientas que parsean la sintaxis de definición de clases, esperando recibir un formato dado. Esas herramientas deberían modificarse para permitir una sintaxis extensible, o para usar los mensajes expuestos en el MOP de las clases.

El trabajo que queda para una integración definitiva en Pharo está fuera del alcance de este trabajo, y tomará un par de años. Está relacionado con la mejora general de calidad de las herramientas existentes. Pharo es una plataforma compleja compuesta de unos 400 paquetes y que agrega varios subproyectos. Modificar todos los elementos requeridos es un proceso continuo que requiere tiempo, ya que ese proceso siempre debería proveer versiones útiles a los usuarios de la plataforma. Este trabajo se manejará junto con la comunidad de Pharo y su consorcio industrial.

### 8.2. About design

En la solución propuesta, se decidió separar la implementación de traits de las clases del kernel mediante el uso de una metaclase distinta para las clases traiteadas. De esta forma, todo el comportamiento relacionado con traits se carga como una biblioteca. Otra alternativa posible es tener una metaclase extensible en el kernel del lenguaje. Sin embargo, hacer una metaclase extensible en el kernel requiere tener un kernel más complejo que tener una implementación más simple.

Como se explicó al presentar la solución (Sección 3.4), se decidió definir el comportamiento requerido para implementar traits en un trait, llamado `TTraitedClass`, que se incluye cada vez que se crea una clase traiteada. El trait `TTraitedClass` incluye todos los métodos que deberían sobreescribirse en la clase `Class` para soportar clases traiteadas. Al hacerlo, se permite tener la capacidad de heredar de cualquier clase del sistema, y esto sin modificar la clase `Class`.

La solución extiende el MOP existente con mensajes para calcular el origen de los elementos de una clase. Esta decisión hace que las herramientas sean independientes del mecanismo de reuso usado. Si una herramienta dada quiere modificar la definición de un elemento, la clase que lo define es fácilmente alcanzable y modificable.

Se decidió tener la definición de traits **como clases**. Otra alternativa es la definición de traits como objetos independientes. Como es deseable tener polimorfismo entre traits y clases para compartir el soporte de herramientas, ambos deberían implementar el mismo MOP. Aprovechar que los traits son clases permite una implementación simple del Metaobject Protocol compartido.

Se decidió tener **dos diccionarios de métodos distintos**. Al hacerlo, la VM usa el diccionario de métodos con los traits aplanados, y las herramientas solo ven los métodos a través del MOP. Esta decisión crea una separación entre el modelo de runtime esperado por la VM y el modelo lógico expuesto por las herramientas. Además, al restringir a las herramientas a usar el diccionario de métodos solo con los métodos definidos localmente, se permite reutilizar la implementación de todos los métodos en la clase `Class`; ya que esos métodos esperan que los diccionarios de métodos tengan solo métodos definidos localmente.

La solución requiere una versión modificada del class builder y del class installer. Esta versión permite la extensión del proceso de construcción de clases.

## 9. Related Work

Otros lenguajes, como Scala, Rust, Groovy o Self, implementan traits como parte de su kernel de lenguaje. Están integrados en el lenguaje y no se pueden descargar. Además, los traits en Scala, Rust y Self no soportan la misma semántica de composición que los traits definidos en Schärli et al. y Ducasse et al. (los papers de referencia). Su semántica implementada está más cerca de los **mixins**.

Existen lenguajes, como Javascript, Python y Racket, que también implementan traits como una biblioteca cargable. Sin embargo, estas implementaciones se desarrollaron para un kernel de lenguaje sin soporte de traits. Entonces, no hay necesidad de modificar el kernel de lenguaje existente ni las herramientas dadas. Finalmente, las herramientas para estos lenguajes no dan soporte a traits.

Existen soluciones para agregar traits a lenguajes existentes con tipado estático, y especialmente a Java. Sin embargo, estas soluciones se enfocan en los tipos y en los desafíos de chequeo de tipos introducidos por los traits. Modifican el compilador o generan código equivalente al que usaría traits. No provee herramientas ni soporte extensible para el desarrollo usando traits.

Pharo ya incluye una implementación de traits funcional (la de Ducasse, Nierstrasz, Schärli, Wuyts y Black, DNS+06), aunque esta implementación está fuertemente acoplada con el kernel del lenguaje, haciendo imposible convertirla en una biblioteca cargable (Lienhard).

**Talents** son traits específicos de instancia. El prototipo temprano de implementación (Ressia, Bergel y Nierstrasz) provee formas de expandir Pharo mediante una biblioteca cargable, pero esta solución tiene un gran impacto en el rendimiento de ejecución, ya que el lookup de métodos se resuelve en el lenguaje sin aprovechar las optimizaciones de rendimiento de la VM. Además, los métodos de talent se definen como strings planos pasados como argumentos a métodos, y no hay soporte de herramientas de ningún tipo.

Razavi et al. ya extendieron un lenguaje existente mediante la definición de metaclases personalizadas. Sin embargo, aplicaron esa solución para implementar **Adaptive Object Models**. No aplican la solución para modificar los mecanismos de reuso disponibles en el lenguaje.

Cazzola et al. también dieron soporte para modificar la semántica del lenguaje y los mecanismos de reuso a través de módulos cargables dinámicamente. Sin embargo, su solución está centrada en una arquitectura de interpretador modular. Modifican el lenguaje modificando el interpretador del mismo. Esta solución no es aplicable al caso de este paper por razones de rendimiento.

Bouraqadi et al. muestran cómo se extienden las clases en un lenguaje reflectivo usando metaclases. Las metaclases se usan entonces como un mecanismo de extensión para agregar nuevas propiedades a las clases, sobre una base por clase. Esta nueva implementación también aprovecha las metaclases para implementar traits, tal como ellos lo hicieron por ejemplo con Mixins. A diferencia de ellos, este trabajo también analiza el impacto de cambiar la implementación de traits actual en el entorno de programación y en sus herramientas provistas. También se introduce un nuevo MOP reflectivo modular que evita que las nuevas extensiones de lenguaje modifiquen el runtime existente manteniendo la compatibilidad.

Malayeri et al. presentan **CZ**, una alternativa a los traits para resolver el problema del diamante. Usa una forma alternativa de combinar clases que extiende la herencia múltiple con restricciones e imports. Sin embargo, esta solución está centrada en el desarrollo de un lenguaje y no provee herramientas ni una biblioteca de runtime.

Tesone et al. muestran una arquitectura general para implementar extensiones modulares a un lenguaje de programación preservando el rendimiento mientras se corre en una VM de herencia simple. Muestra cómo un diseño modular de los mecanismos subyacentes (es decir, el class builder) soporta la introducción de mixins, traits y herencia múltiple. La implementación presentada en este artículo actual usa la misma idea (es decir, metaclases) y un proceso de construcción de clases modular. Sin embargo, es menos general y está enfocada en traits. Además, resuelve el problema más general del desajuste de API entre clases y traits, y diseña un Metaobject Protocol adecuado que tiene en cuenta a los traits.

## 10. Conclusion

En este paper, se abordó el problema de cómo modularizar una parte central del entorno Pharo. Se tomó la implementación existente de traits y se la separó del kernel del lenguaje. Una vez identificados todos los mecanismos, se propuso una implementación novedosa que usa una especialización de las metaclases del kernel para soportar traits. Esta nueva implementación expone un MOP común con las clases existentes. Dado que esta implementación está basada en los mecanismos de implementación de clases, ofrece mejor soporte de herramientas.

El uso de la solución propuesta y su implementación permitió reducir el tamaño y la complejidad del kernel de lenguaje de Pharo. Esta reducción acelera el proceso de bootstrap en un 30%. Un proceso de bootstrap más rápido mejora la capacidad de modificar el lenguaje, la infraestructura y las herramientas. Este resultado es crucial para tener un entorno de programación mantenido por la comunidad que esté completamente testeado y construido constantemente desde el código fuente. Los resultados de este impacto son notorios diariamente en los resultados del proceso de Integración Continua de Pharo.

Finalmente, tener una biblioteca de traits stateful independiente y cargable también logra otros dos objetivos: (1) produce un aumento de la modularidad del sistema, permitiendo a los usuarios seleccionar las características usadas por sus aplicaciones, y (2) abre la puerta a la investigación e implementación de modelos de reuso alternativos usando técnicas dinámicas y de ligadura tardía (*late-binding*) similares. El primer objetivo permite a los usuarios de Pharo producir fácilmente configuraciones a medida para su desarrollo y sus aplicaciones finales.

## Acknowledgements

El siguiente trabajo está apoyado por el proyecto I-Site ERC-Generator Multi 2018-2022. Los autores agradecen el apoyo financiero de la Métropole Européenne de Lille.

## References

El paper cierra con una lista de referencias bibliográficas (entre ellas trabajos de Bracha & Cook sobre mixin-based inheritance, Ducasse-Nierstrasz-Schärli-Wuyts-Black sobre Traits, Bergel-Ducasse-Nierstrasz-Wuyts sobre Stateful Traits, Lienhard sobre Bootstrapping Traits, Kiczales-des Rivières-Bobrow sobre el Art of the Metaobject Protocol, Bouraqadi-Ledoux-Rivard sobre Safe metaclass programming, Tesone-Polito-Fabresse-Bouraqadi-Ducasse sobre mecanismos de reuso modulares en una VM de herencia simple, entre otras) que sustentan las distintas afirmaciones técnicas e históricas hechas a lo largo del texto.
