Aparte del repaso y la profundización de las cosas que ya habían aparecido (el method lookup y super, mixines, conceptos de metaprogramación y reflexión, mirrors), vimos dos temas nuevos principales:

Motivados a partir de pensar una manera de implementar comportamiento específico para las clases, llegamos a ver cómo Ruby implementa sus singleton classes, que permiten resolver este problema y además definir comportamiento específico para un objeto en particular. Accedemos a la singleton class de un objeto enviándole el mensaje singleton_class.
Vimos maneras maneras de modificar comportamiento dinámicamente (ej. mediante el mensaje define_method o con el mecanismo de clases abiertas). También hablamos un poco del concepto de "bloque" en Ruby, y cómo usar los mensajes instance_exec y class_exec.

En la documentación de Ruby pueden encontrar información sobre todos esos mensajes.


Por otro lado, en la carpeta de apuntes en Google Drive (ver mensaje fijado en este canal) creé una sub-carpeta con bibliografía.
Dejo aquí un resumen de lo que hay 👇
Artículos sobre metaprogramación

📄 Concepts and Experiments in Computational Reflection

Este es un artículo fundacional acerca de la reflexión computacional. De aquí sacamos la definición de sistema reflexivo (que vimos durante la clase de hoy) y de otros conceptos (secciones 1 a 4). En la sección 7 se mencionan las distintas propiedades deseables de un sistema reflexivo orientado a objetos que explicamos en clase. También aparece el concepto de "meta-objeto", que está relacionado con la idea de mirror. Recomendamos leer el artículo completo, pero las secciones 1 a 4 son de lectura obligatoria.


---

📄 Mirrors: Design Principles for Meta-level Facilities of Object-Oriented Programming Languages

Aquí se describe el concepto de mirror, que es algo que apareció durante nuestro ejercicio de reimplementación del method lookup y que vimos durante la clase de hoy. La idea está basada (en parte) en la primera propiedad deseable para un sistema reflexivo descripta en el artículo de Pattie Maes (la separación entre los niveles base y reflexivo de un sistema). En el artículo se postulan algunos principios de diseño para todo sistema orientado a objetos que sea reflexivo.

Este artículo está bueno para repasar el concepto, aunque no es de lectura obligatoria. Sin embargo, sí es necesario que tengan bien en claro la idea de la necesidad (a nivel diseño) de tener una buena separación entre el nivel base y el nivel meta de un sistema reflexivo. Pueden repasar esto en este artículo y/o en el de Pattie Maes.

---

📄 Safe Metaclass Programming

Este articulo explica en detalle el problema de comptaibilidad entre metaniveles, que es lo que vimos que justifica la jerarquía paralela entre clases y sus singleton classes (que juegan el rol, en ese contexto, de metaclases). El artículo además plantea algunos problemas con este tipo de soluciones. Las secciones más importantes (para el contexto de esta materia) son la 1 y la 2. La sección 3 da ejemplos en distintos lenguajes, y a partir de la sección 4 se propone un modelo para resolver los problemas que se plantean. Solamente la sección 2 es lectura obligatoria.

---

Artículo (nuevo) sobre mixines

📄 Mixins in Strongtalk

Este artículo puede servir de repaso para el concepto de mixin. Strongtalk es un lenguaje basado en Smalltalk pero estáticamente tipado. En la sección 2 hay un muy buen resumen del modelo. Otras secciones interesantes: en la sección 4 se discuten aspectos que tienen que ver con la verificación de tipos y los mixines (que tiene que ver con una pregunta que surgió en clase); y en la sección 5 hay una discusión sobre mixines y reflexión.

Este artículo no es de lectura obligatoria y sirve como repaso. Recomendamos leer la sección 2.

---

Por último, también hay algunos artículos que tienen que ver con el concepto de Trait (que es con lo que van a estar trabajando durante el TP 1.

---

El artículo principal es "Traits: A Mechanism for Fine-grained Reuse", y ese es el único que tienen que leer.
Dentro de ese, la sección 4 es la más útil para resolver el TP, porque ahí se da una definición de trait y una especificación (esto lo menciona la nota al pie en el enunciado). Para empezar, es suficiente con leer desde el inicio de la sección 4 hasta el inicio de la 4.1.

Sobre el resto de ese artículo, pegarle una mirada viene muy bien para tener una idea de la intención de los autores y ver a qué apuntaban:
Las secciones anteriores (1 a 3) presentan el contexto sobre el concepto de trait, y lo comparan con los mixines y la herencia múltiple.
Las secciones siguientes (5 a 9) tienen una discusión sobre cómo los traits solucionan los problemas que se describen en las secciones anteriores, algunas experiencias en cuanto a su implementación, y conclusiones.
Y con respecto a los otros dos artículos, son completamente opcionales y están nada más para tener contexto adicional.
En "Stateful Traits" hablan sobre una extensión al modelo que plantearon, en donde los traits pueden describir estado interno. Luego, "A new modular implementation for Stateful Traits" trata sobre la última implementación de traits con estado que hicieron para Pharo (un dialecto de Smalltalk).

Nosotros en el TP vamos a implementar traits que soportan estado, pero sin que eso requiera ningún esfuerzo adicional (gracias a cómo Ruby representa las variables de instancia). Lo único que hay que saber es que cuando en el primer artículo ("Traits: A Mechanism for Fine-grained Reuse") se menciona que los traits no tienen estado y se hacen algunos malabares sobre eso, nada de eso va a ser relevante para nuestro TP