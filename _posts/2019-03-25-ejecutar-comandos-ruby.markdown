---
layout: post
title:  "Ejecutando comandos desde Ruby"
date:   2019-03-25 13:15:57 -0600
excerpt: "Podemos hacer uso de comandos de terminal desde Ruby a través de varias formas"
---

Hay muchas herramientas de línea de comandos (o CLI) que son útiles en el día a día, pero que en ocasiones también podrían ayudarnos a desarrollar otras herramientas más complejas.

Antes de llegar a ello, veamos cómo se pueden ejecutar comandos desde Ruby:

## Backticks (``)

Es la forma más sencilla de ejecutar un comando en Ruby.

Permite interpolación (utilizando `#{}`) y devuelve el resultado del comando ejecutado en forma de cadena.

Por ejemplo:

```ruby
directory = './app'
files = `ls #{directory}`
#=> "admin\nassets\nchannels\ncontrollers\ndecorators\nhelpers\njobs\nlisteners\nmailers\nmodels\nqueries\nresponders\nserializers\nservices\nviews\n"
```

> **Nota**: `%x` hace lo mismo que los backticks, pero con una sintáxis similar a `%q` o `%w`.

## `system`

`system` imprime el resultado del comando y devuelve `true` si el comando se ejecuta correctamente, o `false` si falla:

```ruby
system('ls ./app')
# admin		channels	decorators	jobs		mailers		queries		serializers	views
# assets		controllers	helpers		listeners	models		responders	services
#=> true
```

## `exec`

`exec` ejecuta el comando y reemplaza el proceso actual (`irb` o de la aplicación que esté ejecutando `exec`) por un nuevo proceso (el que fue creado por `exec` para ejecutar el comando).

----

Una herramienta de línea de comandos interesante de usar es [Rubocop](https://github.com/rubocop-hq/rubocop). Rubocop permite hacer análisis estático de código en Ruby según una serie de reglas configurables, y permite retornar los resultados en distintos formatos, entre ellos JSON.

De esta forma, podemos ejecutar Rubocop desde Ruby para obtener los resultados:

```ruby
linter_result = `/bin/bash -c "cd #{directory_name} && rubocop --extra-details --format json #{existing_files.join(' ')}"`
JSON.parse(linter_result)
```

> **Nota**: Uso `/bin/bash -c` para poder ejecutar varios comandos (en este caso `cd` y `rubocop`) dentro de una cadena.

> **Nota**: Para `rubocop` utilizo dos flags: `--extra-details` devuelve una explicación breve de cada regla de Rubocop, y `--format json` devuelve los resultados en formato JSON.

Otro escenario donde podríamos ejecutar comandos desde Ruby es para interactuar con git desde nuestra aplicación Ruby. Por ejemplo, si quisiera clonar un branch de Git de manera local, podría hacer algo como esto:

```ruby
def clone_branch(remote_origin, branch, directory_name)
  `mkdir -p tmp/#{directory_name}`
  `git config --global user.name "Mi Bot de Git"`
  `git config --global user.email "gitbot@example.com"`
  `git clone -b #{branch} --single-branch #{remote_origin} tmp/#{directory_name}`
end
```

Así, podría llamar `clone_branch('https://github.com/hpneo/gmaps', 'master', 'gmaps')`, y Ruby se encargaría de clonar el repositorio.

----

Existen otras herramientas que ofrecen datos interesantes que podríamos incluir en nuestras aplicaciones, y esta es una forma sencilla de poder usarlas si no encontramos una manera de integrarlas utilizando directamente Ruby.