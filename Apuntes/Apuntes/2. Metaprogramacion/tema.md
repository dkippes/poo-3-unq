Durante la clase empezamos a hablar sobre la noción de metaprogramación y metamodelo (el modelo que nos permite construir modelos). Luego exploramos el metamodelo de Ruby usando introspección (ej. mediante mensajes como class, superclass, instance_methods o methods).

Además de explorar las clases y los mixines (definidos a partir de las clases Class y Module), vimos también las maneras que tiene Ruby para representar a los métodos:
UnboundMethods (para métodos tal como los define una clase/mixin para sus instancias)
Methods (para métodos ya ligados a un objeto receptor, que va a jugar el rol de self)

Recuerden que pueden buscar estas clases y mensajes en la documentación de Ruby (https://docs.ruby-lang.org/en/3.4/).