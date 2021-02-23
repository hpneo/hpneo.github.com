---
layout: post
title:  "Pasando datos síncronamente desde Rails hacia React"
date:   2021-02-22 19:30:00 -0500
excerpt: "En aplicaciones Rails+React es posible mandar datos en la carga inicial de la página utilizando vistas de Rails y el DOM."
image: /assets/images/domstate.png
---

![Pasando datos síncronamente desde Rails hacia React](/assets/images/domstate.png)

Una de las cosas que más me gustan de Webpacker es lo fácil es que es integrar React en una aplicación Rails. Si bien hay otras formas de tener una aplicación de React, como CRA, manejar el frontend y el backend en una sola aplicación sigue siendo el enfoque con el que más me siento cómodo.

Una de las ventajas de tener React integrado con Rails es que podemos enviar datos en la carga inicial de la página sin necesidad de hacer peticiones asíncronas. De esta forma podemos, por ejemplo, enviar datos sobre el usuario actual, una lista de categorías, configuración inicial, etc.

Para lograr esto, vamos a necesitar trabajar tanto por el lado de Rails como por el lado de React; y para eso usaremos el DOM y JSON.

## Renderizando datos para React desde Rails

Al estar en el lado del cliente, podemos usar JavaScript para leer el HTML que devuelve Rails, que luego podemos incluir en cualquier componente. Y para seguir aprovechando las ventajas del lenguaje, usaremos JSON para pasar datos de un ambiente (backend con Rails) a otro (frontend con React).

Para conseguir esto, vamos a crear una etiqueta `<script>` dentro de una vista de Rails:

```erb
<script type="application/json" id="root-initial-state-data">
</script>

<!-- Nuestro nodo raíz, donde React montará la aplicación que hemos construido. -->
<main id="root"></main>
```

Este `<script>` es diferente al que escribimos usualmente, por dos motivos:

* Su atributo `type` es `application/json`, en vez de `text/javascript` (o simplemente no lleva).
* Tiene un atributo `id`.

Esta forma de utilizar `<script>` nos permite incluir código JSON dentro de una página HTML de tal forma que: a) no es visible para el usuario final, y b) puede ser leído por nuestro código en el frontend, o por otras máquinas.

Una vez que tenemos esta etiqueta `<script>` dentro de una vista, podemos incluir código JSON usando algunos métodos de Ruby y Rails:

```erb
<script type="application/json" id="root-initial-state-data">
{
  "currentUser": <%= @current_user.to_json.html_safe %>
}
</script>
```

En este caso, tenemos una propiedad `currentUser`, cuyo valor es una variable `@current_user` que viene de un _controller_ de Rails. Para poder convertir esta variable en JSON, usamos el método `to_json`, y `html_safe` para asegurarnos que el valor JSON devuelto pueda ser insertado dentro de HTML sin problemas.

> **Nota:** Si quieres aprender sobre otros usos de esta técnica, puedes leer sobre [JSON+LD](https://json-ld.org/).

Ahora que ya tenemos los datos disponibles para ser leídos por React, vamos a ver cómo leerlos desde un componente.

## Leyendo datos del DOM con React

Como mencioné líneas arriba, React tiene acceso al DOM, por lo que leer el HTMl renderizado por Rails es relativamente sencillo.

Lo primero que debemos hacer es obtener acceso al elemento que queremos leer, para eso usaremos `getElementById`:

```javascript
const initialDataElement = document.getElementById('root-initial-state-data');
```

Para poder leer el contenido de `initialDataElement` usamos la propiedad `textContent`, lo que nos devuelve una cadena con el contenido de la etiqueta:

```javascript
const initialDataElement = document.getElementById('root-initial-state-data');
initialDataElement.textContent;
// '{ "currentUser": {...} }'
```

Llegado a este punto, solo tenemos una cadena con el contenido, y para poder trabajar con ese contenido como un objeto en JavaScript necesitamos convertirlo usando `JSON.parse`:

```javascript
const initialDataElement = document.getElementById('root-initial-state-data');
const initialData = JSON.parse(initialDataElement.textContent);

initialData.currentUser
// { id: 1, username: ... }
```

Podemos agrupar esta lógica en una función, y quedaría así:

```javascript
function getDOMState(elementID) {
  const initialDataElement = document.getElementById(elementID);
  const initialData = JSON.parse(initialDataElement.textContent);

  return initialData;
}
```

Hasta aquí, solo hemos usado JavaScript para obtener los datos que han sido renderizados por Rails, pero nos falta usarlos en React. Para lograr eso podemos llamar a la función `getDOMState` dentro del componente, como una variable más:

```javascript
function getDOMState(elementID) {
  const initialDataElement = document.getElementById(elementID);
  const initialData = JSON.parse(initialDataElement.textContent);

  return initialData;
}

const initialData = getDOMState('root-initial-state-data');

function Home() {
  const { currentUser } = initialData;

  if (currentUser) {
    return <Dashboard />;
  }

  return <SignIn />;
}
```

Si queremos ir un paso más allá, podemos hacer algunas mejoras a `getDOMState`, para asegurarnos que `JSON.parse` no falle si el elemento del DOM no existe o está vacío. Para eso, haremos los siguientes cambios:

```javascript
function getDOMState(elementID) {
  const initialDataElement = document.getElementById(elementID);

  if (initialDataElement || initialDataElement.textContent) {
    const initialData = JSON.parse(initialDataElement.textContent);

    return initialData;
  }

  return null;
}
```

Con esta condición aprovechamos el _type coercion_ de JavaScript donde un objeto del DOM (`initialDataElement`) y una cadena que no está vacía (`initialDataElement.textContent`) son considerados como `true`.

---

Esta técnica nos permite aprovechar las funcionalidades que nos ofrece un stack donde el backend y el frontend coexisten en una sola aplicación. Expandiendo un poco más esta idea podemos tratar al DOM como un _state_ de React, que no solo pueda ser leído si no re-escrito.