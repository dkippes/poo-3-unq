# Guía de estudio para el primer parcial — POO3 (2026s1)

UNQ — Programación con Objetos 3

> Resumen punto por punto de la guía oficial. El parcial cubre la primera parte de la materia (Ruby). No evalúa reproducción memorística sino comprensión conceptual y uso correcto del vocabulario.

---

## 1. Repaso y profundización de temas previos

### 1.1 ¿Qué es un objeto?

El punto de partida de la materia es revisar críticamente la definición de "objeto". La clave es entender que un objeto se define **a partir del mensaje**, no al revés.

De esa definición central se deducen tres propiedades:

| Propiedad | Qué significa |
|---|---|
| **Identidad** | Cada objeto es único, independientemente de su estado |
| **Implementación interna** | El objeto decide cómo responde a los mensajes (encapsulamiento) |
| **Polimorfismo** | Distintos objetos pueden responder al mismo mensaje de formas distintas |

**Para el parcial:** saber usar el vocabulario con precisión. Distinguir entre objeto, clase, instancia, mensaje, método.

---

### 1.2 Repaso de subclasificación

Temas que entran:

- **Subclasificación como herramienta técnica** (reusar código) *y* como herramienta de modelado conceptual. Son dos usos distintos y no siempre coinciden.
- **Method lookup:** cómo el intérprete busca el método al recibir un mensaje. Empieza en la clase del receptor, sube por la cadena de superclases hasta encontrarlo (o fallar).
- **Limitaciones de la herencia simple:** no se puede heredar de dos clases a la vez. Esto obliga a elegir una jerarquía, y a veces la jerarquía "correcta" para un uso no es correcta para otro.
- **`super`:** no es un objeto distinto, es el mismo receptor (`self`), pero el método lookup empieza desde la superclase de la clase donde está escrito el `super`. Esto es diferente del `inner` de Beta, donde es el padre quien llama al hijo.

**Heurística para subclasificación:**

> "Todo X es un Y" (en tanto respeta el protocolo de Y)

Nota: decir "es un" sería confundir relación entre *clases* con relación entre *instancia y clase*. Ver apéndice al final.

**Bibliografía relevante:**
- Bracha & Cook, *Mixin-based Inheritance*, sección 2 (super vs. inner, Smalltalk vs. Beta)
- Ducasse et al., *Traits: A Mechanism for Fine-grained Reuse*, sección 2 (limitaciones de herencia simple)

---

## 2. Esquemas no tradicionales para definir el comportamiento de los objetos

### 2.1 Mixines

Un **mixin** es una "subclase abstracta": es una función que toma una superclase y devuelve una nueva clase. No puede usarse solo; siempre se aplica *sobre* algo.

```
Mixin ▷ Superclase = nueva clase que extiende Superclase con el comportamiento del Mixin
```

Temas que entran:

- **Mixin como función de superclase a clase.** El mixin define comportamiento pero deja abierta la superclase. Al aplicarlo, se fija la superclase.
- **Requisitos del mixin sobre su superclase.** Un mixin puede llamar a `super` para métodos que espera que la superclase provea. Eso es un "requisito implícito" del mixin.
- **Linearización.** Al componer múltiples mixins, se construye una cadena lineal de herencia. El orden importa. `super` siempre sube por esa cadena.
- **Comparación con herencia múltiple.** La herencia múltiple permite heredar de dos clases reales con estado propio, lo que genera el "problema del diamante". Los mixins evitan esto porque no tienen instancias propias y se linearizan.

**Lo clave:** en los mixins `super` funciona como en herencia simple, pero la cadena se construye en el momento de la composición. Esto es diferente a los traits donde no hay `super`.

**Bibliografía:**
- Bracha & Cook, *Mixin-based Inheritance* (obligatorio, salvo sección 5)
- Bak et al., *Mixins in Strongtalk*, sección 2 (opcional, buen resumen del modelo)

---

### 2.2 Traits

Un **trait** es un conjunto de métodos reutilizables que no puede tener instancias y, en el modelo original, no tiene estado propio.

Temas que entran:

- **Definición de trait:** provee métodos, puede requerir métodos (que la clase que lo use debe implementar). No es una clase, no se puede instanciar.
- **Flattening:** cuando una clase usa un trait, los métodos del trait se "aplanan" en la clase como si fueran definidos ahí directamente. No hay `super` entre clase y trait — los métodos del trait son métodos de la clase.
- **Comparación con mixins:** los mixins linearrizan (hay `super`), los traits aplanan (no hay `super`). Los traits explicitan los conflictos; los mixins los resuelven automáticamente por orden.
- **Roles según Ducasse et al.:**
  - **Trait:** unidad de reutilización de comportamiento puro
  - **Clase:** combina superclase + estado + traits + "glue methods" (métodos de pegamento que resuelven conflictos)
- **Operadores de composición:**

| Operador | Nombre | Qué hace |
|---|---|---|
| `+` | Suma | Combina dos traits; si hay conflicto, queda sin resolver (requiere intervención) |
| `▷` | Override | El trait de la izquierda tiene precedencia sobre el de la derecha |
| `−` | Exclusión | Elimina un método de la composición |
| `@` | Alias | Renombra un método en la composición |

- **Resolución de conflictos:** es **explícita**, no automática. Si dos traits definen el mismo método, la clase debe resolver el conflicto usando los operadores o definiendo su propio método que los sobreescribe.

**Nota importante del profe:** si en el parcial dicen "los traits no pueden / no deben tener estado", están ignorando los papers de Stateful Traits. El estado no era una restricción esencial, sino un desafío de implementación que luego se resolvió.

**Bibliografía:**
- Ducasse et al., *Traits: A Mechanism for Fine-grained Reuse* — lo más importante: sección 1 (intro), núcleo de sección 4 (hasta 4.1), y sección 4.3 para la "ecuación" de clases.
- TP 1 y su resolución.

---

### 2.3 Closures

Un **closure** es una expresión lambda abierta + el contexto léxico que la cierra.

Temas que entran:

- **Concepto de closure:** una lambda "abierta" tiene variables libres. Al capturar el contexto en el que fue creada, esas variables quedan ligadas. Eso es el closure.
- **Captura del contexto léxico y de `self`:** el closure captura las variables del entorno donde fue definido (no donde se ejecuta), incluyendo `self`.
- **Ligadura del `return`:**

| Tipo | `return` liga... | En Ruby |
|---|---|---|
| Full closure / proc | De manera **no local** (sale del método que lo contiene) | `proc`, bloques |
| Lambda | De manera **local** (sale solo del lambda) | `lambda` |

- **`LocalJumpError`:** ocurre cuando se invoca un `return` no local desde un proc/bloque, pero el método que lo contenía ya terminó (el contexto ya no existe).

- **Por qué importa la ligadura no local:** permite implementar estructuras de control como `if`, `while`, etc. de manera no primitiva (como métodos del lenguaje). Un `return` dentro del bloque de un `if` debe salir del método que contiene el `if`, no del `if` mismo.

**No hay bibliografía específica.** Vieron el tema en clase y en el framework de testing donde usaron retorno no local para detener la ejecución tras una falla de aserción sin usar excepciones.

---

## 3. Metaprogramación

### 3.1 Introducción a la metaprogramación

- **Metaprograma:** programa que razona sobre (o modifica) otro programa (o sobre sí mismo).
- **Reflexión:**
  - **Introspección:** leer información sobre el sistema en tiempo de ejecución (ej. `object.class`, `object.respond_to?(:foo)`)
  - **Intercesión:** modificar el comportamiento del sistema en tiempo de ejecución (ej. `define_method`, reemplazar el method lookup)
- **Modelo vs. metamodelo:**
  - **Modelo:** las clases y objetos de la aplicación (lo que programa el usuario)
  - **Metamodelo:** las clases que representan al propio lenguaje (Class, Module, Method, etc.)
- **Sistema reflexivo:** sistema que tiene una self-representation causalmente conectada consigo mismo. Ver artículo de Pattie Maes (secciones 1-4 obligatorias).
- **Distinción entre meta-niveles:** el nivel base (objetos del dominio) y el nivel reflexivo (objetos que representan al sistema). Consecuencias de juntarlos vs. separarlos:
  - **Juntos (ej. clase mezclada con metaclase):** más confuso, más difícil de razonar sobre el sistema
  - **Separados (ej. mirrors):** más claro, más modular, pero más verboso

**Bibliografía:**
- Maes, *Concepts and Experiments in Computational Reflection* — secciones 1-4 obligatorias (la 7 menciona propiedades deseables de un sistema reflexivo OO y el concepto de meta-objeto)
- Bracha & Ungar, *Mirrors* (opcional) — describe el concepto de mirror como separación explícita entre nivel base y meta

---

### 3.2 Aspectos estructurales de los metamodelos

- **Metamodelo de Ruby:**
  - Todo es un objeto, incluyendo las clases (instancias de `Class`)
  - `Class` hereda de `Module`, que hereda de `Object`
  - La API de metaprogramación permite introspección (`methods`, `instance_variables`, `superclass`) e intercesión (`define_method`, `method_missing`, clases abiertas)
  - **Distinción importante:** saber cuándo un mensaje es reflexivo y cuándo no; cuándo es introspección y cuándo es intercesión.
  - **Cosas para mejorar:** el metamodelo de Ruby tiene inconsistencias (ej. la jerarquía de singleton classes no es perfectamente paralela a la de clases regulares)

- **Singleton classes en Ruby:** cada objeto tiene asociada una singleton class (también llamada eigenclass) que es la clase "propia" de ese objeto. Los métodos definidos directamente sobre un objeto viven en su singleton class. Las **metaclases de Smalltalk** son un caso particular de singleton class (la singleton class de una clase).

- **Jerarquía paralela entre clases y metaclases:** si `B` hereda de `A`, entonces la metaclase de `B` debe heredar de la metaclase de `A`. Esta propiedad garantiza **compatibilidad ascendente**: los mensajes que entiende `A` como clase también los entiende `B` como clase (por herencia en el nivel meta). Ver Safe Metaclass Programming (secciones 1-2 obligatorias) para el problema de compatibilidad inter-nivel.

**Bibliografía:**
- Bouraqadi et al., *Safe Metaclass Programming* — secciones 1-2 obligatorias, 3 da ejemplos en distintos lenguajes, 4+ propone un modelo para resolver los problemas

---

### 3.3 Formas de representar y modificar el comportamiento dinámicamente

- **`Method` vs. `UnboundMethod`:**
  - `Method`: método ya ligado a un receptor específico. Se puede llamar directamente.
  - `UnboundMethod`: método sin receptor. Para usarlo hay que ligarlo (`bind`) a un objeto compatible.
  - `bind`/`unbind` tienen restricciones: solo se puede ligar un `UnboundMethod` a instancias de la clase donde fue definido (o subclases).
  - **Por qué existe esta distinción:** un método puede acceder a `self` y a variables de instancia; para ejecutarlo correctamente necesita saber *sobre qué objeto* opera.

- **`instance_exec` vs. `module_exec` / `class_exec`:**
  - `instance_exec`: evalúa un bloque en el contexto de una instancia (el `self` dentro del bloque es la instancia)
  - `class_exec` / `module_exec`: evalúa un bloque en el contexto de una clase o módulo (el `self` dentro del bloque es la clase/módulo)
  - **Por qué ambas:** a veces se quiere definir comportamiento en el contexto de un objeto cualquiera; otras veces específicamente en el contexto de una clase (ej. para usar `define_method` dentro del bloque).

- **Clases abiertas:** en Ruby se puede re-abrir una clase en cualquier momento y agregar/modificar métodos. Esto permite modificar el comportamiento de instancias ya existentes.

- **`define_method`:** permite definir un método dinámicamente, pasando un bloque o proc como cuerpo del método. Útil para metaprogramación donde el nombre del método se calcula en runtime.

**El parcial evalúa aspectos conceptuales, no sintácticos** (ej. por qué es necesario distinguir entre `Method` y `UnboundMethod`, qué pasaría si solo tuviéramos `instance_exec` pero no `class_exec`).

---

### 3.4 Comparación de metamodelos y maneras de entender la programación

**Smalltalk:**
- Casi todo se resuelve mediante envío de mensajes (incluyendo el `if`, que es un mensaje enviado a un booleano)
- Se programa en un **ambiente vivo**: el sistema corre mientras se lo modifica
- El concepto de metaclase está **reificado** (es un objeto de primera clase en el sistema)

**Java:**
- Muchas cosas rompen el paradigma: los números no son objetos, los arrays no son colecciones, los operadores aritméticos y `new` no son mensajes
- Metamodelo mínimo: las clases se representan solo con `java.lang.Class`
- Los métodos estáticos no son métodos reales (no hay polimorfismo en ellos, no tienen `this` como receptor real)

**Ruby:**
- Está "en el medio" del espectro
- Permite tanto usar la sintaxis (`class X; end`) como los elementos del paradigma (`X = Class.new`) para hacer lo mismo

**Distinciones conceptuales importantes:**

- **Lenguaje vs. sistema/ambiente de programación:** pensar el programa como artefacto lingüístico estático (texto) vs. como proceso en ejecución dinámico. Smalltalk privilegia lo segundo; la mayoría de los lenguajes modernos, lo primero.
- **Resolución paradigmática vs. sintáctica:** resolver un problema usando solo los conceptos del paradigma (objetos, mensajes) vs. agregar sintaxis especial (palabras reservadas, construcciones del lenguaje).
- **Sintaxis flexible:** Ruby permite expresar muchas cosas sin agregar keywords, aprovechando que casi todo es un mensaje.

**Lo que el profe evalúa:** que hayan desarrollado un **criterio propio** para distinguir estos enfoques y puedan posicionarse con argumentos.

---

## Apéndice: Comentarios sobre la subclasificación

### Dos relaciones que se confunden

En el dominio hay dos relaciones distintas:

**1. Relación de generalización** (entre conceptos/clases):
- Se corresponde con la relación **subclase-superclase**
- También llamada "subsunción" o "subordinación conceptual"
- En teoría de conjuntos: `ConceptoEspecífico ⊆ ConceptoGeneral`
- Ejemplo: "Caja de ahorros ⊆ Cuenta bancaria" (todo conjunto de cajas de ahorros está incluido en el conjunto de cuentas bancarias)

**2. Relación de ejemplificación** (entre una cosa y un concepto):
- Se corresponde con la relación **instancia-clase**
- También llamada "es un" o "ejemplificación"
- En teoría de conjuntos: `individuo ∈ Concepto`
- Ejemplo: "pepita ∈ Golondrina" (pepita pertenece al conjunto de las golondrinas)

### El vínculo entre ambas

```
ConceptoEspecífico ⊆ ConceptoGeneral
          ⟺
(∀ individuo ∈ ConceptoEspecífico) individuo ∈ ConceptoGeneral
```

Es decir: "Caja de ahorros ⊆ Cuenta bancaria" equivale a decir "toda caja de ahorros es una cuenta bancaria".

### La confusión frecuente

Decir "**es un**" (sin el "todo") confunde ambas relaciones: mezcla la relación entre una instancia y su clase con la relación entre dos clases. La heurística correcta para subclasificación es:

> "**Todo** X es un Y" — relación entre clases (generalización)

No:

> "X **es un** Y" — relación entre instancia y clase (ejemplificación)

### Consecuencia práctica

La subclasificación correcta no es solo "reusar código" — es afirmar que **todo objeto de la subclase se comporta como un objeto de la superclase** (principio de sustitución de Liskov). Si eso no se cumple, la subclasificación es un abuso técnico aunque haga andar el código.

---

## Lista de lecturas

### Obligatorias

| Paper | Qué leer | Por qué |
|---|---|---|
| Maes (1987), *Concepts and Experiments in Computational Reflection* | Secciones 1-4 obligatorias, resto recomendado | Definición de sistema reflexivo, reflexión procedural vs. declarativa, meta-objetos |
| Bouraqadi et al. (1998), *Safe Metaclass Programming* | Sección 2 obligatoria, 1-3 recomendado | Compatibilidad entre metaniveles, jerarquía paralela de metaclases |
| Bracha & Cook (1990), *Mixin-based Inheritance* | Todo salvo sección 5 | Modelo de mixins, comparación super vs. inner, linearización |
| Ducasse et al. (2006), *Traits: A Mechanism for Fine-grained Reuse* | Sección 4 (referencia principal), recomendable leer completo | Operadores de composición, resolución de conflictos, "ecuación" de las clases |

### Opcionales

| Paper | Por qué leerlo |
|---|---|
| Bracha & Ungar (2004), *Mirrors* | Concepto de mirror, separación nivel base / meta |
| Bak et al. (2002), *Mixins in Strongtalk* | Buen resumen del modelo de mixins (sección 2) |
| Bergel et al. (2006), *Stateful Traits* | Extensión de traits para tener estado interno |
| Tesone et al. (2020), *A new modular implementation for Stateful Traits* | Implementación en Pharo 7.0 vía metaclases especializadas |

---

## Checklist de repaso

Antes del parcial, verificá que podés responder:

- [ ] ¿Por qué se define "objeto" a partir del mensaje y no al revés?
- [ ] ¿Cuál es la diferencia entre `super` en Smalltalk y `inner` en Beta?
- [ ] ¿Por qué la herencia simple tiene limitaciones? ¿Qué problemas resuelven los mixins y los traits?
- [ ] ¿Qué es un mixin? ¿Cómo funciona `super` dentro de un mixin?
- [ ] ¿Qué es el flattening en traits? ¿Por qué no hay `super` entre clase y trait?
- [ ] ¿Cómo se resuelve un conflicto en traits? ¿Por qué es explícito y no automático?
- [ ] ¿Los traits pueden tener estado? ¿Por qué inicialmente no lo tenían?
- [ ] ¿Qué es un closure? ¿Cuál es la diferencia entre un proc y un lambda en Ruby respecto al `return`?
- [ ] ¿Cuándo ocurre un `LocalJumpError`?
- [ ] ¿Qué es un sistema reflexivo? ¿Qué significa "causalmente conectado"?
- [ ] ¿Qué diferencia hay entre introspección e intercesión?
- [ ] ¿Qué es una singleton class? ¿Cómo se relaciona con las metaclases de Smalltalk?
- [ ] ¿Por qué es necesaria la jerarquía paralela entre clases y metaclases?
- [ ] ¿Cuál es la diferencia entre `Method` y `UnboundMethod`? ¿Por qué existe esa distinción?
- [ ] ¿Cuál es la diferencia entre `instance_exec` y `class_exec`?
- [ ] ¿Qué significa resolver algo "de manera paradigmática" vs. "de manera sintáctica"?
- [ ] ¿Qué gana y qué pierde cada lenguaje (Smalltalk, Java, Ruby) respecto al paradigma?
