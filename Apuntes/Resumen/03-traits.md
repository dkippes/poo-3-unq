# Traits

## Qué es un Trait

Un **trait** es una unidad de reutilización de comportamiento, similar a un mixin pero con diferencias clave.

Definición formal (Schärli et al., 2003): un trait es un conjunto de métodos que puede componerse con otros traits o clases, con un mecanismo explícito de resolución de conflictos.

---

## Diferencias clave con Mixines

| Característica | Mixin (Ruby) | Trait |
|---|---|---|
| Estado | Puede tener variables de instancia | Originalmente sin estado (stateless) |
| Conflictos | Resuelto por linearización (último gana) | Resolución explícita por el programador |
| Composición | `include` inserta en la cadena | `flattening`: los métodos se copian a la clase |
| Orden de inclusión | Importa (afecta el method lookup) | No importa (el resultado es el mismo) |

---

## Flattening

La característica más importante de los traits es el **flattening semántico**:
el resultado de componer un trait con una clase es **equivalente** a haber definido esos métodos directamente en la clase.

```
clase C con trait T  ≡  clase C con los métodos de T copiados directamente
```

Esto implica:
- No hay una cadena de herencia adicional
- El `super` dentro de un trait siempre se refiere a la superclase de la clase que lo usa, no al trait

---

## Resolución de conflictos

Si dos traits `T1` y `T2` definen el mismo método `foo`, la clase que los compone **debe** resolver el conflicto explícitamente:

1. **Exclusión**: ignorar el método de uno de los traits
2. **Alias**: renombrar uno de los métodos para evitar la colisión
3. **Override**: definir el método directamente en la clase (pisa a los de los traits)

```
Clase C usa T1 + T2:
  - alias T1#foo como foo_from_t1
  - excluir T2#foo
  - definir foo propio
```

Esto es más explícito que la linearización de los mixines, donde el orden de inclusión decide silenciosamente.

---

## Traits con estado (Stateful Traits)

La definición original de Schärli et al. era stateless.
Trabajos posteriores (Bergel, Ducasse, et al.) extendieron los traits para soportar **estado**:
- El trait puede declarar variables de instancia
- La clase que lo usa "hereda" esas variables
- Se mantiene el flattening: es como si las variables estuvieran definidas en la clase

En Ruby, los módulos (que actúan como mixines) **ya pueden tener estado**, así que en la práctica Ruby se acerca más a los stateful traits que a los traits puros.

---

## Cuándo usar traits vs mixines

- **Mixin**: cuando querés herencia real (con lookup chain) y no te importa el orden
- **Trait**: cuando querés composición explícita sin cadena de herencia, y necesitás resolver conflictos de forma clara

En Ruby puro se usan módulos (mixines). Los traits son un concepto teórico que algunos lenguajes implementan de forma más pura (Scala los llama traits, Rust usa traits también aunque son más parecidos a interfaces).

---

## Ejemplo conceptual

```
Trait Atacante: { ataque, atacar_a(otro) }
Trait Defensor: { defensa, recibir_daño(d) }

Clase Guerrero:
  usa Atacante + Defensor
  (sin conflictos → se hace flattening directo)

Clase Guerrero final:
  ataque, atacar_a(otro), defensa, recibir_daño(d)
  (como si los hubieras escrito directamente en Guerrero)
```
