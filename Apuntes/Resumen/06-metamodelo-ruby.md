# Metamodelo de Ruby

## La jerarquía base

Ruby tiene una jerarquía de clases base que forma el metamodelo:

```
Module
  ↑
Class  ←────────────────────────────────┐
  ↑                                     │ (instancia de)
BasicObject ──(instancia de)──────────→ Class
  ↑
Kernel (módulo incluido en Object)
Object
  ↑
Guerrero  [con mixines: Atacante, Defensor]
  ↑
Espadachin

atila ──(instancia de)──→ Guerrero
```

**Referencias:**
- Flecha roja (→): instancia de
- Flecha azul (↑): superclase

---

## Relaciones clave

| Objeto | Instancia de | Superclase de |
|---|---|---|
| `atila` | `Guerrero` | — |
| `Guerrero` | `Class` | `Object` |
| `Espadachin` | `Class` | `Guerrero` |
| `Object` | `Class` | `BasicObject` |
| `BasicObject` | `Class` | — (raíz) |
| `Class` | `Class` | `Module` |
| `Module` | `Class` | `Object` |

Nota: `Class` es instancia de sí misma (`Class.class == Class`). Es uno de los puntos autoreferenciales del metamodelo.

---

## Kernel

`Kernel` es un módulo que está incluido en `Object`.
Provee métodos como `puts`, `p`, `require`, `lambda`, `proc`, `raise`, etc.
Por estar en `Object`, todos los objetos en Ruby tienen acceso a estos métodos.

---

## Los mixines en el metamodelo

Los módulos incluidos en una clase aparecen en `ancestors`:

```ruby
class Guerrero
  include Atacante
  include Defensor
end

Guerrero.ancestors
# => [Guerrero, Defensor, Atacante, Object, Kernel, BasicObject]
```

(el último `include` queda más cerca de la clase en la cadena)

---

## Method lookup en el metamodelo (sin singleton classes)

Cuando `atila` recibe un mensaje, Ruby recorre:
```
atila ──(instancia de)──→ Guerrero → Defensor → Atacante → Object → Kernel → BasicObject
```

Es decir: primero la clase, luego los mixines en orden inverso al de inclusión, luego la superclase, y así.

---

## ¿Por qué Class hereda de Module?

Porque una clase **es un módulo** con capacidades adicidas:
- Puede instanciarse (`new`)
- Tiene una superclase explícita

Un `Module` no puede instanciarse, pero puede incluirse. `Class` agrega `new` y `superclass` a `Module`.

---

## El metamodelo como "parte del modelo"

La gracia del metamodelo de Ruby (y de Smalltalk) es que **el metamodelo está escrito en sí mismo**.
`Class` es una instancia de `Class`. `Module` es una instancia de `Class`. Hay un ciclo autoreferencial que cierra el sistema.

Esto contrasta con Java, donde el metamodelo está fuera del lenguaje (la JVM lo implementa en C++).
Ver `09-comparacion-metamodelos.md`.
