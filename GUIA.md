# Guia POO3 - UNQ 2026s1

## Estado de archivos descargados

### Apuntes de clase (todos descargados ✓)
| Archivo | Clase |
|---------|-------|
| `Apuntes/0. Introduccion/` | Presentación, conceptos fundamentales, intro Ruby |
| `Apuntes/1. Herencia y Mixines/Algunas definiciones.pdf` | Clase 1 |
| `Apuntes/1. Herencia y Mixines/Clase 1 - Repaso herencia.pdf` | Clase 1 (29/3) |
| `Apuntes/2. Metaprogramacion/Clase 2 - Intro Metaprogramación (resumen).pdf` | Clase 2 (11/4) |
| `Apuntes/3. Singleton Classes/Clase 3 - Singleton classes y otras cosas (resumen).pdf` | Clase 3 |

### Papers de bibliografía (todos descargados ✓)

| Archivo | Obligatorio | Qué leer |
|---------|-------------|----------|
| `Apuntes/Bibliografía/Metaprogramación/Concepts and Experiments in Computational Reflection.pdf` | **SI** | Secciones 1–4 obligatorio; sección 7 también |
| `Apuntes/Bibliografía/Metaprogramación/Safe Metaclass Programming.pdf` | **SI** (parcial) | Solo sección 2 obligatorio; secciones 1 y 3 recomendadas |
| `Apuntes/Bibliografía/Traits/Traits_ A Mechanism for Fine-grained Reuse.pdf` | **SI** (para TP) | Inicio sección 4 hasta 4.1 para el TP; secciones 1–3 y 5–9 para contexto |
| `Apuntes/Bibliografía/Metaprogramación/Mirrors_ Design Principles...pdf` | No | Repaso del concepto de mirror; útil para clase 3 |
| `Apuntes/Bibliografía/Mixines/Mixin-based Inheritance.pdf` | No (recomendado) | Saltearse sección 5 |
| `Apuntes/Bibliografía/Mixines/Mixins in Strongtalk.pdf` | No | Solo sección 2 como repaso de mixines |
| `Apuntes/Bibliografía/Traits/Stateful Traits.pdf` | No | Opcional, contexto extra para TP |
| `Apuntes/Bibliografía/Traits/A new modular implementation for Stateful Traits.pdf` | No | Opcional, implementación en Pharo |

### Repos de código de clase (descargado ✓)
- `Apuntes/1. Herencia y Mixines/age-of-mixins-ruby/` — código visto en clase (clase 1, mixines)

---

## Recorrido por unidad temática

### Unidad 1 — Herencia y Mixines (Clases 1)

**Leer:**
1. `Apuntes/Algunas definiciones.pdf`
2. `Apuntes/Clase 1 - Repaso herencia.pdf`
3. Capítulo de Programming Ruby sobre módulos: https://ruby-doc.org/docs/ruby-doc-bundle/ProgrammingRuby/book/tut_modules.html
4. `Apuntes/Bibliografía/Mixines/Mixin-based Inheritance.pdf` (saltearse sección 5)
5. `Apuntes/Bibliografía/Mixines/Mixins in Strongtalk.pdf` — sección 2

**Ver:**
- Video herencia múltiple: https://www.youtube.com/watch?v=JAINyb6IwPE

**Código:**
- `Apuntes/1. Herencia y Mixines/age-of-mixins-ruby/` — explorar ramas `mixines` y `anti_clases`

**Práctica:**
- `Practica/mixines/2026s1-mixines-ruby-dkippes/`
- `Practica/mixines/Ejercicio Mixines - Recuperación.pdf`

---

### Unidad 2 — Metaprogramación e Introspección (Clase 2)

**Leer:**
1. `Apuntes/Clase 2 - Intro Metaprogramación (resumen).pdf`
2. `Apuntes/Bibliografía/Metaprogramación/Concepts and Experiments in Computational Reflection.pdf` — secciones 1–4 **obligatorio**, sección 7 también
3. `Apuntes/Bibliografía/Metaprogramación/Mirrors_ Design Principles...pdf` — opcional, repaso de mirrors

**Conceptos clave de esta unidad:**
- Metamodelo de Ruby: `class`, `superclass`, `instance_methods`, `methods`
- `UnboundMethod` vs `Method` (ver docs: https://docs.ruby-lang.org/en/3.4/)
- Sistema reflexivo: definición de Pattie Maes
- Meta-objetos y mirrors

**Práctica:**
- `Practica/2026s1-graficador-ruby-dkippes-3/` — graficador de jerarquías usando introspección
- `Practica/2026s1-method-lookup-ruby-dkippes/` — reimplementación del method lookup con mirrors

---

### Unidad 3 — Singleton Classes y Comportamiento Dinámico (Clase 3)

**Leer:**
1. `Apuntes/Clase 3 - Singleton classes y otras cosas (resumen).pdf`
2. `Apuntes/Bibliografía/Metaprogramación/Safe Metaclass Programming.pdf` — sección 2 **obligatorio**; secciones 1 y 3 recomendadas

**Conceptos clave:**
- `singleton_class` — comportamiento específico por objeto/clase
- Jerarquía paralela de metaclases (justifica la estructura de singleton classes)
- `define_method` y clases abiertas (open classes)
- Bloques en Ruby, `instance_exec`, `class_exec`

---

### TP Grupal — Traits (Metaprogramación)

Enunciado: `Practica/TP Grupal Metaprogramacion/TP Metaprogramación 2026s1 - Traits.pdf`

**Leer (en este orden):**
1. `Apuntes/Bibliografía/Traits/Traits_ A Mechanism for Fine-grained Reuse.pdf`
   - Empezar por inicio sección 4 hasta inicio sección 4.1 (definición formal + especificación) — **mínimo para arrancar**
   - Secciones 1–3: contexto, comparación con mixines y herencia múltiple — recomendado
   - Secciones 5–9: implementación, experiencias, conclusiones — recomendado
2. `Apuntes/Bibliografía/Traits/Stateful Traits.pdf` — opcional, contexto extra (los traits del TP soportan estado gracias a Ruby)

**Nota importante del profe:** cuando el artículo diga que los traits no tienen estado, ignorarlo para el TP. Ruby maneja variables de instancia de forma que eso no es un problema.

---

## Recursos permanentes

| Recurso | URL |
|---------|-----|
| Docs Ruby 3.4 | https://docs.ruby-lang.org/en/3.4/ |
| RSpec Core docs | https://rspec.info/documentation/3.13/rspec-core/ |
| RSpec matchers | https://rspec.info/features/3-13/rspec-expectations/built-in-matchers/ |
| Drive con apuntes y bibliografía | https://drive.google.com/drive/folders/1zoXihCueXUFDyRimWmXzNb_zhtnQKaef |

---

## Estructura de carpetas

```
poo3-unq/
├── Apuntes/
│   ├── 0. Introduccion/             ← presentación, conceptos fundamentales, intro Ruby
│   ├── 1. Herencia y Mixines/       ← Clase 1 PDFs + age-of-mixins-ruby (código de clase)
│   ├── 2. Metaprogramacion/         ← Clase 2 PDF
│   ├── 3. Singleton Classes/        ← Clase 3 PDF
│   └── Bibliografía/                ← papers académicos (Metaprogramación, Mixines, Traits)
└── Practica/
    ├── Ejercicio de escritura/
    ├── Ejercicio Ruby - Carrito de compras/
    ├── mixines/
    │   └── 2026s1-mixines-ruby-dkippes/
    ├── 2026s1-graficador-ruby-dkippes-3/    ← introspección
    ├── 2026s1-method-lookup-ruby-dkippes/   ← mirrors + method lookup
    └── TP Grupal Metaprogramacion/          ← TP Traits
```
