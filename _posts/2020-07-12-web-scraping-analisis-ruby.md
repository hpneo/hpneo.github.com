---
layout: post
title:  "Web scraping y análisis de datos con Ruby"
date:   2020-07-12 13:30:00 -0500
excerpt: "Con ActiveWorksheet y ScrapKit pude construir un pequeño análisis de ofertas inmobiliarias en las últimas semanas"
image: /assets/images/m2-cli.png
---

![Web scraping y análisis de datos con Ruby](/assets/images/m2-cli.png)

> **Nota:** Lo utilizado en este post es estrictamente de uso personal.

Buscar un lugar para vivir siempre es complicado, porque hay tantas variables a considerar antes de tomar una decisión. No solo es importante el precio, y la zona, si no también el número de habitaciones o incluso la cantidad de baños y el área construida.

Me he mudado varias veces en los últimos años, y algo que trato de hacer antes de elegir un lugar es analizar todas las ofertas disponibles.

Para esto, lo que hacía era buscar anuncios de lugares para vivir y volcar la información a mano en una hoja de cálculo, luego ir descartando los lugares más alejados, o con peor relación de m2/precio. Pero este es un proceso manual y tedioso que puede automatizarse. Y para automatizarlo me apoyo en 2 bibliotecas que construí en Ruby: **[ScrapKit](https://hpneo.dev/scrap_kit)** y **[ActiveWorksheet](https://hpneo.dev/active_worksheet)**.

[ScrapKit](https://hpneo.dev/scrap_kit) permite hacer web scraping y mapear selectores del DOM a propiedades de un objeto plano, mientras que [ActiveWorksheet](https://hpneo.dev/active_worksheet) lee un archivo CSV o XLS/XLSX y lo convierte a objetos de Ruby, donde cada fila es un registro y las columnas se vuelven propiedades de estos registros.

## Usando ScrapKit

Esta es la parte más _delicada_ del proceso, ya que implica hacer web scraping, que no es más que automatizar la extracción de información de una página web. Algunos sitios web no permiten usar esta técnica, pero ya que es para uso personal, espero que no haya problemas.

En mi caso, estoy usando [Nexo Inmobiliario](http://nexoinmobiliario.pe/) para obtener la información de las ofertas inmobiliarias que hay actualmente.

ScrapKit en su forma más básica me permite mapear selectores del DOM a atributos. Por ejemplo, puedo mapear el selector `.Project-header h1` al atributo `title`.

ScrapKit tiene una característica más poderosa, que permite mapear estructuras complejas usando el atributo especial `selector`, el cual es una colección de selectores que se va internando en el DOM hasta que cumpla una condición establecida.

Por ejemplo, en `[".Project-available-model", { ".name_tipology": "Departamento tipo A" }]`, ScrapKit busca dentro de `.Project-available-model` algún elemento que cumpla con el selector `.name_tipology` y que tenga el valor `Departamento tipo A`.

Sabiendo todo esto, podemos crear una "receta" de ScrapKit de la siguiente forma:

```ruby
url = "https://nexoinmobiliario.pe/proyecto/venta-de-departamento-123-en-lima"
type = "Departamento tipo A"

# Defino el mapeo de selectores a atributos a través de una "receta".
recipe = ScrapKit::Recipe.load(
  url: url,
  attributes: {
    title: ".Project-header h1",
    id: "#project_id",
    stage: ".bx-data-project.box-st > table > tbody > tr:nth-child(4) > td:nth-child(2)",
    due_date: ".bx-data-project.box-st > table > tbody > tr:nth-child(5) > td:nth-child(2)",
    latitude: "#latitude",
    longitude: "#longitude",
    info: {
      selector: [".Project-available-model", { ".name_tipology": type }],
      children_attributes: {
        tipology: "span.name_tipology",
        bedrooms: "span.bedroom",
        area: "span.area",
        price: "span.price"
      }
    }
  }
)

# Obtengo el resultado ejecutando la receta.
output = recipe.run
```

Dado que es un proceso que toma tiempo, es recomendable hacerlo una vez y guardar estos datos en algún lado para procesarla después. Y aquí es donde entra [ActiveWorksheet](https://hpneo.dev/active_worksheet).

## Usando ActiveWorksheet

ActiveWorksheet permite definir una clase en Ruby que se comporta de manera similar a un objeto de ActiveResource o ActiveRecord, con la diferencia que, mientras ActiveResource lee los datos desde un endpoint y ActiveRecord hace lo mismo desde una base de datos, ActiveWorksheet lo hace desde un archivo CSV, XLS o XLSX.

Los datos obtenidos por ScrapKit fueron guardados en un archivo en la ruta `~/departamentos.csv`, que tiene este formato:

```csv
Date,ID,Project,Model,Bedrooms,Area,Price,Due Date,Stage,Latitude,Longitude
2020-07-12,1823,SARAY 2,TIPO K,3,70.24 m2,"S/ 347,568","15 de Diciembre, 2021",En construcción,-12.087613139816469,-77.07117197819613
```

Con ActiveWorksheet creo una clase llamada `Project` que herede de `ActiveWorksheet::Base`, donde cada columna del CSV es automáticamente mapeado a un atributo:

```ruby
require "active_worksheet"

class Project < ActiveWorksheet::Base
  self.source = "~/departamentos.csv"

  # Estos métodos son usados para obtener y calcular valores en base a lo que viene del archivo CSV.
  def currency
    self.price.split(" ").first
  end

  def price_as_number
    self.price.split(" ").last.gsub(",", "").to_f
  end

  def area_as_number
    self.area.gsub(" m2", "").to_f
  end

  def price_per_m2
    (self.price_as_number / self.area_as_number).round(2)
  end
end
```

Con el método de clase `all`, ActiveWorksheet devuelve una colección de instancias de `Project`, una por cada fila del archivo CSV. Así, puedo filtrar por los registros que no cumplen los requisitos mínimos que busco:

```ruby
projects = Project.all
  .select { |project| project.date == Date.today }
  .select { |project| project.bedrooms > 1 }
  .select { |project| project.price_as_number <= 380_000 }
  .select { |project| project.area_as_number >= 60 }
  .select { |project| project.stage != "En planos" }

sorted_projects = projects.sort_by(&:price_as_number)
```

Los siguientes pasos serían: empezar a recopilar los datos de manera semanal, si se quiere hacer un análisis de los precios en el tiempo, y presentar los datos en un formato más legible para poder decidir mejor. Para ello desarrollé una herramienta de línea de comandos que está alojada en este [repositorio de GitHub](https://github.com/hpneo/m2).
