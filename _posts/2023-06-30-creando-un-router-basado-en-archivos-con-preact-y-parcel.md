---
layout: post
title:  "Creando un router basado en archivos con Preact, preact-router y Parcel"
date:   2023-06-30 19:30:00 -0500
excerpt: "Con un plugin de Parcel podemos tener nuestro propio router basado en archivos, como los usados en Next.js o Remix."
image: /assets/images/fs-router.png
---

![Creando un router basado en archivos con Preact, preact-router y Parcel](/assets/images/fs-router.png)

> Para este ejercicio estoy utilizando [Parcel](https://parceljs.org/), una _build tool_ simple de usar y configurable a partir de plugins.

Un _router_ basado en archivos permite que una aplicación _client-side_ pueda armar sus rutas utilizando archivos como base. Esta convención es utilizada por frameworks como [Next.js](https://nextjs.org/) o [Remix](https://remix.run/), por nombrar algunos.

Gracias a esta convención, organizar archivos se vuelve una tarea mucho más sencilla, ya que existe una relación un poco más directa entre las rutas que ven los usuarios en su navegador con la estructura de archivos que ven los programadores al desarrollar una aplicación web frontend.

Como un ejercicio para aprender cómo funciona el plugin de _resolver_ de [Parcel](https://parceljs.org/), vamos a ver cómo crear un router basado en archivos utilizando Preact y [`preact-router`](https://www.npmjs.com/package/preact-router).

> Este acercamiento a tener un _router_ basado en archivos es experimental y más un ejercicio inicial que una implementación lista para producción.

El resultado final será que lograremos tener una aplicación con rutas completamente funcional con una estructura de archivos como esta:

```sh
./app/organizations/[subdomain]/layout.js
./app/organizations/[subdomain]/page.js
./app/organizations/[subdomain]/courses/layout.js
./app/organizations/[subdomain]/courses/page.js
./app/organizations/[subdomain]/courses/[identifier]/layout.js
./app/organizations/[subdomain]/courses/[identifier]/page.js
./app/organizations/[subdomain]/courses/[identifier]/cohorts/page.js
./app/organizations/[subdomain]/courses/[identifier]/edit/page.js
./app/organizations/[subdomain]/users/page.js
./app/organizations/new/page.js
./app/organizations/(course_editor)/[subdomain]/courses/[identifier]/[section]/[lesson]/layout.js
./app/organizations/(course_editor)/[subdomain]/courses/[identifier]/[section]/[lesson]/page.js
```

Y un archivo `index.js` como este:

```js
import { render } from "preact";
import Router from "@hpneo/router";

render(<Router />, document.body);
```

## Creando un _resolver_ de Parcel

Un _resolver_ en Parcel toma un specifier (lo que comúnmente es la ruta de un archivo o el nombre de un módulo de NPM) y devuelve un resultado que luego es usado por otros [_resolvers_](https://parceljs.org/features/plugins/#resolvers) o por [_transformers_](https://parceljs.org/features/plugins/#transformers).

El _resolver_ que escribiremos interceptará los `import` a `@hpneo/router` y devolverá el código que le indiquemos, también llamado [_virtual module_](https://parceljs.org/plugin-system/resolver/#virtual-modules), porque el módulo definido por el _specifier_ no existe "físicamente" como un archivo en la carpeta del proyecto.

Lo primero que haremos será escribir la estructura básica de un _resolver_:

```js
// ./parcel-resolver-router/index.mjs
import { Resolver } from "@parcel/plugin";
import path from "path";

export default new Resolver({
  async resolve({ specifier, options }) {
    if (specifier === "@hpneo/router") {
      const code = ``;

      return {
        filePath: path.join(options.projectRoot, "router.js"),
        code: code
      };
    }
  }
});
```

Este _resolver_ actualmente solo devuelve un _virtual module_ vacío. Sin embargo, contiene 2 detalles importantes a tener en cuenta: `specifier` es el nombre del módulo que estamos interceptando, y `options` contiene una propiedad llamada `projectRoot`, que devuelve la ruta desde donde Parcel está ejecutándose.

El siguiente paso es obtener todos los archivos que nos servirán para armar nuestras rutas. En este caso usaremos la convención de Next.js y su nuevo [App Router](https://nextjs.org/docs/app/building-your-application/routing), y usaremos [`fast-glob`](https://www.npmjs.com/package/fast-glob) para obtener todos los archivos llamados `page.js`.

El siguiente bloque de código va dentro del método `resolve` de nuestro plugin:

```js
import glob from "fast-glob";

const files = await glob("./app/**/page.js", {
  ignore: ["node_modules"],
  cwd: options.projectRoot,
});

// [
//   './app/organizations/[subdomain]/page.js',
//   './app/organizations/new/page.js',
//   './app/organizations/[subdomain]/courses/page.js',
//   './app/organizations/[subdomain]/users/page.js',
//   './app/organizations/[subdomain]/courses/[identifier]/page.js',
//   './app/organizations/[subdomain]/courses/[identifier]/cohorts/page.js',
//   './app/organizations/[subdomain]/courses/[identifier]/edit/page.js',
//   './app/organizations/[subdomain]/courses/[identifier]/[section]/[lesson]/page.js'
// ]
```

Una vez que tenemos las rutas de todos los archivos, vamos a iterar por cada uno de ellos para obtener la ruta de la aplicación a partir de la ruta del archivo:

```js
// ./parcel-resolver-router/index.mjs
const DYNAMIC_ROUTE_SEGMENT_PATTERN = /\[([a-zA-Z]*)\]/;
const CATCH_ALL_SEGMENT_PATTERN = /\[(\.\.\.([a-zA-Z]*))\]/;
const ROUTE_GROUP_PATTERN = /\(([a-zA-Z_-]*)\)/;

function buildRouteFromFilePath(filePath) {
  const pageRoute = filePath
    .replace(/^\.\/app/, "")
    .replace(/\/page\.js$/, "")
    .split("/")
    .map((segment) =>
      segment
        .replace(CATCH_ALL_SEGMENT_PATTERN, ":$2*")
        .replace(DYNAMIC_ROUTE_SEGMENT_PATTERN, ":$1")
        .replace(ROUTE_GROUP_PATTERN, "")
    )
    .filter(Boolean)
    .join("/");

  // Las rutas de aplicación deben empezar con '/'
  return pageRoute.startsWith("/") ? pageRoute : `/${pageRoute}`;
}
```

Como lo que estamos retornando en el _resolver_ es un _virtual module_, vamos a actualizar el código dentro del método `resolve` de nuestro plugin:

```js
const files = await glob("./app/**/page.js", {
  ignore: ["node_modules"],
  cwd: options.projectRoot,
});

const code = `const pages = [
  ${files
    .map((pagePath) => {
      const pageRoute = buildRouteFromFilePath(pagePath);

      return `{
        route: "${pageRoute}",
        component: require("${pagePath}")
      }`;
    })
    .join(",\n")}
];`;
```

Con estos cambios, la variable `pages` de nuestro _virtual module_ contiene un _array_ de objetos con 2 propiedades: `route` y `component`, que es como vamos a importar cada página.

## Soporte para layouts

Una vez que tenemos las rutas con sus respectivos módulos importados, vamos a obtener los _layouts_, si existieran. Un _layout_ es un componente que envuelve al componente de una página y a todos los componentes hijo que compartan la misma ruta.

En nuestro caso, si tenemos un _layout_ en `./app/organizations/[subdomain]/layout.js`, el componente que se exporta desde ese archivo va a estar presente al renderizar las siguientes rutas:

* `/organizations/:subdomain`
* `/organizations/:subdomain/courses`
* `/organizations/:subdomain/courses/:identifier`
* `/organizations/:subdomain/courses/:identifier/cohorts`
* `/organizations/:subdomain/courses/:identifier/edit`
* `/organizations/:subdomain/courses/:identifier/:section/:lesson`
* `/organizations/:subdomain/users`

Sin embargo, el _layout_ no se renderizará en la siguiente ruta:

* `/organizations/new`

Para obtener los _layouts_ para cada página, haremos lo siguiente:

```js
import fsSync from "fs";

const code = `const pages = [
  ${files
    .map((pagePath) => {
      const pageRoute = buildRouteFromFilePath(pagePath);
      const layoutPath = pagePath.replace(/page\.js$/, "layout.js");
      const hasLayout = fsSync.existsSync(layoutPath);
      const layout = hasLayout ? `require("${layoutPath}")` : "null";

      return `{
        route: "${pageRoute}",
        component: require("${pagePath}"),
        layout: ${layout}
      }`;
    })
    .join(",\n")}
];`;
```

## Soporte para route groups

Los _route groups_ permiten agrupar rutas de manera lógica sin afectar la URL final en la aplicación, y sirven para poder tener [múltiples _root layouts_](https://nextjs.org/docs/app/building-your-application/routing/route-groups#creating-multiple-root-layouts).

Dado que los _route groups_ se definen con carpetas con nombre siguiente el formato `(nombre)`, solo tenemos que verificar si cumple esa condición con una expresión regular:

```js
const ROUTE_GROUP_PATTERN = /\(([a-zA-Z_-]*)\)/;

const code = `const pages = [
  ${files
    .map((pagePath) => {
      const pageRoute = buildRouteFromFilePath(pagePath);
      const layoutPath = pagePath.replace(/page\.js$/, "layout.js");
      const hasLayout = fsSync.existsSync(layoutPath);
      const layout = hasLayout ? `require("${layoutPath}")` : "null";
      const isPartOfRouteGroup = !!pagePath.match(ROUTE_GROUP_PATTERN);

      return `{
        route: "${pageRoute}",
        component: require("${pagePath}"),
        layout: ${layout},
        isPartOfRouteGroup: ${isPartOfRouteGroup}
      }`;
    })
    .join(",\n")}
];`;
```

## Renderizando rutas con `preact-router`

Ahora que ya tenemos una colección de rutas, vamos a usar `preact-router` para renderizarlas en una aplicación de Preact.

Pero antes de hacer eso debemos entender que hay rutas que pueden estar anidadas, por lo que debemos convertir nuestro _array_ en un árbol. Para eso, agregamos la siguiente función al código del _virtual module_:

```js
// Este bloque de código va dentro de la variable `code`, como parte del código del virtual module:
import sortBy from "lodash/sortBy";
import partition from "lodash/partition";

function createRoutesFromPages(pages) {
  const routes = [];

  // Dado que las rutas son anidadas, es mejor si ordenamos nuestra colección de rutas de menor a mayor cantidad de caracteres.
  // Esto nos sirve para poder trabajar primero las rutas padre y luego las rutas hijas.
  let sortedPagesByRouteLength = sortBy(pages, (page) => page.route.length);

  while (sortedPagesByRouteLength.length > 0) {
    const page = sortedPagesByRouteLength.shift();

    // Usamos la función de Lodash llamada `partition` para dividir las rutas restantes entre
    // las que sí son hijas de la ruta actual y las que no
    const [childrenPages, otherPages] = partition(
      sortedPagesByRouteLength,
      (childPage) => {
        // Si una ruta es parte de un route group, la ponemos al mismo nivel que la ruta actual
        if (childPage.isPartOfRouteGroup) {
          return false;
        }

        return childPage.route.startsWith(page.route);
      }
    );

    sortedPagesByRouteLength = otherPages;

    routes.push({
      ...page,
      children: createRoutesFromPages(childrenPages),
    });
  }

  return routes;
}
```

Dado que las rutas van a estar anidadas, necesitamos un componente especial que pueda ser usando del router de `preact-router`. Este componente se llama `Route` y va a manejar todos los posibles escenarios de nuestra aplicación:

```js
// Este bloque de código va dentro de la variable `code`, como parte del código del virtual module:
import { h, Fragment } from "preact";
import Router from "preact-router";

// El componente `<Router />` de `preact-router` espera que sus componentes hijo tengan un prop `path`, así que lo pasamos acá
function Route({ path, component, layout, childRoutes = [], ...routeProps }) {
  childRoutes = Array.isArray(childRoutes) ? childRoutes : [childRoutes];

  // Creamos un elemento de Preact usando `h` de `preact`.
  // Como veremos luego, `component` es el componente que exporta cada archivo page.js
  const element = h(component, { path });
  // `layout` también es un componente, pero si no existe usamos `<Fragment />` de `preact`
  const LayoutOrFragment = layout ?? Fragment;

  // Si esta ruta no tiene rutas hija, simplemente renderizamos el componente con su layout
  if (childRoutes.length === 0) {
    return <LayoutOrFragment {...routeProps}>{element}</LayoutOrFragment>;
  }

  // Pero si la ruta tiene rutas hija, creamos un nested router, y le pasamos las rutas hija como un array de `<Route />`
  return (
    <LayoutOrFragment {...routeProps}>
      <Router>
        {element}
        {childRoutes.map((page) => (
          <Route
            key={page.route}
            // Si una de las rutas hija tiene más rutas hija, debemos hacer que el path soporte routers anidados
            path={page.children.length === 0 ? page.route : `${page.route}/:rest*`}
            component={page.component.default}
            layout={page.layout?.default}
            childRoutes={page.children}
          />
        ))}
      </Router>
    </LayoutOrFragment>
  );
}
```

Finalmente, creamos nuestro componente `<ApplicationRouter />`, el cual tendrá todas las rutas creadas con `<Route />` en base al resultado de `createRoutesFromPages`:

```js
// Este bloque de código va dentro de la variable `code`, como parte del código del virtual module:
function ApplicationRouter() {
  const routes = createRoutesFromPages(pages);

  if (routes.length === 0) {
    return null;
  }

  return (
    <Router>
      {routes.map((page) => (
        <Route
          key={page.route}
          // Si una de las rutas hija tiene más rutas hija, debemos hacer que el path soporte routers anidados
          path={page.children.length === 0 ? page.route : `${page.route}/:rest*`}
          component={page.component.default}
          layout={page.layout?.default}
          childRoutes={page.children}
        />
      ))}
    </Router>
  );
}

export default ApplicationRouter;
```

Y si bien con esto ya tenemos nuestro propio router basado en archivos, hay un último punto que debemos tener en cuenta: Si levantamos el proyecto con Parcel y agregamos luego nuevos archivos `page.js` o `layout.js`, nuestro router no reflejará las nuevas rutas. Esto es debido a que los _plugins_ de Parcel usan una caché, por lo que debemos invalidar la caché de Parcel bajo ciertos escenarios.

## Invalidar caché de Parcel

Para invalidar la caché de nuestro _resolver_, debemos agregar ciertas propiedades al objeto que retorna la función `resolve`:

```js
import { Resolver } from "@parcel/plugin";
import path from "path";

export default new Resolver({
  async resolve({ specifier, options }) {
    if (specifier === "@hpneo/router") {
      const glob_pattern = "./app/**/page.js";
      const glob_layout_pattern = "./app/**/layout.js";

      const files = await glob(glob_pattern, fs, {
        ignore: ["node_modules"],
        cwd: options.projectRoot,
      });
      const layouts = await glob(glob_layout_pattern, fs, {
        ignore: ["node_modules"],
        cwd: options.projectRoot,
      });

      const code = `...`;

      return {
        filePath: path.join(options.projectRoot, "router.js"),
        code: code,
        // Le decimos a Parcel que invalide cualquier archivo que cumpla con los patrones glob para `page.js` o `layout.js`
        invalidateOnFileCreate: [
          { glob: glob_pattern },
          { glob: glob_layout_pattern },
        ],
        // Le decimos a Parcel que invalide cualquier cambio en los archivos `page.js` o `layout.js` dentro de "./app":
        invalidateOnFileChange: [
          ...files.map((filePath) => path.join(options.projectRoot, filePath)),
          ...layouts.map((filePath) => path.join(options.projectRoot, filePath)),
        ],
      };
    }
  }
});
```

Si quisieramos soportar otras [convenciones de nombres de archivos](https://nextjs.org/docs/app/building-your-application/routing#file-conventions), como `error.js` o `template.js`, también debemos incluirlos aquí en `invalidateOnFileCreate` y `invalidateOnFileChange`.

---

El código completo del plugin de Parcel lo puedes encontrar en este Gist: [https://gist.github.com/hpneo/c9e9e61e9d530d6c412163f20d8a7df4](https://gist.github.com/hpneo/c9e9e61e9d530d6c412163f20d8a7df4)
