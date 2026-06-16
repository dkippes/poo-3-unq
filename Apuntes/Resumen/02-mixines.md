# Mixines

## Qué es un mixin

Un **mixin** es un módulo (en Ruby, `module`) que puede ser incluido en una clase para agregarle comportamiento.
No es una clase: no se puede instanciar directamente.

Definición formal (Bracha & Cook, 1990): un mixin es una **función de superclase a clase**.
Es decir, un mixin `M` toma una clase `C` y produce una nueva clase `M(C)` que tiene el comportamiento de `M` más el de `C` como superclase.

---

## Solución al problema de Guerrero/Muralla/Fantasma

```ruby
module Atacante
  def ataque
    # comportamiento de ataque
  end
end

module Defensor
  def defensa
    # comportamiento de defensa
  end
end

class Fantasma
  include Atacante
end

class Muralla
  include Defensor
end

class Guerrero
  include Atacante
  include Defensor
end
```

Diagrama:
```
  <<mixin>>        <<mixin>>
  Atacante         Defensor
  (ataque)         (defensa)
     ↑  ↑             ↑  ↑
     |  |             |  |
 Fantasma  Guerrero  Muralla
           (ambos)
```

---

## Cómo funciona: linearización

Cuando incluís un mixin en una clase, Ruby **no hace herencia múltiple real**.
En cambio, inserta el módulo en la **cadena de superclases** (MRO — Method Resolution Order).

```ruby
class Guerrero
  include Defensor  # se incluye primero → va más arriba en la cadena
  include Atacante  # se incluye después → queda más cerca de Guerrero
end

Guerrero.ancestors
# => [Guerrero, Atacante, Defensor, Object, Kernel, BasicObject]
```

La regla es: el último módulo incluido queda más cerca en la cadena.

### Method lookup con mixines

Cuando `atila` (instancia de `Guerrero`) recibe un mensaje:
```
atila → singleton_class(atila) → Guerrero → Atacante → Defensor → Object → Kernel → BasicObject
```

---

## include vs prepend vs extend

| Mecanismo | Dónde inserta el módulo | Efecto |
|---|---|---|
| `include` | Después de la clase en la cadena | El método de la clase tiene prioridad sobre el del módulo |
| `prepend` | Antes de la clase en la cadena | El método del módulo tiene prioridad sobre el de la clase |
| `extend` | En la singleton class del objeto receptor | Agrega métodos al objeto específico (o a la clase si se usa en el cuerpo de la clase) |

```ruby
module Saludador
  def saludar
    "Hola!"
  end
end

class Persona
  prepend Saludador  # Saludador va primero en la cadena
end
```

---

## Ventajas sobre herencia múltiple

- **Sin ambigüedad**: la linearización es determinista (siempre hay un orden claro)
- **Sin diamante**: el mismo módulo no puede aparecer dos veces en la cadena
- **Composición limpia**: podés combinar comportamientos sin crear jerarquías artificiales

---

## Diferencia con Traits

Los mixines en Ruby **sí pueden tener estado** (variables de instancia).
Los traits (en su definición original) son stateless y tienen mecanismos explícitos de resolución de conflictos.
Ver `03-traits.md`.

---

## super en mixines

`super` dentro de un método de un mixin sube al siguiente en la cadena de ancestors, igual que con clases normales.

```ruby
module Atacante
  def ataque
    super + 10  # llama al ataque de quien esté arriba en la cadena
  end
end
```
