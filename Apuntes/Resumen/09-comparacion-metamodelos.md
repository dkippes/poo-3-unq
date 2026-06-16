# Comparación de Metamodelos: Ruby vs Smalltalk vs Java

## El eje de comparación

La pregunta es: **¿cómo representa cada lenguaje a las clases y al método lookup?**

Tres ejes:
1. ¿Las clases son objetos de primera clase?
2. ¿Hay metaclases? ¿Se generan automáticamente?
3. ¿El sistema está cerrado sobre sí mismo?

---

## Ruby

### Metamodelo
- Toda clase es una instancia de `Class`
- Toda clase tiene una **singleton class** automática (`#NombreClase`)
- Hay una jerarquía paralela de singleton classes que espeja la jerarquía de clases
- `Class` es instancia de `Class` (autoreferencial)
- El metamodelo está accesible y modificable en runtime

### Jerarquía paralela
```
Clase normal       Singleton class
──────────         ───────────────
Guerrero      →    #Guerrero
  ↑                    ↑
Object        →    #Object
  ↑                    ↑
BasicObject   →    #BasicObject     ← excepción: su superclase es Class
                        ↑
                       Class
                        ↑
                       Module
```

### Method lookup
Para un objeto normal `atila`:
```
#atila → Guerrero → [mixines] → Object → Kernel → BasicObject
```

Para una clase `Guerrero` recibiendo un mensaje:
```
#Guerrero → #Object → #BasicObject → Class → Module → Object → Kernel → BasicObject
```

### Lenguaje vs Sistema
Ruby se piensa como un **lenguaje**: el código se compila/interpreta, hay fases bien separadas.
Aunque es muy reflexivo, no está pensado como un "sistema vivo" por defecto.

---

## Smalltalk

### Historia
- **Smalltalk-72**: sin clases como las conocemos hoy, bootstrapping manual
- **Smalltalk-78** → **Smalltalk-80** → **Squeak** → **Cuis-Smalltalk** (línea actual)
- Creado en Xerox PARC, pionero en OOP

### Metamodelo
Mucho más simétrico y consistente que Ruby o Java.

```
Behavior
  ↑
ClassDescription
  ↑              ↑
Class         Metaclass
```

- Cada clase `C` tiene automáticamente su **metaclase** `C class`
- `C class` es instancia de `Metaclass`
- `Metaclass` es instancia de `Metaclass class`, que a su vez es instancia de `Metaclass` (ciclo que cierra el sistema)

### Características únicas
- **Todo es un objeto y todo es un mensaje**: incluso el `if` es un mensaje (`ifTrue:ifFalse:`)
- No hay "primitivas" expuestas al programador: hasta los enteros son objetos
- El sistema es **reflexivo completo**: el IDE, el debugger, el propio intérprete son objetos en el mismo sistema

### Sistema vivo
Smalltalk se piensa como un **sistema**: el programa corre continuamente y se modifica a sí mismo.
No hay distinción nítida entre "tiempo de compilación" y "tiempo de ejecución".
Guardás una imagen (snapshot) del sistema completo, incluyendo todos los objetos en memoria.

---

## Java

### Metamodelo
El más simple de los tres.

- Hay una sola clase en el metalevel: `java.lang.Class`
- Toda clase es una instancia de `Class` (incluyendo `Class` misma)
- No hay metaclases automáticas por clase

```
Object  ──(instancia de)──→  Class
Guerrero ─(instancia de)──→  Class
Class   ──(instancia de)──→  Class (autoreferencial)
```

### Métodos estáticos (static)
- Los métodos `static` **no son mensajes** en el sentido OOP
- No hay polimorfismo a nivel de clase
- No hay `this` en un método estático
- El "method lookup" para métodos estáticos es puramente por tipo estático (decidido en tiempo de compilación)

### Consecuencias del metamodelo simple
- No se puede agregar métodos a clases en runtime (sin bytecode manipulation)
- No hay reflexión de las metaclases (sólo de las clases)
- Más seguro y predecible, pero menos flexible

---

## Tabla comparativa

| Característica | Ruby | Smalltalk | Java |
|---|---|---|---|
| Clases como objetos | Sí | Sí | Sí (`Class`) |
| Metaclases automáticas | Sí (singleton classes) | Sí (`X class`) | No |
| Métodos de clase como mensajes | Sí (en singleton class) | Sí (en metaclase) | No (static) |
| Sistema cerrado sobre sí mismo | Parcialmente | Completamente | No |
| Modificable en runtime | Sí (open classes) | Sí (sistema vivo) | Muy limitado |
| Modelo "lenguaje" vs "sistema" | Lenguaje | Sistema | Lenguaje |

---

## La distinción Lenguaje vs Sistema

- **Lenguaje**: el programa es un texto que se transforma y ejecuta. Hay fases separadas (parse, compile, run).
- **Sistema**: el programa es un conjunto de objetos vivos que se modifican continuamente. No hay fases separadas.

Smalltalk es el ejemplo más puro de **sistema**.
Ruby y Java son **lenguajes** (aunque Ruby es mucho más dinámico que Java).
