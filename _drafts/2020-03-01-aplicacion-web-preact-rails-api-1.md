---
layout: post
title: "Creando una aplicación web con Preact y Rails API (I)"
date: 2020-03-01 10:30:00 -0500
excerpt:
image:
---

Los _side projects_ ayudan a poder probar tecnologías nuevas. ¿Qué tal si quiero cambiar una parte del stack? Invertir ese tiempo en el trabajo no es muy rentable, a menos que existan razones suficientes para ello. E incluso así, hacerlo también implica cambiar la forma de trabajo del resto del equipo.

Por eso ahora quiero probar haciendo un side project, con un stack diferente, e ir aprendiendo en el camino. Y para esto quiero usar [Preact](https://preactjs.com/) y [Ruby on Rails API](https://guides.rubyonrails.org/api_app.html). ¿Por qué?

Siempre me ha gustado Preact, lo he usado algunas veces, y más allá de la ventaja de ser _solo_ 3kB, quisiera probar otra forma de crear aplicaciones basadas en componentes. Y he usado Ruby on Rails API en los últimos proyectos en el trabajo, pero nunca he estado desde el `rails new`, así que **quiero usar esta oportunidad para aprender** esos pequeños detalles de la configuración que se pierden luego de unos meses de empezado el trabajo.

Entonces, ¿de qué trata mi _side project_?

## El proyecto

<section class="two-columns">
  <aside class="mermaid">
  classDiagram
    class User
    User: +String username
    User: +String email
    User: +String provider
    User: +String uid
    User: +Integer hourlyRate

    class Project
    Project: +User createdBy
    Project: +String name
    Project: +String description
    Project: +Integer budgeted
    Project: +Integer actual
    Project: +String status

    class ProjectTask
    ProjectTask: +Project project
    ProjectTask: +String content
    ProjectTask: +Integer hours
    ProjectTask: +Boolean done

    User "1" --> "0..*" Project
    Project "1" --> "1..*" ProjectTask
  </aside>
  <aside>
    <p>El proyecto se llama <strong>Simple Budget</strong>. Es una aplicación web orientada a freelancers que permitirá crear estimados para proyectos de software basados en horas.</p>
    <p>Cada usuario va a poder crear proyectos, crear las tareas para cada proyecto, y definir el número de horas que estima va a demorar cada tarea.</p>
    <p>Este diagrama representa cada modelo, los atributos que debería tener y como se relacionan entre sí.</p>
  </aside>
</section>

## Haciendo seguimiento

Antes de empezar a instalar dependencias y escribir código, necesito definir las tareas que voy a hacer. Esto me va a servir para tener una idea de a dónde quiero ir con el proyecto y mantener foco en lo que necesito hacer.

Para esto probaré **GitHub Projects**, que es una herramienta gratuita que ya viene integrada con los repositorios, issues y pull requests de GitHub. Así voy a mantener toda la información de mi proyecto en un solo lugar, e incluso podré tener algunos procesos automatizados, como el pasar una tarea de "To do" a "In progress" si un pull request es abierto y asociado a dicha tarea.

![Mis primeras tareas en Github Projects](/assets/images/github-projects.png)

## Empezando el proyecto

Para este proyecto utilizaré la fórmula de API + SPA.

La API (Application Programming Interface) estará hecha en Ruby on Rails utilizando [JSON:API](https://jsonapi.org/). Es la especificación que usamos en el trabajo y me ha gustado como se encarga de tomar decisiones con respecto al estilo de las respuestas, o cómo incluir información extra de un modelo (como sus asociaciones).

La SPA (Single Page Application) estará hecha con Preact. En el caso del front, es un terreno nuevo para mí, en el sentido que hay muchas opciones disponibles, y no hay un stack establecido, como sí es el caso de Rails. También usaré JSON:API para consumir el backend.

En algunos proyectos he visto que se usa Ruby on Rails con [Webpacker](https://github.com/rails/webpacker) para integrar una SPA sobre una vista de Rails, y de paso encargarse del SSR (Server Side Rendering), pero en este caso prefiero tener dos aplicaciones por separado para probar un enfoque nuevo. No creo que SSR sea algo que importe en este proyecto en particular, pero si fuera el caso, posiblemente utilice este enfoque de Rails + Webpacker.

### Frontend con Preact

Para empezar la creación del frontend, decidí utilizar [`preact-cli`](https://github.com/preactjs/preact-cli). Es similar a [`create-react-app`](https://github.com/facebook/create-react-app), en el sentido que genera una base sobre la cual trabajar. Siento que el código y la configuración _boilerplate_ de `preact-cli` es más simple que la de `create-react-app` y requiere un poco más de trabajo para ajustarlo a mis gustos.

Para usar `preact-cli` debo ejecutar estos comandos:

```bash
npm install -g preact-cli
preact create simple-budget
```

> **Nota:** La última versión estable de `preact-cli` en este momento es la 2.2.1, la cual contiene Preact 8. Preact X es la nueva versión de Preact y trae nuevas funcionalidades:
>
> * Fragments
> * componentDidCatch
> * preact/hooks addon
> * preact/test-utils addon
> * createContext API
>
> He querido probar Preact X desde hace mucho tiempo, así que utilizaré un comando para actualizar Preact a su última versión:
>
> ```bash
> npm i -D preact-cli@rc && npm i preact@latest preact-router@latest preact-render-to-string@latest && npm rm preact-compat
> ```

Luego, ejecutaré el siguiente comando para levantar la aplicación:

```bash
npm run dev # o "yarn dev" si uso Yarn
```

### Backend con Ruby on Rails API

Para usar Ruby on Rails en modo API debo ejecutar el siguiente comando:

```bash
rails new simple-budget-api --api --database=postgresql
```

La implementación que usaré en Rails será [JSONAPI::Resources](http://jsonapi-resources.com/), con la que estoy más familiarizado, y se integra bastante bien con Rails. Para eso agregaré la gema `jsonapi-resources` en mi Gemfile:

```ruby
gem 'jsonapi-resources', '0.9.10'
```

Luego debo instalar la dependencia ejecutando:

```bash
bundle install
```

Por último, para iniciar la aplicación debo ejecutar:

```bash
rails server
```

---

Hasta aquí, ya tengo la base de ambas aplicaciones, lo que sigue es empezar con las tareas definidas en el tablero del proyecto en GitHub.