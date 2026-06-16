# Metaprogramación — Conceptos Fundamentales

## Qué es un metaprograma

Un **metaprograma** es un programa que tiene a otro programa (o a sí mismo) como dato.
Opera sobre la **representación** del programa, no sobre los objetos del dominio.

Ejemplos:
- Un compilador es un metaprograma (toma código fuente y lo transforma)
- Un framework de testing que inspecciona clases para encontrar métodos de test
- Un ORM que genera métodos dinámicamente a partir de columnas de una tabla

> Advertencia: No usar metaprogramación para resolver problemas que no pertenecen al dominio de la programación. Si el problema es de negocio, resolverlo con objetos. Si el problema es sobre el programa mismo, ahí tiene sentido la metaprogramación.

---

## Reflexión

La **reflexión** es la capacidad de un programa de **razonar sobre sí mismo** en tiempo de ejecución.

Hay dos tipos:

### Introspección
Capacidad de **observar** la propia estructura sin modificarla.

```ruby
atila.class          # => Guerrero (qué clase soy)
atila.is_a?(Guerrero) # => true
atila.respond_to?(:atacar_a)  # => true
Guerrero.instance_methods     # => lista de métodos
Guerrero.ancestors            # => cadena de herencia
```

### Intercesión
Capacidad de **modificar** la propia estructura o comportamiento en tiempo de ejecución.

```ruby
Guerrero.define_method(:saludar) { "Hola!" }
atila.extend(OtroModulo)
```

---

## Modelo y Metamodelo

| Concepto | Definición | Ejemplo en Ruby |
|---|---|---|
| **Dominio** | Los objetos del problema | `atila`, `muralla`, `espadachin` |
| **Modelo** | La representación de los objetos | `Guerrero`, `Muralla`, `Espadachin` (las clases) |
| **Metamodelo** | La representación del modelo mismo | `Class`, `Module`, `Method` |

El metamodelo de Ruby responde: ¿cómo están representadas las clases? ¿cómo se buscan los métodos?

---

## Sistema reflexivo

Un **sistema reflexivo** es aquel donde el metamodelo es accesible y modificable desde el propio programa.

Ruby es altamente reflexivo:
- Las clases son objetos de primera clase
- Podés agregar métodos a una clase en runtime
- Podés inspeccionar la cadena de herencia en runtime
- Podés ejecutar bloques en el contexto de cualquier objeto

Smalltalk es el ejemplo más puro de sistema reflexivo: incluso la sintaxis del `if` es un envío de mensajes.

---

## API de introspección en Ruby

```ruby
# Sobre objetos
objeto.class                    # clase del objeto
objeto.is_a?(Clase)             # ¿es instancia de Clase o subclase?
objeto.kind_of?(Clase)          # igual que is_a?
objeto.instance_of?(Clase)      # ¿es instancia EXACTA de Clase (no subclases)?
objeto.respond_to?(:mensaje)    # ¿entiende ese mensaje?

# Sobre clases
Clase.instance_methods          # métodos de instancia
Clase.superclass                # superclase directa
Clase.ancestors                 # toda la cadena (incluye módulos)
Clase.include?(Modulo)          # ¿tiene ese módulo incluido?

# Sobre métodos
objeto.method(:nombre)          # objeto Method ligado
Clase.instance_method(:nombre)  # objeto UnboundMethod
```

---

## La distinción "todo X es un Y"

Un error común es confundir:
- "**Guerrero** es subclase de **Object**" (relación de herencia entre clases)
- "**atila** es instancia de **Guerrero**" (relación objeto-clase)

En el metamodelo de Ruby, estas son dos relaciones completamente distintas.
Ver `06-metamodelo-ruby.md` para el diagrama completo.
