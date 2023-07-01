---
layout: post
title:  "Creando un clon de Storybook desde 0 con Parcel"
date:   2023-06-27 20:10:00 -0500
excerpt: "Usando Parcel, podemos crear una versión simple de Storybook"
image: /assets/images/storybook.png
---

![Creando un clon de Storybook desde 0 con Parcel](/assets/images/storybook.png)

[Storybook](https://storybook.js.org/) permite mostrar en un solo lugar los componentes que utilizas en tu aplicación. Esto es útil si tienes componentes reutilizables y quieres documentar cómo se ven y cómo se usan sin necesidad de ir al código.

Como un ejercicio para aprender cómo funcionan el plugin de _transformer_ de [Parcel](https://parceljs.org/), vamos a crear un clon de Storybook.

## Component Story Format (o CSF)

El primer paso es entender que Storybook funciona leyendo archivos llamados _stories_ (o historias), los cuales son archivos de JavaScript cuyo nombre termina en `.stories.js`. Esto es una convención y puede cambiarse en la configuración de Storybook; pero lo importante es que, al fin y al cabo, estos archivos siguen un formato llamado [**Component Story Format**, o CSF](https://storybook.js.org/docs/react/api/csf).

Un ejemplo básico es CSF es este archivo que llamaremos `button.stories.js`:

```js
// Importamos el componente que usaremos en nuestras stories
import { Button } from "../src/ui";

// Definimos un objeto llamado meta (el nombre no importa), que luego exportaremos como default.
const meta = {
  title: "Components/Button",
  component: Button,
};

// Definimos una variable Basic que también exportaremos.
export const Basic = {
  args: {
    variant: "primary",
    children: "Click Me!",
    loading: false,
  },
  render(args) {
    return <Button {...args} />;
  },
};

// Finalmente exportamos meta como default.
export default meta;
```

La _story_ más simple posible, siguiendo el Component Story Format, tiene 2 exports: un _default export_ y un _named export_. El _default export_ contiene meta datos sobre la _story_, como el componente del que se está escribiendo, o el título de la _story_.

Por otro lado, el _named export_ es un objeto que tiene 2 propiedades: `args` y `render`. A su vez, `args` es un objeto que contiene los _props_ del componente que el usuario puede configurar mediante un panel de control, y `render` es una función que retorna el componente del que estamos escribiendo, y que también recibe una copia de `args` como argumento. Cada vez que los valores de `args` cambian, Storybook vuelve a llamar a `render` y el componente se re-renderiza.

Ahora que ya sabemos como funciona una _story_, vamos a crear nuestro propio Storybook.

## Obteniendo todos los archivos `.stories.js`

> Para este ejercicio estoy utilizando [Parcel](https://parceljs.org/), una _build tool_ simple de usar y configurable a partir de plugins.

Para poder listar todas las _stories_ que existen en nuestro proyecto, necesitamos primero importar todos los archivos JavaScript que terminen en `.stories.js`. Para lograr esta primera tarea usaremos un paquete llamado [`@parcel/resolver-glob`](https://www.npmjs.com/package/@parcel/resolver-glob).

Los _resolvers_ se encargan de resolver un [_import_](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import), ya sea convirtiendo el _specifier_ de un `import` (lo que comunmente es el nombre del módulo) en la ruta absoluta de un archivo o retornando código. Esto último se conoce como _virtual module_, porque el módulo definido por el _specifier_ no existe "físicamente" como un archivo.

Lo que hace `@parcel/resolver-glob` es utilizar el _specifier_ como si fuera un patrón llamado _glob_, y retorna un objeto con todos los módulos que cumplen con el patrón.

```js
import * as stories from "../stories/**/*.stories.js";

console.log(stories);
// {
//   button: {
//     default: { title: "Components/Button", component: ... },
//     Basic: { args: { variant: "primary", children: "Click Me!", loading: false }, render: ... }
//   }
// }
```

Para poder utilizar un _resolver_ dentro de nuestra aplicación, debemos agregarlo en la configuración de Parcel (`.parcelrc`):

```json
{
  "extends": "@parcel/config-default",
  "resolvers": ["@parcel/resolver-glob", "..."]
}
```

Una vez que ya tenemos los módulos importados en nuestro archivo, debemos obtener más información sobre los componentes que vamos a probar en las _stories_.

## Obteniendo información de los componentes con `react-docgen`

Si bien ya importamos los módulos de nuestras _stories_, es muy poco lo que podemos hacer con ellas, excepto quizá renderizar los _named exports_.

Si quisieramos crear una interfaz gráfica que nos permita cambiar los valores de las propiedades definidas en `args`, necesitamos saber de qué tipo de dato es cada atributo de ese objeto. Dado que `args` en realidad representa los valores que tienen los _props_ de un componente, usamos [`react-docgen`](https://www.npmjs.com/package/react-docgen) para obtener la información de los props directamente del componente. ¿Cómo podemos hacer eso?

En la sección anterior vimos los _resolvers_ de Parcel, pero Parcel tiene otro tipo de plugin que es igual o más útil: los _transformers_. Un _transformer_ toma un _asset_ (por ejemplo: un módulo de JavaScript) y lo transforma, pudiendo retornar código que es cualquier otra cosa excepto el código original del asset. En este caso, vamos a usar un _transformer_ para obtener información del componente, mediante `react-docgen`.

Lo que hace este _transformer_ es obtener el código fuente del asset con `asset.getCode()`, para luego pasárselo a `ReactDocGen`. Luego, `ReactDocGen` analiza el código y retorna información de los _props_ del componente, utilizando la propiedad `propTypes` del componente.

> Si deseas hacer lo mismo pero tus componentes están escritos en TypeScript, puedes usar [`react-docgen-typescript`](https://www.npmjs.com/package/react-docgen-typescript).

Una vez que obtenemos la información de los props de un componente, usamos una expresión regular para cambiar el código original e incluir lo devuelto por `ReactDocGen` como una propiedad más del component. Otra forma de lograr el mismo resultado es manipulando el AST con alguna biblioteca que permita eso, como `@babel/core`.

Finalmente, reemplazamos el código original del _asset_ por el editado, con `asset.setCode(output)`. No debemos olvidar asegurarnos que el _asset_ sea de tipo `js`, para que Parcel siga procesando el archivo en caso hayan otros _transformers_ en su _pipeline_.

```js
import { Transformer } from "@parcel/plugin";
import ReactDocGen from "react-docgen";

export default new Transformer({
  async transform({ asset }) {
    try {
      const source = await asset.getCode();
      const code = ReactDocGen.parse(source);

      const output = source.replace(
        /export default ([a-zA-Z]*)/,
        (substring, group) => {
          return `${group}.__docgenInfo = ${JSON.stringify(
            code
          )};\n\n${substring}`;
        }
      );

      asset.type = "js";
      asset.setCode(output);
    } catch (error) {}

    return [asset];
  },
});
```

A diferencia de los _resolvers_, los _transformers_ pueden ser definidos como parte de un _pipeline_ específico, los cuales son diferenciados utilizando _globs_. En este caso, quiero que mi _transformer_ solo sea utilizado en los archivos `.js` dentro de la carpeta `src/ui`.

```json
{
  "extends": "@parcel/config-default",
  "resolvers": ["@parcel/resolver-glob", "..."],
  "transformers": {
    "src/ui/*.js": ["./parcel-transformer-react-docgen/index.mjs", "..."]
  }
}
```

Los `"..."` al final del pipeline le dicen a Parcel que debe seguir procesando esos archivos con el pipeline por defecto para archivos `.js`.

---

Con este pequeño ejercicio hemos aprendido a utilizar los plugins _resolvers_ de Parcel, y cómo escribir nuestro propio _transformer_. Con estos 2 plugins, cubrimos la funcionalidad básica de Storybook, pero se puede extender utilizando otras bibliotecas, como [`@storybook/csf-tools`](https://www.npmjs.com/package/@storybook/csf-tools).
