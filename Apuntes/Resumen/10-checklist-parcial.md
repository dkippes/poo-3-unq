# Checklist para el Primer Parcial

Basado en la Guía de estudio oficial (2026s1). Marcá cada ítem cuando lo tengas claro.

---

## Bloque 1: Repaso y bases de OOP

### ¿Qué es un objeto?
- [ ] Definición desde el paradigma de mensajes (no desde "instancia de clase")
- [ ] Identidad, encapsulamiento, polimorfismo
- [ ] Por qué no es lo mismo "tiene tal atributo" que "responde a tal mensaje"

### Subclasificación / Herencia
- [ ] Qué significa que una clase sea subclase de otra
- [ ] Cómo funciona el method lookup con herencia simple
- [ ] Para qué sirve `super` y cómo funciona
- [ ] Limitaciones de la herencia simple (jerarquía forzada, anti-clases, repetición)
- [ ] Por qué la herencia múltiple genera el problema del diamante
- [ ] Qué no tiene sentido modelar con herencia (anti-patrón "es-un" forzado)

---

## Bloque 2: Esquemas no tradicionales de comportamiento

### Mixines
- [ ] Qué es un mixin (función de superclase a clase)
- [ ] Cómo funciona `include` en Ruby: linearización de la cadena
- [ ] Por qué el orden de los `include` importa (el último incluido queda primero en ancestors)
- [ ] Diferencia entre `include`, `prepend` y `extend`
- [ ] Cómo resuelve el problema del diamante (un módulo no aparece dos veces en ancestors)
- [ ] Cómo `super` funciona a través de mixines

### Traits
- [ ] Qué es flattening y por qué es distinto a la linearización de mixines
- [ ] Cómo se resuelven los conflictos entre traits (exclusión, alias, override)
- [ ] Diferencia entre traits stateless (original) y stateful traits (extensión posterior)
- [ ] En qué se parecen y en qué se diferencian los módulos de Ruby con los traits

### Closures
- [ ] Qué es una closure (código + contexto léxico capturado)
- [ ] Diferencia entre un bloque sintáctico y un `Proc` (objeto)
- [ ] Diferencia entre `proc` y `lambda` (semántica del `return` y la aridad)
- [ ] Qué es `self` dentro de un bloque (léxico: el del contexto de definición)
- [ ] `LocalJumpError`: cuándo ocurre y por qué

---

## Bloque 3: Metaprogramación

### Conceptos generales
- [ ] Qué es un metaprograma (programa que tiene a otro programa como dato)
- [ ] Diferencia entre introspección e intercesión
- [ ] Qué es un modelo y qué es un metamodelo
- [ ] Qué significa que un sistema sea reflexivo

### Aspectos estructurales del metamodelo de Ruby
- [ ] La jerarquía base: `BasicObject`, `Object`, `Kernel`, `Module`, `Class`
- [ ] Por qué `Class` hereda de `Module`
- [ ] Por qué `Class` es instancia de `Class`
- [ ] Para qué sirve `Kernel`

### Singleton classes
- [ ] Qué es una singleton class y para qué sirve
- [ ] Cómo acceder a ella (`obj.singleton_class`, `class << obj`)
- [ ] Que las clases también tienen singleton class (#NombreClase)
- [ ] La jerarquía paralela de singleton classes
- [ ] La regla de la superclase de una singleton class:
  - Si el objeto no es clase → su superclase es la clase del objeto
  - Si el objeto es clase → su superclase es la singleton class de su superclase
  - Excepción: `#BasicObject` apunta a `Class`

### Method lookup completo
- [ ] El camino para un objeto normal: `#obj → clase → mixines → superclase → ...`
- [ ] El camino para una clase: `#Clase → #SuperClase → ... → #BasicObject → Class → Module → Object → ...`
- [ ] "El camino desde la singleton class agrega un prefijo al camino desde la clase"

### Modificar comportamiento dinámicamente
- [ ] Clases abiertas: agregar métodos a cualquier clase en runtime
- [ ] `define_method`: definir métodos con closures
- [ ] `Method` vs `UnboundMethod`: ligado vs sin ligar a receptor
- [ ] `bind` y `unbind`: religar un método a otro objeto
- [ ] `instance_exec`: evaluar bloque con self = un objeto (métodos van a su singleton class)
- [ ] `class_exec`: evaluar bloque con self = una clase (métodos van a la clase)
- [ ] `method_missing` + `respond_to_missing?`

---

## Bloque 4: Comparación de metamodelos

- [ ] Metamodelo de Smalltalk: `Behavior → ClassDescription → Class/Metaclass`, sistema cerrado
- [ ] Metamodelo de Java: sólo `Class`, sin metaclases, métodos static no son mensajes
- [ ] Diferencia entre "lenguaje" (Ruby, Java) y "sistema" (Smalltalk)
- [ ] Por qué en Smalltalk el `if` es un envío de mensajes

---

## Tips del parcial (de la guía oficial)

- El parcial es **conceptual**, no memorístico. No te van a pedir que memorices código.
- Las preguntas son del estilo: "¿qué pasa cuando...?", "¿por qué este diseño tiene este problema?", "¿cómo resolvería X?"
- Tener claro el **método lookup** en cada situación es clave.
- La distinción "instancia de" vs "subclase de" aparece siempre.
- Saber explicar las diferencias entre los tres metamodelos (Ruby, Smalltalk, Java) con ejemplos.
