---
layout: post
title:  "Estructuras de datos para React"
date:   2020-07-14 20:30:00 -0500
excerpt: "Manejar estructuras de datos complejas en React puede ser complicado, aquí algunos tips."
image: /assets/images/data-structures.png
---

![Estructuras de datos para React](/assets/images/data-structures.png)

React [_recomienda_](https://es.reactjs.org/docs/update.html#overview) [evitar mutar los datos](https://reactjs.org/docs/optimizing-performance.html#the-power-of-not-mutating-data) de una aplicación. Es por eso que técnicas como usar el _spread operator_ son bastante utilizadas para manejar datos en React, pero conforme pasamos del código de ejemplo, nos podemos dar cuenta de lo complicado que puede llegar a ser manipular la data en cada vista.

Por ejemplo, tenemos el clásico _to-do list_:

```js
import React, { useState } from "react";

function TodoList() {
  const [items, setItems] = useState([]);

  return (
    <ul>
      {items.map(item => <li key={item}>{item}</li>)}
    </ul>
  );
}
```

En este ejemplo, los elementos de esta lista están guardados en un array, o arreglo, y para poder agregar un elemento a `items`, podemos hacer:

```js
const [items, setItems] = useState([]);

function addItem(newItem) {
  const updatedItems = [...items, newItem];

  return setItems(updatedItems);
}
```

Y si queremos eliminar un elemento nos topamos con un problema. **¿Cómo se hace?**

## Eliminar un elemento de una colección:

Para eliminar un elemento de una colección tenemos 2 opciones:

### Usando `splice`

[`[].splice`](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Objetos_globales/Array/splice) permite eliminar elementos de un array en base a su índice.

El primer parámetro de `splice` es el índice o posición donde se empieza a contar los elementos a eliminar, mientras que el segundo parámetro es el número de elementos a eliminar:

```js
function removeItem(indexToRemove) {
  const updatedItems = [...items];
  updatedItems.splice(indexToRemove, 1);

  return setItems(updatedItems);
}
```

### Usando `filter`

[`[].filter`](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Objetos_globales/Array/filter) permite filtrar los elementos de un array en base a una condición.

En este caso, la condición es _"todos los elementos excepto el que tenga el índice a eliminar"_:

```js
function removeItem(indexToRemove) {
  const updatedItems = items.filter((_item, index) => index !== indexToRemove);

  return setItems(updatedItems);
}
```

## Reemplazar un elemento de la colección

Para reemplazar un elemento dentro de una colección también tenemos 2 opciones, muy similar a la forma como eliminamos elementos.

### Usando `splice`

`splice` tiene un tercer parámetro que puede ser uno o más elementos que se van a insertar en la posición definida por el primer parámetro (en nuestro ejemplo, `indexToReplace`).

En este caso, lo que le decimos a `splice` es: _"elimina **1** elemento en la posición `indexToReplace`, y además agrega `newItem` en la posición `indexToReplace`"_:

```js
function replaceItem(indexToReplace, newItem) {
  const updatedItems = [...items];
  updatedItems.splice(indexToReplace, 1, newItem);

  return setItems(updatedItems);
}
```

### Usando `map`

[`[].map`](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Objetos_globales/Array/map) toma un array, itera por todos sus elementos, aplicando una función sobre cada uno de los elementos, y guarda el resultado en un nuevo array.

En este caso, lo que le decimos a `map` es: _"Devuelve los mismos elementos, a menos que el índice de la iteración actual sea el índice a reemplazar"_:

```js
function replaceItem(indexToReplace, newItem) {
  const updatedItems = items.map((item, index) => {
    if (index === indexToReplace) {
      return newItem;
    }

    return item;
  });

  return setItems(updatedItems);
}
```

## Reemplazar la propiedad de un objeto

Ahora supongamos que los elementos del _to-do list_ son objetos que no solo guardan el texto de cada tarea, si no otras propiedades, como fecha de creación, si está marcado como completado o no, etc.

Para poder cambiar las propiedades de un elemento de esa lista, vamos a tener que manipular un nuevo tipo de estructura de datos, que en este caso es un objeto.

Asumiendo que cada elemento de la lista tiene esta estructura:

```js
{
  text: "Aprendiendo sobre estructuras de datos",
  isCompleted: false,
  createdAt: "2020-07-15T00:42:03.338Z"
}
```

Podemos modificar una propiedad de un objeto usando el [_spread operator_](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator), que nos permite crear una copia del objeto que queremos modificar, sin cambiar el objeto anterior:

```js
function markAsCompleted(item, isCompleted) {
  const newItem = { ...item };
  newItem.isCompleted = isCompleted;

  return newItem;
}
```

## Relacionar dos colecciones

Vayamos un paso más allá. Ahora la data viene de una API y tienes 2 recursos: `todoLists` y `todoItems`.

Los `todoLists` tienen la siguiente estructura:

```js
const todoLists = [
  {
    id: 1,
    name: "Aprendiendo React"
  },
  {
    id: 2,
    name: "Aprendiendo Ruby"
  }
]
```

Y los `todoItems` tienen la siguiente estructura:

```js
const todoItems = [
  {
    id: 1,
    todoListId: 1,
    text: "Aprendiendo sobre estructuras de datos",
    isCompleted: false,
    createdAt: "2020-07-15T00:42:03.338Z"
  },
  {
    id: 2,
    todoListId: 1,
    text: "React hooks",
    isCompleted: false,
    createdAt: "2020-07-15T00:45:03.338Z"
  },
  {
    id: 3,
    todoListId: 1,
    text: "React Context",
    isCompleted: false,
    createdAt: "2020-07-15T00:46:03.338Z"
  },
  {
    id: 4,
    todoListId: 2,
    text: "Metaprogramación en Ruby",
    isCompleted: false,
    createdAt: "2020-07-15T00:48:03.338Z"
  }
]
```

En este caso, tenemos tres elementos para la lista con ID 1, y un elemento para el ID 2. **¿Cómo podríamos hacer para poder mostrar ambos recursos en una vista?**

Supongamos que queremos mostrar algo así:

```markdown
* Aprendiendo React
  - [ ] Aprendiendo sobre estructuras de datos
  - [ ] React hooks
  - [ ] React Context
* Aprendiendo Ruby
  - [ ] Metaprogramación en Ruby
```

Para lograr esto podemos usar [`[].filter`](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Objetos_globales/Array/filter):

```js
const todoListsWithItems = todoLists.map((todoList) => {
  const listWithItems = { ...todoList };

  listWithItems.todoItems = todoItems.filter((todoItem) =>
    todoItem.todoListId === todoList.id
  );

  return listWithItems;
});
```

De esta forma, podemos tener una estructura de datos mucho más fácil de usar dentro de un componente de React, como por ejemplo:

```js
function TodoListsSummary() {
  // const todoLists =  [...];
  // const todoItems =  [...];
  // const todoListsWithItems = todoLists.map((todoList) => {...;

  return todoListsWithItems.map((listWithItems) => (
    <article key={listWithItems.id}>
      <h3>{listWithItems.name}</h3>
      <ul>
        {listWithItems.todoItems.map((todoItem) => (
          <li key={todoItem.id}>{todoItem.text}</li>
        ))}
      </ul>
    </article>
  ));
}
```

---

Hay varias bibliotecas que permiten manejar estructuras de datos de manera inmutable, como [Immutable](https://immutable-js.github.io/) o [Immer](https://immerjs.github.io/immer/), pero es interesante conocer cómo podemos manejar este tipo de estructuras de manera relativamente sencilla sin depender de bibliotecas de terceros.