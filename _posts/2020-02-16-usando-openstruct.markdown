---
layout: post
title:  "Usando OpenStruct"
date:   2020-02-16 10:00:00 -0500
excerpt: "OpenStruct es una estructura de datos de Ruby que permite crear objetos que reciban cualquier atributo de manera dinámica."
image: /assets/images/openstruct.png
---

![Usando OpenStruct](/assets/images/openstruct.png)

Ruby tiene varias formas de simplificar algunas partes del desarrollo, como por ejemplo el manejo de atributos en un objeto.

Nos podemos encontrar en la situación donde queremos dejar de usar hashes y manejar la data como si fueran objetos de Ruby. De esta forma podemos hacer el código un poco más simple o legible.

Por ejemplo, teniendo este array:

```ruby
pages = [
  {
    title: "Aprendiendo a usar arrays en JavaScript",
    slug: "/2020-02-15-javascript-arrays.html"
  },
  {
    title: "APIs de Internacionalización en JavaScript",
    slug: "/2019-05-13-apis-internacionalizacion.html"
  },
  {
    title: "Ejecutando comandos desde Ruby",
    slug: "/2019-03-25-ejecutar-comandos-ruby.html"
  },
  {
    title: "Usando Higher-Order Components",
    slug: "/2019-03-19-usando-hocs.html"
  }
]
```

Podríamos hacer:

```ruby
pages.map { |page| page[:slug] }
```

O, si convertimos `pages` a un array de `OpenStruct`, podríamos hacer:

```ruby
pages.map(&:slug)
```

---

OpenStruct permite crear objetos que leen y guardan atributos de manera dinámica, utilizando metaprogramación.

Para usar OpenStruct, creamos una instancia de `OpenStruct`:

```ruby
require "ostruct"

page = OpenStruct.new(title: "", slug: "")
#=> #<OpenStruct title="", slug="">
```

Con la estructura anterior, podemos hacer:

```ruby
require "ostruct"

pages = [
  {
    title: "Aprendiendo a usar arrays en JavaScript",
    slug: "/2020-02-15-javascript-arrays.html"
  },
  {
    title: "APIs de Internacionalización en JavaScript",
    slug: "/2019-05-13-apis-internacionalizacion.html"
  },
  {
    title: "Ejecutando comandos desde Ruby",
    slug: "/2019-03-25-ejecutar-comandos-ruby.html"
  },
  {
    title: "Usando Higher-Order Components",
    slug: "/2019-03-19-usando-hocs.html"
  }
]

pages = pages.map do |page|
  OpenStruct.new(page)
end

pages.map(&:slug)
#=> ["/2020-02-15-javascript-arrays.html", "/2019-05-13-apis-internacionalizacion.html", "/2019-03-25-ejecutar-comandos-ruby.html", "/2019-03-19-usando-hocs.html"]
```

Si queremos ir más allá, podemos crear una clase que herede de `OpenStruct`:

```ruby
require "ostruct"
require "date"

class Page < OpenStruct
  def created_at
    date_as_string = self.slug.slice(1, 10)
    date_parts = date_as_string.split("-").map(&:to_i)

    Date.new(date_parts[0], date_parts[1], date_parts[2])
  end
end

pages = [
  #...
]

pages = pages.map do |page|
  Page.new(page)
end

pages_created_in_2019 = pages.filter { |page| page.created_at.year == 2019 }
#=> [#<Page title="APIs de Internacionalizacion en JavaScript", slug="/2019-05-13-apis-internacionalizacion.html">, #<Page title="Ejecutando comandos desde Ruby", slug="/2019-03-25-ejecutar-comandos-ruby.html">, #<Page title="Usando Higher-Order Components", slug="/2019-03-19-usando-hocs.html">]
```

De esta forma, podemos hacer más escribiendo un poco menos, ahorrándonos varios `attr_accessor` y definir un método `initialize` para `Page`.