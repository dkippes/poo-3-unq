# Intercesión — Modificar Comportamiento Dinámicamente

## Qué es la intercesión

La **intercesión** es la capacidad de **modificar** la estructura o comportamiento de un programa en tiempo de ejecución.
En Ruby, esto incluye agregar métodos, redefinir clases, y ejecutar bloques en contextos arbitrarios.

> Regla de oro: No usar metaprogramación para resolver problemas que no pertenecen al dominio de la programación.

---

## Clases abiertas (Open Classes)

En Ruby, **toda clase puede ser reabierta y modificada** en cualquier momento.

```ruby
class String
  def palindrome?
    self == self.reverse
  end
end

"racecar".palindrome?  # => true
```

Esto se llama **monkey patching** cuando se modifica una clase que no es propia (como String, Integer, Array).
Es poderoso pero peligroso: podés romper comportamiento esperado por otras partes del código.

Una práctica más segura es usar **refinements** (módulos que "refinan" una clase sólo en el ámbito donde se activan con `using`).

---

## define_method

Permite definir métodos dinámicamente en tiempo de ejecución.

```ruby
class Guerrero
  [:atacar, :defender, :huir].each do |accion|
    define_method("puede_#{accion}?") do
      true
    end
  end
end

Guerrero.new.puede_atacar?   # => true
Guerrero.new.puede_defender? # => true
```

`define_method` recibe un bloque. El bloque captura el contexto donde fue creado (es una closure).
Esto es la diferencia con `def`: `def` crea un contexto nuevo, `define_method` hereda el contexto.

---

## Method vs UnboundMethod

### UnboundMethod
- Representa un método que **define una clase o mixin** para sus instancias
- **No está ligado** a un objeto receptor (`self`) aún
- Para ejecutarlo, hay que **ligarlo** a un objeto primero con `bind`

Restricción: si el método proviene de una clase, sólo se puede ligar a instancias de esa clase o subclases.
Si proviene de un mixin (módulo), no hay restricciones.

```ruby
metodo_sin_ligar = Guerrero.instance_method(:atacar_a)
# => #<UnboundMethod: Guerrero#atacar_a>

metodo_ligado = metodo_sin_ligar.bind(atila)
metodo_ligado.call(otro)
```

### Method
- Representa un método **ligado a un objeto receptor** (`self` ya está definido)
- Se ejecuta directamente con `.call`

```ruby
metodo = atila.method(:atacar_a)
# => #<Method: Guerrero#atacar_a (bound to atila)>

metodo.call(otro)
```

### bind y unbind

```ruby
# Sacar un Method de un objeto y desligarlo
metodo_ligado = atila.method(:atacar_a)
metodo_desligado = metodo_ligado.unbind
# => UnboundMethod

# Ligarlo a otro objeto (debe ser compatible en tipo)
metodo_religado = metodo_desligado.bind(otro_guerrero)
metodo_religado.call(enemigo)
```

---

## Familia eval/exec — cambiar el self de un bloque

### instance_exec

Evalúa un bloque en el contexto de un **objeto**.
Dentro del bloque, `self` es el objeto receptor del `instance_exec`.
Los métodos definidos dentro del bloque se definen en la **singleton class** del objeto receptor.

```ruby
atila.instance_exec do
  def habilidad_especial
    "Conquista"
  end
end

atila.habilidad_especial  # => "Conquista"
gengis.habilidad_especial # => NoMethodError (sólo atila lo tiene)
```

### class_exec / module_exec

Evalúa un bloque en el contexto de una **clase o módulo**.
Dentro del bloque, `self` es la clase/módulo receptor.
Los métodos definidos se definen en la clase/módulo (disponibles para todas las instancias).

```ruby
Guerrero.class_exec do
  def grito_de_guerra
    "¡Ataque!"
  end
end

Guerrero.new.grito_de_guerra  # => "¡Ataque!"
```

### Comparación

| Método | self dentro del bloque | Métodos definidos en... |
|---|---|---|
| `objeto.instance_exec` | el objeto | singleton class del objeto |
| `Clase.class_exec` | la clase | la clase (métodos de instancia) |
| `Clase.instance_exec` | la clase (como objeto) | singleton class de la clase (métodos de clase) |

---

## method_missing

Si ningún método es encontrado en el method lookup, Ruby llama a `method_missing`.

```ruby
class Flexible
  def method_missing(nombre, *args)
    if nombre.to_s.start_with?("puede_")
      true
    else
      super
    end
  end

  def respond_to_missing?(nombre, include_private = false)
    nombre.to_s.start_with?("puede_") || super
  end
end
```

Siempre que se redefina `method_missing`, también hay que redefinir `respond_to_missing?`.

---

## Resumen de herramientas de intercesión

| Herramienta | Para qué |
|---|---|
| Clases abiertas | Agregar métodos a clases existentes |
| `define_method` | Definir métodos dinámicamente con closures |
| `Method` / `UnboundMethod` | Capturar, desligar y religar métodos |
| `instance_exec` | Ejecutar un bloque como si fuera un método del objeto |
| `class_exec` | Ejecutar un bloque como si fuera código de definición de clase |
| `method_missing` | Interceptar mensajes no encontrados |
