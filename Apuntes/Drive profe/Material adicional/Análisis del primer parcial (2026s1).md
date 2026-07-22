# Análisis del primer parcial — POO3 (2026s1)

> Análisis pregunta por pregunta con respuestas correctas y precisas, basadas en la corrección del profesor y en la bibliografía obligatoria. El objetivo es entender *por qué* cada respuesta anterior fue incorrecta y qué se esperaba.

---

## Preguntas obligatorias

---

### Pregunta 1 — Diferencias entre mixins y traits

**Enunciado:** Explicá, de manera breve pero técnica y precisa, las diferencias entre mixins y traits.

**Corrección del profe:** Faltó la linearización. La definición de conflicto era incorrecta ("concatenación de selectores" no tiene sentido). "Selector" = nombre de un mensaje (en Ruby, un Symbol como `:foo`). No apareció la palabra "método".

---

**Respuesta correcta:**

Un **mixin** es una función que toma una clase (su superclase) y retorna una nueva clase. Formalmente: `M ▷ C = M(C)`, donde `M` es el mixin y `C` la superclase sobre la que se aplica. El mixin no existe por sí solo; siempre se aplica sobre una clase concreta.

Cuando se componen múltiples mixins, el resultado es una **linearización**: una cadena de herencia construida en el momento de la composición. Por ejemplo, aplicar `Logging ▷ (Timestamped ▷ Base)` produce la cadena:

```
ClaseResultante → Logging(Timestamped(Base)) → Timestamped(Base) → Base
```

Dentro de esta cadena, `super` funciona como en herencia simple: sube al siguiente eslabón de la cadena. Esto tiene una consecuencia importante: si el mismo mixin se incluye más de una vez en una jerarquía, no hay recursión infinita, porque cada aplicación genera un eslabón distinto en la cadena (el mixin en sí no referencia a ninguna superclase fija).

Los **conflictos** (cuando dos mixins definen un método con el mismo selector, es decir, el mismo nombre de mensaje) se resuelven automáticamente por el orden de linearización: el método del mixin que quedó más arriba en la cadena gana.

Un **trait** es un conjunto de métodos que se compone con una clase mediante **flattening** (aplanado): semánticamente, es como si los métodos del trait hubieran sido definidos directamente en la clase. El trait no ocupa un lugar en la cadena de herencia; por eso, no hay `super` entre la clase y el trait.

Un **conflicto** en traits ocurre cuando dos traits (o un trait y la clase) definen un método con el mismo selector. La resolución es siempre **explícita**: la clase debe intervenir usando los operadores de composición:

| Operador | Función |
|---|---|
| `+` | Combina dos traits; deja el conflicto sin resolver si hay métodos con el mismo selector |
| `▷` | El trait izquierdo tiene precedencia sobre el derecho |
| `−` | Excluye un método de la composición |
| `@` | Alias: renombra un método en la composición |

Los métodos definidos directamente en la clase siempre tienen precedencia sobre los aportados por traits.

**Tabla comparativa:**

| | Mixin | Trait |
|---|---|---|
| ¿Qué es? | Función de clase → clase | Conjunto de métodos |
| Lugar en la herencia | Sí (linearización) | No (flattening) |
| `super` | Funciona (sube la cadena) | No aplica entre clase y trait |
| Resolución de conflictos | Automática (por orden) | Explícita (por operadores) |
| Estado | Puede tenerlo | No (en modelo original); sí en stateful traits |

**Bibliografía:** Bracha & Cook, *Mixin-based Inheritance*; Ducasse et al., *Traits: A Mechanism for Fine-grained Reuse*.

---

### Pregunta 2 — ¿Es el sistema reflexivo?

**Enunciado:** Alguien afirma que su lenguaje es reflexivo porque los programas se expresan como archivos de texto, se pueden leer y escribir desde el programa con `File`, y eso incluye los archivos que representan al programa mismo. ¿Es suficiente para concluir que el sistema es reflexivo?

**Corrección del profe:** Va por ese lado, pero habría que elaborarlo más.

---

**Respuesta correcta:**

No, no es suficiente. La afirmación confunde dos cosas distintas: tener acceso a los *archivos de texto* que describen el programa no es lo mismo que tener una **self-representation causalmente conectada** con el sistema en ejecución.

Según Maes (*Concepts and Experiments in Computational Reflection*), para que un sistema sea reflexivo necesita cumplir dos condiciones:

1. Incorporar estructuras internas que **representen** (aspectos de) sí mismo (la self-representation).
2. Esas estructuras deben estar **causalmente conectadas** con el sistema: si el sistema cambia, la representación se actualiza; y si se modifica la representación, el comportamiento del sistema cambia en consecuencia.

El problema con el argumento del enunciado es que leer un archivo de texto con `File.read` produce un `String`: una cadena de caracteres que no tiene ninguna conexión con el proceso en ejecución. Si el proceso se modifica en memoria, el archivo no se actualiza solo. Si se modifica el archivo, el proceso que ya está corriendo no se ve afectado. No hay conexión causal en ninguna dirección.

Para que hubiera reflexión real, sería necesario que:
- La representación del proceso en ejecución fuera accesible como un objeto de primera clase en el sistema, y
- Modificar esa representación tuviera un efecto real e inmediato sobre el comportamiento del proceso.

Esto sí ocurre, por ejemplo, en Smalltalk: podés inspeccionar y modificar el diccionario de métodos de una clase mientras el sistema está corriendo, y el cambio afecta inmediatamente a todos los objetos de esa clase. Eso sí cumple con la conexión causal.

En conclusión: que el programa sea un archivo de texto legible y modificable es una propiedad del lenguaje (representación de programas como texto), pero no implica reflexión. La reflexión requiere conexión causal entre la representación y el proceso en ejecución.

---

### Pregunta 3 — Problema en el metamodelo

**Enunciado:** Hay un lenguaje con el metamodelo del diagrama (Object instancia de MetaObject, Guerrero instancia de MetaGuerrero, ambas metaclases instancias de Class, pero MetaGuerrero NO hereda de MetaObject). Explicá qué problema concreto aparece. Justificá dando un ejemplo de un caso que fallaría.

**Corrección del profe:** No describió bien el problema. Hay bibliografía específica al respecto. La descripción de lo que hace Ruby estaba bien pero no respondía el problema.

---

**Respuesta correcta:**

El problema es la **ausencia de jerarquía paralela entre clases y metaclases**. En el diagrama, `Guerrero` es subclase de `Object`, pero `MetaGuerrero` (la metaclase de `Guerrero`) no hereda de `MetaObject` (la metaclase de `Object`). Esto rompe la **compatibilidad ascendente** entre metaniveles.

La compatibilidad ascendente establece que si `Guerrero` es subclase de `Object`, entonces todo mensaje que `Object` entiende *como clase* debería ser también entendido por `Guerrero` *como clase*. Esto es necesario para que el polimorfismo funcione correctamente en el nivel de las clases (no solo en el nivel de las instancias).

**Ejemplo concreto que fallaría:**

Supongamos que `MetaObject` define un método de clase específico, por ejemplo `:describe_class`, que retorna información sobre la clase (nombre, número de instancias, etc.). Este método es propio de `MetaObject`, no de `Class`.

- `Object describe_class` → busca `:describe_class` en `MetaObject` → lo encuentra → OK.
- `Guerrero describe_class` → busca `:describe_class` en `MetaGuerrero` → no lo encuentra; `MetaGuerrero` no hereda de `MetaObject` → falla.

El problema no es solo que falle la búsqueda: es que **se rompe el polimorfismo a nivel de clases**. Si tengo una variable que puede contener `Object` o cualquier subclase suya (como `Guerrero`), debería poder enviarle `:describe_class` de manera polimórfica. Pero sin la jerarquía paralela, eso no está garantizado: `Object` lo entiende pero `Guerrero` no.

Dicho de otra forma: la relación de herencia entre clases debería verse reflejada en el nivel meta. Si no lo está, el polimorfismo funciona para las instancias pero se corta cuando las clases mismas son el receptor del mensaje.

**Lo que hace Ruby para evitar este problema:**

En Ruby, si `B` es subclase de `A`, entonces la singleton class de `B` (equivalente a su metaclase) es subclase de la singleton class de `A`. Es decir, la jerarquía de singleton classes es paralela a la jerarquía de clases. Esto garantiza que cualquier método de clase definido en `A` sea heredado automáticamente por `B` como clase, sin necesidad de redefinirlo.

**Bibliografía:** Bouraqadi et al., *Safe Metaclass Programming*, secciones 1-2.

---

### Pregunta 4 — Clases abiertas y comportamiento dinámico

**Enunciado:**
a. ¿Qué mecanismo del paradigma orientado a objetos hace que agregar/redefinir un método en una clase afecte incluso a las instancias ya creadas?
b. Si en plena ejecución de un método se redefine ese mismo método en la clase, ¿la ejecución en curso se ve afectada? Justificá basándote en (a).

**Corrección del profe:** La reflexión no es la respuesta a (a). La pregunta es por el mecanismo del *paradigma OO* que hace posible esto. La respuesta a (b) no estaba justificada.

---

**Respuesta correcta:**

**Apartado (a):**

El mecanismo es el **method lookup dinámico**: la búsqueda del método que corresponde a un mensaje se realiza *en tiempo de ejecución*, en el momento en que se recibe el mensaje, consultando el estado actual del diccionario de métodos de la clase.

Una instancia de `Guerrero` no guarda una copia de los métodos de su clase: guarda una *referencia* a la clase `Guerrero`. Cuando recibe un mensaje, el intérprete busca el método en el diccionario de métodos de `Guerrero` (y sube por la cadena de herencia si no lo encuentra) en ese preciso momento. Si el diccionario fue modificado (por clases abiertas, `define_method`, etc.) antes de que llegue el mensaje, la búsqueda encontrará la versión nueva. La instancia no necesita "enterarse" del cambio; el mecanismo de lookup lo maneja transparentemente.

Esto funciona porque el *método* no está en el objeto sino en la clase. El objeto solo es el receptor; la búsqueda del comportamiento siempre se delega a la clase en tiempo de ejecución.

**Apartado (b):**

La ejecución *en curso* de un método no se ve afectada por una redefinición de ese mismo método mientras se ejecuta.

Razón basada en (a): cuando el método empezó a ejecutarse, el lookup ya encontró el método y comenzó a ejecutarlo. Esa activación (ese frame de ejecución) ya tiene el código del método anterior. Redefinir el método en la clase solo modifica el diccionario de métodos para *futuros* lookups.

Sin embargo, si durante la ejecución del método en curso se envía un mensaje a `self` (por ejemplo, una llamada recursiva o un send a otro método del mismo objeto), *ese* lookup se realizará en el momento del envío y encontrará el método redefinido. Es decir: la activación actual no se interrumpe, pero cualquier send futuro ya usará la nueva definición.

---

## Preguntas bonus

---

### Pregunta 5 — Introspección vs. intercesión

**Enunciado:** Dentro de la reflexión computacional, distinguimos la introspección y la intercesión. Explicá brevemente en qué consiste cada una, dando ejemplos.

**Corrección del profe:** La respuesta era incorrecta. Leer archivos de texto no implica introspección. Se necesita conexión causal desde el proceso en ejecución hacia la representación. Para intercesión se necesita además la relación inversa. Faltó relacionar con bibliografía.

---

**Respuesta correcta:**

Ambas son formas de **reflexión computacional** (Maes), y ambas requieren que haya una **self-representation causalmente conectada** con el proceso en ejecución. La diferencia está en la dirección de esa conexión causal:

**Introspección** — El sistema *lee* información sobre su propio estado de ejecución. La conexión causal va del proceso hacia la representación: el estado del proceso se refleja en la representación, que puede ser consultada.

Ejemplos en Ruby:
```ruby
obj.class                  # → retorna la clase de obj en tiempo de ejecución
obj.respond_to?(:save)     # → consulta si obj entiende ese selector
obj.instance_variables     # → lista las variables de instancia actuales
String.instance_methods    # → lista los métodos de instancia de String
```

**Intercesión** — El sistema *modifica* su propio comportamiento a través de la self-representation. La conexión causal va en ambas direcciones: se puede leer (introspección) y además las modificaciones afectan causalmente al proceso en ejecución.

Ejemplos en Ruby:
```ruby
String.define_method(:shout) { upcase + "!!!" }
# → modifica el diccionario de métodos de String; afecta a todas las instancias existentes

class Guerrero
  def method_missing(selector, *args)
    puts "No entiendo #{selector}"
  end
end
# → intercepta cualquier mensaje no reconocido; cambia el comportamiento del sistema
```

**Lo que NO es introspección:** leer el archivo fuente del programa con `File.read`. El resultado es un `String` sin conexión causal con el proceso: si el proceso cambia en memoria, el archivo no se actualiza; si se modifica el archivo, el proceso no se ve afectado.

**Bibliografía:** Maes, *Concepts and Experiments in Computational Reflection*, secciones 2-4.

---

### Pregunta 6 — Restricciones del mixin sobre su dominio

**Enunciado:** Un mixin puede entenderse como una función que recibe una clase y retorna una clase. ¿Qué restricciones puede establecer esa función sobre su dominio? Justificá con ejemplos.

**Corrección del profe:** El mixin no incluye una superclase fija, entonces no puede haber recursión infinita si está incluido más de una vez. Si la jerarquía tiene un límite (BasicObject), simplemente pasaría dos veces por el mixin.

---

**Respuesta correcta:**

Un mixin es una función `M: Clase → Clase`. Su dominio —el conjunto de clases que puede recibir como argumento— está restringido de forma **implícita** por el uso de `super` dentro de los métodos del mixin.

Cuando un método del mixin llama a `super`, está asumiendo que la clase que recibe como argumento (su superclase en la linearización) provee un método con ese mismo selector. Si la superclase no lo provee, el lookup fallará al llegar al tope de la cadena.

**Ejemplo:**

```ruby
module Logging
  def save
    puts "Guardando..."
    super         # requiere que la superclase tenga #save
  end
end

# Válido: ActiveRecord::Base provee #save
class Articulo < ActiveRecord::Base
  include Logging
end

# Inválido: Object no tiene #save; super fallará
class Cosa < Object
  include Logging
end
Cosa.new.save  # → NoMethodError: super: no superclass method `save`
```

La restricción sobre el dominio es entonces: **la clase argumento (o algún ancestro suyo) debe proveer los métodos que el mixin llama mediante `super`**. Estos son los "requisitos" del mixin.

**Sobre incluir el mismo mixin dos veces:**

El mixin no contiene una referencia fija a ninguna superclase; la superclase se fija en el momento de la aplicación. Si el mismo mixin se incluye más de una vez en una jerarquía, simplemente se pasa dos veces por los métodos del mixin durante el lookup. No hay recursión infinita porque la cadena de herencia tiene un límite finito (en Ruby, `BasicObject`). Al llegar ahí, si `super` no encuentra el método, lanza un `NoMethodError` normal.

**Bibliografía:** Bracha & Cook, *Mixin-based Inheritance*.

---

### Pregunta 7 — Identidad con comportamiento idéntico

**Enunciado:** Dos objetos entienden exactamente los mismos mensajes y responden siempre con los mismos resultados y efectos. Con la definición de objeto que dimos en la materia, ¿alcanza esto para decir que son el mismo objeto? ¿Podría cambiar la respuesta con otro enfoque?

**Corrección del profe:** ¿Cómo sabemos si la identidad es la misma? Si es enviando un mensaje, daría el mismo resultado. Si no es enviando un mensaje, ya estamos fuera del enfoque puro.

---

**Respuesta correcta:**

Con la definición de objeto centrada en el **mensaje** —donde lo único que puede hacerse con un objeto es enviarle un mensaje, y un objeto se caracteriza enteramente por cómo responde a los mensajes que entiende— la respuesta es: **no podemos distinguirlos**, y desde este enfoque, serían el mismo objeto.

La razón es que no existe ninguna operación disponible (dentro del enfoque puro de mensajes) que pueda diferenciarlos. Si enviamos cualquier mensaje a ambos, obtenemos el mismo resultado. Incluso si intentáramos enviarles un mensaje como `:equal?` o `:object_id`, si responden igual, no hay forma de distinguirlos mediante el paradigma.

**¿Puede cambiar la respuesta con otro enfoque?**

Sí. Si abandonamos el enfoque puro de mensajes y consideramos la **identidad** como una propiedad primitiva del objeto —por ejemplo, su dirección de memoria o un identificador interno del sistema— entonces dos objetos podrían tener identidades distintas aunque se comporten de forma idéntica. En ese caso serían objetos distintos que se comportan igual.

Este es el caso en muchos lenguajes concretos: en Ruby, `equal?` compara identidad por referencia (dirección de memoria), y `==` compara comportamiento/valor. Dos objetos pueden pasar `==` pero fallar `equal?`.

La distinción importante es: en el enfoque puro orientado a mensajes (como lo presenta la materia), la identidad solo puede determinarse enviando un mensaje. Si ese mensaje produce el mismo resultado en ambos objetos, el enfoque no puede afirmar que son distintos.

---

### Pregunta 8 — JavaScript, `this` y Method vs. UnboundMethod

**Enunciado:** En JavaScript hay muchas complicaciones con cómo se liga `this`. Relacioná este problema con la ausencia de la distinción que hace Ruby entre `Method` y `UnboundMethod`. ¿Podría haberse evitado si JavaScript hubiese hecho esa distinción?

**Corrección del profe:** La respuesta fue incorrecta y confusa. "Selector de una clase" no tiene sentido.

---

**Respuesta correcta:**

En Ruby, un **método** puede estar en dos estados:

- **`Method`** (método ligado): está asociado a un receptor específico. Sabe a qué objeto corresponde `self`. Puede ejecutarse directamente.
- **`UnboundMethod`** (método no ligado): es el código del método sin receptor. Para ejecutarlo, primero debe ligarse (`bind`) a un objeto. Solo puede ligarse a instancias de la clase donde fue definido (o subclases).

Esta distinción existe porque los métodos acceden a `self` y a las variables de instancia del receptor; sin un receptor concreto, esa información no existe.

**El problema de JavaScript:**

En JavaScript, las funciones (equivalentes a métodos) no tienen un receptor fijo. `this` se determina dinámicamente según *cómo* se invoca la función, no según *dónde* fue definida:

```javascript
const guerrero = {
  nombre: "Aragorn",
  saludar: function() { return this.nombre; }
};

const fn = guerrero.saludar;
fn();  // → undefined (this es el objeto global, no guerrero)
guerrero.saludar();  // → "Aragorn" (this es guerrero)
```

El problema es que `fn` es equivalente a un `UnboundMethod` de Ruby, pero JavaScript no lo hace explícito. El programador no sabe, al ver `const fn = guerrero.saludar`, que `this` quedó desligado. El error solo aparece al ejecutar.

**¿Hubiera ayudado la distinción de Ruby?**

Sí. Si JavaScript hubiera hecho explícita la distinción entre función ligada y no ligada:
- Extraer una función de un objeto produciría un `UnboundMethod` explícito.
- Para poder llamarla, habría que ligarla primero a un receptor (`bind`).
- El intento de llamar a un `UnboundMethod` sin ligar produciría un error en el momento de la llamada, con un mensaje claro ("método no ligado"), en lugar del comportamiento silencioso de `this = undefined`.

JavaScript sí ofrece `Function.prototype.bind()` como mecanismo explícito de ligado, pero no lo impone: las funciones "no ligadas" siguen siendo invocables y ligan `this` dinámicamente, lo que es la fuente de la confusión.

**Bibliografía:** documentación oficial de Ruby sobre `Method` y `UnboundMethod`.

---

## Resumen de errores conceptuales comunes

| Concepto | Error frecuente | Corrección |
|---|---|---|
| **Selector** | Confundirlo con método o con definición | Selector = nombre del mensaje (en Ruby, un Symbol como `:save`) |
| **Conflicto en traits** | "Dos selectores iguales concatenados" | Conflicto = dos traits proveen un método con el mismo selector |
| **Reflexión** | "Puedo leer el código fuente, soy reflexivo" | Se necesita conexión causal con el *proceso en ejecución* |
| **Introspección** | Leer archivos | Leer el estado del *proceso vivo* (`obj.class`, `respond_to?`, etc.) |
| **Mecanismo OOP para clases abiertas** | "Es la reflexión" | Es el **method lookup dinámico**: el método se busca en tiempo de ejecución |
| **Mixin dos veces en jerarquía** | "Recursión infinita" | No hay recursión: el mixin no tiene superclase fija; simplemente pasa dos veces por la cadena |
| **Identidad en enfoque puro de mensajes** | Asumir que hay otra forma de verificar identidad | En el enfoque puro, solo se puede verificar enviando mensajes; si responden igual, son indistinguibles |
