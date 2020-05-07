---
layout: post
title:  "APIs de Internacionalización en JavaScript"
date:   2019-05-13 09:15:57 -0500
excerpt: "Con las APIs de Internacionalización del navegador podemos formatear fechas y números para diferentes idiomas"
image: /assets/images/intl.png
---

![APIs de Internacionalización en JavaScript](/assets/images/intl.png)

Cuando hablamos de internacionalización en front-end nos encontramos con algunos retos. Dejando de lado la traducción de etiquetas, botones y textos en general, nos podemos topar con escenarios donde necesitemos mostrar fechas o monedas para diferentes idiomas o países.

Felizmente existe un conjunto de APIs que nos facilitan este trabajo, y si antes necesitábamos bibliotecas de terceros para manejar estas tareas, con `Intl` podemos hacerlo nativamente, sin necesidad de agregar más dependencias a nuestra aplicación.

## `Intl​.Date​Time​Format`

[`Intl​.Date​Time​Format`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DateTimeFormat) nos permite formatear una fecha según el idioma que elijamos. Por ejemplo, las fechas en Estados Unidos tienen el formato `mes/día/año`, mientras que en Gran Bretaña y países de Hispanoamérica, el formato es `día/mes/año`.

```js
const date = new Date();

const formatterUnitedStates = new Intl.DateTimeFormat('en-US');
const formatterGreatBritan = new Intl.DateTimeFormat('en-GB');
const formatterPeru = new Intl.DateTimeFormat('es-PE');
const formatterSpain = new Intl.DateTimeFormat('es-ES');

console.log(formatterUnitedStates.format(date));
// 5/13/2019
console.log(formatterGreatBritan.format(date));
// 13/05/2019
console.log(formatterPeru.format(date));
// 13/5/2019
console.log(formatterSpain.format(date));
// 13/5/2019
```

## `Intl​.Relative​Time​Format`

[`Intl​.Relative​Time​Format`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RelativeTimeFormat) es una API interesante que nos permite formatear cantidades de tiempo relativas. Por ejemplo, si queremos mostrar _"2 días antes"_ o _"1 día después"_ de manera más _humana_ podemos hacerlo con esta API.

```js
const formatterEN = new Intl.RelativeTimeFormat('en');
const formatterENAuto = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });
const formatterES = new Intl.RelativeTimeFormat('es');
const formatterESAuto = new Intl.RelativeTimeFormat('es', { numeric: 'auto' });

console.log(formatterEN.format(1, 'day'));
// in 1 day
console.log(formatterENAuto.format(1, 'day'));
// tomorrow
console.log(formatterEN.format(1, 'week'));
// in 1 week
console.log(formatterENAuto.format(1, 'week'));
// next week

console.log(formatterES.format(1, 'day'));
// dentro de 1 día
console.log(formatterESAuto.format(1, 'day'));
// mañana
console.log(formatterES.format(1, 'week'));
// dentro de 1 semana
console.log(formatterESAuto.format(1, 'week'));
// la próxima semana
```

## `Intl​.Number​Format`

[`Intl​.Number​Format`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/NumberFormat) es un API que nos ayuda a formatear números, monedas y porcentajes. El caso de uso más común es cuando tenemos una aplicación que realiza cálculos de montos y la moneda varía según el cliente. Por ejemplo, si queremos mostrar un monto en dólares, en pesos mexicanos o en soles peruanos.

> **Nota:**: Esta API no realiza conversión entre monedas, solo se encarga de formatear el monto que se le pase.

```js
const total = 145.50;

const formatterDollars = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' });
const formatterSoles = new Intl.NumberFormat('es-PE', { style: 'currency', currency: 'PEN' });

console.log(formatterDollars.format(total));
// $145.50
console.log(formatterSoles.format(total));
// S/ 145.50
```

> **Nota:**: El resultado de `format` depende no solo de `currency`, si no también del idioma que le pases, por ejemplo:

```js
const total = 145.50;

const formatterDollars = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' });
const formatterSoles = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'PEN' });
const formatterPesos = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'MXN' });

console.log(formatterDollars.format(total));
// $145.50
console.log(formatterSoles.format(total));
// PEN 145.50
console.log(formatterPesos.format(total));
// MX$145.50
```

## `Intl​.List​Format`

[`Intl​.List​Format`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ListFormat) nos ayuda a mostrar una lista de elementos de manera _humana_. Por ejemplo, si queremos convertir `['manzana', 'uva', 'sandía']` a _"manzana, uva y sandía"_ o _"manzana, uva o sandía"_, esta API nos facilita esta tarea.

```js
const items = ['Iron Man', 'Captain America', 'Thor'];

const formatterENAnd = new Intl.ListFormat('en', { type: 'conjunction' });
const formatterENOr = new Intl.ListFormat('en', { type: 'disjunction' });
const formatterESAnd = new Intl.ListFormat('es', { type: 'conjunction' });
const formatterESOr = new Intl.ListFormat('es', { type: 'disjunction' });

console.log(formatterENAnd.format(items));
// Iron Man, Captain America, and Thor
console.log(formatterENOr.format(items));
// Iron Man, Captain America, or Thor
console.log(formatterESAnd.format(items));
// Iron Man, Captain America y Thor
console.log(formatterESOr.format(items));
// Iron Man, Captain America o Thor
```

> **Nota:**: Existe [un bug en español](https://github.com/tc39/proposal-intl-list-format/issues/45) para _"y"_ u _"o"_ cuando la última palabra empieza con _"i"_/_"hi"_ u _"o"_/_"ho"_, respectivamente.

----

Usualmente, para formatear fechas o monedas teníamos que recurrir a algunas bibliotecas, o escribir código más complejo. Con las APIs de Internacionalización nos ahorramos agregar más dependencias a nuestra aplicación.

De todas formas, es importante revisar el [soporte de estas APIs en los navegadores](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl#Browser_compatibility) antes de decidir usarlas en producción.