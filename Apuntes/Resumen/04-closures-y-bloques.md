# Closures y Bloques en Ruby

## Qué es una closure

Una **closure** (cierre) es una función que captura el entorno léxico donde fue creada.
Esto significa que "recuerda" las variables locales del contexto en el que fue definida, incluso después de que ese contexto haya terminado.

Componentes:
1. El código a ejecutar (los mensajes)
2. El contexto léxico (las variables libres capturadas)

---

## Bloques en Ruby

Ruby tiene una sintaxis particular: los **bloques sintácticos** se pasan como parte de un mensaje (no son objetos en sí mismos).

```ruby
[1, 2, 3].each { |x| puts x }

[1, 2, 3].each do |x|
  puts x
end
```

El bloque no es un objeto. Para tratarlo como objeto, necesitás convertirlo en un `Proc`.

---

## Proc (instancias de la clase Proc)

Un `Proc` **sí es un objeto**. Se puede crear con `Proc.new`, `proc { }` o el operador `lambda { }`.

```ruby
mi_proc = Proc.new { |x| x * 2 }
mi_proc.call(5)  # => 10

mi_lambda = lambda { |x| x * 2 }
mi_lambda.call(5)  # => 10
```

---

## Proc vs Lambda — diferencias importantes

| Característica | Proc | Lambda |
|---|---|---|
| `return` | Sale del método que lo contiene (no-local) | Sale sólo del lambda (local) |
| Aridad | No valida cantidad de argumentos | Valida cantidad de argumentos |
| Tipo | `Proc` | `Proc` (con `lambda?` == true) |

### El return no-local del Proc

```ruby
def metodo
  mi_proc = Proc.new { return "desde el proc" }
  mi_proc.call
  "nunca llega acá"  # esto nunca se ejecuta
end

metodo  # => "desde el proc"
```

Si el Proc sobrevive al método que lo creó y luego se llama, el `return` no-local no tiene a dónde volver → `LocalJumpError`.

### El return local del Lambda

```ruby
def metodo
  mi_lambda = lambda { return "desde el lambda" }
  mi_lambda.call
  "esto SÍ se ejecuta"
end

metodo  # => "esto SÍ se ejecuta"
```

---

## self en un bloque

El `self` dentro de un bloque es el `self` del contexto donde fue **definido** el bloque (captura léxica), no donde es ejecutado.

```ruby
class MiClase
  def crear_bloque
    proc { self }  # self capturado = la instancia de MiClase
  end
end

obj = MiClase.new
bloque = obj.crear_bloque
bloque.call  # => la instancia obj, no quien llama al bloque
```

Esto cambia si usás `instance_exec` o `class_exec`, que reemplazan el `self` del bloque.
Ver `08-intercesion.md`.

---

## Bloques y define_method

`define_method` acepta un bloque como implementación del método.
El bloque captura el contexto donde fue creado, lo que permite géneros de métodos dinámicos.

```ruby
class MiClase
  [:foo, :bar, :baz].each do |nombre|
    define_method(nombre) do
      "soy #{nombre}"
    end
  end
end

MiClase.new.foo  # => "soy foo"
MiClase.new.bar  # => "soy bar"
```

---

## Resumen rápido

- **Bloque sintáctico**: se pasa a un método con `{ }` o `do...end`. No es un objeto.
- **Proc**: objeto que representa un bloque. `return` sale del método contenedor.
- **Lambda**: Proc con semántica de función. `return` sale sólo del lambda.
- **self en bloque**: léxico (del contexto de definición), salvo que se use `instance_exec`/`class_exec`.
