# Resumen para el Primer Parcial — POO3 UNQ

> Basado en: Guía de estudio FULL con respuestas, Modelo de parcial real, y resúmenes propios.
> El parcial es **conceptual**, no memorístico. Manejá los conceptos y el vocabulario con precisión.

---

## ÍNDICE RÁPIDO

1. [Objetos y Paradigma](#1-objetos-y-paradigma)
2. [Herencia y Subclasificación](#2-herencia-y-subclasificación)
3. [Mixines](#3-mixines)
4. [Traits](#4-traits)
5. [Closures y Bloques](#5-closures-y-bloques)
6. [Metaprogramación — Conceptos](#6-metaprogramación--conceptos)
7. [Metamodelo de Ruby y Singleton Classes](#7-metamodelo-de-ruby-y-singleton-classes)
8. [Intercesión — Modificar comportamiento](#8-intercesión--modificar-comportamiento)
9. [Metamodelos Comparados: Ruby vs Smalltalk vs Java](#9-metamodelos-comparados-ruby-vs-smalltalk-vs-java)
10. [Mirrors](#10-mirrors)
11. [Respuestas modelo para el parcial](#11-respuestas-modelo-para-el-parcial)

---

## 1. Objetos y Paradigma

### Definición de objeto (la que importa)

> **Un objeto es una representación esencial de una cosa de un dominio de interés.**

- **Cosa**: todo aquello de lo que podemos decir algo
- **Dominio de interés**: el conjunto de cosas relevantes para el problema
- **Representación esencial**: captura lo que hace que la cosa sea lo que es

La definición desde mensajes (la más pura):
- Un programa OO es **un conjunto de objetos que colaboran enviándose mensajes**
- Sólo hay objetos. Lo único que pueden hacer es **enviar y recibir mensajes**.
- Un objeto queda definido por su **protocolo** (conjunto de mensajes que entiende)

### Tres propiedades deducibles

| Propiedad | Qué es | Nota clave |
|---|---|---|
| **Identidad** | Cada objeto es único, distinguible de todos los demás | No confundir con *igualdad* (que es relativa y redefinible) |
| **Encapsulamiento** | La implementación es privada; nadie externo puede acceder | Permite cambiar el *cómo* sin afectar el *qué* |
| **Polimorfismo** | Distintos objetos responden al mismo mensaje cada uno a su manera | Consecuencia directa de definir por protocolo + privacidad |

### Encapsulamiento — Importante para el parcial

El encapsulamiento separa **qué** hace un objeto (su protocolo público) de **cómo** lo hace (su implementación interna).
Nadie externo puede ni debe conocer la implementación.
Cambiar la implementación no afecta a nadie, siempre que se respete el protocolo.

---

## 2. Herencia y Subclasificación

### Dos caras de la herencia

- **Cara técnica ("mecánica")**: reusar comportamiento mediante el *method lookup*.
- **Cara conceptual (modelado)**: representa una teoría del conocimiento sobre el dominio.

**Heurística para subclasificar**: "**todo X es un Y**" (en tanto X se comporte *como* Y, respetando su protocolo).
Atención: "todo X es un Y" cuantifica sobre *instancias*, habla de una relación entre *clases*. No confundir con "es un" (relación instancia-clase).

### Method Lookup con herencia simple

Cuando `atila` recibe un mensaje:
```
atila → Espadachin → Guerrero → Object → Kernel → BasicObject
```
Si no lo encuentra en ningún lado → `NoMethodError`

### super

`super` invoca la versión del mismo método que está más arriba en la cadena.
La subclase tiene el control: puede agregar, reemplazar o complementar el comportamiento del padre.

### Limitaciones de la herencia simple

| Problema | Descripción |
|---|---|
| **Jerarquía forzada** | Se mete comportamiento "demasiado arriba", poluyendo la superclase |
| **Anti-clases** | Una subclase que rompe el contrato (hereda y deshabilita). Viola Liskov. |
| **Código repetido** | Hay que duplicar features que no provienen de un ancestro común |
| **Herencia múltiple** | Posible solución conceptual, pero genera el **Problema del Diamante** |

### Problema del Diamante

```
       A
      / \
     B   C       B y C overridean el mismo método de A
      \ /
       D          ¿Cuál versión usa D? → Ambigüedad irresoluble
```

La herencia múltiple genera ambigüedad. Por eso la mayoría de los lenguajes la evitan.

---

## 3. Mixines

### Qué es un mixin

**Definición formal (Bracha & Cook)**: Un mixin es una **función de superclase a clase**.
Toma una clase `S` (la superclase) y devuelve una subclase de `S` con un cuerpo determinado.
Es una "subclase abstracta": no sabe quién será su superclase concreta al momento de definirse.

En Ruby: los módulos (`module`) actúan como mixines.

### Cómo funciona: Linearización

`include` **no copia métodos**: inserta una referencia al módulo en la cadena de búsqueda (ancestors).

```ruby
class Guerrero
  include Defensor   # se incluye primero → va más lejos en la cadena
  include Atacante   # se incluye último → queda más cerca
end

Guerrero.ancestors
# => [Guerrero, Atacante, Defensor, Object, Kernel, BasicObject]
```

**Regla**: el último incluido queda más cerca de la clase.

### Method lookup con mixines

```
atila → #atila → Guerrero → Atacante → Defensor → Object → Kernel → BasicObject
```

### include vs prepend vs extend

| Mecanismo | Posición en la cadena | Efecto |
|---|---|---|
| `include` | *después* de la clase | La clase tiene prioridad sobre el módulo |
| `prepend` | *antes* de la clase | El módulo tiene prioridad sobre la clase |
| `extend` | En la singleton class del receptor | Agrega métodos al objeto específico (o a la clase si se usa dentro de ella) |

### Ventaja sobre herencia múltiple

- **Sin diamante**: el mismo módulo no puede aparecer dos veces en la cadena (linearización determinista)
- El orden de inclusión define quién gana en caso de conflicto (sin ambigüedad)
- **Costo**: hay que encontrar un orden total adecuado (que puede no existir)

### super en mixines

`super` dentro de un mixin sube al siguiente en la cadena de ancestors (late binding).
La misma instancia de mixin puede hacer `super` hacia distintas superclases dependiendo de en qué clase se incluya.

---

## 4. Traits

### Qué es un trait

**Definición (Ducasse et al.)**: Un trait es un **conjunto de métodos**, separado de toda jerarquía de clases.
- Provee métodos y puede *requerir* métodos (que la clase que lo usa debe proveer)
- **Sin estado**: no tienen variables de instancia (aunque luego se extendió con Stateful Traits)

### Flattening — la propiedad clave

> Un método no sobre-escrito que viene de un trait tiene **exactamente la misma semántica** que si estuviera escrito directamente en la clase.

Consecuencias:
- El trait **no introduce un nivel nuevo** en la cadena de method lookup
- `super` dentro de un trait apunta a la **superclase de la clase que lo usa**, no a otro trait
- Los traits son **divorciados de la jerarquía**

### Resolución de conflictos

Si dos traits definen el mismo método, la clase **debe** resolver explícitamente:

| Operación | Efecto |
|---|---|
| **Override** (`▷`) | La clase define el método y pisa al del trait |
| **Exclusión** (`−`) | Quita el método de uno de los traits |
| **Alias** (`→`) | Da otro nombre a un método del trait para evitar colisión |

### Ecuación de las clases con traits

```
Clase = Superclase + Estado + Traits + Glue methods
```

- **Superclase**: herencia simple para jerarquía conceptual
- **Estado**: variables de instancia (viven en la clase, no en el trait)
- **Traits**: unidades de reuso
- **Glue methods**: conectan traits entre sí, adaptan métodos, resuelven conflictos

### Mixines vs Traits — diferencias clave

| | Mixin (Ruby) | Trait |
|---|---|---|
| Unidad | Subclase abstracta | Conjunto de métodos |
| Composición | Lineal / ordenada | No ordenada (varios a la vez) |
| Conflictos | Resueltos por el orden (silencioso) | El cliente los resuelve **explícitamente** |
| Estado | Puede tener | Sin estado (originalmente) |
| `super` | Sube en la cadena del mixin | Apunta a la superclase de la clase |
| Jerarquía | Se inserta en la cadena | Divorciados de la jerarquía (flattening) |

### Stateful Traits

La definición original era stateless. Trabajos posteriores (Bergel et al.) extendieron los traits con estado.
**Afirmar que "los traits no pueden tener estado" es un error**: los Stateful Traits existen y son válidos.
El cliente siempre retiene el control de la composición.

---

## 5. Closures y Bloques

### Qué es una closure

> Una closure es una **expresión lambda "abierta"** junto con el **contexto léxico que la cierra**.

Componentes:
1. El **código** a ejecutar
2. El **contexto léxico** (variables libres capturadas del entorno donde fue definida)

Una lambda es "abierta" cuando hace referencia a variables que no son sus parámetros. La closure las "cierra" capturando esas variables del entorno.

### Bloques en Ruby

```ruby
[1, 2, 3].each { |x| puts x }    # bloque sintáctico
```

El bloque sintáctico **no es un objeto**. Para tratarlo como objeto → convertirlo en `Proc`.

### Proc vs Lambda

```ruby
mi_proc   = Proc.new { |x| x * 2 }
mi_lambda = lambda { |x| x * 2 }
```

| Característica | Proc | Lambda |
|---|---|---|
| `return` | **No-local**: sale del método contenedor | **Local**: sale sólo del lambda |
| Aridad | No valida cantidad de argumentos | Valida cantidad de argumentos |
| `lambda?` | false | true |

### Return no-local del Proc

```ruby
def metodo
  mi_proc = Proc.new { return "desde el proc" }
  mi_proc.call
  "nunca llega acá"
end
# => "desde el proc"
```

Si el Proc **sobrevive al método** que lo creó y se llama después → `LocalJumpError` (no hay a dónde volver).

### self en un bloque

El `self` dentro de un bloque es el `self` del contexto donde fue **definido** (captura léxica), no donde se ejecuta.
Esto cambia si se usa `instance_exec` o `class_exec`.

### Método vs Closure — relación conceptual

Un método (`def`) **no captura el contexto léxico** (no puede acceder a variables locales de afuera).
Sin embargo, un `Method` (objeto que representa a un método) **sí se parece a una closure** porque:
- Está **ligado a un receptor** (`self` ya está definido)
- Puede ejecutarse con `.call`
- Captura el valor de `self` (el objeto al que pertenece)

La diferencia: una closure captura variables libres arbitrarias; un `Method` sólo captura `self`.

---

## 6. Metaprogramación — Conceptos

### Qué es un metaprograma

> Un metaprograma es un **programa que genera, manipula o define a otros programas**.

Ejemplos: compiladores, frameworks de testing, ORMs, linters, refactoring automático.

**Regla de oro**: No usar metaprogramación para resolver problemas que no pertenecen al dominio de la programación.

### Reflexión

> La reflexión es la capacidad de un programa de **razonar sobre sí mismo** (y opcionalmente modificarse) en tiempo de ejecución.

Tiene dos modos:

| Modo | Qué hace | Ejemplos en Ruby |
|---|---|---|
| **Introspección** | *Leer* la auto-representación | `atila.class`, `respond_to?`, `ancestors`, `methods` |
| **Intercesión** | *Modificar* la auto-representación | `define_method`, reabrir clases, `singleton_class` |

### Modelo vs Metamodelo

| Nivel | Qué contiene | Ejemplo |
|---|---|---|
| **Modelo** | Los objetos del dominio | `atila`, `muralla`, `espadachin` |
| **Metamodelo** | Los conceptos del lenguaje | `Class`, `Module`, `Method`, `superclass` |

Cuando programamos → resolvemos el **modelo**.
Cuando metaprogramamos → operamos sobre el **metamodelo**.

### Sistema reflexivo

Un sistema reflexivo tiene una **auto-representación** (*self-representation*) que está **causalmente conectada** con el sistema:
- Si la representación cambia → el sistema cambia en consecuencia
- Si el sistema cambia → la representación lo refleja

Esto implica dos propiedades:
1. El sistema siempre tiene una representación **precisa** de sí mismo
2. Estado y computación están siempre **en conformidad** con esa representación

### ¿Leer archivos = sistema reflexivo?

**No es suficiente** que un programa pueda leer/escribir sus propios archivos fuente para ser reflexivo.
Lo que importa es la **conexión causal**: si modifico la representación, ¿el sistema cambia su comportamiento?
Leer un archivo de texto no modifica el comportamiento del programa en ejecución.
Un sistema reflexivo real (como Smalltalk) sí tiene esa conexión causal directa.

---

## 7. Metamodelo de Ruby y Singleton Classes

### La jerarquía base

```
BasicObject   ← raíz de la herencia (superclase es nil)
  ↑
Object        ← raíz práctica
  ↑
...tu clase...

Kernel        ← módulo incluido en Object (puts, lambda, proc, require...)
Module        ← representa módulos
  ↑
Class         ← subclase de Module, representa clases
```

- Toda clase es instancia de `Class`
- `Class` es instancia de sí misma (`Class.class == Class`)
- `Class` hereda de `Module` porque una clase *es* un módulo con capacidades extra (`new`, `superclass`)

### Singleton Classes (Eigenclasses / Metaclases del objeto)

> Todo objeto en Ruby tiene su propia clase privada llamada **singleton class** (#NombreObjeto).

```ruby
def atila.habilidad_especial   # va a la singleton class de atila
  "Conquista total"
end

atila.singleton_class          # acceso explícito
class << atila; end            # bloque para trabajar dentro de ella
```

**Para qué sirven**: agregar métodos a un objeto individual sin afectar a todos los de su clase.

### Singleton class de las CLASES

Las clases también son objetos → también tienen singleton class.
La singleton class de `Guerrero` (#Guerrero) contiene los **métodos de clase**:

```ruby
class Guerrero
  def self.crear(nombre)   # def self.x → define en #Guerrero
    new(nombre)
  end
end
```

### Diagrama completo del metamodelo de Ruby (notación de la cátedra)

La cátedra usa esta notación (igual que en las slides):
- Caja azul: clase normal
- Caja verde: singleton class
- `*` sobre una flecha roja: "apunta a la clase Class" (instancia de Class, sin dibujar la flecha larga)
- `...` verde: más singleton classes (la torre continúa)
- Flecha azul: superclase
- Flecha roja: instancia de
- Flecha verde: singleton class de

```
                                Module ──────────→ #Module → ...
                                  ↑         ←──────────↑
                                Class ←─────→  #Class  → ...
          *                       ↑
BasicObject ─────────────────→ #BasicObject → ...
  ↑ (Kernel incluido)              ↑
Object ──────────────────────→ #Object ──→ ...
  ↑                                ↑
Guerrero ────────────────────→ #Guerrero → ...
(Atacante, Defensor)               ↑
  ↑                            #Espadachin → ...
Espadachin ──────────────────→ 

atila ──→ #atila → ...
           ↑ (superclase de #atila = Guerrero)
```

**Leyendo el diagrama completo (como lo muestra la cátedra):**

| Objeto | Instancia de | Superclase de (herencia) | Singleton class |
|---|---|---|---|
| `atila` | `Guerrero` | — | `#atila` |
| `Guerrero` | `Class` (`*`) | `Object` | `#Guerrero` |
| `Espadachin` | `Class` (`*`) | `Guerrero` | `#Espadachin` |
| `Object` | `Class` (`*`) | `BasicObject` (con Kernel) | `#Object` |
| `BasicObject` | `Class` (`*`) | `nil` | `#BasicObject` |
| `Class` | `Class` (se apunta a sí misma) | `Module` | `#Class` |
| `Module` | `Class` (`*`) | `Object` | `#Module` |
| `#Guerrero` | `Class` (`*`) | `#Object` | `...` |
| `#Object` | `Class` (`*`) | `#BasicObject` | `...` |
| `#BasicObject` | `Class` (`*`) | `Class` ← **EXCEPCIÓN** | `...` |
| `#atila` | `Class` (`*`) | `Guerrero` | `...` |

### Method Lookup completo

**Regla**: La singleton class agrega un **prefijo estricto** al camino que arranca en la clase del objeto.

**Para un objeto normal (`atila` recibe un mensaje):**
```
atila → #atila → Guerrero → Defensor → Atacante → Object → Kernel → BasicObject
```
(#atila tiene superclase Guerrero, no #Guerrero, porque atila no es una clase)

**Para una clase (`Guerrero` recibe un mensaje, ej: `Guerrero.new`):**
```
Guerrero → #Guerrero → #Object → #BasicObject → Class → Module → Object → Kernel → BasicObject
```
(Los `#X` tienen como superclase al `#` de su superclase)

**Formulación simplificada (Clase 3):**
> "El camino a partir de la singleton class **agrega un prefijo estricto** al camino a partir de la respuesta a `class`"

Sin singleton class: `Guerrero → Class → Module → Object → Kernel → BasicObject`
Con singleton class: `#Guerrero → #Object → #BasicObject → Class → Module → Object → Kernel → BasicObject`

### Regularidades del metamodelo

1. **Todo objeto tiene singleton class** (incluidas las clases, y las singleton classes también tienen la suya → la "torre" con `...`)

2. **Superclase de la singleton class**:
   - Si el objeto **NO es una clase** → su superclase es la **clase del objeto**
     - `#atila` tiene superclase `Guerrero` (no `#Guerrero`)
   - Si el objeto **ES una clase** → su superclase es la **singleton class de su superclase**
     - `#Guerrero` tiene superclase `#Object`
     - `#Object` tiene superclase `#BasicObject`
     - `#Espadachin` tiene superclase `#Guerrero`
   - **Excepción**: `#BasicObject` tiene superclase `Class` (rompe el patrón para cerrar el sistema)

---

## 8. Intercesión — Modificar comportamiento

### Clases abiertas (Open Classes)

En Ruby toda clase puede reabrirse y modificarse en cualquier momento:

```ruby
class String
  def palindrome?
    self == self.reverse
  end
end
"racecar".palindrome?  # => true
```

Como `include` hace referencia (no copia), al redefinir un método en runtime **todas las instancias ya creadas** ven el nuevo comportamiento inmediatamente.

**Monkey patching**: modificar clases que no son propias (ej: String, Integer). Poderoso pero peligroso.

### define_method

Permite definir métodos dinámicamente con un bloque (closure):

```ruby
[:atacar, :defender].each do |accion|
  define_method("puede_#{accion}?") { true }
end
```

A diferencia de `def`: captura el contexto léxico y el nombre puede calcularse en runtime.

### Method vs UnboundMethod

| | `Method` | `UnboundMethod` |
|---|---|---|
| Estado | Ligado a un receptor (`self` definido) | No ligado (sin receptor) |
| Ejecutar | `.call` directamente | Primero `.bind(objeto)`, luego `.call` |
| Obtener | `objeto.method(:nombre)` | `Clase.instance_method(:nombre)` |

**Restricciones de bind**:
- Si proviene de una **clase**: sólo se puede ligar a instancias de esa clase o subclases
- Si proviene de un **módulo**: no hay restricciones

**Por qué existe esta distinción**: permite manipular, transplantar y reutilizar comportamiento de forma controlada, manteniendo la seguridad de que el objeto receptor cumpla las precondiciones.

### instance_exec vs class_exec

| Método | `self` dentro | Métodos definidos en... |
|---|---|---|
| `objeto.instance_exec` | el objeto | singleton class del objeto |
| `Clase.class_exec` | la clase | la clase (métodos de instancia para todas las instancias) |

**Por qué existen ambas**: son niveles distintos del metamodelo. Sin `class_exec` no podríamos agregar métodos de instancia a una clase de forma dinámica y limpia.

### method_missing

Si ningún método se encuentra en el lookup, Ruby llama a `method_missing`:

```ruby
def method_missing(nombre, *args)
  if nombre.to_s.start_with?("puede_")
    true
  else
    super  # siempre llamar super si no manejamos el mensaje
  end
end

def respond_to_missing?(nombre, include_private = false)
  nombre.to_s.start_with?("puede_") || super
end
```

**Siempre** redefinir `respond_to_missing?` junto con `method_missing`.

---

## 9. Metamodelos Comparados: Ruby vs Smalltalk vs Java

### Smalltalk — "Todo es mensaje / Sistema vivo"

- Todo se resuelve enviando mensajes, **incluso el `if`** (`ifTrue:ifFalse:` es un mensaje a un booleano)
- Se programa en un **ambiente vivo**: no hay distinción nítida entre compilación y ejecución
- Las clases tienen **metaclases reificadas** (de primera clase): la metaclase de `C` es `C class`
- Metamodelo: `Behavior → ClassDescription → Class/Metaclass`
- **Gana**: máxima uniformidad, reflexión potente, expresividad
- **Pierde**: puede ser menos familiar/eficiente, exige todo el ambiente

### Java — "Muchas cosas fuera del paradigma"

- Los números son primitivos (no objetos), los arrays no son colecciones, `new` no es un mensaje
- Metamodelo mínimo: **sólo `Class`**, sin metaclases automáticas por cada clase
- Los métodos `static` **no son mensajes**: no hay `this`, no hay polimorfismo sobre ellos
- **Gana**: simplicidad, performance, tipado estático, familiaridad
- **Pierde**: uniformidad, expresividad, poder reflexivo; rompe el paradigma en varios puntos

### Ruby — "En el medio del espectro"

- Casi todo es objeto, hay reflexión potente (como Smalltalk)
- Conserva sintaxis y construcciones más tradicionales
- No reifica la metaclase: usa **singleton classes** (más generales/anónimas)
- **Gana**: pragmatismo, familiaridad sin abandonar la uniformidad
- **Pierde**: algo de la coherencia total de Smalltalk

### Tabla comparativa

| | Ruby | Smalltalk | Java |
|---|---|---|---|
| Clases como objetos | Sí | Sí | Sí (`Class`) |
| Metaclases automáticas | Sí (singleton classes) | Sí (`X class`) | No |
| Métodos de clase como mensajes | Sí | Sí | No (static) |
| Sistema cerrado sobre sí mismo | Parcialmente | Completamente | No |
| Modificable en runtime | Sí (open classes) | Sí (sistema vivo) | Muy limitado |
| Modelo "lenguaje" vs "sistema" | Lenguaje | Sistema | Lenguaje |

### Lenguaje vs Sistema — distinción importante

- **Lenguaje**: el programa es un **texto estático** que se transforma y ejecuta. Hay fases separadas (parse, compile, run). Ej: Java, Ruby.
- **Sistema**: el programa es un conjunto de **objetos vivos** que se modifican continuamente. No hay fases separadas. Ej: Smalltalk.

Esta distinción explica por qué pensamos "orientados a clases" (el texto) en lugar de "orientados a objetos" (la ejecución).

---

## 10. Mirrors

### Los tres principios de diseño (Bracha & Ungar)

Un sistema es *mirror-based* si sus facilidades meta-nivel respetan:

1. **Encapsulamiento**: las facilidades meta-nivel deben encapsular su implementación. El código cliente no debe depender de detalles del sistema reflexivo.

2. **Estratificación** (*Stratification*): las facilidades meta-nivel deben estar **separadas** de la funcionalidad de nivel base. Se las puede agregar o quitar. Una app desplegada puede prescindir de ellas.

3. **Correspondencia ontológica**: la ontología de las facilidades meta-nivel debe corresponderse con la del lenguaje que manipulan.

### Mirror = objeto meta separado

Un **mirror** es un objeto del nivel meta a través del cual se reflexiona sobre un objeto base, **sin mezclar ambos niveles**.

En Ruby (sin mirrors): el propio objeto tiene los mensajes reflexivos (`atila.class`, `atila.respond_to?...`).
Con mirrors: hay un objeto separado `Mirror(atila)` que expone esas capacidades.

### ¿Qué se gana y qué se pierde?

**Con reflexión en el propio objeto (Ruby)**:
- ✅ Más conveniente: `obj.class` en lugar de `Mirror.of(obj).getMirror.getClass`
- ✅ Sintaxis más simple y directa
- ❌ Mezcla el nivel base con el meta (viola estratificación)
- ❌ Los métodos reflexivos "contaminan" el protocolo del objeto de dominio
- ❌ No se pueden separar/distribuir fácilmente las capacidades meta

**Con mirrors (objeto separado)**:
- ✅ Separación clara de niveles (principio de estratificación)
- ✅ Las capacidades meta pueden implementarse de forma independiente (incluso remotamente)
- ✅ Se pueden agregar/quitar sin tocar el objeto base
- ❌ Más verboso y burocrático de usar
- ❌ Introduce indirección

---

## 11. Respuestas modelo para el parcial

### Pregunta 1 — Method lookup con mixines/traits

#### Estructura del diagrama

```
Mixin M:       bar → "M",    foo → "U"
Clase A:       foo → "O",    bar → "A"
Trait T:       bar → "T",    foo → super + "!"
Clase B:       bar → super + foo    (B hereda A, B include M, B usa T)
```

- `B` **hereda** de `A`
- `B` **incluye** el mixin `M` (la flecha de B se bifurca: rama izquierda → M, rama superior → A)
- `B` **usa** el trait `T` (con flattening — los métodos de T quedan "dentro" de B)
- `B` tiene su propio `bar` = `super + foo` → **pisa** el `bar` de T
- `B` **no define `foo`** → hereda `foo` de T por flattening: `T#foo = super + "!"`

**Cadena de ancestors de B:**
```
B → M → A → Object → Kernel → BasicObject
```
(M incluido por B queda entre B y A; T es flattened, no aparece en ancestors)

#### Trace paso a paso de `B.new.bar`

**1. Se llama `B#bar`** = `super + foo`

**2. Evaluar `super`:**
- `super` busca `bar` por encima de B en la cadena de ancestors
- Cadena: `B → M → A → ...`
- El primero que tiene `bar` es **M** → `M#bar` = `"M"`
- ∴ `super` = `"M"`

**3. Evaluar `foo`:**
- Se busca `foo` empezando desde B (el `self`)
- B no define `foo` propio, pero T (flattened en B) sí: `T#foo` = `super + "!"`
- Se ejecuta `T#foo`:
  - `super` en T#foo → con flattening, `super` busca `foo` por encima de B en la cadena de ancestors
  - Cadena: `B → M → A → ...`
  - El primero que tiene `foo` es **M** → `M#foo` = `"U"`
  - ∴ `T#foo` = `"U"` + `"!"` = `"U!"`
- ∴ `foo` = `"U!"`

**4. Resultado:**
```
B#bar = super + foo = "M" + "U!" = "MU!"
```

> **`B.new.bar` retorna `"MU!"`**

#### Por qué están "O" y "A" en el diagrama (el truco del ejercicio)

`A#foo` = `"O"` y `A#bar` = `"A"` **nunca se llaman** en este trace.
El punto es que M (incluido por B) intercepta ambas búsquedas antes de llegar a A.
Si el alumno no ve que B incluye M y asume que super va directo a A, responde `"AO!"` — respuesta incorrecta.

#### Claves conceptuales para explicar

1. **B incluye M**: M queda entre B y A en la cadena → `super` en B#bar encuentra M#bar antes que A#bar.
2. **Flattening del trait**: T no agrega un nivel en la cadena → T#foo está "dentro" de B.
3. **`super` dentro de T#foo**: busca `foo` por encima de B en la cadena → encuentra M#foo = "U".
4. **B pisa T#bar**: B define su propio `bar`, que tiene prioridad sobre el de T.

**Proceso general a explicar en el parcial:**
1. Identificar la clase del receptor y su cadena de ancestors
2. Con traits: el trait hace flattening — sus métodos se comportan como si estuvieran en la clase
3. `super` dentro de un mixin → sube al siguiente en la cadena linearizada (late binding)
4. `super` dentro de un trait → apunta a la superclase de la clase que usa el trait (no a otro trait)
5. La clase tiene prioridad sobre los métodos provistos por traits

---

### Pregunta 2 — Métodos como closures

**En Ruby, los métodos definidos con `def` NO capturan el contexto léxico:**

```ruby
x = 1
def m1
  # x no existe acá
end
```

**Sin embargo, un `Method` (objeto) se parece a una closure porque:**

- Está **ligado a un receptor** (`self` está definido)
- Se puede pasar como objeto, almacenar, invocar con `.call`
- Captura el valor de `self` (el objeto al que está ligado)

**Similitudes con una closure:**
- Ambos son unidades de comportamiento encapsuladas
- Ambos tienen un contexto de ejecución definido (`self` en el método, variables libres en la closure)
- Ambos se pueden ejecutar en otro momento, en otro contexto

**Diferencia clave:**
- Una closure captura **variables libres arbitrarias** del entorno léxico
- Un `Method` sólo captura **`self`** (no tiene acceso a variables locales del contexto de definición)
- Un método se liga a `self` dinámicamente al recibirlo; una closure captura `self` léxicamente

---

### Pregunta 3 — ¿Leer archivos = reflexivo?

**No es suficiente.**

Un sistema reflexivo requiere que la **auto-representación esté causalmente conectada** con el sistema:
- Cambiar la representación → el sistema cambia
- El sistema cambia → la representación se actualiza

Leer/escribir archivos de texto cumple la parte estructural (el programa puede "ver" su propio texto), pero **no garantiza la conexión causal**:
- Escribir en el archivo fuente no modifica el comportamiento del programa *en ejecución*
- El archivo y el sistema en ejecución son dos cosas distintas

Para ser realmente reflexivo necesita algo como Smalltalk: donde modificar una clase *en el sistema* modifica inmediatamente el comportamiento de todos los objetos.

**Conclusión**: es un sistema que puede *leer su representación* pero no necesariamente *actuar sobre ella*. No alcanza para ser reflexivo en el sentido formal de Maes.

---

### Pregunta 4 — Encapsulamiento en Ruby con send/instance_variable_get

**¿Hay encapsulamiento en Ruby a pesar de `send` y `instance_variable_get`?**

**Sí, el encapsulamiento se mantiene**, porque:

El encapsulamiento es un principio de **diseño**, no una barrera técnica absoluta.
El hecho de que *se pueda* violar no significa que *no exista*.

- El protocolo público del objeto sigue siendo el mecanismo normal y esperado de interacción
- `send` con métodos privados y `instance_variable_get` son **capacidades reflexivas** que existen para metaprogramación, debugging y herramientas del lenguaje
- Usarlas para acceder a implementación privada es deliberadamente ir *"por fuera del sistema"*, del mismo modo que en C se puede castear punteros para romper abstracciones

El encapsulamiento protege contra el **acoplamiento accidental**, no contra la **violación deliberada**.

**Si se argumenta que no hay encapsulamiento**: una alternativa sería usar el enfoque de **mirrors** (Bracha & Ungar): separar las capacidades reflexivas a un objeto meta separado que requiera autorización explícita, de modo que el protocolo del objeto base no quede contaminado con métodos reflexivos.

---

### Pregunta 5 (bonus) — Introspección vs Intercesión

| | Definición | Ejemplos en Ruby |
|---|---|---|
| **Introspección** | Capacidad de *leer/observar* la propia estructura sin modificarla | `obj.class`, `respond_to?`, `ancestors`, `methods`, `instance_variables` |
| **Intercesión** | Capacidad de *modificar* la propia estructura o comportamiento | `define_method`, reabrir clase, `singleton_class`, `remove_method` |

Ambas son formas de **reflexión**. La introspección es la parte de lectura; la intercesión es la de escritura.

---

### Pregunta 6 (bonus) — Mirrors: análisis de diseño

**Reflexión en el propio objeto (Ruby style)**:
- Se gana: comodidad, sintaxis directa, menos código
- Se pierde: mezcla niveles (base y meta en el mismo objeto), viola estratificación, los métodos reflexivos "contaminan" el protocolo del dominio

**Enfoque mirrors (objeto separado)**:
- Se gana: separación limpia de niveles, las capacidades meta pueden ser distribuidas/remotas, se pueden agregar o quitar sin tocar el objeto base
- Se pierde: verbosidad, indirección, más código para operaciones simples

**Trade-off central**: conveniencia vs. pureza de diseño y separación de responsabilidades.

---

### Pregunta 7 (bonus) — Programa como texto estático vs sistema vivo

**Visión A — Artefacto lingüístico (Java, la mayoría)**:
- El programa es un texto que describe un cómputo. Hay fases separadas: escritura, compilación, ejecución.
- Más familiar, predecible, fácil de razonar estáticamente.
- Resuelve mejor: tipado estático, análisis en tiempo de compilación, tooling convencional.

**Visión B — Sistema vivo (Smalltalk, Lisp)**:
- El programa es un conjunto de objetos que se modifican continuamente. No hay separación nítida entre "el programa" y "su ejecución".
- Permite reflexión completa, modificación en tiempo de ejecución, experimentación interactiva.
- Resuelve mejor: sistemas que se adaptan, herramientas que inspeccionan y modifican el propio sistema.

**Lo que A resuelve mejor que B**: análisis estático, seguridad de tipos, tooling convencional, performance predecible.
**Lo que B resuelve mejor que A**: reflexión real, sistemas auto-modificables, entornos de desarrollo interactivos.

---

### Pregunta 8 (bonus) — JavaScript `this` y Method/UnboundMethod

En Ruby existe la distinción **`Method`** (ligado a un receptor) vs **`UnboundMethod`** (sin receptor).
Un `Method` siempre sabe quién es `self`.

En JavaScript, **`this` se liga dinámicamente** al momento de la llamada, no al momento de la definición.
Esto significa que una función puede ser llamada con distintos `this` dependiendo de cómo se la invoca.

**Si JavaScript hiciera la distinción**:
- Una función ligada a un objeto (equivalente a `Method`) siempre sabría su `this` → menos confusión
- Una función no ligada (equivalente a `UnboundMethod`) requeriría un paso explícito de ligadura antes de ejecutarse
- Esto haría explícito lo que hoy es implícito y fuente de bugs

**¿Se habrían evitado todas las confusiones?** Parcialmente. El problema de `this` en JavaScript tiene múltiples causas (llamadas por referencia, callbacks, etc.), y la distinción Method/UnboundMethod resolvería algunos pero no todos los casos (p.ej., no resuelve el problema de `this` dentro de callbacks de forma automática, a menos que se adopte ligadura léxica como en arrow functions).

---

## CHECKLIST RÁPIDO ANTES DEL PARCIAL

- [ ] Sé explicar qué es un objeto sin mencionar "clase" ni "atributos"
- [ ] Sé trazar el method lookup paso a paso para cualquier jerarquía con mixines
- [ ] Sé la diferencia entre `include`, `prepend` y `extend`
- [ ] Sé qué es flattening y en qué se diferencia de la linearización
- [ ] Sé la diferencia entre `proc` y `lambda` (el `return`)
- [ ] Sé qué es y cuándo ocurre `LocalJumpError`
- [ ] Sé la diferencia entre introspección e intercesión con ejemplos
- [ ] Sé explicar qué requiere un sistema para ser realmente reflexivo (conexión causal)
- [ ] Sé el camino completo del method lookup con singleton classes
- [ ] Sé las dos regularidades de las singleton classes (superclase si es objeto vs si es clase)
- [ ] Sé la diferencia entre `Method` y `UnboundMethod` y las restricciones de `bind`
- [ ] Sé la diferencia entre `instance_exec` y `class_exec`
- [ ] Sé comparar Ruby, Smalltalk y Java en sus metamodelos
- [ ] Sé explicar "lenguaje" vs "sistema" y dar ejemplo de cada uno
- [ ] Sé los tres principios de Mirrors y qué se gana/pierde
