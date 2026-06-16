# Herencia y Subclasificación

## Qué es un objeto

Un objeto se define por sus **mensajes** (no por sus atributos ni su tipo).
Tres propiedades esenciales:
- **Identidad**: cada objeto es único
- **Encapsulamiento**: el estado interno no es accesible directamente
- **Polimorfismo**: distintos objetos pueden responder al mismo mensaje de formas distintas

> Importante: la relación "instancia de" (un objeto con su clase) es distinta de la relación "subclase de" (entre clases). No confundirlas.

---

## Subclasificación / Herencia

La **subclase** hereda el comportamiento de la superclase.
La subclase puede:
- Usar métodos tal como están en la superclase
- **Overridear** (redefinir) un método
- Llamar al método original de la superclase con `super`

### Method lookup con herencia simple

Cuando `atila` recibe el mensaje `atacar_a(otro)`:

1. Ruby busca el método en la **clase** de `atila` (ej: `Espadachin`)
2. Si no lo encuentra, sube a la **superclase** (`Guerrero`)
3. Sigue subiendo (`Object`, `BasicObject`)
4. Si no lo encuentra en ningún lado → `NoMethodError`

```
atila → Espadachin → Guerrero → Object → BasicObject
```

---

## Problema motivador (Age of Empires)

Tenemos: **Guerrero**, **Espadachín** (subclase de Guerrero), **Muralla**, **Fantasma**

| Unidad | vida/defensa/recibir_daño | atacar/fuerza/ataque |
|---|---|---|
| Guerrero | sí | sí |
| Espadachín | heredado | redefinido |
| Muralla | sí | NO |
| Fantasma | NO | sí (100 fijo) |

El problema: Muralla y Guerrero comparten `vida/defensa/recibir_daño`, y Guerrero y Fantasma comparten `atacar/ataque`, pero no hay una sola jerarquía que modele esto bien con herencia simple.

---

## Intentos fallidos con herencia simple

### 1. Jerarquía forzada
Meter a `Defensor` como superclase de `Guerrero` y `Muralla`, y a `Atacante` como clase separada.
- Problema: `Muralla` hereda `ataque` de `Defensor` aunque no debería atacar.
- Solución "parche": redefinir `ataque` en `Muralla` para lanzar error → **Anti-clase**.

### 2. Anti-clases
Una subclase que **rompe** el contrato de la superclase (hereda algo y lo deshabilita).
Viola el principio de sustitución de Liskov.
```ruby
class Muralla < Defensor
  def ataque
    raise "Las murallas no atacan!"
  end
end
```
Esto es un smell: si necesitás una anti-clase, la jerarquía está mal modelada.

### 3. Delegación
Crear clases `ImplAtacante` e `ImplDefensor` con la implementación, y que `Guerrero`, `Fantasma` y `Muralla` deleguen a ellas.
- Funciona, pero es verboso y no es herencia real.
- Cada clase necesita métodos wrapper que sólo llaman al delegado.

### 4. Feature flags
Meter todo en una sola clase `Unidad` con booleanos `es_atacante?` / `es_defensor?`.
- El código se llena de ifs.
- Polimorfismo cero.

### 5. Repetir código
Copiar `ataque` en `Guerrero` y en `Fantasma`.
- Obviamente no escala.

### 6. Herencia múltiple
```
Fantasma hereda de Atacante
Guerrero hereda de Atacante y Defensor
Muralla hereda de Defensor
```
Funciona conceptualmente, pero genera el **Problema del Diamante**.

---

## El Problema del Diamante (Deadly Diamond of Death)

```
       A
      / \
     B   C
      \ /
       D
```

Si `B` y `C` overridean un método de `A`, y `D` hereda de ambos: ¿cuál versión usa `D`?
Ambigüedad irresoluble sin reglas arbitrarias. Por eso la mayoría de los lenguajes evitan la herencia múltiple.

---

## La solución: Mixines

En lugar de herencia múltiple, Ruby usa **mixines** (módulos incluidos en la cadena de herencia).
Ver `02-mixines.md`.
