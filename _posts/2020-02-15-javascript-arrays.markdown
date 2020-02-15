---
layout: post
title:  "Aprendiendo a usar arrays en JavaScript"
date:   2020-02-15 18:15:00 -0500
excerpt: "Los arrays (o arreglos) son un tipo especial de objetos y representan colecciones que pueden guardar cualquier tipo de dato."
---

![Aprendiendo a usar arrays en JavaScript](/assets/images/arrays-js.png)

Los _arrays_ (o arreglos) son un tipo especial de objetos y representan colecciones que pueden guardar cualquier tipo de dato en JavaScript.

Los arrays tienen una propiedad llamada `length`, que guarda el tamaño, o número de elementos que contiene dicho array, y utilizan los corchetes (`[]`) para acceder a un elemento del arreglo a través de su índice. En JavaScript, el índice de los arrays empieza en 0.

## Trabajando con arrays

Al ser objetos, los arrays tienen una serie de métodos que sirven para manipularlos:

1. [`join`](#join)
1. [`pop`](#pop)
1. [`push`](#push)
1. [`indexOf`](#indexOf)
1. [`reverse`](#reverse)
1. [`concat`](#concat)
1. [`find`](#find)
1. [`forEach`](#forEach)
1. [`map`](#map)
1. [`filter`](#filter)
1. [`reduce`](#reduce)

### `join`

Une todos los elementos de un array en una cadena, utilizando un _separador_.

```javascript
const dateParts = [2020, 2, 15];

dateParts.length;
// 3

dateParts.join('-');
// "2020-2-15"
```

### `pop`

Quita el último elemento del array y retorna su valor.

```javascript
const dateParts = [2020, 2, 15];

dateParts.pop();
// 15

dateParts;
// [2020, 2]
```

> **Nota:**: `pop` es uno de los tantos métodos que modifican el array original.

### `push`

Agrega un elemento al final del array original y retorna el nuevo tamaño.

```javascript
const dateParts = [2020, 2, 15];

dateParts.pop();
// 15

dateParts.push(14);
// 3

dateParts;
// [2020, 2, 14]
```

### `indexOf`

Devuelve el índice (o posición) en el array del primer elemento que sea igual al argumento que recibe.

```javascript
const dateParts = [2020, 2, 15];

dateParts.indexOf(2);
// 1

dateParts.indexOf(15);
// 2
```

### `reverse`

Modifica el array invirtiendo sus elementos y retorna el array invertido.

```javascript
const dateParts = [2020, 2, 15];

dateParts.reverse();
// [15, 2, 2020]

dateParts;
// [15, 2, 2020]
```

### `concat`

Agrega elementos a una copia del array original y devuelve la copia con los nuevos elementos agregados.

```javascript
const dateParts = [2020, 2, 15];

dateParts.concat(18, 15, 0);
// [2020, 2, 15, 18, 15, 0]

dateParts;
// [2020, 2, 15]
```

### `find`

Devuelve el primer elemento que cumpla con una condición pasada como función.

```javascript
const dateParts = [2020, 2, 15];

dateParts.find(part => part > 1900);
// 2020
```

### `forEach`

Recorre cada elemento del array a través de una función.

```javascript
const dateParts = [2020, 2, 15];

dateParts.forEach(part => console.log(part));
// 2020
// 2
// 15
```

### `map`

Devuelve un nuevo array, donde cada elemento es el resultado de aplicar una función sobre cada elemento del array original.

```javascript
const dateParts = [2020, 2, 15];

dateParts.map(part => part + 2);
// [2022, 4, 17]
```

### `filter`

Devuelve un nuevo array, cuyos elementos son aquellos elementos del array original que cumplen con una condición pasada como función.

```javascript
const dateParts = [2020, 2, 15];

dateParts.filter(part => part < 1990);
// [2, 15]
```

### `reduce`

`reduce` permite trabajar sobre todos los elementos de un array y devolver un resultado, que puede ser _cualquier cosa_.

Como en los métodos anteriores, recibe una función como argumento, solo que en esta ocasión el parámetro es llamado _acumulador_. El _acumulador_ empieza con un valor inicial y acumula el valor retornado por cada iteración. Al final, `reduce` retorna el valor del _acumulador_.

Un ejemplo bastante común es sumar un array de números:

```javascript
const numbers = [1, 2, 3, 4, 5, 6];

numbers.reduce((accumulator, number) => accumulator + number);
// 21
```

El valor inicial de un acumulador es pasado como segundo parámetro, luego de la función. Si no se pasa ningún valor inicial, el valor inicial es el primer elemento del array.

Para nuestro ejemplo, ya que `dateParts` es un array que contiene año, mes y día, podemos generar un objeto que tenga la misma información pero estructurada de otra forma. Para eso creamos `dateKeys`, un array que contiene los nombres de las propiedades asociados al año, mes y día:

```javascript
const dateParts = [2020, 2, 15];
const dateKeys = ["year", "month", "day"];

dateParts.reduce((accumulator, part, index) => {
  const key = dateKeys[index];

  accumulator[key] = part;

  return accumulator;
}, {});
// {year: 2020, month: 2, day: 15}
```

> **Nota:**: `find`, `forEach`, `map` y `filter` tienen una función como primer parámetro, que a su vez puede recibir hasta 3 argumentos, en el siguiente orden: `item`, `index` y `array`. `reduce` tiene un argumento adicional (el _acumulador_) que va antes de `item`.
