# POO 3 - UNQ

Repositorio de ejercicios de la materia Programación Orientada a Objetos 3 (UNQ). Lenguaje: Ruby + RSpec.

## Setup inicial (solo la primera vez)

### 1. Verificar Ruby
```bash
ruby --version
# Esperado: ruby 3.4.x
```

### 2. Instalar RSpec
Si gem falla con error de SSL:
```bash
brew install openssl
echo 'export SSL_CERT_FILE=$(brew --prefix)/etc/openssl@3/cert.pem' >> ~/.zshrc
source ~/.zshrc
```
Luego:
```bash
gem install rspec
```

### 3. Inicializar RSpec en el proyecto
```bash
rspec --init
```
Esto crea `.rspec` y `spec/spec_helper.rb`.

---

## Estructura del proyecto

```
poo3-unq/
├── src/               # código Ruby
│   └── mi_clase.rb
├── spec/              # tests RSpec
│   ├── spec_helper.rb
│   └── mi_clase_spec.rb
└── README.md
```

---

## Ejecutar código Ruby

```bash
ruby archivo.rb
```

---

## Ejecutar tests con RSpec

```bash
# Correr todos los tests
rspec

# Correr un archivo específico
rspec spec/mi_clase_spec.rb

# Correr con output detallado
rspec --format documentation
```

---

## Template base de una clase

```ruby
class MiClase
  def initialize(valor)
    @valor = valor
  end

  def valor
    @valor
  end
end
```

## Template base de un test

```ruby
require 'rspec'
require_relative '../src/mi_clase'

describe MiClase do
  it 'hace algo esperado' do
    objeto = MiClase.new(42)
    expect(objeto.valor).to eq(42)
  end
end
```

---

## Referencias

- [Documentación Ruby 3.4](https://docs.ruby-lang.org/en/3.4/)
- [RSpec Core](https://rspec.info/documentation/3.13/rspec-core/)
- [RSpec Matchers](https://rspec.info/features/3-13/rspec-expectations/built-in-matchers/)
