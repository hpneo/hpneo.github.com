---
layout: post
title:  "Usando Higher-Order Components"
date:   2019-03-19 01:15:57 -0600
excerpt: "A veces necesitamos reutilizar lógica dentro de varios componentes de React. Los Higher-Order Components son una manera elegante de lograrlo."
# categories: jekyll update
---

![Usando Higher-Order Components](/assets/images/hocs.png)

A veces debemos reutilizar lógica dentro de varios componentes de React.

Por ejemplo, tengo dos problemas:

* ¿Qué pasa si quiero acceder a la información del usuario logueado para ocultar / mostrar componentes según su nivel de acceso?
* Tengo una aplicación multi-región donde necesito mostrar información de moneda y fechas según el idioma del usuario.

Y esa implementación la tengo que hacer, no solo en una, si no en **varios** componentes. ¿Qué opciones tengo?

Bueno, una de ellas es pasar _props_ de componentes padre a hijo, pero quizá no es la solución ideal porque tendrías que pasar _props_ en componentes que no los necesitan:

```jsx
const App = ({ currentUser }) => (
  <div>
    <Header />
    <Body>
      <Dashboard currentUser={currentUser} />
      <Settings currentUser={currentUser} />
    </Body>
  </div>
);

// Usando el componente:
<App currentUser={...} />
```

Por ejemplo, `<App />` no necesita el _prop_ `currentUser` pero es necesario pasarlo ahí para que `<Dashboard />` y `<Settings />` lo utilicen, y puede que internamente en estos componentes pase lo mismo.

Otra solución podría ser crear componentes utilitarios que guarden esa información y que contengan lógica, pero esta solución obliga a utilizar un componente por cada pequeña porción de tu aplicación que requiera hacer este cambio:

```jsx
const Dashboard = ({ currency }) => (
  <div>
    <p>Tu saldo total es <FormatCurrency currency={currency} value={150.25} /></p>
    <p>Debes <FormatCurrency currency={currency} value={10.10} /></p>
  </div>
);

// Usando el componente:
<Dashboard currency='USD' />
```

Y en este caso, ¡sigues repitiendo código! Cada vez que necesitamos darle formato de moneda a un número, tenemos que llamar al componente `<FormatCurrency />`, pasándole el _prop_ `currency`.

Aquí es donde entran los [**Higher-Order Components**](https://reactjs.org/docs/higher-order-components.html).

Un Higher-Order Components es, como dice la documentación oficial:

> [...] **a function that takes a component and returns a new component**.

Es decir, una función que toma un componente como parámetro y retorna un nuevo componente.

Un Higher-Order Component (o HOC) encapsula lógica y crea un componente intermedio que renderiza el componente que pasas como parámetro, con los mismos _props_ que recibiría el componente original, y posiblemente agregando nuevos _props_, propios del HOC.

Por ejemplo, hagamos un HOC que lee la URL actual y pasa los _search params_ como _prop_:

```jsx
const withLocationInfo = (Component) => {
  class ComponentWithLocationInfo extends React.Component {
    render() {
      // Leo la URL de la página actual
      const url = new URL(document.location);
      // Obtengo los search params y los convierto a un array con Array.from
      const searchParams = Array.from(url.searchParams.entries());

      // Renderizo el componente original con sus props + un nuevo prop llamado searchParams
      return <Component {...this.props} searchParams={searchParams} />;
    }
  }

  return ComponentWithLocationInfo;
};

export default withLocationInfo;
```

> **Nota**: Ten en cuenta que `Array.from(url.searchParams.entries())` devuelve un array del tipo `[['q', 'react'], ['topic', 'javascript']]`.

Y para utilizarlo, defino un componente y llamo al HOC de la siguiente forma:

```jsx
class SearchBox extends React.Component {
  render() {
    // Leo el prop searchParams agregado por el HOC
    const { searchParams } = this.props;
    // Utilizo [].find para obtener el search param con el nombre query
    const query = searchParams.find(pair => pair[0] === 'query');

    return <input name='query' value={query[1]} />;
  }
}

// Ejecuto el HOC withLocationInfo y le paso SearchBox como argumento
export default withLocationInfo(SearchBox);
```

Para luego utilizar el componente en:

```jsx
class SearchForm extends React.Component {
  render() {
    return (
      <form>
        <SearchBox />
        <input type='submit' value='Buscar' />
      </form>
    );
  }
}
```

Pero tener accesibles los _search params_ no solo sirven para cajas de búsqueda, también puedes usarlo para resaltar palabras dentro de un texto, mostrar un encabezado, hacer un paginador, etc. La ventaja es que la lógica que hace el _trabajo sucio_ está encapsulado en una sola función y solo debemos llamarla para aumentar las capacidades de nuestro componente mediante nuevos _props_.

---

Ahora veamos cómo resolver las situaciones presentadas originalmente:

> **¿Qué pasa si quiero acceder a la información del usuario logueado para ocultar / mostrar componentes según su nivel de acceso?**

Podemos crear un HOC que tome la información del usuario logueado de alguna fuente (digamos, Redux o el Local Storage):

```jsx
const withCurrentUserInfo = (Component) => {
  class ComponentWithCurrentUserInfo extends React.Component {
    render() {
      // Leemos la información guardada en el Local Storage, si existe
      const currentUser = localStorage['CURRENT_USER']
        ? JSON.parse(localStorage['CURRENT_USER'])
        : {};

      return <Component {...this.props} currentUser={currentUser} />;
    }
  }

  return ComponentWithCurrentUserInfo;
}

export default withCurrentUserInfo;
```

O en el caso de utilizar Redux, podemos hacer algo como:

```jsx
const withCurrentUserInfo = (Component) => {
  class ComponentWithCurrentUserInfo extends React.Component {
    render() {
      const { currentUser } = this.props;

      return <Component {...this.props} currentUser={currentUser} />;
    }
  }

  const mapStateToProps = (state) => {
    return {
      currentUser: state.auth.currentUser,
    };
  };

  return connect(mapStateToProps)(ComponentWithCurrentUserInfo);
}

export default withCurrentUserInfo;
```

En ambos casos, podemos agregar más lógica y devolver otros _props_ si fuera necesario (por ejemplo, un _prop_ `isAdmin` para saber si `currentUser` es un usuario administrador, o un _prop_ `can` que sea un función que verifique permisos).

Con este HOC, puedes encapsular la lógica de acceso a una fuente de datos para obtener información del usuario logueado y aumentar las capacidades del componente original sin repetir código.

---

> **Tengo una aplicación multi-región donde necesite mostrar información de moneda y fechas según el idioma del usuario.**

Para este caso en particular, podemos utilizar la misma mecánica del HOC anterior y agregarle unos _props_ que permitan mostrar la información según el idioma del usuario:

```jsx
const LANGUAGES = {
  mx: 'es-MX',
  us: 'en-US',
  pe: 'es-PE',
};

const CURRENCIES = {
  mx: 'MXN',
  us: 'USD',
  pe: 'PEN',
};

const withLocalization = (Component) => {
  class ComponentWithLocalization extends React.Component {
    render() {
      // Obtenemos la información del usuario logueado (que viene de withCurrentUserInfo)
      const { currentUser } = this.props;

      // Obtenemos su país, idioma y tipo de moneda
      const country = currentUser.country;
      const language = LANGUAGES[country];
      const currency = CURRENCIES[country];

      // Definimos funciones de formato de moneda y fecha con la Web API Intl
      const formatCurrency = (value) =>
        new Intl.NumberFormat(language, { style: 'currency', currency: currency }).format(value);

      const formatDate = (value) =>
        new Intl.DateTimeFormat(language).format(value);

      return (
        <Component
          {...this.props}
          formatCurrency={formatCurrency}
          formatDate={formatDate}
        />
      );
    }
  }

  return withCurrentUserInfo(ComponentWithLocalization);
}

export default withLocalization;
```

> **Nota**: [`Intl.NumberFormat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/NumberFormat) y [`Intl.DateTimeFormat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DateTimeFormat) son Web APIs _relativamente_ nuevas pero muy interesantes.

Y utilizar el HOC de la siguiente manera:

```jsx
class ExpensesDetail extends React.Component {
  render() {
    const { formatCurrency, formatDate } = this.props;

    return (
      <table>
        <thead>
          <tr>
            <th>Concepto</th>
            <th>Monto</th>
            <th>Fecha</th>
          </tr>
        </thead>
        <tbody>
          {
            expenses.map(expense => (
              <tr key={expense.id}>
                <td>{expense.description}</td>
                <td>{formatCurrency(expense.amount)}</td>
                <td>{formatDate(expense.created_at)}</td>
              </tr>
            ))
          }
        </tbody>
      </table>
    );
  }
}

export default withLocalization(ExpensesDetail);
```

De esta forma, conseguimos encapsular lógica de formato de monedas y fechas en un solo componente, y así poder reutilizarlo en otros componentes cuando sea necesario (por ejemplo, para darle formato a las fechas de un log de actividades, o para dar formato de moneda a información de saldos).