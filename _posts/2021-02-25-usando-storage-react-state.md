---
layout: post
title:  "Persistiendo estados de React con Web Storage"
date:   2021-02-25 08:30:00 -0500
excerpt: "Podemos usar localStorage y sessionStorage para guardar estados de React que persistan luego de recargar una página."
image: /assets/images/uselocalstoragestate.png
---

![Persistiendo estados de React con Web Storage](/assets/images/uselocalstoragestate.png)

La API de Web Storage permite guardar datos dentro del navegador en forma llave/valor. Con Web Storage podamos persistir datos en el navegador y volver a leerlos incluso después de haber recargado una página, lo cual es útil si queremos guardar datos sobre configuración de usuario, o incluso usarlo como caché.

Existen dos tipos de Web Storage, `localStorage` y `sessionStorage`, y ambos comparten las mismas propiedades, métodos y eventos. Así mismo, ambos comparten las siguientes características:

* Crean un espacio por cada origen (un origen está formado por el protocolo y el host).
* Cada espacio no comparte información con otros espacios de otros orígenes.
* Los datos persisten las recargas de página.
* Cada espacio está limitado a guardar 5MB como máximo.

La principal diferencia entre ambos tipos es que **`sessionStorage` borra los datos al momento que el tab o la ventana del navegador se cierra**, mientras que `localStorage` mantiene los datos guardados hasta que son manualmente eliminados.

Como ambos tienen una interfaz similar, vamos a usar `localStorage` en este post, pero el mismo código se puede aplicar para `sessionStorage` (teniendo en cuenta sus limitaciones).

## Accediendo a elementos del Storage

Si revisamos `localStorage` vamos a ver que parece ser solo un objeto con algunas entradas:

![Pasando datos síncronamente desde Rails hacia React](/assets/images/localstorage.png)
_Mi `localStorage` desde https://developer.mozilla.org/_

Esto significa que podemos acceder a las entradas de `localStorage` como si accediéramos a cualquier objeto en JavaScript:

```js
localStorage['banner.developer_needs.embargoed_until'];
// "1604634596534"
```

Sin embargo, `localStorage` también tiene un método llamado `getItem` que cumple la misma función:

```js
localStorage.getItem('banner.developer_needs.embargoed_until');
// "1604634596534"
```

## Escribiendo en el Storage

Como ya vimos, `localStorage` se comporta como un objeto en JavaScript, lo que significa que también podemos guardar valores de la misma forma que guardamos valores en un objeto:

```js
localStorage['banner.developer_needs.embargoed_until'] = Date.now();
// 1614099381524
```

Y de igual forma, `localStorage` tiene un método `setItem` que hace exactamente lo mismo:

```js
localStorage.setItem('banner.developer_needs.embargoed_until', Date.now());
// undefined
```

Hasta aquí no hay nada fuera de lo común con Web Storage, _excepto_ por un tema bastante particular: **Los valores son guardados como cadenas**. Volviendo al ejemplo anterior:

```js
localStorage['banner.developer_needs.embargoed_until'];
// "1614099592460"

localStorage.getItem('banner.developer_needs.embargoed_until');
// "1614099592460"
```

Sabemos que `Date.now()` devuelve un número, pero al acceder a la propiedad `banner.developer_needs.embargoed_until` de `localStorage`, lo que obtenemos es una cadena. ¿Qué pasa si queremos guardar un objeto dentro de `localStorage`?

```js
const userConfiguration = {
  receiveNotifications: false,
  displaySubscriptionBanner: true,
  enableGeolocation: false,
};

localStorage.setItem('userConfiguration', userConfiguration);

// Ahora tratamos de leer el objeto guardado en localStorage.
// La línea después de `localStorage.getItem...` es el resultado:
localStorage.getItem('userConfiguration');
// "[object Object]"
```

Como vemos, en vez de obtener el objeto `userConfiguration` tenemos una cadena `"[object Object]"`. Esto es por el _type coercion_ de JavaScript, que convierte el objeto en una cadena. Felizmente, tenemos una forma fácil de convertir objetos en cadenas: `JSON.stringify`.

`JSON.stringify` convierte cualquier valor de JavaScript a una cadena usando su representación en JSON, que es justamente lo que necesitamos ahora.

```js
const userConfiguration = {
  receiveNotifications: false,
  displaySubscriptionBanner: true,
  enableGeolocation: false,
};

localStorage.setItem('userConfiguration', JSON.stringify(userConfiguration));

localStorage.getItem('userConfiguration');
// "{"receiveNotifications":false,"displaySubscriptionBanner":true,"enableGeolocation":false}"
```

Luego de esta extensa introducción, podemos ver el tema que da título a este post: **¿Cómo podemos usar `localStorage` para persistir algunos estados de React?**

## Creando nuestro propio _hook_ en React

Los [_hooks_](https://es.reactjs.org/docs/hooks-intro.html) son una funcionalidad de las tantas ofrecidas por React para poder trabajar dentro de los componentes creados a través de funciones. En particular nos interesa usar el _hook_ `useState`, con el cual podemos leer y escribir estados dentro de un componente.

Supongamos que queremos guardar la configuración del usuario en nuestra aplicación, y para eso tenemos el siguiente componente:

```jsx
import React, { useState } from 'react';
import UserConfigurationForm from './UserConfigurationForm';

function UserConfigurationDashboard() {
  const [userConfiguration, setUserConfiguration] = useState({});

  return (
    <UserConfigurationForm onSubmit={(data) => setUserConfiguration(data)} />
  );
}
```

El componente `<UserConfigurationDashboard />` está usando `useState` para crear un estado que almacenará la configuración del usuario. Este estado luego es actualizado cuando el componente `<UserConfigurationForm />` lanza un evento `onSubmit`.

Este componente funciona sin problemas pero, si refrescamos la página, el estado de `<UserConfigurationDashboard />` vuelve a su valor inicial (`{}`). Para evitar esto usaremos `localStorage`.

El primer paso será hacer que el valor inicial de el estado venga de `localStorage`:

```jsx
import React, { useState } from 'react';
import UserConfigurationForm from './UserConfigurationForm';

function UserConfigurationDashboard() {
  const initialUserConfiguration = localStorage.getItem('userConfiguration');
  const [userConfiguration, setUserConfiguration] = useState(initialUserConfiguration);

  return (
    <UserConfigurationForm onSubmit={(data) => setUserConfiguration(data)} />
  );
}
```

Como ya sabemos, los valores de `localStorage` son guardados como cadenas, por lo que debemos convertirlos a objetos usando la contraparte de `JSON.stringy`: la función `JSON.parse`:

```jsx
import React, { useState } from 'react';
import UserConfigurationForm from './UserConfigurationForm';

function UserConfigurationDashboard() {
  const initialUserConfiguration = JSON.parse(localStorage.getItem('userConfiguration'));
  const [userConfiguration, setUserConfiguration] = useState(initialUserConfiguration);

  return (
    <UserConfigurationForm onSubmit={(data) => setUserConfiguration(data)} />
  );
}
```

Ahora nuestro estado `userConfiguration` siempre va iniciar con el valor que venga del `localStorage`, pero ahora notamos algo: Realmente no estamos persistiendo cualquier nuevo valor ingresado por el usuario. Esto es porque en ninguna parte hemos usado `setItem`.

Para sincronizar nuestro estado con `localStorage` vamos a usar un nuevo _hook_ llamado `useEffect`. Este _hook_ tiene dos partes: una función (también llamada _callback_) y un arreglo de dependencias; y ejecuta el _callback_ cuando cualquier elemento del arreglo de dependencias es actualizado.

En este caso, cada vez que el estado `userConfiguration` se actualice, lo grabaremos en el `localStorage`:

```jsx
import React, { useState, useEffect } from 'react';
import UserConfigurationForm from './UserConfigurationForm';

function UserConfigurationDashboard() {
  const initialUserConfiguration = JSON.parse(localStorage.getItem('userConfiguration'));
  const [userConfiguration, setUserConfiguration] = useState(initialUserConfiguration);

  useEffect(() => {
    // Recuerda usar `JSON.stringify` para convertir un objeto en cadena con formato JSON.
    localStorage.setItem('userConfiguration', JSON.stringify(userConfiguration));
  }, [userConfiguration]);

  return (
    <UserConfigurationForm onSubmit={(data) => setUserConfiguration(data)} />
  );
}
```

Ahora que ya tenemos la lógica lista podemos trabajar en crear nuestro propio _hook_, y para hacer eso vamos a mover esta lógica a una nueva función:

```jsx
import React, { useState, useEffect } from 'react';
import UserConfigurationForm from './UserConfigurationForm';

// La convención es que todos los hooks deben empezar con `use`.
// Le pasamos un `defaultValue` como valor por defecto, en caso la entrada no exista en `localStorage`.
function useUserConfiguration(defaultValue) {
  const initialUserConfiguration = JSON.parse(localStorage.getItem('userConfiguration')) || defaultValue;
  const [userConfiguration, setUserConfiguration] = useState(initialUserConfiguration);

  useEffect(() => {
    // Recuerda usar `JSON.stringify` para convertir un objeto en cadena con formato JSON.
    localStorage.setItem('userConfiguration', JSON.stringify(userConfiguration));
  }, [userConfiguration]);

  return [userConfiguration, setUserConfiguration];
}

function UserConfigurationDashboard() {
  const [userConfiguration, setUserConfiguration] = useUserConfiguration({});

  return (
    <UserConfigurationForm onSubmit={(data) => setUserConfiguration(data)} />
  );
}
```

De esta forma hemos encapsulado parte de la lógica del componente dentro de un _hook_ e incluso aprovechamos en agregar soporte para valores por defecto. Pero como eso no es suficiente, vamos a hacerlo un poco más genérico y reutilizable:

```jsx
import React, { useState, useEffect } from 'react';
import UserConfigurationForm from './UserConfigurationForm';

function useLocalStorageState(localStorageKey, defaultValue) {
  const initialValue = JSON.parse(localStorage.getItem(localStorageKey)) || defaultValue;
  const [value, setValue] = useState(initialValue);

  useEffect(() => {
    localStorage.setItem(localStorageKey, JSON.stringify(value));
  }, [value]);

  return [value, setValue];
}

function UserConfigurationDashboard() {
  const [userConfiguration, setUserConfiguration] = useLocalStorageState('userConfiguration', {});

  return (
    <UserConfigurationForm onSubmit={(data) => setUserConfiguration(data)} />
  );
}
```

---

Con este _hook_ propio hemos aprendido a trabajar con la API de Web Storage dentro de React. Si bien el ejemplo usa `localStorage`, el mismo código puede usarse con `sessionStorage`; e incluso se puede extender `useLocalStorageState` para poder elegir entre una y otra opción para persistir nuestros estados.