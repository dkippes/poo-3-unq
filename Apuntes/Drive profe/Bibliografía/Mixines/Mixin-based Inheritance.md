# Mixin-based Inheritance
Gilad Bracha & William Cook — OOPSLA 1990

> Resumen detallado en español, organizado siguiendo la estructura original del paper, para poder seguirlo en paralelo con el PDF. No es traducción literal sino una explicación fiel y completa, en mis propias palabras, de cada sección.

## Abstract

El paper interpreta los mecanismos de herencia de Smalltalk, Beta y CLOS como usos distintos de una misma construcción subyacente. Smalltalk y Beta se diferencian principalmente en la dirección de crecimiento de la jerarquía de clases. Estos mecanismos se subsumen en un nuevo modelo de herencia basado en la composición de mixins (subclases abstractas). Esta forma de herencia también puede codificar una jerarquía de herencia múltiple al estilo CLOS, aunque los cambios a esa jerarquía codificada que violarían el encapsulamiento son difíciles de realizar. La aplicación práctica de la herencia basada en mixins se ilustra con un bosquejo de extensión a Modula-3.

## 1 Introduction

Existen muchos mecanismos de herencia para lenguajes orientados a objetos: desde la herencia simple clásica de Smalltalk, pasando por el prefijado más seguro de Beta, hasta las combinaciones complejas y potentes de herencia múltiple de CLOS. Todos comparten un modelo de objetos similar y ven la herencia como mecanismo de programación incremental, pero difieren en el tipo de cambios incrementales que soportan.

- En **Smalltalk**, las subclases pueden agregar métodos nuevos o reemplazar métodos existentes del padre; no hay relación necesaria entre el comportamiento de las instancias de una clase y el de sus subclases. Los métodos de la subclase pueden invocar cualquier método original de la superclase vía `super`.
- En **Beta**, una definición de subpatrón (subclase) se ve como una extensión de un patrón prefijo definido previamente. También se pueden definir métodos nuevos, pero los métodos del prefijo no pueden reemplazarse; en cambio, el prefijo puede usar el comando `inner` para invocar el código del método extendido que provee el subpatrón. Como el código del prefijo se ejecuta siempre en cualquiera de sus extensiones, Beta impone un grado de consistencia de comportamiento entre un patrón y sus subpatrones.

El mecanismo subyacente de herencia es el mismo para Beta y Smalltalk. La diferencia está en si las extensiones a una definición existente tienen precedencia sobre las definiciones previas y pueden referirse a ellas (Smalltalk), o si la definición heredada tiene precedencia sobre las extensiones y puede referirse a ellas (Beta). Esto muestra que Beta y Smalltalk tienen jerarquías de herencia invertidas: una subclase de Smalltalk se refiere a su padre usando `super`, igual que un prefijo de Beta se refiere a sus subpatrones usando `inner`.

En **CLOS** (y su predecesor Flavors), se pueden fusionar varias clases padre durante la herencia. El grafo de ancestros de una clase se "lineariza" para que cada ancestro aparezca una sola vez. Con la combinación de métodos estándar para métodos primarios, se usa `call-next-method` para invocar el siguiente método en la cadena de herencia.

CLOS soporta los **mixins** como una técnica útil para construir sistemas a partir de atributos mezclables. Un mixin es una subclase abstracta, es decir, una definición de subclase que puede aplicarse a distintas superclases para crear una familia relacionada de clases modificadas. Por ejemplo, se puede definir un mixin que agregue un borde a una clase ventana; ese mixin podría aplicarse a cualquier tipo de ventana para crear una clase "ventana con borde". Semánticamente, los mixins están muy relacionados con los prefijos de Beta.

La linearización ha sido criticada por violar el encapsulamiento, porque puede cambiar las relaciones padre-hijo entre las clases de la jerarquía de herencia. Pero la técnica de mixin en CLOS depende directamente de la linearización y de la modificación de esas relaciones padre-hijo. En lugar de evitar los mixins porque violan el encapsulamiento, los autores argumentan que la linearización es solo una técnica de implementación de los mixins que oscurece su verdadera naturaleza como abstracciones.

Mediante una generalización modesta de los modelos de herencia de Smalltalk y Beta, se deriva una forma de herencia basada en la composición de mixins. Esta herencia basada en mixins soporta tanto la flexibilidad de Smalltalk como la seguridad de Beta, y soporta la codificación directa de jerarquías de herencia múltiple de CLOS sin duplicar definiciones de subclases. Sin embargo, como la jerarquía se codifica como una colección explícita de cadenas de herencia linearizadas en lugar de un único grafo de herencia, algunos cambios a la jerarquía (especialmente si violan la noción de encapsulamiento de Snyder) no se pueden hacer fácilmente.

**Estructura del paper:** la sección 2 discute los lenguajes de herencia simple Smalltalk y Beta y muestra que soportan usos muy distintos de una misma construcción subyacente. La sección 3 analiza la herencia múltiple y la linearización en CLOS, con foco especial en el soporte de mixins. La sección 4 presenta un mecanismo de herencia generalizado que soporta el estilo de herencia de Beta, Smalltalk y CLOS, con soporte explícito para mixins. La sección 5 bosqueja una extensión a Modula-3 que ilustra el uso de esta herencia generalizada. Finalmente, la sección 6 resume las conclusiones.

## 2 Single Inheritance Languages

### 2.1 Smalltalk Inheritance

La herencia en Smalltalk es un mecanismo de derivación incremental de clases, adaptado de Simula, y sirve como el mecanismo de herencia prototípico. La sutileza principal del proceso de herencia es la interpretación de las variables especiales `self` y `super`. `self` representa la recursión o autoreferencia dentro de la instancia de objeto que se está definiendo. El paper se enfoca en la interpretación de `super`.

Ejemplo: una clase `Person` con variable de instancia `name` y un método `display` que muestra el nombre. Una subclase `Graduate` agrega una variable `degree` y extiende `display` para mostrar primero (vía `super display`) el nombre y luego el título. Un graduado llamado "A. Smith" con título "Ph.D." respondería al método display invocando el display de Graduate, que invoca el display de Person mediante `super display` para mostrar el nombre, y luego muestra el título: el resultado neto sería "A. Smith Ph.D.". También sería posible anteponer el nombre con un título como "Dr." mostrándolo antes de llamar a `super`.

La subclase Graduate especifica solamente en qué se diferencia de Person. Esta diferencia puede indicarse explícitamente como un **delta**, o conjunto de cambios; en este caso el delta es simplemente el nuevo método display. La definición original también es solo un método display. Al combinarse, el nuevo método display reemplaza al original.

Para formalizar esto, los objetos se representan como registros cuyos campos contienen métodos. La notación `{a1↦v1,...,an↦vn}` representa un registro con campos `a1...an` y valores asociados `v1...vn`. `r.a` representa la selección del campo `a` del registro `r`. La combinación de registros es un operador binario `⊕` que forma un nuevo registro con los campos de sus dos argumentos, donde el valor proviene del argumento izquierdo en caso de que el mismo campo esté presente en ambos. Por ejemplo: `{a↦3, b↦'x'} ⊕ {a↦true, c↦8}` reemplaza el valor de `a` y da `{a↦3, b↦'x', c↦8}`.

Para interpretar `super`, es necesario que el delta (las modificaciones) pueda acceder al método original heredado de Person. Esto se logra pasando los métodos de la clase padre como parámetro al delta. El mecanismo de herencia resultante es una combinación asimétrica de un delta paramétrico `∆` y una especificación de padre `P`:

```
C = ∆(P) ⊕ P
```

Esta es una forma de herencia simple: `P` se refiere al padre heredado mientras que `∆` es un conjunto explícito de cambios. Las dos apariciones de `P` no significan que se instancia dos veces, sino que su información se usa en dos contextos: para interpretar `super` y para proveer métodos a la subclase. Suprimiendo la interpretación de las variables de instancia ocultas, el ejemplo anterior queda así:

```
P = {display ↦ name.display}
∆(s) = {display ↦ s.display, degree.display}
∆(P) = {display ↦ name.display, degree.display}
```

Aunque los deltas se introdujeron para aclarar la especificación del mecanismo de herencia, no son elementos independientes de un programa Smalltalk: no pueden existir por sí solos, siempre son parte de una definición de subclase, que tiene un padre explícito.

En Smalltalk, una subclase de Person puede reemplazar completamente el método display por, por ejemplo, una rutina que muestre la hora del día. En la herencia de Smalltalk, **la subclase tiene el control**: no hay forma de definir Person de modo que obligue a sus subclases a invocar su método display como parte de su propia operación de display.

### 2.2 Beta Inheritance

La herencia en Beta está diseñada para dar seguridad frente al reemplazo de un método por otro completamente distinto. Se soporta mediante el "prefijado" (*prefixing*) de definiciones. Beta usa una única construcción definicional, el *pattern*, para expresar tipos, clases y métodos; dado que esta generalidad puede confundir, el paper usa una sintaxis simplificada que distingue los distintos roles.

Ejemplo recodificado en Beta:

```
Person: class
(# name: string;
   display: virtual proc
     (# do name.display; inner #);
#);

Graduate: class Person
(# degree: string;
   display: extended proc
     (# do degree.display; inner #);
#);
```

La definición de Graduate está "prefijada" por Person. Person es el **superpatrón** de Graduate, y Graduate es **subpatrón** de Person. `display` se declara `virtual`, lo que significa que puede extenderse en un subpatrón — no que pueda redefinirse arbitrariamente, como en la mayoría de los lenguajes orientados a objetos.

El comportamiento del método display de Person es mostrar el nombre y luego ejecutar la instrucción `inner`. Para una instancia simple de Person, que no tiene comportamiento inner, esa instrucción es una operación nula (no-op). Cuando se define un subpatrón de Person, la instrucción `inner` ejecutará el método display correspondiente del subpatrón.

El subpatrón Graduate extiende el comportamiento del display de Person proveyendo el comportamiento inner. Para una instancia G de Graduate, el efecto inicial de `G.display` es el mismo que el de Person: se ejecuta el método original de Person. Después de mostrar el nombre, se ejecuta el procedimiento inner provisto por Graduate para mostrar el título. El uso de `inner` dentro de Graduate vuelve a interpretarse como no-op; solo tendría efecto si el método display se extendiera por un subpatrón de Graduate. Es imposible lograr que se imprima un título como "Dr." antes del nombre heredando de Person, porque la decisión de invocar `inner` después del nombre ya está fijada en el método display de Person. **En el prefijado de Beta, el prefijo controla el comportamiento del resultado.**

La interpretación del patrón Person es como una definición paramétrica de atributos, `P′`. El parámetro representa cualquier definición inner provista por los subpatrones. Para una instancia de Person, la parte inner de `P′` está ligada al registro de métodos nulos: `P′(∅)`.

Un subpatrón especifica atributos adicionales que también pueden referirse a cualquier comportamiento inner posterior de subpatrones futuros. Si los atributos definidos en el subpatrón se especifican mediante `∆′`, el resultado de prefijar por `P′` es la composición:

```
C′(inner) = P′(∆′(inner) ⊕ inner) ⊕ ∆′(inner)
```

Es decir, la interpretación `C′` del subpatrón, dado un parámetro inner, es el resultado de combinar la especificación del superpatrón `P′` con los cambios especificados por `∆′`. Al aplicar `P′` a `∆′(inner)⊕inner`, la especificación inner de `P′` queda ligada a los campos del subpatrón combinados con cualquier campo adicional provisto por subpatrones posteriores. Los métodos del prefijo tienen precedencia sobre el sufijo. Para el ejemplo, examinando los usos reales de inner, la ecuación se simplifica a:

```
P′(i) = {display ↦ name.display, i.display}
∆′(i) = {display ↦ degree.display, i.display}
C′(i) = {display ↦ name.display, degree.display, i.display}
```

Esta formulación no codifica directamente la restricción de que `inner` dentro de un método `m` solo puede referirse al método sufijo llamado `m`. En ese sentido, `inner` es menos general que el `super` de Smalltalk, pero la restricción se justifica por la seguridad que se busca. Una formalización alternativa que sí captura esta restricción representa cada método como una función de su comportamiento inner; el prefijado se define entonces como combinación de registros donde los campos duplicados se "componen". El formalismo resultante es equivalente al dado arriba, bajo la condición de que los campos de `P′` y `∆′` solo accedan a los campos correspondientes de inner.

### 2.3 Comparing Smalltalk and Beta

Los mecanismos de herencia de Smalltalk y Beta son orientaciones distintas de un mismo mecanismo subyacente: un operador binario no asociativo, `▷`, que aplica super/inner y combina atributos:

```
∆ ▷ P = ∆(P) ⊕ P
```

La relación entre Beta y Smalltalk se demuestra comparando las interpretaciones de la herencia en ambos lenguajes:

```
C = ∆ ▷ P             (Smalltalk)
C′(∅) = P′ ▷ ∆′(∅)     (Beta)
```

`∆` representa la información nueva explícita aportada por la subclase/subpatrón, mientras que `P` representa los atributos originales aportados por la superclase/superpatrón. El operador de combinación `▷` favorece los valores de su argumento izquierdo en caso de atributo duplicado.

Es claro que el mecanismo de herencia es el mismo; solo cambia la dirección de crecimiento. En Smalltalk los atributos nuevos son favorecidos y pueden reemplazar a los heredados; en Beta se favorecen los atributos originales. La herencia de Beta funciona en la dirección opuesta a la de la mayoría de los lenguajes orientados a objetos, debido a esta inversión de roles entre superpatrones/subpatrones y subclases/superclases. El paper incluye una figura (Figura 1) que ilustra esta inversión mostrando las relaciones semánticas en Smalltalk y Beta cuando una superclase se coloca arriba de una de sus subclases, incluyendo la interpretación de la autoreferencia, que es implícita en las referencias a variables (`var`) de Beta. Ninguna de las dos direcciones de herencia puede expresar a la otra, y cada una tiene sus ventajas y desventajas.

## 3 Multiple Inheritance and Mixins

### 3.1 CLOS Inheritance

CLOS soporta un mecanismo rico de herencia múltiple. El paper se enfoca solo en la combinación de métodos estándar y los métodos primarios.

Ejemplo recodificado en CLOS:

```lisp
(defclass Person () (name))
(defmethod display ((self Person))
  (display (slot-value self 'name)))

(defclass Graduate (Person) (degree))
(defmethod display ((self Graduate))
  (call-next-method)
  (display (slot-value self 'degree)))
```

`defclass` incluye el nombre de la clase nueva, la lista de sus superclases y la lista de sus variables de instancia. La lista de argumentos de `defmethod` define la clase sobre la que se define el método. La combinación de métodos, simple pero efectiva, se soporta con `call-next-method`, que cumple el rol de `super` en Smalltalk. Pero, igual que `inner` en Beta, `call-next-method` solo da acceso al siguiente método en la cadena de herencia con el mismo selector de mensaje.

Una clase CLOS puede heredar de más de un padre, por lo que un mismo ancestro puede heredarse más de una vez. Ejemplo: las siguientes clases hacen que Person se herede dos veces en Research-Doctor:

```lisp
(defclass Doctor (Person) ())
(defmethod display ((self Doctor))
  (display "Dr. ")
  (call-next-method))

(defclass Research-Doctor (Doctor Graduate) ())
```

Si no se tiene cuidado, el método display de Person se ejecutaría dos veces, y un Research-Doctor mostraría "Dr. A. SmithA. Smith Ph.D.". Para resolver esto, CLOS **lineariza** el grafo de ancestros de una clase, produciendo una lista de herencia en la que cada ancestro aparece una sola vez. El grafo de ancestros de Research-Doctor se lineariza a: Research-Doctor, Doctor, Graduate, Person. Esto también resuelve el problema del orden de invocación de métodos, porque las clases ancestro quedan en un orden lineal.

Cada conjunto de definiciones de métodos puede invocar métodos posteriores en la secuencia linearizada vía `call-next-method`. Si la especificación de los padres `P1,...,Pn` está dada por `∆1,...,∆n`, la interpretación `C` de la subclase se define iterando el operador de herencia sobre la lista:

```
C = ∆1 ▷ (∆2 ▷ (··· ▷ (∆n ▷ ∅)))
```

Cada especificación de la lista se aplica al resultado de la especificación anterior y se combina con ella. Los mecanismos de combinación de métodos más complejos de CLOS (como métodos *before*/*after*) también pueden modelarse en este marco.

El proceso de linearización ha sido criticado por violar el encapsulamiento. Un argumento es que la relación entre una clase y sus padres declarados puede modificarse durante la linearización: en el ejemplo, la clase Graduate queda ubicada entre Doctor y Person, contradiciendo la declaración explícita de que Doctor hereda directamente de Person. Solo conociendo toda la jerarquía de clases el programador puede prever esto.

Mediante la linearización, una jerarquía de herencia múltiple de CLOS se reduce a una colección de cadenas de herencia, cada una interpretable con herencia simple. Pero un cambio pequeño en la jerarquía original de CLOS puede producir una colección de cadenas de herencia muy distinta. Esto es especialmente cierto si los cambios violan la noción de encapsulamiento de Snyder, como cuando una clase base se divide en dos clases. Un problema menos grave es que una misma clase puede aparecer en muchas cadenas, así que si la colección se implementara en un lenguaje de herencia simple, las subclases tendrían que duplicarse. Para eliminar esa duplicación, el modelo de herencia simple debe generalizarse para permitir nombrar y reutilizar explícitamente los deltas definidos por las subclases.

### 3.2 Mixin Programming

Esta sección discute una técnica de programación común en CLOS llamada **mixins**. Un mixin es una subclase abstracta que puede usarse para especializar el comportamiento de varias clases padre. Suele hacerlo definiendo métodos nuevos que realizan alguna acción y luego llaman a los métodos correspondientes del padre. Los mixins son muy similares a los deltas introducidos informalmente en la sección 2.1.

Ejemplo: la noción de "tener un título de posgrado" como parte del nombre puede escribirse como un mixin independiente:

```lisp
(defclass Graduate-mixin () (degree))
(defmethod display ((self Graduate-mixin))
  (call-next-method)
  (display (slot-value self 'degree)))
```

Este ejemplo ilustra una característica de los mixins: invocan `call-next-method` aunque no parecen tener ningún padre. Esto obviamente causaría un error si se creara una instancia del mixin directamente. La linearización ubica al mixin en una cadena de herencia antes que otras clases que soporten ese método. Esto ocurre en la nueva definición de Graduate: como Graduate-mixin aparece antes que Person, el método display de Person será invocado por el display de Graduate-mixin.

```lisp
(defclass Graduate (Graduate-mixin Person) ())
```

En CLOS, los mixins son simplemente una **convención de codificación** y no tienen estatus formal. Aunque el uso de `call-next-method` sin un padre vinculado localmente es un indicio claro de que una clase es un mixin, el concepto no tiene definición formal, y cualquier clase podría usarse como mixin si aporta comportamiento parcial.

Usando Graduate-mixin, ahora es posible definir distintos tipos de clases con comportamiento "graduado": por ejemplo, un perro guardián podría tener un título de una escuela de obediencia:

```lisp
(defclass Guard-Dog (Graduate-mixin Dog) ())
```

Ni Smalltalk ni Beta soportan completamente los mixins. En Smalltalk, el efecto de un mixin puede lograrse creando subclases explícitas y copiando el código del mixin en la subclase, lo cual impide compartir código y la abstracción. En Beta, una clase individual se parece mucho a un mixin, pero no puede adjuntarse a clases definidas independientemente: la clase cliente debe construirse con el mixin como prefijo. Si se necesita una familia de versiones "mezcladas" de una misma clase, hay que copiar toda la clase para cada mixin prefijado. Entonces, en Smalltalk hay que copiar el mixin, mientras que en Beta hay que copiar la clase base — esto es consistente con el análisis de la dirección de crecimiento en Beta y Smalltalk hecho antes.

La programación con mixins aprovecha la herencia múltiple de una manera sutil y poco intuitiva: los mixins dependen de la linearización para ubicarse en el lugar adecuado de la cadena de herencia y para insertar otras clases entre el mixin y sus padres. Cuando los mixins se ven como subclases abstractas, o definiciones de clase parametrizadas por su padre, queda claro que la linearización cumple el rol de "aplicación", ligando el parámetro formal de padre del mixin a una clase concreta. Este proceso de abstracción y aplicación puede hacerse más explícito generalizando el mecanismo de herencia común a Smalltalk y Beta.

## 4 Inheritance as Composition of Mixins

Los mixins son la base de un mecanismo de herencia composicional que generaliza a Smalltalk y Beta, soportando además la codificación de una versión encapsulada de una jerarquía de herencia múltiple al estilo CLOS. La idea básica de la generalización es tomar a los mixins como la construcción definicional primaria. La herencia se formula entonces como composición de mixins. Los atributos nuevos pueden componerse al estilo Smalltalk o al estilo Beta (sobrescribiendo o extendiendo). Como los mixins y la composición son explícitos, no hace falta linearización implícita: el programador elegiría explícitamente el orden de todos los componentes mixin. Si un componente se compone más de una vez, aparecerá como copias múltiples en el resultado; la duplicación se evita aplicando explícitamente dos componentes a un padre compartido.

El operador de composición de mixins, escrito `⋆`, es el operador de herencia de Beta, pero usado en una forma un poco más general. La composición de mixins toma dos mixins como parámetros y devuelve un nuevo mixin como resultado:

```
M1 ⋆ M2 = fun(i) M1(M2(i) ⊕ i) ⊕ M2(i)
```

En caso de conflicto, `⋆` da prioridad al primer parámetro. En `M1`, super/inner queda ligado durante la operación de herencia a `M2`. En `M2`, super/inner queda ligado al parámetro formal `i` del resultado. Asumiendo que el operador básico de combinación de atributos `⊕` es asociativo, `⋆` también es asociativo. Además, si `⊕` fuera conmutativo, `⋆` también lo sería.

Las clases ordinarias se ven como mixins degenerados que no usan su parámetro inner/super. Así, los mixins generalizan a las clases de Smalltalk, los patrones de Beta y los mixins al estilo CLOS. Las clases abstractas se ven como mixins que se refieren a campos no definidos en `self`. Un mixin es **completo** si no se refiere a su parámetro padre y define en sí mismo todos los campos a los que se refiere; si no, es **parcial**. Solo los mixins completos pueden instanciarse de forma significativa.

## 5 Application to an Existing Language

### 5.1 Choice of Language

Eligieron **Modula-3** como base para una extensión que incorpora herencia basada en mixins. Modula-3 es adecuado para esta extensión porque soporta herencia simple y es fuertemente tipado. La herencia simple se generaliza naturalmente a herencia basada en mixins, y el tipado fuerte da un marco en el que los mixins pueden usarse de forma segura y eficiente.

### 5.2 Modula-3 Inheritance

Modula-3 soporta herencia vía "object types", que son análogos aproximados a las clases en la mayoría de los lenguajes orientados a objetos. Ejemplo:

```
type Person =
  object name: string
  methods display() := displayPerson
  end;

type Graduate = Person
  object degree: string
  methods display := displayGraduate
  end;

procedure displayPerson(self: Person) =
begin
  self.name.display();
end displayPerson;

procedure displayGraduate(self: Graduate) =
begin
  Person.display(self);
  self.degree.display()
end displayGraduate;
```

Person define una variable de instancia `name` y un método `display`. El método se define dando un nombre, seguido de una "signature" o lista de parámetros formales (en este caso vacía), y luego se le asigna un valor: un procedimiento definido por separado, `displayPerson`. Si `o` es un objeto de tipo Person, `o.display()` se interpreta como `displayPerson(o)`.

La definición de Graduate tiene dos partes: una definición preexistente, Person, y una modificación dada por la cláusula `object...methods...end`. Graduate es **subtipo** de Person (su **supertipo**); hereda de Person pero incluye una sobreescritura (*override*) del método display. La sobreescritura nombra al método sobreescrito y le asigna un nuevo valor (`displayGraduate`); no se da signature porque siempre es idéntica a la del método correspondiente en el supertipo. Graduate puede referirse a los métodos sobreescritos de Person mediante la sintaxis `Person.methodname` — esto es similar a `super` en Smalltalk, pero más general.

Una cláusula `object...methods...end` corresponde a la noción de delta discutida antes. Como en Smalltalk, los deltas no pueden definirse independientemente de un padre. La siguiente sección presenta una extensión en la que esos deltas se vuelven construcciones independientes.

### 5.3 Extending Modula-3

Se extiende Modula-3 generalizando los object types a mixins. Un mixin puede ser una modificación explícita, de la forma `object...methods...end`, o puede ser el resultado de combinar dos mixins previamente definidos:

```
Mixin = object...methods...end | Mixin1 ⋆ Mixin2
```

La sintaxis concreta usada en los ejemplos difiere de la notación usada hasta ese punto en tres aspectos: (1) se invierte el orden de los operandos del operador mixin, dando prioridad al operando de la derecha; (2) la operación mixin no se escribe explícitamente, sino que es implícita entre cada par de mixins en una definición; (3) se agrega una cláusula `super` opcional a las modificaciones. Los primeros dos cambios reflejan la sintaxis existente de Modula-3, lo que ayuda a que la extensión sea compatible "hacia arriba" con el lenguaje existente. El tercer cambio es por motivos de chequeo de tipos. La sintaxis resultante es:

```
Mixin′ = object...methods...end
       | object...methods...super...end
       | Mixin′2 Mixin′1
```

Ejemplo equivalente al ejemplo de mixin de CLOS dado antes:

```
type GraduateMixin =
  object degree: string
  methods display := displayGraduateMixin
  super display() := No_Op
  end;

mixin procedure
displayGraduateMixin(self: GraduateMixin) =
begin
  super.display()
  self.degree.display();
end displayGraduateMixin;

procedure No_Op(self: root) = begin end No_Op;

type Graduate = Person GraduateMixin;
```

Como GraduateMixin se define independientemente de cualquier padre, no se puede inferir la signature de `display`, así que debe darse en una cláusula `super` especial. De igual modo, no se conoce el valor sobreescrito de display, así que se le puede asignar un valor por defecto: aquí es `No_Op`, un procedimiento que funciona con cualquier tipo porque está definido sobre `root`, la raíz de la jerarquía de tipos. `displayGraduateMixin` se refiere al método display sobreescrito a través de la pseudovariable `super`, con la sintaxis `super.methodname`. Los procedimientos que referencian a `super` se distinguen con la palabra clave `mixin procedure`.

En el código, GraduateMixin juega un rol similar al de una subclase en Smalltalk. Invirtiendo la posición de GraduateMixin en la definición de Graduate, su rol pasa a ser el de un subpatrón de Beta. Esto se ilustra con PersonMixin funcionando como superpatrón:

```
type PersonMixin =
  object name: string
  methods display := displayPersonMixin
  super display() := No_Op
  end;

type Graduate = GraduateMixin PersonMixin;

mixin procedure
displayPersonMixin(self: PersonMixin) =
begin
  self.name.display();
  super.display()
end displayPersonMixin;
```

Aquí PersonMixin tiene el control al combinarse con GraduateMixin. `Graduate.display()` invoca `displayPersonMixin`, donde `super.display()` llama a `displayGraduateMixin`. En `displayGraduateMixin`, `super.display` usará el valor por defecto `No_Op`, lo que corresponde a una cláusula inner vacía en Beta.

Estos ejemplos tienen una ventaja importante sobre sus equivalentes en Smalltalk y Beta: **todas las partes de la definición pueden reutilizarse sin copiarse textualmente**.

Ejemplo final, recodificando el ejemplo de herencia múltiple de CLOS:

```
type Doctor =
  object
  methods display := displayDoctor
  super display() := No_Op
  end;

type ResearchDoctor =
  PersonMixin GraduateMixin Doctor;

mixin procedure displayDoctor(self: Doctor) =
begin
  display("Dr. ");
  super.display()
end displayDoctor;
```

Notar cómo la secuencia lineal de definiciones se da explícitamente, sin depender de la linearización automática.

#### 5.3.1 Typing

Esta subsección presenta las reglas de tipado para mixins en la extensión de Modula-3. El tipado de mixins no se había abordado en trabajos previos, porque los mixins no se habían introducido antes en un lenguaje fuertemente tipado.

La identidad de tipos se define como en Modula-3: dos tipos son idénticos si sus definiciones expandidas son idénticas.

La relación de subtipado sobre mixins, `T ≪ S` (T es subtipo de S, o S es supertipo de T), se define así:

1. `object...end ≪ root`. Todos los mixins son subtipos de root.
2. Si `T1 = T2 T3` entonces `T1 ≪ T2` y `T1 ≪ T3` (donde `=` denota identidad de tipos).
3. `≪` es reflexiva y transitiva.

Por ejemplo, `ResearchDoctor ≪ Doctor`, `ResearchDoctor ≪ GraduateMixin`, `ResearchDoctor ≪ PersonMixin`. Lo menos obvio es que si `PGMixin = PersonMixin GraduateMixin`, entonces `ResearchDoctor ≪ PGMixin`. Esto se sigue de que `ResearchDoctor = PGMixin Doctor` por la definición de identidad de tipos. Como el operador de combinación de mixins `⋆` es asociativo, esto se refleja en las reglas de subtipado.

Reglas adicionales para la composición de mixins:

- Un método debe mencionarse en la cláusula `super` de un tipo si fue sobreescrito. En la práctica, una sobreescritura puede omitirse de la cláusula `super` si su signature puede inferirse del contexto. Esta excepción se hace por compatibilidad con código Modula-3 existente.
- La pseudovariable `super` solo puede usarse en procedimientos declarados como `mixin procedure`. El primer parámetro del procedimiento debe ser de un tipo que incluya una sobreescritura para cada método referenciado a través de `super`.
- Un `mixin procedure` solo puede invocarse como método. Esto garantiza que no hay forma de acceder desde fuera de la instancia a los métodos sobreescritos de una instancia de mixin.

Todas las reglas dadas en esta sección pueden verificarse estáticamente, lo cual es condición necesaria para la seguridad y para una implementación eficiente. Estas reglas son específicas de Modula-3; una extensión de otro lenguaje seguramente diferiría en muchos detalles. Sin embargo, la estrategia básica de generalizar los object types (o clases, en otros lenguajes) a mixins es fundamental para cualquier extensión de este tipo.

## 6 Conclusion

Los mecanismos de herencia de Beta, Smalltalk y CLOS representan tres decisiones de diseño distintas para la herencia. Aunque en la superficie son muy distintos, los autores identifican una estructura subyacente común: este mecanismo combina dos conjuntos de atributos de modo que las definiciones de atributos duplicadas toman el valor de uno de los conjuntos, donde ese valor usado puede referirse al valor que está siendo eliminado.

Beta y Smalltalk soportan ambos herencia simple, en la que una única definición existente puede extenderse con atributos nuevos. En Smalltalk, los atributos nuevos pueden reemplazar a los existentes, accesibles directamente vía `super`. En cambio, Beta prohíbe que las extensiones reemplacen atributos existentes; una nueva definición para un atributo existente solo tiene efecto al ser invocada cuando el atributo original ejecuta el comando `inner`. Estos dos mecanismos tienen relaciones inversas entre la definición heredada y las extensiones: la relación subclase/superclase de Smalltalk es análoga a la relación superpatrón/subpatrón de Beta, donde `super` es análogo a `inner`.

CLOS soporta herencia múltiple, en la que varias definiciones existentes pueden combinarse. Para evitar la duplicación de componentes, CLOS lineariza el conjunto de componentes primitivos en las definiciones heredadas. Esta lista lineal de componentes se combina luego mediante el mismo mecanismo que subyace a Smalltalk y Beta. Una desventaja de la linearización es que pueden cambiar las relaciones entre los componentes primitivos. Sin embargo, los autores muestran que la linearización es la base de la útil técnica de programación con mixins.

Proponen que el mecanismo de herencia subyacente —que aparece en dos formas restringidas distintas en Beta y Smalltalk, y queda oculto detrás de la linearización en CLOS— se use como fundamento de una construcción de herencia general. En esta formulación, los mixins se convierten en la construcción definicional básica, y la herencia se interpreta como composición de mixins. Como la composición de mixins es explícita, no surge el problema de que la linearización viole el encapsulamiento.

No parece difícil extender Beta y Smalltalk para soportar mixins y herencia generalizada. Este trabajo podría aplicarse a CLOS, que ya soporta mixins, para hacerlos más explícitos y menos susceptibles a problemas de encapsulamiento. Un bosquejo de extensión a Modula-3 ilustra un diseño posible para mixins y herencia generalizada.

## References

El paper cierra con una lista de 19 referencias bibliográficas (trabajos de Cardelli, Cook, Dahl/Myhrhaug/Nygaard sobre Simula, DeMichiel/Gabriel sobre CLOS, Goldberg/Robson sobre Smalltalk-80, Keene sobre CLOS, Kristensen et al. sobre Beta, Moon sobre Flavors, Snyder sobre encapsulamiento, Wand sobre tipos, Wegner/Zdonik, entre otros) que sustentan las distintas afirmaciones técnicas e históricas hechas a lo largo del texto.
