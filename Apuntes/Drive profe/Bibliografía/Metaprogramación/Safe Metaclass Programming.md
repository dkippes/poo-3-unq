# Safe Metaclass Programming

Noury M. N. Bouraqadi-Saâdani, Thomas Ledoux & Fred Rivard — OOPSLA 1998

> Resumen detallado en español, organizado siguiendo la estructura original del paper, para poder seguirlo en paralelo con el PDF. No es traducción literal sino una explicación fiel y completa, en mis propias palabras, de cada sección.

## Abstract

En un sistema donde las clases son objetos de primera clase, las clases son instancias de otras clases llamadas **metaclases**. Un beneficio importante de usar metaclases es poder asignarle a una clase **propiedades** (por ejemplo: ser abstracta, ser final, trazar mensajes particulares, soportar herencia múltiple) de forma independiente del código de base (el código de las instancias normales). Pero cuando herencia e instanciación están involucradas simultáneamente y de forma explícita, la comunicación entre clases y sus instancias plantea el problema de la **compatibilidad de metaclases** (*metaclass compatibility*). Algunos lenguajes (como Smalltalk) resuelven este problema de compatibilidad, pero no permiten asignar propiedades específicas a las clases fácilmente. Otros lenguajes (como CLOS) sí permiten asignar propiedades específicas a las clases, pero no resuelven bien el problema de compatibilidad.

En este paper los autores describen un nuevo modelo de organización a nivel de metaclases, llamado **the compatibility model** (el modelo de compatibilidad), que supera esta dificultad. Permite hacer **metaclass programming seguro** (*safe metaclass programming*), porque hace posible asignar propiedades específicas a las clases asegurando al mismo tiempo la compatibilidad de metaclases. De esta forma se puede aprovechar el poder expresivo de las metaclases para construir software confiable. Además, extienden este modelo de compatibilidad para permitir la reutilización y composición segura de propiedades de clase. Esta extensión está implementada en **NeoClassTalk**, una extensión de Smalltalk totalmente reflectiva.

**Keywords:** Metaclases, compatibilidad, propiedades específicas de clase, propagación de propiedades de clase.

## 1 Introduction

La idea de partida es que programar con metaclases trae beneficios importantes. Un uso interesante de las metaclases es poder asignarle **propiedades específicas** a una clase. Por ejemplo: una clase puede ser abstracta, puede tener una única instancia permitida, puede trazar los mensajes que recibe, puede definir pre/post-condiciones en sus métodos, puede prohibir que se redefinan ciertos métodos, etc. Todas estas propiedades pueden implementarse usando metaclases, lo que permite personalizar el comportamiento de las clases sin tocar el código de base.

Desde un punto de vista arquitectónico, usar metaclases organiza una aplicación en **niveles de abstracción**. Cada nivel describe y controla al nivel inmediatamente inferior, con el que está conectado causalmente (si cambia el nivel de arriba, cambia el de abajo). Las clases (reificadas como objetos) se comunican con otros objetos, incluyendo sus propias instancias. Así, una clase puede enviar mensajes a sus instancias, y una instancia puede enviar mensajes a su clase. Este tipo de envío de mensajes entre niveles se llama **comunicación inter-nivel** (*inter-level communication*).

Sin embargo, una herencia hecha sin cuidado en un nivel puede romper la comunicación inter-nivel, generando un problema llamado **the compatibility issue** (el problema de compatibilidad). Los autores identifican **dos tipos simétricos** de problemas de compatibilidad:

- **upward compatibility** (compatibilidad hacia arriba): este problema fue nombrado *metaclass compatibility* por Nicolas Graube.
- **downward compatibility** (compatibilidad hacia abajo).

Ambos tipos de problemas de compatibilidad son obstáculos importantes para la programación con metaclases, de los que cualquier programador debería ser consciente.

Ningún lenguaje existente que trabaje con metaclases permite, hoy, asignar propiedades específicas a las clases asegurando al mismo tiempo la compatibilidad:

- **CLOS** permite asignar cualquier propiedad a las clases, pero no asegura la compatibilidad.
- **SOM** y **Smalltalk** sí abordan el problema de compatibilidad, pero a cambio introducen el problema de la **propagación de propiedades de clase** (*class property propagation*): una propiedad asignada a una clase se propaga automáticamente a sus subclases. Por lo tanto, en SOM y Smalltalk una clase no puede tener una propiedad específica solo para ella: por ejemplo, si se le asigna la propiedad de "ser abstracta" a una clase Smalltalk dada, sus subclases también se vuelven abstractas, lo cual no es deseable.

Esto deja al programador ante un dilema: usar un lenguaje que permite asignar propiedades específicas pero sin asegurar compatibilidad, o usar un lenguaje que asegura compatibilidad pero sufre el problema de propagación.

En este paper se presenta **the compatibility model**, un modelo que permite hacer *safe metaclass programming*: es decir, hace posible asignar propiedades específicas a las clases sin comprometer la compatibilidad. Además, evita la propagación de propiedades de clase: una clase puede recibir propiedades específicas sin que eso tenga ningún efecto colateral sobre sus subclases.

Este modelo de compatibilidad fue implementado en **NeoClassTalk**, una extensión de Smalltalk que introduce muchas características, incluyendo metaclases explícitas. Sus experimentos mostraron que el modelo de compatibilidad permite a los programadores aprovechar plenamente el poder expresivo de las metaclases. Este trabajo dio como resultado: (i) una herramienta que permite a un programador no familiarizado con metaclases manejar de forma transparente las propiedades específicas de clase, y (ii) un enfoque que permite reutilizar y componer propiedades de clase.

**Organización del paper:** la sección 2 presenta el problema de compatibilidad con ejemplos que muestran su importancia. La sección 3 muestra cómo los lenguajes existentes abordan (o no) el problema de compatibilidad, y cómo lidian con el problema de propagación de propiedades. La sección 4 describe la solución de los autores (el compatibility model) y la ilustra con un ejemplo. La sección 5 trata la reutilización y composición de propiedades específicas de clase dentro del modelo de compatibilidad, y bosqueja el uso del modelo extendido tanto para programadores de base como para programadores de metanivel. La última sección contiene un resumen final.

## 2 Inter-level communication and compatibility

Los autores definen **inter-level communication** (comunicación inter-nivel) como cualquier envío de mensaje entre clases y sus instancias (ver Figura 1 del paper). Los objetos clase pueden interactuar con otros objetos enviando y recibiendo mensajes; en particular, una instancia puede enviarle un mensaje a su clase, y una clase puede enviarle un mensaje a alguna de sus instancias. Usan Smalltalk para ilustrar el problema.

Hay dos mensajes que permiten esta comunicación inter-nivel en Smalltalk: **new** y **class**. Cuando se usa alguno de los dos, los objetos involucrados pertenecen a niveles de abstracción distintos:

- Un objeto que recibe el mensaje **class** devuelve su clase. El método `class` permite entonces "subir un nivel" (*go one level up*). Ejemplo (extraído de Visual Works Smalltalk), un método de instancia donde se le envía el mensaje `class` al receptor:

```smalltalk
"el mensaje name es enviado a la clase:"
Object>>printOn: aStream
    | title |
    title := self class name.
    ...

"el mensaje daysInYear es enviado a la clase:"
Date>>daysInYear
    "Answer the number of days in the year represented by the receiver."
    ^ self class daysInYear: self year
```

- Una clase que recibe el mensaje **new** devuelve una nueva instancia. El método `new` permite entonces "bajar un nivel" (*go one level down*). Ejemplo, dos métodos de clase que envían mensajes a las instancias recién creadas:

```smalltalk
"el mensaje at:put: es enviado a una instancia nueva:"
ArrayedCollection class>>with: anObject
    | newCollection |
    newCollection := self new: 1.
    newCollection at: 1 put: anObject.
    ^newCollection

"el mensaje on: es enviado a una instancia nueva:"
Browser class>>openOn: anOrganizer
    self openOn: (self new on: anOrganizer) withTextState: nil
```

Entonces, en Smalltalk la comunicación inter-nivel se materializa enviando los mensajes `new` y `class`. Otros lenguajes donde las clases también son reificadas (como CLOS y SOM) permiten un envío de mensajes similar.

Como estos mensajes de comunicación inter-nivel están embebidos dentro de métodos, son heredados cada vez que se heredan esos métodos. Asegurar la **compatibilidad** significa garantizar que esos métodos no van a provocar ninguna falla en las subclases, es decir, que todos los mensajes enviados siempre van a ser comprendidos por el receptor. Identifican dos tipos de compatibilidad: *upward compatibility* y *downward compatibility*.

### 2.1 Upward compatibility

Supongamos que `A` implementa un método `i-foo` que le envía el mensaje `c-bar` a la clase del receptor (ver Figura 2). `B` es subclase de `A`. Cuando se le envía `i-foo` a una instancia de `B`, es la clase `B` la que recibe el mensaje `c-bar`. Para evitar cualquier falla, `B` debería entender el mensaje `c-bar` (es decir, `MetaB` debería implementar o heredar un método `c-bar`).

```smalltalk
A>>i-foo
    ↑ self class c-bar
```

Gráficamente (Figura 2 del paper): hay una jerarquía de instanciación `MetaA → A` y `MetaB → B`, y una jerarquía de herencia `A → B` (B subclase de A). El método `i-foo` está definido en `A` y envía `c-bar` "hacia arriba", a su propia metaclase. El signo de pregunta en la figura está sobre la flecha `MetaA → MetaB`: la pregunta es si `MetaB` entiende `c-bar`, es decir, si hereda (o no) de `MetaA`.

**Definición de upward compatibility:** sea `B` una subclase de la clase `A`, `MetaB` la metaclase de `B`, y `MetaA` la metaclase de `A`. La compatibilidad hacia arriba está asegurada para `MetaB` y `MetaA` si y solo si: todo mensaje posible que no lleve a un error para ninguna instancia de `A`, tampoco llevará a un error para ninguna instancia de `B`.

En criollo: si todo método de instancia de `A` (heredable por `B`) que envía mensajes "hacia arriba" a su clase funciona sin errores para `A`, debe seguir funcionando sin errores cuando ese mismo método se ejecuta sobre una instancia de `B`. Para eso, `MetaB` tiene que entender al menos todos los mensajes que `MetaA` entiende, lo cual normalmente se logra si `MetaB` hereda de `MetaA`.

### 2.2 Downward compatibility

Supongamos que `MetaA` implementa un método `c-foo` que le envía el mensaje `i-bar` a una instancia recién creada (ver Figura 3). `MetaB` se crea como subclase de `MetaA`. Cuando se le envía `c-foo` a `B` (una instancia de `MetaA`... en este caso `B` es instancia de `MetaB`), `B` va a crear una instancia que va a recibir el mensaje `i-bar`. Para evitar cualquier falla, las instancias de `B` deberían entender el mensaje `i-bar` (es decir, `B` debería implementar o heredar el método `i-bar`).

```smalltalk
MetaA>>c-foo
    ↑ self new i-bar
```

Gráficamente (Figura 3): `MetaA → MetaB` es relación de herencia (MetaB hereda de MetaA), y `MetaA → A`, `MetaB → B` son relaciones de instanciación. El método `c-foo` definido en `MetaA` crea una instancia nueva (`self new`) y le envía `i-bar`. La pregunta en la figura está sobre la flecha `A → B`: ¿entiende `B` el mensaje `i-bar`?

**Definición de downward compatibility:** sea `MetaB` una subclase de la metaclase `MetaA`. La compatibilidad hacia abajo está asegurada para dos clases `B` (instancia de `MetaB`) y `A` (instancia de `MetaA`) si y solo si: todo mensaje posible que no lleve a un error para `A`, no llevará a un error para `B`.

En criollo: si la metaclase `MetaA` tiene un método de clase que, al crear una instancia nueva, le manda un mensaje a esa instancia recién creada (como `i-bar`), y `MetaB` hereda ese método, entonces las instancias de `B` (clase que es instancia de `MetaB`) tienen que entender ese mensaje también, o se rompe en runtime.

Ambos problemas (upward y downward) son simétricos: uno mira si la metaclase del hijo entiende lo que entiende la metaclase del padre (upward); el otro mira si la clase hija (vista como instancia) entiende lo que entiende la clase padre, cuando el método problemático está definido a nivel de las metaclases (downward).

## 3 Existing models

En esta sección, los autores muestran por qué ninguno de los modelos existentes permite asignar propiedades específicas a las clases asegurando, al mismo tiempo, la compatibilidad.

### 3.1 CLOS

Cuando se (re)define una clase en CLOS, se llama a la función genérica **validate-superclass**, antes de que se guarden las superclases directas. Por defecto, `validate-superclass` devuelve verdadero si la metaclase de la clase nueva es la misma que la metaclase de la superclase, es decir, **las clases y sus subclases deben tener la misma metaclase**. Por lo tanto, las incompatibilidades se evitan, pero la programación con metaclases queda muy restringida (Figura 4: una jerarquía donde A y B comparten obligatoriamente la misma metaclase MetaA).

En la práctica, los programadores suelen redefinir `validate-superclass` para que siempre devuelva verdadero. Así, los programadores de CLOS tienen total libertad para usar una metaclase específica para cada clase, y pueden asignar propiedades específicas a las clases — pero el costo es que tienen que estar siempre atentos al problema de compatibilidad, porque CLOS no los protege de él.

### 3.2 SOM

**SOM** es un producto compatible con CORBA de IBM, basado en una arquitectura de metaclases que sigue el modelo ObjVlisp. Las metaclases de SOM son explícitas y pueden tener muchas instancias, por lo que los usuarios tienen total libertad para organizar sus jerarquías de metaclases.

#### 3.2.1 Compatibility issue in SOM

SOM fomenta la definición y el uso de metaclases explícitas introduciendo un concepto único llamado **derived metaclasses** (metaclases derivadas), que resuelve el problema de **upward compatibility**. En tiempo de compilación, SOM determina automáticamente una metaclase apropiada que asegura la compatibilidad hacia arriba. Si es necesario, SOM crea automáticamente una nueva metaclase llamada metaclase derivada.

Supongamos que se quiere crear una clase `B`, instancia de `MetaB` y subclase de `A` (instancia de `MetaA`). SOM detecta un problema de upward compatibility, porque `MetaB` no hereda de la metaclase de `A` (`MetaA`). Por lo tanto, SOM crea automáticamente una metaclase derivada (`Derived`), usando **herencia múltiple**, para soportar todos los métodos y variables de clase necesarios (Figura 5 del paper). Cuando una instancia de `B` recibe `i-foo`, sube un nivel y le envía `c-bar` a la clase `B`. `B` entiende el mensaje `c-bar` porque su metaclase (`Derived`) es una metaclase derivada que hereda tanto de `MetaB` como de `MetaA`.

Sin embargo, SOM **no provee ninguna política ni mecanismo** para manejar la **downward compatibility**. Supongamos que `MetaB` se crea como subclase de `MetaA` (Figura 6). El método `c-foo`, heredado por `MetaB`, le envía el mensaje `i-bar` a una instancia nueva. Cuando la clase `B` recibe el mensaje `c-foo`, va a ocurrir un error en tiempo de ejecución, porque sus instancias no entienden el mensaje `i-bar`. Es decir: SOM resuelve un problema (upward) pero deja el otro (downward) completamente sin resolver.

#### 3.2.2 Class property propagation in SOM

SOM no permite asignarle una propiedad a una clase dada sin que esa misma propiedad se le asigne también a sus subclases. Los autores llaman a este defecto **the class property propagation problem** (el problema de propagación de propiedades de clase). Lo ilustran mostrando cómo las metaclases derivadas causan, de forma implícita, una propagación no deseada de propiedades de clase.

Supongamos que la clase `A` (Figura 7) es una clase **released** (liberada), es decir, no debería modificarse más — esta propiedad es útil en entornos de desarrollo multi-programador, para fines de versionado. Para evitar cualquier cambio, `A` es instancia de la metaclase `Released`. Sea `B` una clase que tiene una única instancia: `B` es instancia de la metaclase `SoleInstance`. Pero como `B` es subclase de `A`, SOM crea a `B` como instancia de una metaclase derivada automáticamente, que hereda tanto de `SoleInstance` como de `Released`. Así, en cuanto se crea `B`, queda automáticamente "bloqueada" y se comporta como una clase liberada — **¡no podemos definir ningún método nuevo sobre ella!** La propiedad de "released" se propagó a `B` sin que nadie lo pidiera, solo por el mecanismo automático de metaclases derivadas.

### 3.3 Smalltalk-80

En Smalltalk, las metaclases están parcialmente ocultas y son creadas automáticamente por el sistema. Cada metaclase es **no compartible** (*non-sharable*) y está fuertemente acoplada con su única instancia. Por eso, la jerarquía de metaclases es paralela a la jerarquía de clases, y se genera implícitamente cada vez que se crean clases.

#### 3.3.1 Compatibility issue in Smalltalk-80

Usando jerarquías paralelas, el modelo de Smalltalk asegura **tanto la compatibilidad hacia arriba como hacia abajo**. Cualquier código que use `new` o `class` se hereda y funciona correctamente. Cuando se crea la clase `B`, subclase de `A` (Figura 8), Smalltalk genera automáticamente la metaclase de `B` ("B class") como subclase de "A class", la metaclase de `A`. Supongamos que `A` implementa un método `i-foo` que le envía `c-bar` a la clase del receptor. Si se le envía `i-foo` a una instancia de `B`, la clase `B` recibe el mensaje `c-bar`. Gracias a las jerarquías paralelas, "B class" entiende el mensaje `c-bar`, así que la upward compatibility queda asegurada. De manera análoga, la downward compatibility también queda asegurada gracias a la jerarquía paralela.

#### 3.3.2 Class property propagation in Smalltalk-80

Como las metaclases son manejadas automática e implícitamente por el sistema, Smalltalk reduce drásticamente la oportunidad de cambiar el comportamiento de las clases, haciendo que la programación con metaclases sea "anecdótica". Igual que en SOM, Smalltalk no permite asignarle una propiedad a una clase sin propagarla a sus subclases.

```smalltalk
A class>>new
    self error: 'I am Abstract'
```

En la Figura 9: la clase `A` es abstracta porque sus subclases deben implementar algunos métodos para completar el comportamiento de instancia. `B` es una clase concreta, ya que implementa todo ese conjunto de métodos. Supongamos que queremos forzar la propiedad de abstracción de `A`. Para prohibir instanciar `A`, definimos el método de clase `A class>>new`, que lanza un error. Desgraciadamente, "B class" hereda el método `new` de "A class". Como resultado, **al intentar crear una instancia de B también se lanza el error** — ¡aunque `B` sea concreta y debería poder instanciarse sin problema! (Los autores aclaran en una nota que este ejemplo es deliberadamente simple — se podría evitar redefiniendo `new` en "B class" — pero esa "solución" es en realidad una anomalía de herencia que aumenta el costo de mantenimiento.)

## 4 The compatibility model

De los modelos anteriores, solo el de Smalltalk (con sus jerarquías paralelas) asegura compatibilidad completa. Pero no permite asignar propiedades específicas a las clases. Por otro lado, solo el modelo de CLOS permite asignar propiedades específicas a las clases, pero no asegura compatibilidad. Los autores creen que ambos objetivos pueden lograrse a la vez con un nuevo modelo que hace una **separación clara entre compatibilidad y propiedades específicas de clase**.

Ilustran la idea refactorizando el ejemplo de la Figura 9. Crean una nueva metaclase llamada **"Abstract + A class"**, subclase de "A class" (Figura 10). La clase `A` se redefine como instancia de esta nueva metaclase. Como "Abstract + A class" redefine el método `new` para lanzar un error, `A` no puede tener ninguna instancia. Sin embargo, como "B class" **no** es subclase de "Abstract + A class" (sino solo de "A class"), la clase `B` sigue siendo concreta — el error de abstracción no se propaga. Esta generalización del esquema es lo que llaman **the compatibility model**.

> Nota de convención: en el resto del paper, los nombres de las metaclases que definen alguna propiedad de clase se denotan concatenando el nombre de la propiedad, el símbolo `+` y el nombre de la superclase. Por ejemplo, "Abstract + A class" es subclase de "A class", que define la propiedad de abstracción llamada `Abstract`.

### 4.1 Description of the compatibility model

El modelo de compatibilidad extiende el modelo de Smalltalk separando dos preocupaciones (concerns): la **compatibilidad** y las **propiedades específicas de clase**.

- Una jerarquía de metaclases paralela a la jerarquía de clases asegura tanto la compatibilidad hacia arriba como hacia abajo, igual que en Smalltalk.
- Se introduce una "capa" extra de metaclases para asignarle propiedades a las clases de forma **local**. Las clases son instancias de metaclases que pertenecen a esta capa.

El modelo de compatibilidad se basa entonces en **dos "capas" de metaclases**, cada una dedicada a una única preocupación:

- **Compatibility concern (preocupación de compatibilidad):** la abordan las metaclases organizadas en una jerarquía paralela a la jerarquía de clases. Estas metaclases se llaman **compatibility metaclasses** (metaclases de compatibilidad). Definen todo el comportamiento que debe propagarse a todas las (sub)clases. Todos los métodos de clase que les envían mensajes a las instancias deben definirse en estas metaclases; y también todos los mensajes enviados a las clases por sus instancias deben definirse en estas metaclases.
- **Specific class properties concern (preocupación de propiedades específicas de clase):** la abordan metaclases que definen las propiedades específicas de clase. Estas metaclases se llaman **property metaclasses** (metaclases de propiedad). Una clase con una propiedad específica es instancia de una metaclase de propiedad que hereda de la correspondiente metaclase de compatibilidad. La metaclase de propiedad **no** está conectada con otras metaclases de propiedad, ya que define una propiedad específica de esa clase en particular.

La Figura 11 muestra el modelo de compatibilidad aplicado a una jerarquía de dos clases: `A` y `B`. Son, respectivamente, instancias de las metaclases "AProperty + AClass" y "BProperty + BClass". "AProperty + AClass" define propiedades específicas de la clase `A`, mientras que "BProperty + BClass" define propiedades específicas de la clase `B`. Como "AProperty + AClass" y "BProperty + BClass" no están conectadas por ningún vínculo entre sí, **no ocurre la propagación de propiedades de clase**. Así, `A` y `B` pueden tener propiedades distintas.

Como "AProperty + AClass" y "BProperty + BClass" son subclases de las metaclases de compatibilidad AClass y BClass (que sí son paralelas a la jerarquía A → B), tanto la compatibilidad hacia arriba como hacia abajo quedan garantizadas. Supongamos que `A` define dos métodos de instancia `i-foo` e `i-bar`. El método `i-foo` le envía el mensaje `c-bar` a la clase del receptor. El método `i-bar` es enviado a una instancia nueva por el método `c-foo`. Como la jerarquía de metaclases AClass/BClass es paralela a la jerarquía de clases A/B, no hay falla de comunicación inter-nivel.

En definitiva: la idea clave es que las propiedades "viven" en una capa separada (las property metaclasses), que cuelga de la capa de compatibilidad pero **no se conecta horizontalmente entre clases hermanas**, evitando así que una propiedad asignada a `A` se filtre hacia `B`.

### 4.2 Example: Refactoring the Smalltalk-80 Boolean hierarchy

La jerarquía de Boolean de Smalltalk-80 se muestra en la Figura 12: `Boolean` es una **clase abstracta** que define un protocolo compartido por `True` y `False`. `True` y `False` son clases concretas que **no pueden tener más de una instancia**. Estas propiedades (ser abstracta, tener una única instancia) son implícitas en Smalltalk. Usando el modelo de compatibilidad, los autores refactorizan la jerarquía de Boolean para hacer explícitas estas propiedades.

Primero crean **"Boolean class"**, que es una metaclase de compatibilidad. El segundo paso es crear la metaclase de propiedad **"Abstract + Boolean class"**, que fuerza a que la clase `Boolean` sea abstracta. Finalmente, construyen la clase `Boolean` instanciando la metaclase "Abstract + Boolean class".

Para refactorizar la clase `False`, primero crean la metaclase **"False class"**, como subclase de "Boolean class", para asegurar la compatibilidad. El segundo paso consiste en crear la metaclase de propiedad **"SoleInstance + False class"**, que fuerza a que la clase `False` tenga a lo sumo una instancia. Por último, crean la clase `False` instanciando la metaclase "SoleInstance + False class". La clase `True` se refactoriza de la misma manera. El resultado de reconstruir toda la jerarquía de Boolean se muestra en la Figura 13.

## 5 Reuse and composition within the compatibility model

Experimentaron el modelo de compatibilidad en **NeoClassTalk**, un Smalltalk totalmente reflectivo. Rápidamente se encontraron con la necesidad de **reutilizar y componer propiedades de clase**: clases no relacionadas, que pertenecen a jerarquías distintas, pueden tener las mismas propiedades, y una clase dada puede tener muchas propiedades a la vez.

En la sección anterior, tanto la clase `True` como la clase `False` tienen la misma propiedad (tener una única instancia), pero a cada clase de la jerarquía de Boolean se le asignó **solo una** propiedad. En la práctica, una clase necesita que se le asignen **muchas** propiedades. Por ejemplo, la clase `False` no solo debería tener una única instancia, sino que además no debería poder ser subclaseada (es decir, debería ser **final**, en terminología de Java). Entonces hace falta reutilizar y componer estas propiedades de clase respetando el modelo de compatibilidad.

En esta sección proponen una extensión del modelo de compatibilidad que soporta reutilización y composición de propiedades de clase. Cualquier lenguaje donde las clases sean objetos regulares puede integrar este modelo de compatibilidad extendido; NeoClassTalk fue usado como primera plataforma de experimentación.

### 5.1 Reuse of class properties

En Smalltalk, como las metaclases se comportan distinto de las clases, se definen como instancias de una clase particular llamada **meta-metaclase**, llamada `Metaclass`. `Metaclass` define el comportamiento de todas las metaclases en Smalltalk. Por ejemplo, el nombre de una metaclase es el nombre de su única instancia con la palabra "class" pospuesta:

```smalltalk
Metaclass>>name
    ↑thisClass name, ' class'
```

Los autores aprovechan este concepto de meta-metaclases para **reutilizar propiedades de clase**. Como las metaclases que implementan propiedades distintas tienen comportamientos distintos, necesitan **una meta-metaclase por cada propiedad de clase**. Las metaclases de propiedad que definen la misma propiedad de clase son instancias de la misma meta-metaclase.

Cuando se crea una metaclase de propiedad, la meta-metaclase la inicializa con la definición de la propiedad de clase correspondiente. Así, el código (variables de instancia, métodos, etc.) correspondiente a la definición de la propiedad de clase **se genera automáticamente**. La reutilización se logra creando metaclases de propiedad que definen la misma propiedad de clase como instancias de la misma meta-metaclase, es decir, se inicializan con la misma definición de propiedad de clase.

La raíz de la jerarquía de meta-metaclases se llama **PropertyMetaclass**, y describe la estructura y comportamiento por defecto de las metaclases de propiedad. Por ejemplo, el nombre de una metaclase de propiedad se construye a partir del nombre de la propiedad y el nombre de la superclase:

```smalltalk
PropertyMetaclass>>name
    ↑self class name, '+', self superclass name
```

En la jerarquía de Boolean refactorizada de la sección 4.2, tanto "SoleInstance + False class" como "SoleInstance + True class" definen la propiedad de tener una única instancia. La reutilización se logra definiendo a ambas como instancias de **SoleInstance**, subclase de `PropertyMetaclass` (ver Figura 14). Es decir: `SoleInstance` (la meta-metaclase) "sabe" cómo generar el código necesario (variable de instancia que guarda la única instancia permitida, método que chequea si ya existe, etc.) y cada metaclase de propiedad concreta ("SoleInstance + False class", "SoleInstance + True class") es una instancia de esa meta-metaclase, generada automáticamente con ese comportamiento, sin tener que escribir el código a mano dos veces.

### 5.2 Composition of class properties

Como una clase dada puede tener muchas propiedades, el modelo debe soportar la **composición** de propiedades de clase. Eligieron usar varias metaclases de propiedad organizadas en una **única jerarquía de herencia simple**, donde cada metaclase implementa una propiedad de clase específica.

Para ilustrar esta idea, modifican el vínculo de instanciación de la clase `False` (Figura 15). Definen dos metaclases de propiedad, una por cada propiedad:

- La primera metaclase de propiedad es **"SoleInstance + False class"**, que hereda de la metaclase de compatibilidad "False class".
- La segunda es **"Final + SoleInstance + False class"**, que es la clase de `False`. Se define como subclase de "SoleInstance + False class".

El esquema resultante respeta el modelo de compatibilidad: permite asignarle dos propiedades específicas a la clase `False` y sigue asegurando la compatibilidad.

#### 5.2.1 Conflict management

La composición de metaclases de propiedad no es trivial: hay que lidiar con conflictos que surgen al componer distintas metaclases de propiedad usando herencia. Hay dos tipos de conflictos: **name conflicts** (conflictos de nombre) y **value conflicts** (conflictos de valor).

- **Name conflicts:** ocurren cuando metaclases de propiedad **ortogonales** (es decir, que definen propiedades de clase no relacionadas entre sí) definen variables de instancia o métodos que tienen el mismo nombre. Estos conflictos se evitan adaptando la definición de una nueva metaclase de propiedad según sus superclases. Por ejemplo, aunque "SoleInstance + False class" y "SoleInstance + True class" definen la misma propiedad para sus respectivas instancias (False y True), pueden usar nombres de variables de instancia o de métodos distintos para no chocar.
- **Value conflicts:** ocurren cuando metaclases de propiedad **no ortogonales** definen métodos con el mismo nombre. La mayoría de estos conflictos se evitan haciendo que la jerarquía de metaclases de propiedad actúe como una **cadena de cooperación** (*cooperation chain*): cada metaclase de propiedad se refiere explícitamente a los métodos sobrescritos definidos en sus superclases (en NeoClassTalk, igual que en Smalltalk, esto se logra con la pseudo-variable `super`). Por eso cada metaclase de propiedad actúa como un **mixin**.

#### 5.2.2 Example of cooperation between property metaclasses

Supongamos que se le quieren asignar dos propiedades específicas a la clase `False` (Figura 16): (i) **tracing**, trazar todos los mensajes (`Trace`), y (ii) tener **breakpoints** en métodos particulares (`BreakPoint`). Estas dos propiedades manejan el manejo de mensajes, basándose en NeoClassTalk en la técnica de los "method wrappers" (envoltorios de método). El método `executeMethod:receiver:arguments:` es el punto de entrada para manejar mensajes en NeoClassTalk; customizar `executeMethod:receiver:arguments:` permite especializar el envío de mensajes (un `executeMethod:receiver:arguments:` por defecto es provisto por `StandardClass`, la raíz de todas las metaclases en NeoClassTalk, que simplemente aplica el método sobre el receptor con los argumentos). Así, cuando el objeto `false` recibe un mensaje, la clase `False` recibe el mensaje `executeMethod:receiver:arguments:`.

Según la jerarquía de herencia, (1) primero se hace el trace, luego (2), usando `super`, se hace el breakpoint, y (3) finalmente se ejecuta la aplicación regular del método (de nuevo invocada usando `super`):

```smalltalk
"(3) StandardClass>>executeMethod:receiver:arguments:"
StandardClass>>executeMethod: method receiver: rec arguments: args
    ...

"(2) BreakPoint+False class>>executeMethod:receiver:arguments:"
BreakPoint+False class>>executeMethod: method receiver: rec arguments: args
    method selector == stopSelector
        ifTrue: [self halt: 'Breakpoint for ', stopSelector].
    ↑super executeMethod: method receiver: rec arguments: args

"(1) Trace+BreakPoint+False class>>executeMethod:receiver:arguments:"
Trace+BreakPoint+False class>>executeMethod: method receiver: rec arguments: args
    self transcript show: method selector; cr.
    ↑super executeMethod: method receiver: rec arguments: args
```

Es decir: la metaclase más específica (`Trace+BreakPoint+False class`) loguea el selector en el transcript y delega con `super` a la siguiente metaclase en la cadena (`BreakPoint+False class`), que chequea si el selector coincide con el de un breakpoint configurado y, si es así, detiene la ejecución (`self halt:`); luego delega con `super` a la implementación estándar (`StandardClass`), que finalmente ejecuta el método real sobre el receptor. Cada nivel de la cadena de herencia agrega su comportamiento y delega al siguiente — el mismo patrón de "encadenamiento" que un mixin.

### 5.3 The extended compatibility model

Generalizando los ejemplos anteriores, se define el **extended compatibility model** (modelo de compatibilidad extendido), ver Figura 17. Cada metaclase de propiedad define las variables de instancia y los métodos involucrados en una propiedad única. Las metaclases de propiedad específicas de una clase dada se organizan en una **única jerarquía**. La raíz de esta jerarquía es subclase de una metaclase de compatibilidad. Cada metaclase de propiedad es instancia de una meta-metaclase que describe una propiedad de clase específica, lo cual permite su reutilización (esta jerarquía única se puede comparar con una linearización explícita de metaclases de propiedad compuestas usando herencia múltiple).

La creación, composición y eliminación de metaclases se manejan automáticamente respecto al modelo de compatibilidad extendido. Cada vez que se crea una clase nueva, se crea automáticamente una nueva metaclase de compatibilidad (de la misma forma en que Smalltalk construye su jerarquía paralela de metaclases). La asignación de una propiedad a esa clase resulta en la inserción de una nueva metaclase dentro de su jerarquía de metaclases de propiedad. Esta inserción se hace en dos pasos (la eliminación de una metaclase de propiedad se hace de forma simétrica):

1. primero, la nueva metaclase de propiedad se vuelve subclase de la última metaclase de la jerarquía de metaclases de propiedad;
2. luego, la clase se vuelve instancia de esta nueva metaclase de propiedad.

NeoClassTalk provee protocolos para cambiar dinámicamente la clase de un objeto (`changeClass:`) y la superclase de una clase (`superclass:`). Por lo tanto, la implementación de estos dos pasos es inmediata en NeoClassTalk, y la provee el método `composeWithPropertiesOf:`:

```smalltalk
PropertyMetaclass>>composeWithPropertiesOf: aClass
    self superclass: aClass class.
    aClass changeClass: self.
```

### 5.4 Programming within the extended compatibility model

Distinguen dos tipos de programadores: (i) **"base level programmers"**, que implementan aplicaciones usando el lenguaje y las herramientas de desarrollo, y (ii) **"meta level programmers"**, para quienes el lenguaje en sí mismo es la aplicación.

#### 5.4.1 Base Level Programming

Para hacer que el modelo sea fácil de usar para un "programador de base", el entorno de programación de NeoClassTalk incluye una herramienta que permite asignarle distintas propiedades a una clase dada usando un browser al estilo Smalltalk (Figura 18). Estas propiedades pueden agregarse y quitarse en tiempo de ejecución. El nivel de metaclases se construye automáticamente de acuerdo con la selección hecha por el "programador de base" — es decir, el programador de base no necesita entender ni manipular directamente metaclases: simplemente tilda/destilda propiedades en una interfaz y el sistema gestiona toda la maquinaria de metaclases de compatibilidad y de propiedad por detrás.

#### 5.4.2 Meta Level Programming

Para introducir **nuevas** propiedades de clase, los "programadores de metanivel" deben crear una subclase de la meta-metaclase `PropertyMetaclass`. Esta nueva meta-metaclase almacena las variables de instancia y los métodos que deben definir sus instancias (las metaclases de propiedad). Cuando esta nueva meta-metaclase se instancia, las variables de instancia previas se agregan a la metaclase de propiedad resultante y los métodos se compilan en el momento de la inicialización (una solución más rápida consiste en compilar una sola vez, lo que da "proto-methods": así, cuando una metaclase de propiedad se inicializa, los proto-methods se "copian" al diccionario de métodos de la metaclase de propiedad, permitiendo una instanciación rápida de meta-metaclases. Esto asume que la inicialización es parte del proceso de creación, lo cual es tradicionalmente cierto en casi todos los lenguajes, y en Smalltalk se logra redefiniendo `new` con `super new initialize`).

Por ejemplo, la evaluación de la siguiente expresión crea una metaclase de propiedad — instancia de la meta-metaclase `Trace` — que le asigna la propiedad de trace a la clase `True`:

```smalltalk
Trace new composeWithPropertiesOf: True.
```

Para lograr el trace, los mensajes deben capturarse y luego registrarse (loguearse) en un colector de texto (*text collector*). Por lo tanto, las instancias de `Trace` deben definir una variable de instancia (llamada `transcript`) correspondiente a un colector de texto, y un método que maneje los mensajes. El manejo de mensajes se logra usando el método `executeMethod:receiver:arguments:`, cuyo código fuente ya se presentó en la sección 5.2.2. Estas definiciones se generan cuando se inicializan las metaclases de propiedad, es decir, usando el método `initialize` de la meta-metaclase `Trace`:

```smalltalk
Trace>>initialize
    super initialize.
    self instanceVariableNames: 'transcript '.
    self generateExecuteMethodReceiverArguments.
```

## 6 Conclusion

Considerar a las clases como objetos de primera clase organiza las aplicaciones en distintos niveles de abstracción, lo que inevitablemente plantea problemas de compatibilidad hacia arriba y hacia abajo. Las soluciones existentes que abordan los problemas de compatibilidad (como Smalltalk) no permiten asignarle propiedades específicas a una clase dada sin propagarlas a sus subclases.

El **compatibility model** propuesto en este paper aborda el problema de compatibilidad y permite asignar propiedades específicas a las clases sin propagarlas a sus subclases. Esto se logra gracias a la separación de las dos preocupaciones involucradas: compatibilidad y propiedades de clase. Las compatibilidades hacia arriba y hacia abajo se asegura usando la jerarquía de **metaclases de compatibilidad**, que es paralela a la jerarquía de clases. Las **metaclases de propiedad**, que permiten asignar propiedades específicas a las clases, son subclases de estas metaclases de compatibilidad. Por lo tanto, se puede aprovechar el poder expresivo de las metaclases para definir, reutilizar y componer propiedades de clase en un entorno que soporta **safe metaclass programming**.

Las propiedades de clase mejoran la legibilidad, la reutilización y la calidad del código aumentando la **separación de concerns** (*separation of concerns*). De hecho, permiten una mejor organización de las bibliotecas de clases y de los frameworks para diseñar software confiable. Los autores están fuertemente convencidos de que su modelo de compatibilidad permite la separación de concerns basada en el paradigma de metaclases. Por lo tanto, promueve la construcción de software confiable que es fácil de reutilizar y mantener.
