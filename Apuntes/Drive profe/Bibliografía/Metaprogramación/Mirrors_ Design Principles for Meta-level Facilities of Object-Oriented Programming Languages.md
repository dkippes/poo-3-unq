# Mirrors: Design Principles for Meta-level Facilities of Object-Oriented Programming Languages

Gilad Bracha & David Ungar — OOPSLA 2004 (Vancouver, British Columbia, Canada)

> Resumen detallado en español, organizado siguiendo la estructura original del paper, para poder seguirlo en paralelo con el PDF. No es traducción literal sino una explicación fiel y completa, en mis propias palabras, de cada sección.

## Abstract

Los autores identifican tres principios de diseño para las facilidades de reflection y metaprogramación en lenguajes orientados a objetos:

- **Encapsulación:** las facilidades de meta-nivel deben encapsular su implementación.
- **Estratificación (Stratification):** las facilidades de meta-nivel deben estar separadas de la funcionalidad de base (base-level).
- **Correspondencia ontológica (Ontological correspondence):** la ontología de las facilidades de meta-nivel debe corresponderse con la ontología del lenguaje que manipulan.

Las arquitecturas reflectivas tradicionales/mainstream no respetan estos principios. En cambio, las APIs reflectivas construidas alrededor del concepto de **mirrors** sí los cumplen. Como consecuencia, las arquitecturas basadas en mirrors tienen ventajas significativas respecto de distribución, deployment y metaprogramación de propósito general.

## 1 Introduction

Los lenguajes orientados a objetos tradicionalmente soportan operaciones de meta-nivel como reflection "reificando" elementos del programa (clases, por ejemplo) como objetos que a su vez soportan operaciones reflectivas, tales como `getSuperclass` o `getMethods`.

El paper arranca con un ejemplo típico. En un lenguaje OO típico con reflection (Java, C#, Smalltalk, CLOS), uno puede preguntarle a una instancia por su clase:

```java
class Car {...}
Car myCar = new Car();
int numberOfDoors = myCar.numberOfDoors;
Class theCarsClass = myCar.getClass();
```

Mirando la API de un sistema así, uno esperaría ver algo como:

```java
class Object {
  Class getClass();
  ...
}
class Class {
  Class getSuperclass();
  // muchos otros métodos: getMethods(), getFields() etc.
  ...
}
```

Estas APIs soportan reflection en el centro mismo del sistema: cada objeto tiene al menos un método reflectivo, que lo conecta con `Class` y (probablemente) con todo un subsistema reflectivo. Las operaciones de base y las de meta-nivel **coexisten lado a lado**: el mismo objeto `Class` que contiene constructores y atributos estáticos también responde consultas sobre su nombre, superclase y miembros. El mismo objeto que exhibe comportamiento del dominio del problema también exhibe comportamiento sobre el hecho de ser miembro de una clase (`getClass`).

La tesis central del paper es que la funcionalidad de meta-nivel **debería implementarse separadamente** de la funcionalidad de base, usando objetos llamados **mirrors**. Con mirrors, esa misma API se vería así:

```java
class Object {
  // sin métodos reflectivos
  ...
}
class Class {
  // sin métodos reflectivos
  ...
}
interface Mirror {
  String name();
  ...
}
class Reflection {
  public static ObjectMirror reflect(Object o) {...}
}
interface ObjectMirror extends Mirror {
  ClassMirror getClass();
  ...
}
interface ClassMirror extends Mirror {
  ClassMirror getSuperclass();
  ...
}
```

Y el ejemplo del auto, reescrito en estilo mirror-based:

```java
ObjectMirror theCarsMirror = Reflection.reflect(myCar);
ClassMirror theCarsClassMirror = theCarsMirror.getClass();
ClassMirror theCarsSuperclassMirror = theCarsClassMirror.getSuperclass();
```

A primera vista el cambio parece cosmético, pero tiene dos efectos importantes:

1. **Divorcia** la interfaz de las operaciones de meta-nivel de una implementación particular (porque `ObjectMirror`/`ClassMirror` son interfaces, no clases concretas como `Class`).
2. **Saca** las operaciones de meta-nivel hacia un subsistema separable (uno accede al mundo reflectivo a través de una operación explícita, `reflect`, en lugar de tenerlo ya mezclado en cada objeto).

Cada una de estas dos propiedades encarna un principio de diseño importante: la primera es el principio de **encapsulación** (las facilidades de meta-nivel deben encapsular su implementación) y la segunda es el principio de **estratificación** (las facilidades de meta-nivel deben estar separadas de la funcionalidad de base). Un tercer principio es la **correspondencia estructural** (*structural correspondence*): la estructura de las facilidades de meta-nivel debe corresponder a la estructura del lenguaje que manipulan. Cualquier arquitectura de lenguaje de meta-nivel que respete estos principios es, por definición, un sistema basado en mirrors. Además, los autores proponen el principio de **correspondencia temporal** (*temporal correspondence*): las APIs de meta-nivel deben estratificarse de modo que distingan entre propiedades estáticas y dinámicas del lenguaje que manipulan. Estos dos últimos principios (estructural y temporal) pueden fusionarse en un principio más amplio: la **correspondencia ontológica** (las facilidades de meta-nivel deben corresponder a la ontología del lenguaje que manipulan).

El paper muestra que adherir a estos principios trae ventajas significativas para el desarrollo distribuido y el deployment de aplicaciones. También argumentan que una API reflectiva mirror-based bien diseñada puede servir como una API de metaprogramación de propósito general.

La **Figura 1** del paper (que se reproduce más abajo, en la sección 2) ilustra la diferencia entre un diseño reflectivo tradicional y uno basado en mirrors. En el diseño tradicional, las clases "están a caballo" entre el nivel base y el meta-nivel (un mismo objeto `Class` cumple ambos roles). En un diseño mirror, uno se mueve del nivel base al meta-nivel mediante una operación `reflect`; los niveles quedan claramente separados, y de hecho la presencia de clases en el nivel base ya no es estrictamente necesaria.

El principio de encapsulación es una regla básica de ingeniería de software, pero —como se mostrará— en muchos casos no se aplicó al diseño de las arquitecturas reflectivas incorporadas en los principales lenguajes de programación. El principio de estratificación es bien conocido en la comunidad de reflection ([Maes87]), pero tampoco se respeta consistentemente en la mayoría de los lenguajes. La correspondencia estructural fue elucidada en trabajos previos; aunque en general se respeta, los autores destacan violaciones y sus implicancias para los lenguajes mainstream. La correspondencia temporal está relacionada con la conocida distinción entre reflection en tiempo de compilación, de carga (load-time) y de ejecución (run-time). Estas fases no siempre son aplicables a un lenguaje dado, pero incluso cuando lo son, usualmente el lenguaje solo soporta reflection en tiempo de ejecución.

Este paper es la primera discusión sistemática de los principios de diseño de los sistemas basados en mirrors y sus ventajas concomitantes. Las ventajas de los mirrors incluyen:

- La capacidad de escribir aplicaciones de metaprogramación que son **independientes de una implementación específica** de reflection. Con cuidado, los clientes de metaprogramación pueden interactuar con fuentes de metadata locales o remotas sin ningún cambio en el cliente. Más aún, un cliente puede interactuar con múltiples fuentes de metadata en tiempo de ejecución, e incluso interactuar simultáneamente con meta-objetos de distintas implementaciones.
- La capacidad de obtener metadata de sistemas que se ejecutan en plataformas que no incluyen ellas mismas una implementación reflectiva completa. Ejemplos: dispositivos pequeños con memoria limitada o sistemas embebidos; aplicaciones desplegadas donde preocupaciones de footprint, seguridad o ancho de banda desalentaron o impidieron incluir soporte de reflection.
- La capacidad de agregar/quitar soporte de reflection dinámicamente a/de una computación en ejecución.
- La capacidad de desplegar aplicaciones no reflectivas escritas en lenguajes reflectivos sobre plataformas sin implementación reflectiva, reduciendo footprint o ahorrando tiempo de comunicación.

**Terminología.** Las arquitecturas de lenguajes reflectivos pueden caracterizarse según su soporte para:

1. **Introspection (Introspección).** La capacidad de un programa de examinar su propia estructura.
2. **Self-modification (Auto-modificación).** La capacidad de un programa de cambiar su propia estructura.
3. **Ejecución de código generado dinámicamente.** La capacidad de ejecutar fragmentos de programa que no son estáticamente conocidos. Es un caso especial de (2).
4. **Intercession (Intercesión).** La capacidad de modificar la semántica del lenguaje de programación subyacente desde dentro del lenguaje mismo (el término "intercession" a veces se usa en un sentido más amplio en parte de la literatura, pero aquí se adopta la definición más estrecha basada en [18]).

La **Tabla 1** del paper resume el soporte para estas características en varios sistemas reflectivos mencionados en el paper:

| | Introspection | Self Modification | Intercession |
|---|---|---|---|
| Java Core Reflection | Sí | No | No |
| Smalltalk | Sí | Sí | Muy limitado |
| CLOS | Sí | Sí | Sí |
| JDI | Sí | Limitado | No |
| Strongtalk | Sí | Sí | No |
| Self | Sí | Sí | Muy limitado |

El término **reflection** se usa para situaciones donde un programa se manipula a sí mismo. Los autores usan el término más general **metaprogramming** para describir situaciones donde un programa manipula a un programa (posiblemente distinto). La palabra "programa" en sí misma a veces se usa para dos nociones distintas: una descripción dada en algún lenguaje de programación, y un proceso computacional en ejecución. Los autores llaman a la primera **code** (código) y a la segunda **computation** (computación).

Cada una de las siguientes tres secciones se enfoca en uno de los tres principios identificados. En cada sección se muestran problemas concretos que surgen de violar el principio en discusión, y cómo se resuelven usando mirrors. La sección 2 discute el principio de encapsulación y sus implicancias para la ejecución distribuida. Esto lleva a la necesidad de estratificación, discutida en la sección 3 junto con temas de deployment. La sección 4 trata el principio de correspondencia y los problemas que surgen cuando se descuida. La sección 5 da una discusión general de los temas que surgen en el diseño de sistemas mirror-based. Luego se discute trabajo relacionado y se presentan las conclusiones.

## 2 ENCAPSULATION

Es una regla básica de la ingeniería de software que un módulo no debería depender de los detalles de implementación de otro módulo. Desafortunadamente, los clientes de las APIs reflectivas clásicas dependen de detalles de implementación del sistema reflectivo que usan. Esto se demuestra con un caso de estudio, seguido de un análisis más general.

### 2.1 Case Study: Distribution

Consideremos el siguiente escenario: un programador escribe un *class browser* (navegador de clases) usando reflection. Más adelante, se vuelve necesario poder navegar clases en máquinas remotas. Se quiere reusar la mayor parte posible del código del browser, con la menor adaptación posible.

(Los autores aclaran que no están a favor ni en contra de la distribución transparente — esa controversia queda fuera del alcance del paper. El argumento es que los mirrors son un buen approach para diseñar una API que *sea* "distribution-aware", y que una API reflectiva mirror-based y distribution-aware puede diseñarse de forma que también sirva bien para el caso no distribuido.)

Con ese escenario en mente, se contrastan dos APIs basadas en el mismo lenguaje y la misma máquina virtual: **Java core reflection** y el **Java Debugger Interface (JDI)**.

#### 2.1.1 Java Core Reflection

En Java, las capacidades reflectivas están centradas en la clase `java.lang.Class` (`Class`), extendida por el paquete `java.lang.reflect`. Todas las clases, incluyendo `Class` misma, son instancias de `Class`.

Además de `Class`, existen otras clases que soportan reflection: `java.lang.reflect.Method`, `java.lang.reflect.Field`, `java.lang.reflect.Constructor`. La API reflectiva está definida mediante **clases**, no interfaces, y esto tiene consecuencias importantes (se discute en la sección 2.2 más abajo).

Las clases tienen métodos que soportan introspección: se puede consultar a una clase por su superclase, superinterfaces, fields, métodos, constructores y clases miembro. No es posible alterar la estructura o el código de una clase existente con estas facilidades, ya que no soportan self-modification.

#### 2.1.2 Applying Core Reflection to the Class Browser Problem

Es bastante fácil escribir un browser que, dado el nombre de una clase, permita examinar la estructura de esa clase usando core reflection. Sin embargo, en cuanto se intenta usar ese mismo código sobre clases remotas, aparecen dificultades serias [26].

Como Java core reflection no soporta distribución directamente, el browser necesitaría usar una implementación alternativa. ¿Cómo evitar la necesidad de reescribir completamente el browser?

Una posibilidad, siguiendo [20], es diseñar la API de metaprogramación distribuida con la misma estructura de clases y métodos que core reflection. Entonces la aplicación browser podría cambiar entre la API local y la distribuida simplemente cambiando un único `import` (de `import java.lang.reflect.*` a, por ejemplo, `import com.mycompany.distributed.metaprogramming.*`).

Este approach es problemático: hay que mantener dos copias (ligeramente distintas) del código fuente, dos sets de binarios, el browser duplica su footprint de memoria en tiempo de ejecución, y es muy difícil lograr que las dos versiones cargadas interoperen. Más aún, si el browser interactúa con la API de class loader y oculta esa interacción usando `import`, no será posible.

También podría querer usarse el browser para observar código fuente fuera de una VM en ejecución; los problemas mencionados se agravan aún más (habría tres versiones del fuente, tres sets de binarios, etc.).

Los `import` no resuelven el problema porque no pueden usarse para desacoplar módulos; en cambio, los acoplan de manera localizada.

Otro conjunto de dificultades es específico de la programación distribuida: la aplicación debe poder manejar fallas de red y latencia, lo cual afecta la lógica de la aplicación. También se necesitaría especificar y mostrar las ubicaciones de red de las clases que se están navegando.

En definitiva, la API de core reflection no es adecuada para este propósito.

#### 2.1.3 JDI

El **Java Debug Interface (JDI)** es la capa superior de la *Java Platform Debugger Architecture* (JPDA) [29]. Está diseñada para soportar debugging remoto, pero soporta todas las capacidades de introspección presentes en core reflection, además de formas limitadas de self-modification.

JDI define interfaces que describen todas las entidades de programa que pueden interesarle a un debugger: clases, interfaces, objetos, stack frames, etc. Todas estas interfaces son subtipos de la interfaz `com.sun.jdi.Mirror`. (Notar: esta es justamente la noción de "mirror" que da nombre al paper.)

Un mirror siempre está asociado a una máquina virtual particular en la que existe la entidad reflejada. La interfaz `com.sun.jdi.VirtualMachine` describe un mirror sobre una VM como un todo. Se puede obtener el conjunto de clases e interfaces cargadas en la VM reflejada, el conjunto de threads en la VM, información sobre las capacidades de esa VM, mirrors sobre clases o valores específicos en la VM, etc.

Los objetos se reflejan mediante objetos que implementan la interfaz `com.sun.jdi.ObjectReference` (el equivalente de la interfaz `ObjectMirror` mostrada en la introducción). Es posible leer y escribir los fields del objeto reflejado, invocar sus métodos, y obtener su clase.

Los mirrors remotos sobre objetos plantean problemas de garbage collection distribuido. Por defecto, un objeto reflejado puede ser recolectado por la VM reflejada (es decir, los object mirrors mantienen una referencia débil al objeto reflejado). Es posible sobreescribir este comportamiento por defecto, y también determinar si un objeto fue recolectado.

Los threads se reflejan mediante objetos que implementan `com.sun.jdi.ThreadReference`, que es una especialización de `ObjectReference` (ya que los threads son objetos en la JVM). Las operaciones sobre threads incluyen suspensión y reanudación, y operaciones sobre el call stack del thread.

#### 2.1.4 Scenario Revisited Using JDI

Usando JDI, es directo escribir un class browser que examine clases en procesos separados, en otras máquinas o en la misma máquina virtual.

JDI fue diseñado pensando en debugging distribuido. Debuggear la misma máquina virtual usando JDI no se recomienda, porque debugging implica detener threads en la VM que se está debuggeando, y si JDI corre en la misma VM hay riesgo sustancial de deadlock.

Sin embargo, la introspección estructural usualmente no requiere pausar threads. Incluso si la implementación usual de JDI no se comportara bien en ese caso, podría derivarse una implementación alternativa envolviendo (*wrapping*) core reflection dentro de objetos que implementen las interfaces de JDI según se necesite para soportar introspección estructural.

Concretamente, se pueden construir distintas implementaciones de la interfaz `VirtualMachine` (que refleja una VM particular) que soporten reflection sobre el proceso actual (local) o sobre procesos remotos. Un class browser consultaría una instancia de `VirtualMachine` por las clases cargadas (representadas como una lista de `ClassType`, la interfaz mirror para clases). El código del browser mismo es **ajeno** (oblivious) a la diferencia entre las distintas implementaciones de las interfaces mirror.

Aunque se estableció que JDI puede usarse para reflection no distribuida, no se mostró que sea tan conveniente de usar como core reflection. La dificultad principal es la necesidad de lidiar con excepciones potenciales que solo pueden surgir en uso distribuido. Aun así, los autores argumentan que ese costo se ve superado por los beneficios de tener que aprender una sola API en lugar de dos. Más aún, como se discute en 3.1.3, la experiencia con las APIs de mirrors de Self indica que es posible emplear la misma API para reflection local y distribuida sin penalidad excesiva.

### 2.2 Analysis

El caso de estudio muestra que Java core reflection no soporta herramientas de desarrollo distribuido, mientras que JDI sí. En parte, esto es porque JDI trata ciertos temas específicos de distribución (fallas de red, gestión de memoria distribuida), mientras que core reflection no lo hace.

Estos temas podrían atenderse mediante una implementación alternativa de core reflection que se comunicara con JVMs remotas vía proxies. Ante una falla de red (o latencia inaceptable), las operaciones podrían fallar lanzando una excepción.

El punto clave es que **una implementación alternativa así no es posible** porque la API de core reflection está basada en **clases** en lugar de **interfaces**. Core reflection viola deliberadamente el principio de encapsulación, al hacer que sus clientes dependan de tipos de implementación específicos (clases). Esta dependencia es forzada por el sistema de tipos y evita que los clientes usen implementaciones alternativas de la API de core reflection.

Por supuesto, en un lenguaje de tipado dinámico se podría escribir un proxy que emule la API reflectiva sin preocuparse por las peculiaridades del sistema de tipos. Sin embargo, incluso en lenguajes OO de tipado dinámico, la implementación de un objeto puede exponerse sutilmente de otras formas, como se discute a continuación.

#### 2.2.1 Encapsulating Class Identity

En la mayoría de los lenguajes OO, el método `getClass()` (o su equivalente) expone información sobre la implementación a los clientes. Las aplicaciones pueden llegar a depender de esta información, haciendo muy difícil reemplazar un objeto por otro que tenga la misma funcionalidad pero que sea instancia de una clase diferente.

Un ejemplo simple:

```java
if (aCar.getClass() == Car.class) {...}
```

Esto es muy mala práctica, aunque no es poco común. Este código fallará si `aCar` es instancia de cualquier implementación alternativa de `Car`.

Ahora consideremos un ejemplo en un lenguaje donde las clases tienen estado y código específico de la aplicación (como en Smalltalk o CLOS). La clase `Car` podría tener un método:

```
numberOfCarsMadeIn(y) {...} // retorna el número de autos fabricados en el año y
```

usado típicamente así:

```
n = aCar.getClass().numberOfCarsMadeIn(1999);
```

Si `aCar` es un proxy para un auto remoto, la implementación estándar de `getClass` devolvería `Proxy`, haciendo que el código falle. Está claro que se quiere que `getClass` devuelva un proxy a la clase remota del auto. Hacer eso, sin embargo, plantea un problema: ¿cómo accede el código reflectivo a la clase real de `aCar`, la clase `Proxy`? Se podría definir otro método, `getRealClass`, pero esto simplemente perpetúa el problema original de exponer la identidad de clase. La funcionalidad provista por `getClass` derrota justamente el reuso que se promueve como motivación de la programación orientada a objetos.

> *(Nota al pie del paper: este problema quizás no sea tan importante para quienes valoran sobre todo las capacidades de modelado de la programación orientada a objetos, así que quizás los autores admiten que este paper aborda su tema desde una perspectiva "no escandinava". Pero puede ser que la estratificación que proponen no sea disonante con la separación entre conceptos y fenómenos.)*

La solución es **factorizar la funcionalidad reflectiva fuera de la API de los objetos ordinarios**. Esto es exactamente lo que hacen los mirrors.

Este factoring implica una **descomposición funcional**, en lugar de una clásica orientada a objetos. La implementación del mirror decide cómo reflejar objetos de un tipo dado, en lugar de dejar esa decisión a la implementación de los objetos mismos.

En algunos escenarios, como el ejemplo del class browser, esto es bastante natural: el browser sabe dónde están las clases —localmente, remotamente, en una base de datos, etc.— y puede elegir una *mirror factory* adecuada. En otros casos, como un debugger en un proceso local que se encuentra con un objeto proxy, no es inmediatamente claro qué mirror elegir. Puede ser una preferencia de configuración fijada por el usuario.

De esto se sigue que las mirror factories pueden necesitar despachar según el tipo del objeto. ¿Cómo lo hacen si el acceso a la identidad se les niega, como recomendamos? La respuesta es que la reflection local básica inherentemente no respeta la encapsulación, y puede usarse por aplicaciones reflectivas —incluyendo otras mirror factories— para identificar clases si así lo eligen. Uno podría incluso definir una "public mirror factory" que permita registrar clases indicando qué implementación de mirror usar al reflejar sobre sus instancias.

Hay usualmente otros medios además de `getClass` por los que se puede detectar la identidad de clase de una instancia. Un ejemplo común es el uso de construcciones como `instanceof` con tipos de clase (usar estas construcciones con tipos de interfaz es inofensivo). Ese tipo de uso, y cualquier dependencia en la identidad de clase, debería evitarse en código de aplicación.

La conclusión de los autores es que la separación entre mirrors, en el meta-nivel, y clases, en el nivel base, es necesaria para soportar plenamente la encapsulación. Esta separación es una manifestación del principio de **estratificación**, discutido más a fondo en la siguiente sección.

## 3 STRATIFICATION

Una propiedad de ingeniería deseable de una feature es que no imponga costos cuando no se usa. La adherencia al principio de estratificación soporta este deseo, haciendo fácil eliminar reflection cuando no se necesita. Esto tiene beneficios importantes en el contexto de deployment, como se discute más abajo.

### 3.1 Case Study: Deployment

Al desplegar una aplicación, no siempre es deseable desplegarla junto con todas las facilidades reflectivas disponibles en el lenguaje. La aplicación puede no necesitar esas capacidades en absoluto, o necesitarlas infrecuentemente. En esos casos, puede ser ventajoso reducir el footprint de la aplicación evitando o postergando el deployment de las facilidades reflectivas. Esto es especialmente cierto en plataformas pequeñas como teléfonos móviles, PDAs, smart cards u otros sistemas embebidos.

El objetivo, entonces, es evitar desplegar reflection a menos y hasta que la aplicación realmente la necesite. Se revisa la API reflectiva de Smalltalk-80, se la contrasta con Strongtalk (un sistema Smalltalk mirror-based), y se analiza cómo estas arquitecturas distintas afectan el problema del deployment.

#### 3.1.1 Smalltalk-80

Smalltalk-80 se diferencia de la mayoría de los lenguajes en que un programa no se define declarativamente. En cambio, una computación se define mediante un conjunto de objetos. Las clases capturan estructura compartida entre objetos, pero ellas mismas son objetos, no declaraciones. La única forma de invocar métodos, crear nuevas clases, agregar código a clases, etc., es a través de mensajes a esos objetos.

Las clases de Smalltalk soportan inherentemente self-modification, porque reflection es el único mecanismo disponible para construir y modificar clases. El método `class` está definido para todos los objetos, así que se puede obtener la clase de cualquier instancia. Cada objeto también implementa el método `inspect`, que abre un *inspector* sobre el objeto.

Las clases de Smalltalk no se usan exclusivamente como meta-objetos. Las clases típicamente incluyen métodos y estado específicos de la aplicación. El uso más común de los métodos de clase es la creación de instancias. No hay sintaxis especial para instanciar una clase, ni existe noción de constructor en Smalltalk; en cambio, se usan métodos de clase para crear nuevas instancias.

Como las clases de Smalltalk cumplen tanto roles específicos de la aplicación como roles de meta-nivel en un programa, en general es difícil quitar el soporte de reflection de una aplicación Smalltalk. Esto se discute más en 3.1.3.

#### 3.1.2 Strongtalk

Strongtalk se diferencia de los sistemas Smalltalk tradicionales en varios aspectos, los más relevantes para esta discusión son:

- Adopta el uso de **mirrors** en lugar de la arquitectura reflectiva tradicional.
- Tiene un sistema de tipos estático **opcional** [8][6] basado exclusivamente en interfaces, lo cual soporta el principio de encapsulación.
- Es un sistema basado en **mixins** [7][9][5].

El sistema de mirrors de Strongtalk soporta introspección y self-modification. La clase `Mirror` y sus subclases soportan reflection sobre mixins, clases, tipos, métodos, variables globales, objetos, el stack y frames de activación individuales (*activation records*). Invocando `Mirror>>on:` sobre un objeto se obtiene el mirror apropiado.

Los mixins son la unidad básica de self-modification en la API de mirrors de Strongtalk. Los mixins son adecuados para esta tarea porque son *stateless* (a diferencia de las clases de Smalltalk), y por lo tanto pueden copiarse libremente. Pueden hacerse modificaciones a una copia de un mixin sin ningún efecto sobre la computación en curso. Solo cuando todas las modificaciones están completas se instala la versión modificada, en una operación atómica. Varios mixins modificados pueden instalarse simultáneamente. Este *batching* de modificaciones mejora el rendimiento, pero tiene una ventaja más importante: una serie de modificaciones puede ser consistente como un todo, pero si se hace de manera incremental (pieza por pieza) podría crear versiones intermedias inconsistentes del código, posiblemente llevando a fallas del programa. Este problema se evita gracias al batching de las modificaciones (ver [5] para más detalles).

La funcionalidad reflectiva usual asociada a `Class` está disponible en `ClassMirror`. De manera similar, existen clases mirror especializadas para mixins, *protocols* (el equivalente aproximado de las interfaces en Java), y declaraciones de variables globales.

Mientras que en un sistema Smalltalk ordinario se le pediría a una clase que elimine uno de sus métodos, en Strongtalk se obtendría un mirror sobre la clase usando `Mirror>>on:` y se interactuaría con el mirror, según dicta el principio de estratificación.

Para inspeccionar una instancia ordinaria `o`, no se usa el método `inspect`. En cambio, se invoca el método `Inspector>>launchOn:` sobre el objeto. Esto es crucial para desacoplar la GUI del resto del sistema.

Para determinar la clase de un objeto con fines reflectivos, en lugar de invocar el método `class`, se invoca el método `Reflection>>classOf:` sobre el objeto.

Este último ejemplo merece discusión. En Smalltalk, obtener la clase de un objeto es una operación rutinaria no reflectiva. Los métodos de clase se usan para construir nuevas instancias y para otros fines específicos de la aplicación. Para esos fines específicos de la aplicación, el método `class` puede y debe usarse. A diferencia de los Smalltalk tradicionales, este método puede sobreescribirse en Strongtalk. Esto permite que los objetos oculten detalles de implementación, incluyendo su clase. Por ejemplo, un objeto proxy puede ocultar el hecho de que es una instancia de una clase proxy. (Ver sección 2.2.1 para análisis adicional.)

#### 3.1.3 Analysis

Los mirrors hacen más fácil eliminar la infraestructura reflectiva de una aplicación. Para ver por qué, hay que considerar los problemas tanto en lenguajes de tipado dinámico como estático.

En lenguajes de tipado dinámico que no usan mirrors, puede ser difícil separar las facilidades reflectivas y el entorno de desarrollo de la aplicación. Por ejemplo, la capacidad de agregar nuevos métodos requiere acceso a un compilador de código fuente. Si esta capacidad se coloca en la clase `Class`, se vuelve difícil eliminarla de una aplicación, porque todas las aplicaciones dependen de la clase `Class`. De forma similar, en Smalltalk tradicional, `Object>>inspect` tiende a ligar los inspectores de objetos y un framework de UI a la aplicación.

En general, si las capacidades reflectivas son parte de una clase que tiene usos más allá de la reflection, es difícil quitar esas capacidades reflectivas del sistema de forma segura. Para estar seguro de que una aplicación no usa reflection hay que recurrir a técnicas de inferencia de tipos sofisticadas y costosas [2].

Los mirrors eliminan este problema separando claramente la funcionalidad reflectiva, y moviéndola a lugares que las aplicaciones ordinarias no van a acceder. Entonces es directo establecer que una aplicación no requiere funcionalidad del subsistema reflectivo ni del entorno de desarrollo. Si la aplicación no hace referencia a los puntos de entrada asociados con reflection (por ejemplo, las clases `Mirror` y/o `Reflection` en Strongtalk, la interfaz `com.sun.jdi.Mirror` en JDI, o el método `reflect:` en Self [33][27]), se puede dejar afuera el soporte de reflection.

En lenguajes de tipado estático, eliminar funcionalidad reflectiva de una aplicación antes del deployment es considerablemente más fácil que en lenguajes de tipado dinámico. Sin embargo, si no se usan mirrors pero se quiere evitar desplegar el subsistema reflectivo innecesariamente, hay que determinar estáticamente que reflection no se usará en absoluto en la aplicación. Si reflection no se despliega inicialmente, no será posible modificar después las representaciones existentes de clases, métodos, etc. (ya que se necesitarían capacidades de self-modification para eso). Esta es una limitación real ante la presencia de carga dinámica de clases. Usando mirrors, se puede agregar o quitar la capacidad reflectiva en tiempo de ejecución sin soporte especial, usando carga de clases dinámica. La capacidad de habilitar/deshabilitar dinámicamente el soporte de reflection también es útil desde una perspectiva de seguridad. Por supuesto, reflection no puede desplegarse dinámicamente sin algún grado de soporte de la implementación subyacente.

La capacidad de reflejar una computación que no contiene una API reflectiva se demuestra en **Klein**, una Self VM metacíclica (*metacircular*) que está siendo desarrollada por el segundo autor (Ungar). Klein misma no soporta una API reflectiva. Klein se debuggea usando una GUI de Self corriendo sobre la VM estándar de Self, en un proceso separado. La GUI se comunica con la VM de Klein usando mirrors que se comunican mediante sockets. No se hicieron cambios a la API de mirrors de Self — solo se necesitó una nueva implementación. Esta experiencia apoya la afirmación de los autores (sección 2.1) de que una única API de mirrors puede servir tanto para el caso distribuido como para el local.

En general, se concluye que los mirrors facilitan el deployment. Las ventajas son más pronunciadas para los lenguajes de tipado dinámico, pero los mirrors son ventajosos incluso cuando se usa un sistema de tipos estático.

## 4 ONTOLOGICAL CORRESPONDENCE

### 4.1 Temporal Correspondence

La reflection se define tradicionalmente con referencia a una computación. Naturalmente, una suposición subyacente de las APIs reflectivas es que las entidades reflejadas existen dentro de un contexto en ejecución. Estas APIs por lo tanto soportan operaciones como instanciar una clase, o consultarla por todas sus instancias. Mientras algunas aplicaciones reflectivas (por ejemplo, profilers y debuggers) efectivamente manipulan una computación, otras, como compiladores, navegadores de jerarquías de clases (*class hierarchy browsers*) y *pretty printers*, solo manipulan la estructura de un programa (código).

Es deseable poder correr aplicaciones de esta segunda categoría sobre código que no está embebido en una computación. Un class browser podría usarse para examinar una base de datos de código fuente, por ejemplo. Inversamente, algunas herramientas de metaprogramación pueden asumir la disponibilidad de información de fuente que puede no estar disponible en tiempo de ejecución. Por ejemplo, **Javadoc** [17] espera que los comentarios estén disponibles.

#### 4.1.1 Case Study: Browsing via a Source Database vs. Browsing via Reflection

Si uno escribe un class browser usando Java core reflection, no puede retargetear fácilmente la aplicación para navegar clases descritas en una base de datos de código fuente. La situación es similar a la encontrada en 2.1.2. No se puede crear una implementación alternativa de la API que produzca instancias de `Class`, `java.lang.reflect.Method`, etc. simplemente leyendo código fuente sin compilarlo y cargar las clases en una JVM en ejecución. Este es otro ejemplo de la importancia del principio de encapsulación, pero hay temas adicionales involucrados aquí.

Incluso si fuera posible una implementación alternativa de core reflection, se enfrentarían dificultades. La API de reflection permite invocar métodos, instanciar clases, etc. Estas operaciones no tienen sentido cuando el browser está examinando una base de datos de fuente.

Tampoco se estaría mejor usando JDI, que fue diseñada principalmente para debugging. JDI asume que tiene que haber una computación, con threads en ejecución, de la cual se pueden obtener stack frames, objetos y clases. Adherir al principio de encapsulación es una condición necesaria pero insuficiente para resolver este problema.

Notar que el subconjunto de JDI relacionado con introspección estructural sobre clases es igual de aplicable a clases cuya estructura se extrae de código fuente o de archivos de clase binarios. Si esos elementos de JDI que no dependen de la presencia de una computación se factorizaran en una API separada, una implementación que operara sobre una base de datos de fuente sería directa.

Esto lleva a la observación: ***mirroring code* y *mirroring computation* deberían ser módulos separables de la API de mirrors.** Esta es una manifestación del principio de **correspondencia temporal**. La distinción que hace el lenguaje entre código (tiempo de compilación) y computación (tiempo de ejecución) debería manifestarse en sus APIs de metaprogramación.

La noción de código es útil para reiniciar programas en un estado fresco, para probar propiedades de programas, y especialmente para transportar programas entre procesos, como se discute a continuación.

#### 4.1.2 Distinguishing Code and Computation in Self: Interchange of Programs and Data

El sistema **Self** busca aprovechar las intuiciones de la gente sobre el mundo real para ayudarlas a programar computadoras. Como el mundo real no distingue código y computación —no hay un "switch" compilar/ejecutar en el mundo real— Self intenta unificar programa y computación. Un programa Self es simplemente un conjunto de objetos, y sus mirrors reflejan esa visión del mundo. Entonces, uno podría argumentar que el principio de correspondencia temporal es irrelevante para Self.

Sin embargo, Self incluye el **transporter**, un sistema diseñado para mover "programas" (conjuntos de *slots* conteniendo datos o código) de un mundo de objetos Self a otro [32]. Al construir el transporter, el segundo autor (Ungar) se vio forzado a reconocer que había una necesidad de un "programa" — algo que pudiera describirse y moverse a otro mundo de objetos que le proveyera la nueva funcionalidad. Los objetos agregados al sistema para representar programas (anotaciones, módulos, etc.) son efectivamente meta-objetos que tratan con código en lugar de con computación. A pesar de las mejores intenciones de los autores, al final, cuando llegó el momento de compartir programas, encontraron que este principio aplicaba después de todo.

### 4.2 Structural Correspondence

La correspondencia estructural implica que cada construcción del lenguaje debe estar reflejada (tener su correspondiente meta-objeto). Este principio se reconoce hace mucho en la comunidad de reflection [23]. Sin embargo, en la práctica suele violarse. Se discuten algunos de los problemas que surgen.

#### 4.2.1 Reifying Both Code and Computation

Idealmente, los meta-object protocols introducen un meta-objeto para cada objeto de la computación. Sin embargo, en muchos lenguajes, nociones importantes como módulos, declaraciones de import/export, metadata, tipos y comentarios existen solo en tiempo de compilación. Tales construcciones quedan excluidas por un MOP (*meta-object protocol*) que solo reifica elementos de la computación real. De forma similar, los MOPs de tiempo de compilación tratan solamente con construcciones de tiempo de compilación; el MOP no está presente en tiempo de ejecución, y no puede reificar entidades que existen solo en tiempo de ejecución (ver sección 6.4 para más discusión sobre MOPs de tiempo de compilación).

#### 4.2.2 Mirroring Method Bodies

En la mayoría de los lenguajes, construcciones por debajo del nivel de método, como statements y expresiones, no tienen meta-objetos correspondientes. Esto también pasa en los sistemas de mirrors discutidos en el paper. A nivel de la VM, frecuentemente está disponible el bytecode, y este a menudo puede mapearse de vuelta a código fuente. Esta estrategia se usa típicamente en herramientas como debuggers.

Una verdadera correspondencia estructural implicaría que debería estar disponible una representación de más alto nivel de los cuerpos de método. Esto sería útil para que herramientas que manipulan código fuente, como compiladores, puedan usar una representación estandarizada.

Como el código fuente (o incluso el bytecode) no siempre está disponible, muchos implementadores se alejaron de proveer tal funcionalidad. Sin embargo, a menudo es posible proveerla condicionalmente (es decir, si está disponible) y/o a demanda. Por ejemplo, en JDI, los clientes pueden consultar a un `VirtualMachine` qué tipo de operaciones soporta. Esto permite que JDI defina una API para acceder al bytecode de un método, pero permite también implementaciones que no retienen el bytecode.

#### 4.2.3 Which Language to Reflect?

Los lenguajes de programación que soportan reflection suelen estar implementados sobre una máquina virtual (por ejemplo Java y la JVM, C# y el CLR). No hay que confundir la API de metaprogramación para el lenguaje de la máquina virtual subyacente (el **VML**, *virtual machine language*) con la del lenguaje de alto nivel (el **HLL**, *high level language*) que corre sobre ella. Al diseñar una API de metaprogramación, es importante tener claro cuál es el lenguaje de base. Esto es así independientemente de si la API en cuestión es mirror-based o no.

La reflection tiene que estar soportada por la VM en algún nivel básico, así que una API reflectiva para el VML es un dato dado. Los lenguajes de alto nivel implementados sobre una máquina virtual idealmente deberían incluir su propia API de reflection. Mantener una API reflectiva distinta para el HLL es valiosa por varias razones:

- La API reflectiva del VML puede no mantener los invariantes del HLL, introduciendo así problemas potenciales de seguridad y corrección.
- Hay riesgo de discrepancias entre el VML y el HLL. Tales discrepancias a menudo surgen al implementar construcciones de alto nivel que no son soportadas directamente por la VM.

Un ejemplo prominente son las **clases anidadas (nested classes)** en Java. Implementar clases anidadas requiere generar clases, interfaces, métodos y fields sintéticos que no están presentes en el código fuente original. En algunos casos, los constructores pueden requerir parámetros adicionales.

Tales features deberían ocultarse de los programas HLL, porque exponen detalles de un esquema de traducción específico entre el HLL y el VML. Ese esquema de traducción es un detalle de implementación del cual los programas HLL nunca deberían depender. En particular, esos detalles no deberían filtrarse a través de la API reflectiva.

Como contraejemplo, considerar `java.lang.Class.getMethods`, que devuelve los métodos declarados por una clase. Se devuelven todos los métodos declarados a nivel de VM, sin importar si son sintéticos. Esto expone la estrategia de traducción del compilador Java a los clientes de reflection.

Si múltiples lenguajes fuente se implementan sobre una misma máquina virtual dada, el riesgo de discrepancias entre el lenguaje de la máquina virtual y los distintos lenguajes fuente aumenta. Un ejemplo común es el **overloading** de métodos, típicamente soportado por el compilador del HLL pero no por la VM subyacente. Si dos lenguajes tienen esquemas de resolución de overload diferentes, una única API reflectiva solo soportará correctamente a uno de ellos.

Incluso si el HLL y el VML están en completo acuerdo, es probable que con el tiempo surjan discrepancias, a medida que se agreguen e implementen nuevas características del HLL por el front-end del lenguaje sin soporte correspondiente de la VM. Nuevamente, las clases anidadas y los *generics* en Java son ejemplos de esto.

Para evitar estas dificultades, la API de mirrors de Strongtalk se subdivide en **high-level mirrors** y **low-level mirrors**. Los high-level mirrors reflejan Smalltalk, y los low-level mirrors reflejan las estructuras subyacentes en la máquina virtual. Esta distinción no está presente en ninguna otra API reflectiva de las que los autores tienen conocimiento.

Los high-level mirrors se definen mediante la jerarquía de clases `Mirror`. Los high-level mirrors soportan la semántica a nivel de Smalltalk. Los low-level mirrors se definen mediante la clase `VMMirror` y sus subclases. Los VM mirrors manifiestan diferencias representacionales entre distintos tipos de objetos (por ejemplo, enteros, arrays, clases, mixins, objetos regulares) que quedan ocultas a nivel del lenguaje. Se le puede preguntar a un `ClassVMMirror` por el tamaño físico de sus instancias, o por ejemplo el tamaño de su header. La API de low-level mirror es inherentemente sensible al diseño del lenguaje de la máquina virtual subyacente y a su implementación.

Los autores concluyen que: **debería haber APIs reflectivas distintas para cada lenguaje en un sistema**, en particular para el lenguaje de la máquina virtual subyacente y para cada lenguaje de alto nivel que corra sobre ella. Esta es una instancia del principio de correspondencia estructural.

## 5 ISSUES IN THE DESIGN OF MIRROR-BASED SYSTEMS

### 5.1 Classes vs. Prototypes

#### 5.1.1 Self

Los mirrors fueron introducidos por primera vez en el lenguaje de programación **Self** [33]. Self usa prototipos en lugar de clases, pero, a diferencia de Actors [3], unifica el acceso a estado y comportamiento. Al no tener referencias directas a métodos, el lenguaje base de Self no podía soportar operaciones reflectivas tradicionales e integradas. La omisión por parte de Self de referencias directas a métodos surge de su unificación entre invocación de métodos y acceso/asignación de variables, como se ilustra en la **Figura 2** del paper:

```
Un objeto Self con dos slots, x e y.
  x: rho * theta cos
  y: 17

(a mirror)
```

Enviar `y` a este objeto retorna `17`; enviar `x` a él retorna el producto de `rho` y `cos theta`. No hay forma en el lenguaje base de Self de obtener una referencia a este método.

Un mirror sobre el objeto de arriba: enviarle el mensaje `size` retorna `2`; enviarle `at: 'y'` retorna `17`; enviarle `at: 'x'` retorna **un mirror sobre el método** en el slot `x` (no el resultado de ejecutarlo).

Los diseñadores de Self sintieron que las referencias a métodos no eran orientadas a objetos, porque un método hace siempre lo mismo cuando es invocado, a diferencia de un mensaje enviado a un objeto, donde el objeto decide. Sin embargo, cuando llegó el momento de construir un entorno de programación (un IDE), se hizo evidente que se necesitaba una forma clara de referirse a los métodos.

La solución fue el **"mirror"**. Nombrado originalmente como un juego de palabras con "reflection" y también para sugerir "espejos y humo" (*smoke and mirrors*), la noción original de mirror era un objeto que aparentaría ser un diccionario cuyas entradas estaban nombradas por los nombres de slot del objeto original (el *"reflectee"*) y cuyas entradas contenían mirrors sobre los contenidos de los slots del objeto reflejado, satisfaciendo así el principio de estratificación. Más adelante, se introdujeron los **"slot objects"**. Cuando se le pide la entrada del diccionario para un slot dado, un mirror retorna un objeto que representa al slot. Contiene el nombre del slot, atributos (como si es un *parent slot*), y un mirror sobre el contenido del slot.

Los mirrors de Self soportan introspección y self-modification. Algunos ejemplos:

```
(reflect: anObject) size                            — retorna el número de slots en un objeto
(reflect: anObject) do: [:s | s printLine]           — imprime los slots de un objeto, uno por línea
(reflect: anObject) at: 'newSlotName' Put: (reflect: 17)  — agrega un slot conteniendo 17 al objeto
((reflect: anObject) at: 'fred') setParent: true     — convierte el slot llamado "fred" en parent slot
```

Como observó el co-diseñador de Self, Randall B. Smith, la desventaja de los mirrors es una pérdida de uniformidad: a veces no queda claro si un método nuevo debería aceptar un mirror o un objeto base como argumento. Peor aún, alguna funcionalidad, como imprimir (*printing*), no parece caer claramente ni en el nivel base ni en el meta-nivel. En general, sin embargo, los diseñadores de Self están bastante satisfechos con cómo resultó su diseño estratificado de mirrors.

En este paper, los autores argumentan que los lenguajes mainstream basados en clases se beneficiarían de un modelo de metaprogramación que siga los tres principios presentados. Sin embargo, si se acepta esta premisa, hay que reconocer que existe un problema fundamental con los lenguajes basados en clases tal como los conocemos. (Nota al pie: por supuesto, puede haber otros problemas con los lenguajes basados en prototipos también, que los autores no conocen.) Cada lenguaje de clases que conocen los autores exhibe los problemas asociados con `instanceOf` y los tests de identidad de clase descritos en otras partes del paper. Los autores creen que la mentalidad basada en clases en sí misma arrastra la implicación de que la clase de un objeto es algo que el código cliente debe conocer. Pero justamente ese conocimiento inhibe el reuso. Por otro lado, los lenguajes existentes basados en prototipos, incluso Self, no parecen permitir suficiente latitud al programador para expresar sus intenciones a nivel lingüístico. En consecuencia, coinciden con la visión de Ole Lehrmann Madsen [21] de que el próximo OOPL importante traerá juntos clases y prototipos. En otras palabras: conocen algunas de sus características, pero les falta un ejemplo concreto.

### 5.2 The Role of Types

Se distinguen cuatro categorías de sistemas de tipos:

1. Sistemas de tipos estructurales.
2. Sistemas de tipos nominales basados exclusivamente en interfaces.
3. Sistemas de tipos opcionales.
4. Sistemas de tipado dinámico.

Todos estos enfoques soportan el principio de encapsulación, pero solo una versión obligatoria (*mandatory*) de (2) puede ayudarnos a identificar de forma confiable cuándo la API reflectiva no está siendo realmente usada, lo cual soporta los objetivos de deployment de los autores.

Un sistema de tipos nominal y obligatorio que evite tipos de implementación puede proteger a los diseñadores de algunos errores de diseño que afectan a las arquitecturas reflectivas mainstream actuales. Los autores creen que un sistema así es una buena opción para lenguajes que buscan emplear *typechecking* obligatorio. Sin embargo, muchas consideraciones impactan el diseño del sistema de tipos de un lenguaje, y discutirlas está fuera del alcance de este paper.

Afortunadamente, un diseño cuidadoso de mirrors implica que no es necesario depender del sistema de tipos para separar la API reflectiva. Los beneficios de una API mirror-based pueden obtenerse casi independientemente del sistema de tipos usado, si hay alguno. La restricción clave sobre el sistema de tipos es que **evite depender exclusivamente de tipos de implementación**.

### 5.3 Designing Languages in tandem with Reflection

Por definición, una API reflectiva reifica la ontología de un lenguaje de programación. El principio de correspondencia estructural exige que cada construcción del lenguaje se mapee a una interfaz en la API. Examinando JDI, vemos un framework grande que tiene que reificar una ontología de lenguaje compleja (tipos primitivos, clases, interfaces, control de acceso, paquetes, métodos, constructores, inicializadores, etc.). La complejidad de un lenguaje se vuelve manifiesta en su API reflectiva, y el tamaño de la API está directamente relacionado con el tamaño del lenguaje. Esto se suma al atractivo de los lenguajes simples, con un número pequeño de construcciones muy generales, en contraposición a lenguajes complejos con un gran número de construcciones altamente especializadas.

Dado que la reflection es una necesidad en las aplicaciones modernas, parece plausible sugerir que los lenguajes se diseñen **en tándem** con sus APIs reflectivas. Si la API reflectiva parece demasiado grande y compleja, los diseñadores de lenguajes pueden tomar esto como indicación de que el lenguaje mismo es demasiado grande y complejo.

### 5.4 Metadata

La idea de soporte lingüístico para metadata definida por el usuario ha recibido mucha atención recientemente, con su introducción en C#. Tal soporte ahora también se agregó al lenguaje Java [30]. La metadata en este contexto consiste en datos especificados por el usuario adjuntados a elementos de la fuente del programa, como declaraciones de clases o métodos. El diseño de tales facilidades de metadata plantea muchos de los mismos temas discutidos en este paper —específicamente, la capacidad de examinar metadata en configuraciones distribuidas o cuando la fuente no está cargada.

Los mirrors de Self proveen un hogar acogedor (*"a cozy home"*) para su metadata. Originalmente, Self no tenía metadata especificable por el usuario. Más adelante en el proyecto, ganó la capacidad de que código de usuario asociara un objeto arbitrario (llamado *annotation*) con cualquier objeto o slot. La máquina virtual de Self implementó esta facilidad con espacio extra en sus *maps*, y expuso las anotaciones a través de mirrors. Los métodos a nivel Self en los mirrors implementaron toda la funcionalidad de anotación, como anotaciones get/set, para objetos y slots. Al proveer un lugar de primera clase (*first-class place*) para las operaciones de meta-nivel, el diseñador que elige mirrors se prepara para la futura expansión de las capacidades reflectivas.

### 5.5 Disadvantages of Mirrors

Las arquitecturas mirror-based reifican la distinción entre operaciones de nivel base y de meta-nivel. Cuando esta distinción es incómoda o ambigua, los mirrors pueden simplemente estorbar. Por ejemplo, consideremos un objeto de interfaz de usuario que le permite a un programador inspeccionar los slots de un objeto. Sin mirrors, uno podría esperar un prototipo como: `SlotExaminer newOn: anObject`. Pero en un sistema con mirrors, uno se enfrenta a una elección incómoda: si el mensaje toma un objeto como argumento, el slot examiner no puede usarse con un objeto proxy. Pero si el mensaje toma un mirror como argumento, cada invocación del método debe sufrir la verbosidad de la operación de creación del mirror. En un sistema no uniforme, cada opción tiene desventajas.

La cuestión de qué protocolo usar para un inspector de objetos puede parecer discutible para un verdadero creyente en reflection —al fin y al cabo el inspector está reflejando, así que mándele un mirror y aguante el costo (verbosidad)— pero a veces la línea entre nivel base y meta-nivel puede volverse borrosa, hasta el punto de que no queda distinción en absoluto. Consideremos la operación de imprimir un objeto. Lo que la mayoría consideraríamos una representación impresa razonable no respeta ninguna separación entre base y meta. Por ejemplo, un objeto lista podría imprimirse como "A List containing (a Car, a Truck)". La primera parte de ese string usa el nombre de la clase del objeto (meta-nivel), pero la última parte usa código de iteración de la lista (nivel base). Una arquitectura mirror-based agrega complejidad al código de impresión introduciendo cambios de nivel explícitos en el código. Donde la distinción entre nivel base y meta-nivel falla en modelar el problema a resolver, los mirrors se vuelven una molestia en lugar de una ayuda.

### 5.6 Future Work: The Ultimate Mirror System

Este paper concibe una API de reflection/metaprogramación que:

- Soporta introspección, self-modification e intercession tanto sobre código como sobre computación.
- Incluye capas distintas para reflejar el lenguaje de la máquina virtual y el o los lenguajes de alto nivel.
- Es claramente separable del lenguaje base subyacente, permitiendo que las aplicaciones que no usan la API de reflection/metaprogramación puedan desplegarse independientemente de ella.
- No asume una implementación particular; en cambio, permite de forma transparente uso local o remoto y demuestra soportar múltiples implementaciones.

Un sistema así debería poder soportar un IDE manipulando remotamente una VM de footprint pequeño que no incluya una implementación completa de reflection, como una encontrada en una PDA o teléfono móvil.

Ninguno de los sistemas reflectivos construidos hasta la fecha cumple completamente este objetivo, como puede verse en la Tabla 1 (que destaca la falta de soporte de self-modification y/o intercession) y en la **Tabla 2**, que resume otras propiedades clave de los sistemas mirror-based discutidos en el paper:

| | Compile time | Run time | VML | HLL | Reflects below Method lvl |
|---|---|---|---|---|---|
| Strongtalk | No | Sí | Sí | Sí | No |
| Self | Sí | Sí | No | Sí | No |
| JDI | No | Sí | Mixto | Mixto | Sí |
| APT | Sí | No | No | Sí | No |

En Strongtalk, la API fue diseñada con todos estos objetivos en mente excepto metaprogramación en tiempo de compilación, reflejando por debajo del nivel de método, e intercession. Sin embargo, el desarrollo de Strongtalk cesó antes de que la API de mirrors estuviera completamente madura. Como resultado, nunca se construyó una implementación distribuida que la validara.

En contraste, JDI implementa exitosamente metaprogramación distribuida en producción, pero asume que está operando sobre una representación en tiempo de ejecución, y no hace separación entre la máquina virtual y los lenguajes de alto nivel. Aunque la interfaz de JDI está diseñada para soportar completamente self-modification, las implementaciones reales son más restrictivas, y el soporte de intercession está completamente ausente.

Self todavía carece de soporte completo para facilidades de reflection a nivel de VM, y no soporta completamente ni reflection de grano fino por debajo del nivel de método ni intercession basada en mirrors. Aparte de eso, parece satisfacer los criterios de los autores.

La pregunta de cómo soportar intercession en un entorno mirror-based es intrigante. En lugar de especular, los autores la dejan para investigación futura.

## 6 RELATED WORK

### 6.1 Pluggable Reflection

El trabajo más cercano a este paper es [20], en el que Lorenz y Vlissides abordan deficiencias de los sistemas reflectivos mainstream. El foco principal está en la violación del principio de encapsulación. En lugar de considerar diseños alternativos para reflection y lenguajes, se concentran en una metodología pragmática y herramientas que mejoran el problema para usuarios de lenguajes existentes. Usando patrones y técnicas de componentes, trabajan en reducir el acoplamiento entre reflection y sus clientes. También notan el problema de la correspondencia temporal (aunque la terminología difiere), pero sin ofrecer una solución. No abordan directamente los otros principios de diseño discutidos aquí.

### 6.2 Declarative Metaprogramming

La metaprogramación declarativa [35] hace uso de lenguajes declarativos para metaprogramación. En particular, la **metaprogramación lógica** [36] usa un lenguaje de programación lógica para definir metaprogramas. El lenguaje que se está manipulando no necesita ser un lenguaje declarativo [37]. Cuando la metaprogramación ocurre entre lenguajes distintos, el principio de estratificación se respeta naturalmente. El uso de un lenguaje declarativo evita los problemas sutiles de identidad de clase mencionados en 2.2.1. Es posible construir un sistema de metaprogramación declarativo que obedezca los principios de encapsulación y correspondencia, pero ninguna de las dos propiedades puede tomarse por garantizada.

### 6.3 Lisp

Históricamente, la reflection fue pionerizada en Lisp, y el trabajo estándar sobre la semántica de reflection se hizo en el contexto de Lisp [11]. Los sistemas Lisp orientados a objetos, ejemplificados por CLOS, son los más cercanos a este paper.

La reflection en CLOS se soporta vía un **Meta-Object Protocol (MOP)** [18] que es parte de la definición del lenguaje. Un MOP es un modelo declarativo de la ontología del lenguaje. El MOP se enfoca en soportar reflection, incluyendo introspección, self-modification y, sobre todo, una rica noción de intercession.

Los meta-objetos de CLOS incluyen (entre otros) clases. Como en Smalltalk, las clases se usan tanto para fines de aplicación, como crear nuevas instancias (vía el método `make-instance`) y mantener estado compartido (vía variables `:class`), y pueden tener métodos específicos de aplicación también. Esto contradice el principio de estratificación. El MOP en general respeta la correspondencia estructural, pero solo reifica entidades que tienen semántica en tiempo de ejecución.

### 6.4 Compile-time MOPs

Los MOPs de tiempo de compilación ([12], [13], [31]) tienen dos propiedades clave:

1. Tratan con código: solo definen meta-objetos que reifican entidades que existen en tiempo de compilación.
2. Permiten que el código acceda al MOP mientras el código mismo está siendo compilado. Esto le permite al código influenciar cómo será compilado vía computación en tiempo de compilación, soportando una forma de intercession. El código puede incluso manipular su propia estructura usando el MOP, soportando programación generativa ([14], [28]).

El ítem (1) implica que los meta-objetos provistos por un MOP de tiempo de compilación son necesarios pero no suficientes para soportar el principio de correspondencia. Por otro lado, la habilidad de usar estos meta-objetos en computación de tiempo de compilación no es requerida por ninguno de los principios de diseño discutidos en este paper. Mayor discusión del ítem (2) está fuera del alcance de este paper.

### 6.5 APT

**APT (Annotation Processing Tool)** [4] es una API de metaprogramación de tiempo de compilación diseñada para soportar el procesamiento de metadata. La API es mirror-based: usa interfaces exclusivamente, y soporta encapsulación y estratificación. APT trata explícitamente solo con propiedades de tiempo de compilación del lenguaje fuente (Java), en línea con el principio de correspondencia. Sin embargo, la API no provee acceso a construcciones por debajo del nivel de método. Desafortunadamente, APT no está integrado con una API de reflection en tiempo de ejecución.

### 6.6 C#/.Net

La API de reflection de C# soporta introspección, así como la creación y evaluación dinámica de programas, pero no self-modification ni intercession.

La API está basada mayormente en clases abstractas. Esto permite que implementaciones alternativas se deriven mediante subclasificación. Sin embargo, el principio de encapsulación no se respeta de manera uniforme. En particular, la parte de la API que soporta la construcción dinámica de programas no usa clases abstractas ni interfaces. También ocurre que muchas de las clases abstractas no son completamente abstractas, y por lo tanto fijan ciertas propiedades (especialmente representaciones) para todas las implementaciones. A pesar de estas fallas, parece haber alcance considerable para implementaciones alternativas, al menos para introspección.

No hay una separación clara entre el nivel base y el meta-nivel. Las clases soportan directamente las operaciones reflectivas, y la operación `GetType` está embebida en la raíz de la jerarquía de tipos, `object`, y no puede sobreescribirse. Los tipos de clase también se exponen vía *checked casts*, el operador `typeOf` (el equivalente del `instanceof` de Java), y nociones de identidad de tipo cableadas (*hardwired*).

Aunque la API refleja principalmente la máquina virtual .Net, en lugar del lenguaje C# mismo, hay soporte para construcciones como las enumeraciones que parecen estar en el dominio de los lenguajes de alto nivel. No hay una capa distinta de la API dedicada al lenguaje de alto nivel.

No parece haber separación entre código y computación. Por ejemplo, los métodos soportan una operación `Invoke` que no podría soportarse al examinar clases en una base de datos de código fuente.

### 6.7 Beta

El sistema de metaprogramación de Beta, **Yggdrasil** [24], produce automáticamente jerarquías de clases basadas en una gramática abstracta de sintaxis, en estrecha correspondencia con el principio de correspondencia estructural. Las jerarquías generadas y las herramientas asociadas soportan metaprogramación pero no reflection. **MetaBeta** [10] provee soporte para reflection en tiempo de ejecución, incluyendo intercession. La distinción entre Yggdrasil y MetaBeta está en línea con el principio de correspondencia temporal, pero desafortunadamente las dos APIs no están relacionadas entre sí.

### 6.8 Oberon

La arquitectura reflectiva de Oberon-2 [25] factoriza reflection en un módulo separado, no muy distinto de los sistemas mirror-based. La información reflectiva se accede a través de *riders*, objetos iteradores que soportan el recorrido del programa reificado. Los riders se usan para introspección de las declaraciones del programa y del call stack, y para ejecución dinámica. El sistema no soporta self-modification ni intercession.

A diferencia de los mirrors, los riders no corresponden directamente a entidades individuales en un programa. En cambio, representan secuencias de entidades similares. Los riders corresponden de forma menos directa a la ontología del lenguaje, pero parecen soportar estratificación.

### 6.9 Firewall

Allen Wirfs-Brock et al. [34] discuten las propiedades de un modelo declarativo para programas Smalltalk. El "modelo de objeto abstracto" (*abstract object model*) que proponen parece ser un sistema de mirrors para Smalltalk, implementado como el prototipo **Firewall** para ParcPlace (ahora Cincom) Smalltalk. Discuten las ventajas para el desarrollo distribuido y el deployment; sin embargo, su discusión es específica de Smalltalk y depende críticamente de la noción más general de un modelo de programa declarativo para Smalltalk. No discuten la separación entre high-level mirrors y low-level mirrors, las interacciones con tipado estático y multithreading, ni la relación con prototipos.

Como implica Wirfs-Brock, una definición de lenguaje declarativa es una buena base para un sistema de mirrors limpio. Una parte clave de tal definición es la sintaxis abstracta del lenguaje.

### 6.10 Aspect-Oriented Programming

La programación orientada a aspectos (**AOP**) tiene sus raíces en reflection y en los meta-object protocols en particular. Una visión de AOP es que identifica un subconjunto de operaciones reflectivas que son frecuentemente útiles para el desarrollo de aplicaciones, y busca representar ese subconjunto a nivel base mediante construcciones dedicadas. Como tal, AOP está profundamente preocupada por la distinción entre operaciones de meta-nivel y de nivel base. Sin embargo, AOP se relaciona solo periféricamente con este paper, ya que la preocupación central de los autores es el diseño de APIs de meta-nivel.

## 7 CONCLUSIONS

Se presentaron tres principios de diseño para facilidades de meta-nivel en lenguajes de programación orientados a objetos:

1. **Encapsulation.** Las facilidades de meta-nivel deben encapsular su implementación.
2. **Stratification.** Las facilidades de meta-nivel deben estar separadas de la funcionalidad de nivel base.
3. **Ontological Correspondence.** La ontología de las facilidades de meta-nivel debe corresponder a la ontología del lenguaje que manipulan.

Los sistemas mirror-based encarnan sustancialmente estos principios. Aíslan las capacidades reflectivas de un lenguaje orientado a objetos en objetos intermediarios separados llamados mirrors, que se corresponden directamente con las estructuras del lenguaje, y hacen que el código reflectivo sea independiente de una implementación particular.

Como resultado:

- Los mirrors hacen más fácil el desarrollo remoto/distribuido.
- Los mirrors hacen más fácil el deployment, porque la reflection puede sacarse o agregarse fácilmente, incluso dinámicamente.

Los principios de diseño detrás de los mirrors pueden parecer obvios, y sin embargo no han sido ampliamente aplicados a las APIs reflectivas de los lenguajes orientados a objetos. Los mirrors fueron implementados en varios lenguajes de programación distintos. Esto incluye lenguajes basados en clases, tanto de tipado dinámico como estático, así como el lenguaje basado en prototipos Self en el que fueron originalmente concebidos. Los mirrors han sido demostrados exitosamente en la práctica: se han construido IDEs muy ricos usando reflection mirror-based, así como debuggers de calidad de producción.

El poder completo de los sistemas mirror-based todavía no se ha realizado. Sistemas que soporten completamente metaprogramación tanto de código como de computación, a nivel de la máquina virtual y del lenguaje de alto nivel, todavía no se han demostrado. Sin embargo, el potencial es claro.

En general, los autores creen que las ventajas de los sistemas mirror-based superan ampliamente sus desventajas, y que las APIs de metaprogramación mirror-based deberían ser la norma en los futuros lenguajes orientados a objetos.

## 8 ACKNOWLEDGMENTS / 9 REFERENCES

El paper cierra agradeciendo a los equipos que construyeron los sistemas mirror-based discutidos (equipos de Self, Strongtalk, JDI y APT) y a varios colegas por discusiones y comentarios sobre versiones anteriores del paper. Luego presenta una lista de 37 referencias bibliográficas (trabajos sobre Self, Strongtalk, Smalltalk-80, CLOS y su Meta-Object Protocol, JDI/JPDA, Beta y sus sistemas de metaprogramación Yggdrasil/MetaBeta, Oberon-2, APT, AOP, reflection en Lisp, metaprogramación declarativa y lógica, entre otros) que sustentan las afirmaciones técnicas e históricas hechas a lo largo del texto. No se detallan aquí ya que no son necesarias para seguir la argumentación central del paper.
