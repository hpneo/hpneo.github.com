---
layout: post
title:  "Creando bookmarks de CodeMirror con Preact"
date:   2020-07-25 16:30:00 -0500
excerpt: "Con CodeMirror puedes crear marcas dentro del texto en el editor. Con Preact, puedes agregar interactividad a bajo costo."
image: /assets/images/codemirror-preact.png
---

![Creando bookmarks de CodeMirror con Preact](/assets/images/codemirror-preact.png)

[CodeMirror](https://codemirror.net/) es una de mis bibliotecas favoritas y la que más utilizo cuando se trata de implementar un editor de texto plano. Una de las funcionalidades que estoy explorando ahora es la de los llamados _bookmarks_.

Un _bookmark_ es una marca dentro del editor que está asociado a una posición específica (línea y columna) y puede contener un nodo del DOM. Los _bookmarks_ son útiles para extender la funcionalidad del editor y proveer acciones dentro de un contexto específico (por ejemplo **[`codemirror-colorpicker`](https://github.com/easylogic/codemirror-colorpicker)** agrega un _color picker_ en un _bookmark_ cuando encuentra un color dentro del texto).

Lo interesante es que, al contener un nodo del DOM, el _bookmark_ puede recibir eventos, con lo que podemos usar JavaScript para [agregarle eventos](https://cevichejs.com/3-dom-cssom.html#eventos). O, podríamos usar una biblioteca que nos permita simplificar ese trabajo, e incluso reutilizar otras partes de nuestro código.

Aquí es donde entra [Preact](https://preactjs.com/). Con solo [4kB de tamaño](https://bundlephobia.com/result?p=preact@10.4.6) es ideal para crear componentes como React, pero más ligeros.

Digamos que para el caso que nos ocupa, vamos a hacer un editor de texto que, cuando encuentre un texto **`date:`**, agregue un botón al costado derecho que permita incrustar la fecha y hora local.

## Creando el editor con CodeMirror (y Preact)

Primero debemos crear un editor de texto con CodeMirror. En mi caso, voy a utilizar Preact para crear el _bookmark_ y el editor.

Para esto creo un `<div />` con un `<textarea />`. El `<textarea />` debe tener un `ref`, que luego voy a usar dentro de un hook para crear la instancia de `CodeMirror`.

```jsx
import { useRef, useEffect } from "preact/hooks";
import CodeMirror from "codemirror/lib/codemirror";
import "codemirror/lib/codemirror.css";

const options = {
  lineNumbers: true
};

export default function Editor() {
  const ref = useRef(null);

  useEffect(() => {
    CodeMirror.fromTextArea(ref.current, options);
  }, []);

  return (
    <div class="editor">
      <textarea ref={ref} />
    </div>
  );
}
```

Este primer paso es bastante directo, y si no utilizas Preact (o React), también puedes usarlo con cualquier otra biblioteca, o incluso con JavaScript puro.

## Creando el component del _bookmark_ con Preact

El siguiente paso es crear el componente que irá dentro del _bookmark_. En este caso, el componente es un simple botón que, al hacer click, llamará al prop `onClick`, pasándole la fecha y hora actual como una cadena.

```jsx
export default function NowButton({ onClick }) {
  function handleOnClick() {
    onClick(new Date().toString());
  }

  return <button onClick={handleOnClick}>Now</button>;
}
```

## Creando el _bookmark_

Este paso es el más complejo, porque requiere conocer un poco del API de CodeMirror.

Lo primero que tenemos que hacer es crear la función `createBoomarks`. Esta función va a usar el API de CodeMirror para crear los bookmarks en base a nuestra condición inicial (que haya una línea que empiece con **`date:`**).

Para poder hacer esto debemos permitir que `createBoomarks` reciba como argumento, la instancia de CodeMirror que corresponde al editor.

```js
function createBoomarks(cm) {
}
```

Luego, para poder aplicar los bookmarks sin bloquear el editor, o generar varios cambios dentro del mismo, usaremos el método `operation` de CodeMirror.

Este método permite pasarle un _callback_ que puede contener varios cambios y operaciones en el editor, pero que se aplicará como un solo cambio dentro de CodeMirror, mejorando la performance de nuestras operaciones.

```js
function createBoomarks(cm) {
  cm.operation(() => {});
}
```

El siguiente paso es iterar por cada línea del editor para agregar nuestro _bookmark_, si cumple con la condición:

```js
function createBoomarks(cm) {
  cm.operation(() => {
    cm.eachLine(lineHandle => {
      // Si no cumple la condición, ignoramos el resto del callback.
      if (!lineHandle.text.startsWith("date:")) {
        return;
      }

      // Obtenemos el número de línea y caracter para saber dónde posicionar el bookmark.
      const line = lineHandle.lineNo();
      const ch = lineHandle.text.indexOf(":");

      // Creamos el "widget", que será el contenedor de nuestro componente.
      const widget = document.createElement("div");
      widget.style.display = "inline";
      widget.style.verticalAlign = "middle";
      widget.style.height = "14px";

      // Creamos el bookmark con cm.doc.setBookmark, pasándole la línea y caracter, y el widget.
      cm.state.bookmarks[line] = cm.doc.setBookmark(
        { line, ch: ch + 1 },
        { widget, handleMouseEvents: true }
      );
    });
  });
}
```

Aquí hay algo importante a considerar. Dado que el texto de nuestro editor puede cambiar varias veces (porque el usuario edita el texto), es necesario guardar una referencia de nuestros _bookmarks_ en caso necesitemos eliminarlos antes del siguiente cambio.

Es por eso que CodeMirror nos da un objeto `cm.state`, donde podemos definir estados para el editor. En nuestro caso, usaremos `state` para crear un objeto `bookmarks` donde las llaves serán los números de línea, y los valores serán los _bookmarks_.

> **Nota:** En nuestro caso, las llaves del objeto `cm.state.bookmarks` son los números de línea porque se asume que nuestros _bookmarks_ solo aparecen una vez por línea. Pero en otros casos podría ser una combinación de línea y caracter u otro.

Hasta aquí hemos creado una función que crea un _bookmark_ con un elemento `<div />` que no hace nada. Ahora toca agregarle el componente:

```jsx
function createBoomarks(cm) {
  cm.operation(() => {
    cm.eachLine(lineHandle => {
      // if (!lineHandle.text.startsWith("date:")) {
      //   return;
      // }

      // const line = lineHandle.lineNo();
      // const ch = lineHandle.text.indexOf(":");

      // const widget = document.createElement("div");
      // widget.style.display = "inline";
      // widget.style.verticalAlign = "middle";
      // widget.style.height = "14px";

      // Aquí definimos el valor del prop `onClick`. Recordemos que esta función recibe la fecha y hora local como una cadena.
      function setDate(date) {
        // `cm.doc.replaceRange` va a reemplazar cualquier texto que exista luego de "date:" con el valor que reciba la función `setDate`.
        cm.doc.replaceRange(
          date,
          { line, ch: ch + 1 },
          {
            line,
            ch: ch + date.length
          }
        );
      }

      // Aquí es donde creamos el componente que definimos algunos párrafos arriba.
      const button = <NowButton onClick={setDate} />;

      // cm.state.bookmarks[line] = cm.doc.setBookmark(
      //   { line, ch: ch + 1 },
      //   { widget, handleMouseEvents: true }
      // );

      // Por último, montamos y renderizamos nuestro componente usando la función `render` de Preact.
      render(button, widget);
    });
  });
}
```

Por último, debemos definir un _init hook_. Un _init hook_ en CodeMirror es una función que se va a llamar al crear un editor. Esto es útil si sabemos que el editor tendrá un valor inicial y queremos crear nuestros _bookmarks_ inmediatamente después de crear el editor. Pero no solo eso, también es aquí donde guardaremos un estado con los _bookmarks_.

También vamos a requerir asociarnos a un evento `change`, que permitirá ejecutar nuestra función `createBoomarks` (la que va a crear los _bookmarks_) cada vez que haya un cambio en el editor.

```jsx
CodeMirror.defineInitHook(cm => {
  cm.state.bookmarks = {};
  createBoomarks(cm);
  cm.on("change", () => createBoomarks(cm));
});
```

Líneas arriba había mencionado que podemos requerir borrar todos los _bookmarks_ antes del siguiente cambio. Esto es posible agregando el siguiente código dentro del _callback_ de `cm.operation`:

```js
function createBoomarks(cm) {
  cm.operation(() => {
    Object.keys(cm.state.bookmarks).forEach(lineNumber => {
      // Desmontamos el componente al llamar a la función `render` de preact con un `null`.
      const widget = cm.state.bookmarks[lineNumber].replacedWith;
      render(null, widget);

      // Llamamos al método `clear` de los bookmarks y eliminamos `cm.state.bookmarks[lineNumber]` de la memoria.
      cm.state.bookmarks[lineNumber].clear();
      delete cm.state.bookmarks[lineNumber];
    });

    // cm.eachLine(lineHandle => {
    // ...
  });
}
```

---

**¿Por qué es importante desmontar los componentes antes de llamar a `clear()`?** Por dos motivos:

1. Si solo llamamos a `clear()`, CodeMirror eliminará el nodo del DOM (el famoso `widget`), pero Preact seguirá manteniendo una instancia del componente para ese nodo y no liberará memoria (los ya conocidos _memory leaks_).
2. Al desmontar los componentes con `render(null, widget)` también logramos que los componentes se desuscriban de sus propios _side effects_. Si no desmontamos el componente, un (hipotético) `setInterval` seguirá llamándose incluso luego de haber eliminado el _bookmark_, o un `fetch` podría seguir en ejecución tratando de finalizar un request aunque el editor esté completamente vacío.

---

El código completo quedaría así:

```jsx
import { render } from "preact";
import CodeMirror from "codemirror/lib/codemirror";
import NowButton from "./nowButton";

function createBoomarks(cm) {
  cm.operation(() => {
    Object.keys(cm.state.bookmarks).forEach(lineNumber => {
      const widget = cm.state.bookmarks[lineNumber].replacedWith;
      render(null, widget);
      cm.state.bookmarks[lineNumber].clear();
      delete cm.state.bookmarks[lineNumber];
    });

    cm.eachLine(lineHandle => {
      if (!lineHandle.text.startsWith("date:")) {
        return;
      }

      const line = lineHandle.lineNo();
      const ch = lineHandle.text.indexOf(":");

      const widget = document.createElement("div");
      widget.style.display = "inline";
      widget.style.verticalAlign = "middle";
      widget.style.height = "14px";

      function setDate(date) {
        cm.doc.replaceRange(
          date,
          { line, ch: ch + 1 },
          {
            line,
            ch: ch + date.length
          }
        );
      }

      const button = <NowButton onClick={setDate} />;

      cm.state.bookmarks[line] = cm.doc.setBookmark(
        { line, ch: ch + 1 },
        { widget, handleMouseEvents: true }
      );

      render(button, widget);
    });
  });
}

CodeMirror.defineInitHook(cm => {
  cm.state.bookmarks = {};
  createBoomarks(cm);
  cm.on("change", () => createBoomarks(cm));
});
```

Y aquí un ejemplo en vivo:

<iframe
  src="https://codesandbox.io/embed/magical-dawn-3hw6z?fontsize=14&hidenavigation=1&theme=dark"
  style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
  title="magical-dawn-3hw6z"
  allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
  sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

---

Esto es solo una exploración de lo que podría hacerse con los _bookmarks_ de CodeMirror y una biblioteca de componentes como Preact. Algo que creo que vale la pena seguir revisando es ver cómo lograr mantener un estado entre _unmounts_ del mismo componente, o en su defecto hacer un _diff_ inteligente para borrar solo algunos _bookmarks_ y no todos en cada operación.