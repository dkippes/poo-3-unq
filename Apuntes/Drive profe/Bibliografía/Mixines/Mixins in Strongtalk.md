# Mixins in Strongtalk
Lars Bak, Gilad Bracha, Steffen Grarup, Robert Griesemer, David Griswold, Urs Hölzle — 6 de junio de 2002 (ECOOP 2002 Inheritance Workshop)

> Resumen detallado en español, organizado siguiendo la estructura original del paper, para poder seguirlo en paralelo con el PDF. No es traducción literal sino una explicación fiel y completa, en mis propias palabras, de cada sección.

## Abstract

Describen el uso y la implementación de mixins en el sistema **Animorphic Smalltalk**, una máquina virtual Smalltalk de alto rendimiento y un entorno de programación. Los mixins son la unidad básica de implementación y están soportados directamente por la VM. A nivel de lenguaje, el código puede definirse tanto en mixins como en clases, pero las clases son solo "azúcar sintáctica" para la definición e invocación de mixins. El sistema de tipos de Strongtalk soporta el chequeo de tipos estático opcional de mixins de manera encapsulada. Independientemente del chequeo de tipos, el sistema resultante supera sustancialmente en rendimiento a las implementaciones existentes de Smalltalk.

## 1 Introduction

El concepto de mixin se originó en la comunidad LISP, donde se refería a una clase diseñada para operar con varias superclases distintas. Los mixins de LISP eran clases que respetaban una convención de programación que aprovechaba los algoritmos de linearización de herencia múltiple de Flavors/CLOS. Los mixins no eran una construcción del lenguaje en Flavors ni en CLOS.

En el paper de Bracha & Cook de 1990 ("Mixin-based Inheritance", resumido en el otro documento) los mixins se identificaron como una construcción lingüística formal. Este paper usa el término "mixin" en ese sentido salvo que se indique lo contrario. **Un mixin es una subclase abstracta parametrizada por su superclase.**

Reportan el diseño y la implementación de mixins en el contexto de un Smalltalk de alto rendimiento, el sistema Animorphic Smalltalk. Aunque varios autores estudiaron los mixins desde el punto de vista de los lenguajes de programación, este trabajo se distingue por:

- Una implementación de alto rendimiento.
- Una API reflectiva completa y un entorno de programación, incluyendo un chequeador de tipos opcional e incremental.

En la VM Animorphic, los **mixins** (y no las clases) son la unidad básica de implementación. Esto no implica ninguna penalización de rendimiento; por el contrario, la VM Animorphic supera a otras implementaciones de Smalltalk por un factor sustancial.

A nivel de lenguaje, el código puede definirse tanto en mixins como en clases. Internamente, todo el código de programa se almacena en mixins. Una clase es simplemente azúcar sintáctica para una definición de mixin correspondiente junto con la aplicación de ese mixin a una superclase particular.

El sistema incluye el sistema de tipos opcional de Strongtalk, que soporta el chequeo de tipos estático de la definición y el uso de mixins de manera encapsulada.

Las implementaciones de Smalltalk suelen incluir no solo un sistema de runtime, sino también una librería de clases y un entorno de desarrollo interactivo (IDE); esto también es así en el sistema Animorphic. Se incluye una librería de clases completa al estilo del "blue book" (Goldberg & Robson). Los mixins se usan en puntos clave de la librería, y el entorno de programación soporta la exploración (*browsing*) de declaraciones de mixin. Por debajo de los navegadores (*browsers*) hay una API reflectiva que da acceso a los mixins; de hecho, los mixins son la estructura básica manipulada por esa API.

**Estructura del resto del paper:** la sección 2 discute el modelo básico de programación. La sección 3 muestra ejemplos de uso de mixins. La sección 4 discute el chequeo de tipos (opcional) de mixins. La sección 5 discute las interacciones entre mixins y reflexión. La sección 6 discute la implementación. La sección 7 discute las decisiones de diseño del lenguaje y compara el enfoque con otros. La sección 8 discute brevemente el estado y la historia del proyecto. Finalmente discuten sus contribuciones y sacan conclusiones.

## 2 Basic Model

Un mixin especifica un conjunto de modificaciones (sobreescrituras y/o extensiones) que se aplicarán a un parámetro de superclase. Un mixin se diferencia de una definición de subclase ordinaria en que **abstrae sobre la identidad de su superclase**.

Matemáticamente, un mixin es una función que mapea una clase `S` a una subclase de `S` con un cuerpo de clase particular.

Ejemplo (basado en la clase estándar de librería `Magnitude`; en el sistema Animorphic, `Magnitude` se define como clase por compatibilidad, y luego se usa como mixin):

```
mixin Comparable(G) {
  G subclass {
    ≤ that  ^(self == that) || (self < that)
    > that  ^that < self
    ≥ that  ^that ≤ self
  }
}
```

El mixin `Comparable` abstrae sobre una superclase formal no especificada, `G`. El código del mixin asume que `G` implementa un método booleano `<` que implementa la relación "menor que". `Comparable` puede usarse en distintos puntos de la jerarquía de subclases, invocándolo sobre distintas superclases reales, de forma análoga a invocar una función en varios puntos de un cómputo. Esto se logra mediante `▷`, el operador de invocación de mixin:

```
ComparablePoint = Comparable ▷ Point
Number = Comparable ▷ BasicNumber
```

El operador `▷` toma un mixin `M` y una (super)clase `S`, y produce una nueva clase que modifica esa superclase `S` con el código definido en `M`. El mixin puede invocarse sobre distintas superclases para derivar clases diferentes. A los resultados se los llama **"invocaciones de mixin"** (*mixin invocations*). En todos los casos, el código fuente del mixin se comparte entre todas esas invocaciones, lo que promueve la modularidad.

Ver a los mixins como funciones es útil para entender sus propiedades. Puntos clave del modelo:

- Se distingue entre mixins y clases, igual que se distingue entre funciones y los valores que toman como argumento o producen como resultado. Un mixin no es una clase; debe invocarse sobre un parámetro de superclase real para producir una clase.
- Los mixins no afectan la semántica de la construcción de subclases ni de la búsqueda de métodos (*method lookup*). Esto es esperable, ya que los mixins resultan de aplicar directamente el principio de abstracción procedural a la subclasificación.
- Un mixin puede no ser aplicable a todas las clases, igual que una función no necesariamente está definida para todas las entradas. Por ejemplo, un mixin puede contener llamadas a métodos de `super`, que deben estar definidos en cualquier superclase real que se use. Algunos de estos requisitos pueden expresarse mediante anotaciones de tipo (ver sección 4).
- Un mixin puede componerse con otro mixin para producir un **"mixin compuesto"** (*composite mixin*), de manera completamente análoga a la composición de funciones: `(M1 * M2) ▷ S = M1 ▷ (M2 ▷ S)`.

Toda clase define implícitamente un mixin cuyo cuerpo es idéntico al cuerpo de la clase. El sistema reconoce esto y permite derivar mixins a partir de clases. Una **"derivación de mixin"**, escrita `C mixin`, denota el mixin definido implícitamente por la clase `C`.

Ejemplo de la librería de UI de Animorphic: supongamos que la clase `Region` representa una región en la pantalla. Es natural tener una clase que agrupe varias Regions en una única Region compuesta:

```
Region subclass CompositeRegion...
```

Es útil pensar en esta clase como una colección de Regions. Sin embargo, normalmente no se puede heredar la funcionalidad de `Collection`, porque los widgets gráficos pertenecen a una jerarquía separada. Para reutilizar toda la funcionalidad de la clase `Collection`, se redefine `CompositeRegion` así:

```
(Collection mixin ▷ Region) subclass CompositeRegion...
```

Ahora la superclase de `CompositeRegion` es una invocación de mixin: `Collection mixin ▷ Region`. El mixin invocado aquí es una derivación de mixin, `Collection mixin`. El mixin derivado opera exactamente como si la funcionalidad de la clase `Collection` se hubiera dado como una declaración de mixin explícita.

Los programadores pueden definir código en el contexto de una clase o de un mixin. Definir código en clases suele ser ventajoso porque es un paradigma familiar y relativamente concreto; por otro lado, definir mixins favorece la reutilización. En cualquiera de los dos casos, la posibilidad de usar el código nuevo de una clase en otras jerarquías de clases no se ve afectada.

## 3 Examples of Usage

### 3.1 Example 1: Critical Section

El primer ejemplo (mostrado en la Figura 1 del paper, una captura de pantalla del navegador de mixins sobre el mixin `InstanceCritical`) expresa la funcionalidad de una sección crítica (monitor). Tiene una variable de instancia, `monitor`, que representa el semáforo, con su método de acceso asociado. El mixin incluye un método `critical:` que recibe una clausura (*closure*) como argumento y la ejecuta dentro del monitor.

El navegador muestra las "signatures" de tipo de Strongtalk, entre corchetes angulares, para las variables de instancia y los métodos. Además, para chequear los tipos de la declaración del mixin se necesita información sobre las posibles superclases (ver sección 4.3.1). En este caso, el mixin es aplicable a cualquier clase, es decir, a cualquier subclase de `Object`. Esto se refleja en la parte superior de la ventana del navegador, en el encabezado **"Mixin on Object"**.

Sin mixins, esta funcionalidad habría que duplicarla en cada lugar donde se necesite, o alternativamente agregarla a la clase `Object` —pero sin soporte especializado del lenguaje (como en Java), esto agregaría una sobrecarga significativa a cada objeto. Este diseño no impide una optimización similar.

### 3.2 Example 2: I/O Streams

Consideremos el caso de un stream que puede hacer tanto entrada como salida. ¿Debería `InputOutputStream` estar en la jerarquía de `InputStream` o en la de `OutputStream`? En cualquiera de los dos casos habría que duplicar funcionalidad de una de las dos clases de stream. Este problema se evita usando mixins para los streams de entrada y de salida.

La declaración real de un stream de E/S en el sistema Animorphic es:

```
BasicOutputStream mixin ▷ BasicReadStream subclass BasicReadWriteStream
```

## 4 Typechecking

Los mixins interactúan con el sistema de tipos de varias formas: las declaraciones e invocaciones de mixin deben chequearse, y además, en un sistema de tipos nominal, las declaraciones de mixin afectan la relación de subtipado. El paper aborda cada uno de estos puntos.

Es crucial que, una vez que se chequeó el tipo de una declaración de mixin `M`, las invocaciones del mixin puedan chequearse solo a partir de la interfaz del mixin, sin necesidad de acceder a su código interno. Para esto hace falta entender qué es la interfaz de un mixin, lo cual depende de los conceptos de "signature de mixin" y "signature de clase".

### 4.1 The Signature of a Class

Chequear el tipo de una clase generalmente requiere comparar los tipos de los miembros de la clase con los tipos de los miembros correspondientes en la superclase. En sistemas de tipos que permiten chequear una clase sin acceso al código fuente de la superclase, siempre existe una **"signature de clase"** (usualmente implícita) que da la información necesaria para verificar las restricciones de tipado al heredar de una clase.

Animorphic Smalltalk usa el sistema de tipos de Strongtalk. Los tipos en Strongtalk nunca exponen métodos ni variables privadas. En cambio, la signature de clase de Strongtalk incluye las signatures y visibilidades de todos los métodos y mensajes, y los tipos de todas las variables de instancia y de clase. La signature de clase incluye todos los miembros heredados. De esto se sigue que la signature de clase de una clase `C` es completamente distinta tanto del tipo de una instancia de `C` como del tipo `C class` (el tipo de la única instancia de la metaclase de `C`).

### 4.2 The Signature of a Mixin

Una **signature de mixin** encapsula la información de tipos sobre el mixin que se necesita para chequear su aplicación a una superclase concreta. Esto equivale a la signature declarada de la superclase formal más los tipos de todos los campos y métodos declarados dentro del propio mixin.

Un mixin que requiere superclases que cumplan con la signature de clase `S` tiene a `S` como su **dominio**. Si, dada una clase con signature `S`, el mixin produce una clase de signature `C`, entonces `C` es el **rango** del mixin. El tipo de un mixin con dominio `S` y rango `C` puede escribirse como un tipo función sobre signatures de clase: `S → C`. Sin embargo, es más conveniente representarlo como un par `(S, δ)`, donde `S` es la signature de la superclase (como antes) y `δ` es una signature de clase que solo incluye información sobre los miembros declarados por el propio mixin.

### 4.3 Typechecking a Mixin

El chequeo de tipos de mixins se subdivide en dos partes: el chequeo de las declaraciones de mixin y el chequeo de las invocaciones de mixin.

#### 4.3.1 Typechecking Mixin Declarations

Para chequear el tipo de una declaración de mixin de forma aislada, es necesario que el mixin declare la signature de clase esperada para las superclases potenciales. En la práctica, esto se hace nombrando una clase particular (en la Figura 1, se usa la clase `Object` para este propósito).

Cualquier superclase real debe cumplir con la signature de la clase nombrada. Esto no implica que las superclases reales deban ser subclases de la clase nombrada en la declaración de signature de superclase.

El cuerpo del mixin se chequea entonces asumiendo que la superclase tiene la signature de superclase declarada. Esto permite al chequeador de tipos verificar que:

1. Ningún método del mixin tiene una signature que contradiga la de la signature de superclase declarada. Por ejemplo, si un método declarado en el mixin sobreescribe un método de la signature de superclase, se exige que su signature sea subtipo de la signature del método sobreescrito.
2. Ningún campo del mixin tiene una signature que contradiga la de la signature de superclase declarada. En Smalltalk, los campos quedan expuestos a las subclases, así que un mixin no puede declarar un campo con el mismo nombre que un campo declarado en la signature de superclase.
3. Todas las llamadas a `super` dentro del mixin están bien tipadas (es decir, la signature de superclase soporta todas esas llamadas con la signature de método correcta).
4. Todas las llamadas a `self` dentro del mixin están bien tipadas (es decir, esas llamadas están soportadas o bien por métodos declarados por el propio mixin, o bien por la signature de superclase con la signature de método correcta).

Los dos primeros puntos chequean la relación entre el mixin y su parámetro de superclase; los dos últimos chequean el cuerpo del propio mixin.

Sin embargo, esto en sí mismo no es suficiente, por el **"problema de la información negativa"** (Cardelli & Mitchell, 1989). La superclase real podría incluir campos o métodos adicionales. Como ejemplo, consideremos la clase `PatientStatus` bosquejada así:

```
Object subclass PatientStatus {
  ...
  critical: b ^<Boolean> {...}
}
```

`PatientStatus` pretende representar datos de monitoreo para pacientes de hospital. Para los fines del ejemplo, las únicas propiedades relevantes de `PatientStatus` son que es subclase de `Object` y que tiene un método `critical:` que toma un argumento booleano. Si el argumento es verdadero, se considera que el estado del paciente es crítico, y requiere cuidados intensivos. Ahora consideremos la invocación de mixin:

```
InstanceCritical ▷ PatientStatus
```

Aunque se chequeó que `InstanceCritical` está bien formado asumiendo una superclase con signature `Object`, y `PatientStatus` es subtipo de `Object`, la invocación **no es correcta en cuanto a tipos**. `PatientStatus` define el método `critical:`, que también define `InstanceCritical`, pero los tipos de los argumentos de `InstanceCritical>>critical:` y de `PatientStatus>>critical:` son incompatibles.

Evidentemente, hay que hacer chequeos de tipo adicionales en el momento de la invocación del mixin, como se discute a continuación. Sin embargo, esos chequeos no requieren acceso al código fuente ni del mixin ni de la superclase real: toda la información necesaria la dan la signature de la superclase real y la signature del mixin.

#### 4.3.2 Typechecking Mixin Invocations

En el momento de invocar un mixin, hay que verificar que la superclase real no introduzca miembros cuyo tipo entre en conflicto con los miembros correspondientes del mixin. Esto es, de hecho, una repetición de los chequeos 1 y 2 de la subsección anterior, pero aplicados a la **superclase real**.

#### 4.3.3 Typechecking Mixin Derivations and Compositions

Las derivaciones de mixin no requieren chequeo de tipos especial. La signature del mixin derivado debe inferirse: dada una clase `C` con superclase `S`, el tipo del mixin es `(S, δC)`, donde `δC` puede calcularse como la diferencia entre las signatures de `C` y de `S`.

La composición de mixins `M1 * M2` se chequea igual que se chequearía la invocación de mixin `M1 ▷ (M2 ▷ S)`, donde `S` es la superclase por defecto de `M2`.

### 4.4 Interactions with Subtyping

#### 4.4.1 Motivation

Cuando el subtipado es estructural, los mixins no introducen problemas nuevos respecto del subtipado. Sin embargo, la mayoría de los lenguajes de programación prácticos usan **subtipado nominal**, donde el subtipado se basa en relaciones declaradas explícitamente. Strongtalk comenzó como un sistema de tipos estructural y evolucionó hacia uno nominal con el tiempo. Hay que tener especial cuidado para soportar el uso de mixins en un sistema de tipos nominal.

La regla básica de subtipado usada en lenguajes basados en clases con sistemas de subtipado nominal es:

```
C1 ≤ C2 si y solo si:
  1. C1 == C2, o
  2. C1 es subclase directa de S y S ≤ C2
```

Consideremos la clase `CompositeRegion` mostrada antes, definida como `(Collection mixin ▷ Region) subclass CompositeRegion...`. La regla anterior permite concluir que `CompositeRegion ≤ Region`, pero **no** que `CompositeRegion ≤ Collection`. Claramente, se necesita alguna extensión de las reglas usuales de subtipado nominal.

#### 4.4.2 Mixin Subtyping in Strongtalk

En Strongtalk, toda declaración de clase, y también toda declaración de mixin, introduce implícitamente un tipo con el mismo nombre que la declaración. El tipo definido implícitamente para una clase se conoce como **"protocolo"**, que consiste en un conjunto de signatures de mensaje; es similar a una interfaz en Java. El protocolo de una clase es subtipo directo del protocolo de su superclase, extendido por las signatures de mensaje públicas declaradas por la propia clase.

El tipo definido implícitamente para un mixin es similar, pero los mixins no tienen superclase. En su lugar, se usa el protocolo de la clase usada para declarar la signature de superclase esperada como el supertipo.

Una invocación de mixin `M ▷ S` tiene como tipo implícito un **"tipo mixin"** del mismo nombre, escrito `M ▷ S`. Para fines de subtipado, un tipo mixin `t1 ▷ t2` actúa como un **tipo intersección**. Las reglas de subtipado para tipos mixin son:

1. `t ≤ s1 ▷ s2` si `t ≤ s1` y `t ≤ s2`.
2. `t1 ▷ t2 ≤ s` si `t1 ≤ s` o `t2 ≤ s`.

La intuición de la primera regla es: el conjunto de signatures de mensaje soportadas por `s1 ▷ s2` consiste en las signatures soportadas por `s1`, más cualquier signature adicional heredada de `s2`. `s1 ▷ s2` no soporta ninguna signature de mensaje que no provea ni `s1` ni `s2`. Como `t ≤ s1` y `t ≤ s2`, `t` debe soportar todas las signatures de `s1 ▷ s2`, y por lo tanto es subtipo estructural de `s1 ▷ s2`.

La segunda regla se justifica intuitivamente así: una invocación de mixin `t1 ▷ t2` debe ser subtipo de `t2` — esto lo garantizan las reglas de chequeo de tipos para invocaciones de mixin dadas en la sección 4.3.2. Por transitividad, `t1 ▷ t2 ≤ s` si `t2 ≤ s`. De forma más sutil, `t1 ▷ t2` también es subtipo estructural de `t1`. Para verlo, hay que notar que:

- `t1` debe haberse declarado con una signature de superclase esperada `C`.
- Toda signature de mensaje declarada en `t1` debe estar soportada por `t1 ▷ t2` — no podría haber sido sobreescrita por `t2`.
- Cualquier otra signature de mensaje soportada por `t1` debe haberse heredado de `C`. Pero los requisitos de chequeo de tipos dados en 4.3.2 garantizan que `t2 ≤ C`. Por lo tanto, cualquier signature de mensaje heredada por `t1` desde `C` (o sus subtipos) también se heredará de `t2` hacia `t1 ▷ t2`.

De ahí que `t1 ▷ t2 ≤ s` si `t1 ≤ s` o `t2 ≤ s` se siga directamente. Se mencionan reglas similares dadas para gbeta y Scala.

## 5 Mixins and Reflection

### 5.1 Mixins as Objects

Los mixins están **reificados como objetos** en Animorphic Smalltalk. Toda clase es una invocación de algún mixin, que puede obtenerse a través de la interfaz reflectiva del sistema. Usando reflexión, se puede determinar la estructura de un mixin particular (por ejemplo, qué variables de instancia o métodos define). También se puede averiguar qué invocaciones de ese mixin existen en el sistema, como lo ejemplifica el siguiente fragmento de código, que imprime una lista de todas las clases que invocan la abstracción "monitor" mostrada antes:

```
(Mirror on: InstanceCritical) invocations do:
       [:inv | Transcript show: inv name; cr]
```

Los **"Mirrors"** (espejos) son objetos especiales que se usan para reflejar otros objetos. Habiendo obtenido un espejo sobre `InstanceCritical`, se puede obtener una lista de sus invocaciones y ejecutar una clausura para cada elemento de la lista. El código de la clausura simplemente imprime en la consola (*transcript*) el nombre de cada invocación.

### 5.2 Reflective Update

Los mixins son la **unidad de actualización reflectiva**, porque son la unidad de almacenamiento de código en la VM. Para realizar un cambio reflectivo sobre una clase, primero se hace una copia del mixin de esa clase. Esa copia se modifica como se desee. Finalmente, la copia se **"instala"** atómicamente en lugar del mixin original. En ese momento, todas las invocaciones de ese mixin, sus subclases y las instancias de todas esas clases se actualizan para reflejar los cambios hechos al mixin.

En algunos aspectos, los mixins son más fáciles de usar para la actualización reflectiva que las clases. Un mixin no puede tener instancias, y un objeto mixin no tiene métodos que contengan código de usuario. No hay nada que se pueda hacer con un mixin no instalado salvo reflejarlo o enviarlo para su instalación. Como resultado, se puede hacer una serie de cambios reflectivos sobre un mixin no instalado sin necesidad de actualizar todo el sistema en cada cambio individual. Incluso es posible instalar simultáneamente varios mixins en una única operación atómica.

Esto tiene dos ventajas:

1. Una serie de cambios reflectivos puede constituir una única transacción: ninguno de los cambios se aplica a menos que todos tengan éxito, y todos ocurren a la vez. Muchos cambios de programa siguen este patrón.
2. Si se hacen varios cambios de esquema, es mucho más barato agruparlos (*batch*) y evitar recorridos repetidos del heap para actualizar todas las instancias.

Esto contrasta con el enfoque estándar de Smalltalk, donde cada operación de cambio tiene efecto inmediato.

Es más difícil obtener las propiedades deseadas 1 y 2 si los cambios reflectivos se hacen sobre clases en lugar de mixins. Para ver por qué, examinan dos alternativas: **"scratchpad classes"** (clases borrador) y **"functional update"** (actualización funcional).

**Scratchpad classes:** se podría considerar hacer copias de clases para usarlas como "borradores", con la intención de cambiar su estructura e instalarlas después. Esto es análogo al enfoque que se toma con los mixins, pero falla con las clases por las siguientes razones:

- **Las clases pueden instanciarse.** Imaginemos que se instancia una clase borrador, y luego se le aplican una serie de operaciones de cambio, como agregar variables de instancia y métodos que las acceden. Si los cambios a una clase borrador no tienen efecto inmediato, surge una inconsistencia entre la clase y sus instancias. Intentar invocar un método sobre esa instancia podría llevar a comportamiento indefinido.
- **Las clases son usualmente con estado.** En Smalltalk, las clases pueden tener sus propias variables de instancia y variables de clase. Consideraciones similares aplican en otros lenguajes: Eiffel tiene variables `once` por clase; en Java, las clases se asocian con variables estáticas y locks por clase, etc. Una vez creada una copia borrador de una clase, el estado de la clase original puede evolucionar independientemente del estado de la clase borrador. Cuando se instala la clase borrador, hay que reconciliar de alguna manera su estado con el de la clase que se está actualizando: ¿se preserva el estado de la original, el del borrador, o alguna combinación de ambos?
- **Las clases tienen métodos específicos de la aplicación** (métodos de clase). Esto es cierto no solo en Smalltalk sino también en Java. Si estos métodos pueden invocarse antes de instalar la clase, se pierde la noción de cambio transaccional.

Para evitar estos problemas habría que impedir que esas clases no instaladas se instancien o reciban mensajes específicos de la aplicación de cualquier tipo. Estas restricciones convierten efectivamente a las clases no instaladas en una entidad completamente distinta de una clase normal: se vuelven **"descripciones de clase"** (*class descriptions*) que son independientes de la clase de la que derivan. Esas descripciones no interactúan de ninguna manera con el estado de programa no reflectivo. Estas descripciones son muy parecidas a los mixins, excepto que especifican una superclase particular.

**Functional update:** un enfoque alternativo es que los cambios reflectivos sobre una clase no la alteren destructivamente; en cambio, cada operación de cambio reflectivo devolvería una copia nueva de la clase, con los cambios correspondientes aplicados. Esta solución tiene varias desventajas:

- El copiado involucrado puede ser costoso.
- Las clases copiadas deben tener nombres distintos, que hay que gestionar automáticamente.
- La invocación de métodos de clase todavía podría perturbar los invariantes de la clase original.

Conclusión de los autores: cualquier solución debería realizar los cambios reflectivos sobre una descripción puramente declarativa. En la práctica, los mixins son justamente esa descripción.

## 6 Implementation

### 6.1 Highlights of the Animorphic VM

La VM Animorphic combina un intérprete de alto rendimiento con un compilador dinámico, formando un motor de ejecución de modo mixto. La invocación del compilador dinámico se decide en base a datos de perfilado (*profiling*) obtenidos dinámicamente.

El despacho dinámico de métodos se soporta mediante el uso de cachés en línea polimórficos (PICs, *polymorphic inline caches*). Las clases y los mixins tienen tablas de métodos asociadas que almacenan referencias a sus métodos, pero estas deben distinguirse de las tablas de despacho virtual comúnmente usadas en la implementación de lenguajes como C++ y Java: en el sistema Animorphic **no se usan tablas de despacho virtual**.

### 6.2 Transparent Lookup

En un sistema basado en mixins, una clase no incluye directamente en su tabla de métodos todos los métodos que declara: esos métodos se comparten naturalmente en la tabla de métodos del mixin. En consecuencia, el proceso de búsqueda de métodos (*method lookup*) debe cambiarse para consultar también al mixin. Sin embargo, estos cambios pueden, en principio, localizarse en la rutina que recorre la jerarquía de clases durante la búsqueda de métodos. Los cambios en la búsqueda pueden ser **"transparentes"** para el resto de la VM; en particular, técnicas de cacheo como los PICs siguen operando sin cambios.

En la práctica, las VMs operan sobre estructuras de datos concretas y realizan varias optimizaciones de rendimiento que complican este panorama.

### 6.3 Data Structure

La VM representa los mixins como objetos especiales que contienen la descripción de un "delta" de clase. El mixin contiene descripciones de las variables de instancia y de clase, y un diccionario de métodos donde se almacena inicialmente todo el código.

Todas las invocaciones de mixin contienen: las variables de clase (distintas para cada invocación, pero compartidas por una invocación y todas sus subclases), las variables de instancia propias de la clase (distintas para cada invocación, y de hecho para cada clase), la superclase (única para cada invocación), un puntero al mixin, e información adicional de implementación (como el offset de las variables de instancia, que varía según la superclase).

Uno de los problemas al compartir código entre invocaciones de mixin es que el layout físico de las instancias varía entre invocaciones. En código de alto rendimiento, las referencias a variables de instancia deben convertirse en offsets físicos relativos al comienzo de la instancia. Esto no es posible si el código se comparte entre invocaciones con distinta estructura de instancia. Este problema se resuelve mediante el mecanismo de **"copying down"** (copiado hacia abajo), descrito a continuación.

### 6.4 Copying down Methods

Los métodos que no acceden a variables de instancia ni a `super` se comparten en el mixin sin cambios. Los métodos que sí acceden a variables de instancia pueden tener que especializarse para cada invocación, donde el acceso a la variable de instancia se "personaliza" (*customiza*) según la estructura de las instancias de esa invocación.

Esta "customización" consiste en pedirle al compilador que compile una versión del método en el contexto de una clase particular: la invocación de mixin en cuestión. La versión personalizada debe instalarse en esa invocación. Al proceso de personalización e instalación se lo llama **"copying down"**.

En lugar de copiar los métodos hacia abajo cuando se crea una nueva invocación de mixin, se los copia perezosamente (*lazily*), la primera vez que se invoca el método.

La tabla de métodos del mixin contiene una versión de cada método definido en el mixin, incluso los métodos que acceden a variables de instancia o a `super`. Esta versión debe existir siempre, como una plantilla que las invocaciones del mixin usarán para personalizar el método cuando sea necesario.

Cuando se llama a un método sobre una invocación de mixin, se lo busca; si no existe una versión personalizada, se encuentra la versión del mixin. Esa versión puede entonces personalizarse para la invocación en cuestión, y esa versión personalizada se instala en la invocación.

El "copying down" puede evitarse por completo en ciertas circunstancias:

- Para mixins declarados explícitamente (a diferencia de derivaciones y composiciones de mixin), los métodos que acceden a variables de instancia se compilan asumiendo un offset inicial de 0. Cuando la superclase real de una invocación no tiene variables de instancia, puede usarse la versión compilada para el mixin sin necesidad de "copying down".
- Si un mixin representa una declaración de clase, se asocia el mixin con su **"invocación maestra"** (*master invocation*), que es la clase a partir de la cual se derivó el mixin. La invocación maestra se guarda en una variable de instancia del mixin. Cualquier invocación puede verificar si es la maestra examinando su mixin y viendo si la maestra es idéntica a sí misma. La personalización del método para la invocación maestra se hace "in place" (en el lugar), la primera vez que se llama al método. Por eso la invocación maestra nunca necesita tener métodos copiados hacia abajo. Cuando se invoca un método sobre otra invocación, también se puede evitar el "copying down" si el tamaño de las instancias de la superclase de esa invocación coincide con el de las instancias de la superclase de la invocación maestra.

Los métodos que acceden a `super` deben copiarse hacia abajo, excepto en el caso de la invocación maestra.

### 6.5 The Cost of Mixins

El alto rendimiento de la VM Animorphic es **independiente** del uso de mixins: no se logra gracias al uso de mixins. Al mismo tiempo, los mixins no degradan el rendimiento. Los mixins prácticamente no tienen impacto en el rendimiento.

No hay sobrecarga de espacio por método al usar mixins, siempre que se deriven de clases al estilo Smalltalk convencional. Puede ocurrir una sobrecarga de espacio por método solo para algunos métodos en mixins "regulares", que por diseño suelen tener varias invocaciones. En la práctica, los mixins de propósito general rara vez tienen estado propio o acceden a `super`, así que esta situación es poco frecuente.

Solo hay una ligera sobrecarga por clase, que es despreciable, y la sobrecarga en tiempo de ejecución de la personalización perezosa (*lazy customization*), que también es despreciable, ya que solo ocurre en la primera llamada. La API reflectiva y las clases usadas para gestionar mixins agregan también un pequeño costo fijo.

Los mixins sí tienden a aumentar el grado de polimorfismo del sistema. Se pueden esperar los mismos efectos de rendimiento que en cualquier otro código altamente polimórfico.

## 7 Related Work

- Una descripción muy breve de los mixins del sistema Animorphic se dio en un trabajo anterior de Bracha & Griswold (1996). Los mixins implementados en el sistema Animorphic se basan en la semántica dada en el paper de Bracha & Cook de 1990. Desde ese trabajo temprano se propusieron modelos alternativos de mixins que intentan abordar algunas debilidades de ese modelo. El primero fue la noción de **módulos** propuesta en la tesis doctoral de Bracha de 1992. Desde entonces ha habido una cantidad considerable de investigación sobre mixins.
- **Gbeta** unifica los patrones de Beta con los mixins. Sin embargo, Gbeta incorpora un mecanismo de linearización automática de mixins, a diferencia del modelo de 1990 que usan los autores. Gbeta es un lenguaje estáticamente tipado, y chequea los tipos de las declaraciones e invocaciones de mixin de manera similar a este sistema. La unificación de mixins y clases en Gbeta demuestra un punto importante: usando mixins se puede evitar la necesidad de distinguir entre tipos e implementaciones (por ejemplo, entre clases e interfaces en Java, o entre clases/mixins y protocolos en Strongtalk), conservando al mismo tiempo las ventajas de la clasificación múltiple.
- El lenguaje **Scala** soporta mixins con un modelo similar al de los autores, con una disciplina de tipos relacionada (pero obligatoria). El tratamiento de mixins en Scala difiere en varios aspectos: los mixins nunca se declaran explícitamente; en cambio, se derivan implícitamente de definiciones de clase. En el contexto de la creación de una instancia particular, se puede reemplazar la superclase de una clase, convirtiendo implícitamente la clase en un mixin. Sin embargo, la implementación de Scala difiere sustancialmente de este trabajo, porque Scala se implementa traduciendo a bytecode de la máquina virtual de Java.
- En **Ruby**, los mixins se definen como "módulos", y estos pueden incluirse en otros módulos o en clases. A pesar de la diferencia considerable en la sintaxis superficial, el modelo subyacente de mixins y clases usado en Ruby es esencialmente el mismo que en Animorphic Smalltalk. La librería de Ruby hace un uso significativo de mixins, igual que la librería de Animorphic. Ruby no provee un sistema de tipos ni una implementación de alto rendimiento de mixins.
- Ancona et al. presentan **JAM**, una extensión del lenguaje Java con mixins, conceptualmente similares a los discutidos en este paper. Discuten el chequeo de tipos de mixins según principios similares a los presentados aquí. Sin embargo, no tratan la composición de mixins ni la derivación automática de mixins a partir de clases.
- Ha habido varios otros esfuerzos para chequear tipos de mixins. La intuición esencial para el chequeo de tipos de mixins se bosquejó en el capítulo 3 de la tesis de Bracha de 1992. El trabajo de los autores difiere de esos otros trabajos en que: es parte de un sistema de tipos opcional; está implementado como parte de un chequeador de tipos interactivo e incremental en el contexto de un IDE completo; no tienen una formalización exhaustiva; y dan una explicación de cómo chequear tipos en la composición de mixins.
- Varios investigadores han explorado construcciones similares a mixins con semánticas algo distintas, todas preocupadas por el problema potencial de las **sobreescrituras involuntarias** (*inadvertent overrides*): cuando se define un mixin, el programador no conoce la superclase concreta, por lo que el mixin puede declarar métodos que sobreescriban sin querer métodos de superclases particulares.
  - Steyaert et al. estudiaron los "mixin-methods", un mecanismo para que las clases controlen qué extensiones se les pueden aplicar.
  - Duggan propuso los "mixin modules", una extensión del sistema de módulos de ML.
  - Flatt et al. propusieron una semántica alternativa donde los mixins especifican qué interfaces pretenden sobreescribir.
  - En Extended Moby, las clases solo sobreescriben los métodos conocidos de sus superclases, representando los mixins mediante genericidad; esto evita la sobreescritura no intencionada, pero no permite derivar un mixin de una clase después de definida (a diferencia del sistema de los autores), y depende de un sistema de tipos estático obligatorio, abandonando la subsumción.
  - Bono propuso distinguir sintácticamente los métodos que sobreescriben.
- En el modelo no tipado presentado en este paper, no hay forma de distinguir entre métodos pensados para sobreescribir y otros métodos del mixin; usando tipos es posible advertir sobre sobreescrituras no intencionadas, pero igual ocurren — aunque hay poca experiencia práctica que demuestre el problema de forma concluyente. Ninguna de las propuestas alternativas discute en detalle técnicas de implementación; se adaptan mejor a implementaciones basadas en tablas de despacho, mientras que las implementaciones basadas en búsqueda (*lookup*) podrían requerir esquemas de "name mangling".
- Los autores optaron por una semántica más simple que la de esas propuestas porque: es más adecuada para un lenguaje dinámicamente tipado (u opcionalmente tipado); no observaron en la práctica el problema que esas construcciones más ricas intentan resolver; y son más simples y por lo tanto más comprensibles.

## 8 Status/History

El sistema descrito en este paper se empezó a desarrollar en el otoño de 1994, como parte del proyecto **Animorphic**. El proyecto produjo un sistema Smalltalk de alto rendimiento funcional, que incluía VM, librería al estilo "blue book", un framework de UI y un entorno de desarrollo incremental. A fines del verano de 1996, todo el trabajo sobre Animorphic Smalltalk cesó efectivamente por consideraciones comerciales que pusieron el énfasis en las máquinas virtuales de Java. Para julio de 2002, estaría disponible para descarga una versión pública de Animorphic Smalltalk.

## 9 Conclusions

Presentaron una implementación de alto rendimiento, en funcionamiento, de mixins que respeta el modelo semántico bien definido dado en el paper de 1990. Mostraron que los mixins pueden implementarse eficientemente, con una sobrecarga despreciable en tiempo y espacio comparado con una implementación de alto rendimiento basada en clases de un lenguaje dinámicamente tipado.

También mostraron cómo chequear opcionalmente los tipos de esos mixins de forma estática en el contexto de un sistema de tipos nominal, cómo incorporarlos en una arquitectura reflectiva, y cómo pueden ser útiles en aplicaciones realistas, como un framework de UI funcional y librerías de E/S.

## Acknowledgements

El sistema Animorphic no podría haberse construido sin la inspiración del trabajo pionero de David Ungar y Randy Smith sobre Self. La publicación pública del sistema le debe mucho a la iniciativa de Eliot Miranda. En Sun Microsystems, varias personas contribuyeron a la decisión de liberar el sistema: Larry Abrahams, Rich Green, y especialmente Richard Gabriel.

## References

El paper cierra con una lista extensa de referencias bibliográficas: el propio paper "Mixin-based Inheritance" de Bracha & Cook (1990); trabajos sobre Strongtalk de Bracha & Griswold; el trabajo de Cardelli & Mitchell sobre el problema de la información negativa; trabajos de Duggan sobre mixin modules; de Ernst sobre Gbeta; de Flatt, Krishnamurthi y Felleisen sobre clases y mixins, y sobre Units; el libro de Gosling et al. sobre la especificación de Java; el libro "Smalltalk-80" de Goldberg & Robson; el trabajo de Hölzle, Chambers y Ungar sobre cachés en línea polimórficos; la tesis de Hölzle sobre Self; el libro de Matsumoto sobre Ruby; el libro de Meyer sobre construcción de software orientado a objetos (Eiffel); el manual de SELF; trabajos de Moon sobre Flavors; el reporte de Odersky sobre Scala; el trabajo de Smaragdakis y Batory sobre mixin layers; el trabajo de Steyaert et al. sobre mixin-methods anidados; el libro de Thomas y Hunt sobre Ruby; y el paper de Ungar y Smith sobre Self.
