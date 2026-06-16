# Singleton Classes

## Qué es una singleton class

En Ruby, **todo objeto tiene su propia clase privada**, llamada **singleton class** (también eigenclass o metaclase del objeto).

Se escribe `#NombreObjeto` o se accede con `obj.singleton_class`.

La singleton class es:
- Una clase especial, creada automáticamente por Ruby para cada objeto
- Contiene los métodos definidos específicamente para **ese objeto individual**
- Es la primera que Ruby revisa en el method lookup

```ruby
def atila.presentarse
  "Soy Atila, el guerrero"
end

# Este método vive en #atila (la singleton class de atila)
```

---

## Para qué sirve

Permite agregar métodos a **un objeto individual** sin afectar a todos los de su clase.

```ruby
atila = Guerrero.new
gengis = Guerrero.new

def atila.habilidad_especial
  "Conquista total"
end

atila.habilidad_especial   # funciona
gengis.habilidad_especial  # NoMethodError
```

---

## Singleton classes de las clases

Lo interesante es que **las clases también son objetos**, por lo tanto también tienen singleton class.

La singleton class de `Guerrero` se nota `#Guerrero` y contiene los **métodos de clase** (los que se llaman sobre `Guerrero` directamente, no sobre instancias).

```ruby
class Guerrero
  def self.crear(nombre)  # esto se define en #Guerrero
    new(nombre)
  end
end

Guerrero.crear("Atila")   # método de clase
```

En Ruby, `def self.metodo` es azúcar sintáctica para definir un método en la singleton class de la clase.

---

## El metamodelo completo con singleton classes

```
(verde = singleton class, azul = superclase, rojo = instancia de)

Module  ──→  #Module
  ↑               ↑
Class   ←──  #Class
  ↑               ↑
BasicObject ──→ #BasicObject
  ↑                   ↑
Object      ──→  #Object
  ↑                   ↑
Guerrero    ──→  #Guerrero
  ↑                   ↑
Espadachin  ──→  #Espadachin

atila  ──→  #atila
  (instancia de Guerrero)
  (superclase de #atila es Guerrero)
```

---

## Regularidades del metamodelo

**1. Todo objeto tiene su singleton class.**
Esto incluye las clases mismas, y las singleton classes también tienen su propia singleton class (los `…` en el diagrama).

**2. La superclase de la singleton class depende del tipo de objeto:**
- Si el objeto **no es una clase**: la superclase de su singleton class es su **clase**.
  - `#atila` tiene como superclase a `Guerrero` (no a `#Guerrero`).
- Si el objeto **es una clase**: la superclase de su singleton class es la **singleton class de su superclase**.
  - `#Guerrero` tiene como superclase a `#Object`.
  - Excepción: `#BasicObject` tiene como superclase a `Class` (rompe el patrón para cerrar el sistema).

---

## Method lookup completo (con singleton classes)

Cuando `atila` recibe un mensaje:
```
atila → #atila → Guerrero → Defensor → Atacante → Object → Kernel → BasicObject
```

Cuando `Guerrero` recibe un mensaje (por ejemplo `Guerrero.crear`):
```
Guerrero → #Guerrero → #Object → #BasicObject → Class → Module → Object → Kernel → BasicObject
```

La regla general del method lookup:
1. Ir a la **singleton class** del receptor
2. Seguir la cadena de **superclases** de la singleton class

La singleton class **agrega un prefijo estricto** al camino que empieza en la clase del objeto.

---

## Una singleton class también es una clase

Las singleton classes son instancias de `Class` también (con algún quirk: son instancias de `#Class`, la singleton class de `Class`).
Esto significa que también tienen su propia singleton class, y así infinitamente (de ahí los `…` en el diagrama).
En la práctica no se llega tan profundo nunca.

---

## Resumen

| Concepto | Qué es |
|---|---|
| Singleton class de un objeto | Clase privada del objeto, primer paso del method lookup |
| Singleton class de una clase | Contiene los "métodos de clase" (def self.x) |
| Jerarquía de singleton classes | Espejo paralelo de la jerarquía de clases |
| `obj.singleton_class` | Acceso explícito a la singleton class |
| `class << obj; end` | Bloque para definir cosas dentro de la singleton class |
