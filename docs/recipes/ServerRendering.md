# Серверный рендеринг

Наиболее распространенный случай использования серверного рендеринга — это необходимость обрабатывать _первоначальный рендер_, когда пользователь (или поисковый робот) впервые запрашивает наше приложение. Когда сервер получает запрос, он рендерит требуемый(ые) компонент(ы) в HTML-строку и затем посылает в качестве ответа клиенту. С этого момента клиент берет на себя обязанности рендеринга.

Мы будем использовать React в примерах ниже, но эти приемы также могут быть использованы с другими фреймворками, которые умеют рендерить на сервере.

### Redux на сервере

Когда мы используем Redux вместе с серверным рендерингом, мы должны также отправлять состояние нашего приложения вместе с запросом, так клиент может использовать это в качестве начального состояния. Это важно потому что если мы подгружаем любые данные до генерирования HTML, то мы хотим, чтобы клиент также имел доступ к этим данным. В противном случае, сгенерированная на клиенте разметка не будет совпадать с серверной разметкой, и клиенту придется запрашивать данные снова.

Чтобы отправить данные клиенту, нам надо:

* создать свежий, новый экземпляр Redux-хранилища в ответ на каждый запрос;
* (необязательно) отправить некоторые действия;
* вытянуть состояние из хранилища;
* и затем передать состояние вместе с клиентом.

На клиентской стороне новое Redux-хранилище будет создано и инициализировано вместе с состоянием, переданным сервером.
Работа Redux на серверной стороне состоит **_только_** в том, чтобы передать **начальное состояние** нашего приложения.

## Настройка

В следующем рецепте мы собираемся взглянуть на то, как установить рендеринг на серверной стороне. Мы будем использовать упрощенный [Counter app](https://github.com/reactjs/redux/tree/master/examples/counter) пример в качестве руководства и покажем, как сервер может рендерить состояние досрочно на основе запроса.

### Установка пакетов
Для примера, мы будем использовать [Express](http://expressjs.com/) в качестве простого web-сервера. Также нам надо установить зависимости React для Redux, т.к. они не включены по умолчанию в Redux.

```
npm install --save express react-redux
```

## Серверная сторона

Следующим по плану идет вид нашего сервера. Мы собираемся установить [Express middleware](http://expressjs.com/guide/using-middleware.html), сипользуя [app.use](http://expressjs.com/api.html#app.use) для отлова всех запросов, которые поступают на сервер. Если Вы не знакомы с Express или миддлвэром, то знайте, что наша функция handleRender будет вызываться каждый раз, когда сервер будет получать запрос.

##### `server.js`

```js
import path from 'path'
import Express from 'express'
import React from 'react'
import { createStore } from 'redux'
import { Provider } from 'react-redux'
import counterApp from './reducers'
import App from './containers/App'

const app = Express()
const port = 3000

// This is fired every time the server side receives a request
app.use(handleRender)

// We are going to fill these out in the sections to follow
function handleRender(req, res) { /* ... */ }
function renderFullPage(html, preloadedState) { /* ... */ }

app.listen(port)
```

### Отлавливание запроса

Первая вещь, которую нам надо сделать для каждого запроса — это создать новый экзмепляр Redux-хранилища. Единственно назначение этого экземпляра хранилища — это обеспечивать начальное состояние нашего приложения.

Когда будем рендерить, мы обернем наш главный компонент `<App />` в `<Provider>`, чтобы хранилище было доступно всем компонентам дерева, также как мы делали в [Использование с  React](../basics/UsageWithReact.md).

Ключевой шаг в серверном рендеринге — это рендер HTML-разметки компонента _**до**_ отправки на клиент. Чтобы сделать это, мы используем [ReactDOMServer.renderToString()](https://facebook.github.io/react/docs/top-level-api.html#reactdomserver.rendertostring).

Затем мы получаем начальное состояние из нашего Redux-хранилища с помощью [`store.getState()`](../api/Store.md#getState). Мы увидим, как это передается вместе с нашей функцией `renderFullPage`.

```js
import { renderToString } from 'react-dom/server'

function handleRender(req, res) {
  // Create a new Redux store instance
  const store = createStore(counterApp)

  // Render the component to a string
  const html = renderToString(
    <Provider store={store}>
      <App />
    </Provider>
  )

  // Grab the initial state from our Redux store
  const preloadedState = store.getState()

  // Send the rendered page back to the client
  res.send(renderFullPage(html, preloadedState))
}
```

### Внедрение начальной HTML-разметки и состояния

Конечным шагом серверного рендеринга является внедрение нашей начальной разметки компонента и начального состояния в шаблон для рендеринга на клиенте. Чтобы отправить вместе с состоянием, мы добавив `<script>` тэг, который будет прикреплять `preloadedState` к `window.__PRELOADED_STATE__`.

`preloadedState` затем будет доступно на клиенте через `window.__PRELOADED_STATE__`.

Мы также включаем наш bundle-файл для клиента через тег `<script>`. Это любой выход чтобы передать ваши упакованные инструменты на клиентскую стороны. Это может быть статичный файл или URL для быстрой перезагрузки сервера разработки.

```js
function renderFullPage(html, preloadedState) {
  return `
    <!doctype html>
    <html>
      <head>
        <title>Redux Universal Example</title>
      </head>
      <body>
        <div id="root">${html}</div>
        <script>
          window.__PRELOADED_STATE__ = ${JSON.stringify(preloadedState)}
        </script>
        <script src="/static/bundle.js"></script>
      </body>
    </html>
    `
}
```

>##### Помните о синтаксисе интерполяции строк

>В примере выше мы использовали синтаксис [шаблонных строк](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/template_strings) ES6. Это позволило нам писать многострочные строки и интерполировать значения, но это требует поддерки ES6. Если Вы хотите писать Ваш Node-код, используя ES6, посмотрите документацию [Babel require hook](https://babeljs.io/docs/usage/require/). Или же Вы просто можете использовать ES5.

## Клиентская сторона

Клиентская сторона — очень простая. Всё, что нам нужно сделать, — это захватить начальное состояние из `window.__PRELOADED_STATE__`, и передать его в наш [`createStore()`](../api/createStore.md) в качестве начального состояния.

Давайте посмотрим на наш новый клиентский файл:

#### `client.js`

```js
import React from 'react'
import { render } from 'react-dom'
import { createStore } from 'redux'
import { Provider } from 'react-redux'
import App from './containers/App'
import counterApp from './reducers'

// Grab the state from a global injected into server-generated HTML
const preloadedState = window.__PRELOADED_STATE__

// Create Redux store with initial state
const store = createStore(counterApp, preloadedState)

render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
```

Вы можете установить сборщик на Ваш выбор (Webpack, Browserify, и т.д.), чтобы сгенерировать bundle-файл `static/bundle.js`.

Когда страница загружена, bundle-файл запустится, и [`ReactDOM.render()`](https://facebook.github.io/react/docs/top-level-api.html#reactdom.render) «подцепится» к `data-react-id` атрибуту из сгенерированного сервером HTML. Это подсоединит наш недавно запущенный экземпляр React к виртуальному дереву DOM, используемому на сервере. Теперь у нас то же самое начальное состояние Redux-хранилища, и использование этого же кода для всех компонентов позволит получить в точности такой же реальный DOM.

И это всё! Это всё, что нам нужно сделать, чтобы осуществить серверный рендеринг.

Но результат все еще «голый». Это по существу рендерит статическое представление из динамического кода. Что нам надо сделать дальше — это построить начальное состояние динамически чтобы дать возможность этому срендеренному представлению быть динамическим.

## Подготовка начального состояния

Because the client side executes ongoing code, it can start with an empty initial state and obtain any necessary state on demand and over time. On the server side, rendering is synchronous and we only get one shot to render our view. We need to be able to compile our initial state during the request, which will have to react to input and obtain external state (such as that from an API or database).

### Processing Request Parameters

The only input for server side code is the request made when loading up a page in your app in your browser. You may choose to configure the server during its boot (such as when you are running in a development vs. production environment), but that configuration is static.

The request contains information about the URL requested, including any query parameters, which will be useful when using something like [React Router](https://github.com/reactjs/react-router). It can also contain headers with inputs like cookies or authorization, or POST body data. Let's see how we can set the initial counter state based on a query parameter.

#### `server.js`

```js
import qs from 'qs' // Add this at the top of the file
import { renderToString } from 'react-dom/server'

function handleRender(req, res) {
  // Read the counter from the request, if provided
  const params = qs.parse(req.query)
  const counter = parseInt(params.counter, 10) || 0

  // Compile an initial state
  let preloadedState = { counter }

  // Create a new Redux store instance
  const store = createStore(counterApp, preloadedState)

  // Render the component to a string
  const html = renderToString(
    <Provider store={store}>
      <App />
    </Provider>
  )

  // Grab the initial state from our Redux store
  const finalState = store.getState()

  // Send the rendered page back to the client
  res.send(renderFullPage(html, finalState))
}
```

The code reads from the Express `Request` object passed into our server middleware. The parameter is parsed into a number and then set in the initial state. If you visit [http://localhost:3000/?counter=100](http://localhost:3000/?counter=100) in your browser, you'll see the counter starts at 100. In the rendered HTML, you'll see the counter output as 100 and the `__PRELOADED_STATE__` variable has the counter set in it.

### Async State Fetching

The most common issue with server side rendering is dealing with state that comes in asynchronously. Rendering on the server is synchronous by nature, so it's necessary to map any asynchronous fetches into a synchronous operation.

The easiest way to do this is to pass through some callback back to your synchronous code. In this case, that will be a function that will reference the response object and send back our rendered HTML to the client. Don't worry, it's not as hard as it may sound.

For our example, we'll imagine there is an external datastore that contains the counter's initial value (Counter As A Service, or CaaS). We'll make a mock call over to them and build our initial state from the result. We'll start by building out our API call:

#### `api/counter.js`

```js
function getRandomInt(min, max) {
  return Math.floor(Math.random() * (max - min)) + min
}

export function fetchCounter(callback) {
  setTimeout(() => {
    callback(getRandomInt(1, 100))
  }, 500)
}
```

Again, this is just a mock API, so we use `setTimeout` to simulate a network request that takes 500 milliseconds to respond (this should be much faster with a real world API). We pass in a callback that returns a random number asynchronously. If you're using a Promise-based API client, then you would issue this callback in your `then` handler.

On the server side, we simply wrap our existing code in the `fetchCounter` and receive the result in the callback:

#### `server.js`

```js
// Add this to our imports
import { fetchCounter } from './api/counter'
import { renderToString } from 'react-dom/server'

function handleRender(req, res) {
  // Query our mock API asynchronously
  fetchCounter(apiResult => {
    // Read the counter from the request, if provided
    const params = qs.parse(req.query)
    const counter = parseInt(params.counter, 10) || apiResult || 0

    // Compile an initial state
    let preloadedState = { counter }

    // Create a new Redux store instance
    const store = createStore(counterApp, preloadedState)

    // Render the component to a string
    const html = renderToString(
      <Provider store={store}>
        <App />
      </Provider>
    )

    // Grab the initial state from our Redux store
    const finalState = store.getState()

    // Send the rendered page back to the client
    res.send(renderFullPage(html, finalState))
  })
}
```

Because we call `res.send()` inside of the callback, the server will hold open the connection and won't send any data until that callback executes. You'll notice a 500ms delay is now added to each server request as a result of our new API call. A more advanced usage would handle errors in the API gracefully, such as a bad response or timeout.

### Security Considerations

Because we have introduced more code that relies on user generated content (UGC) and input, we have increased our attack surface area for our application. It is important for any application that you ensure your input is properly sanitized to prevent things like cross-site scripting (XSS) attacks or code injections.

In our example, we take a rudimentary approach to security. When we obtain the parameters from the request, we use `parseInt` on the `counter` parameter to ensure this value is a number. If we did not do this, you could easily get dangerous data into the rendered HTML by providing a script tag in the request. That might look like this: `?counter=</script><script>doSomethingBad();</script>`

For our simplistic example, coercing our input into a number is sufficiently secure. If you're handling more complex input, such as freeform text, then you should run that input through an appropriate sanitization function, such as [validator.js](https://www.npmjs.com/package/validator).

Furthermore, you can add additional layers of security by sanitizing your state output. `JSON.stringify` can be subject to script injections. To counter this, you can scrub the JSON string of HTML tags and other dangerous characters. This can be done with either a simple text replacement on the string or via more sophisticated libraries such as [serialize-javascript](https://github.com/yahoo/serialize-javascript).

## Next Steps

You may want to read [Async Actions](../advanced/AsyncActions.md) to learn more about expressing asynchronous flow in Redux with async primitives such as Promises and thunks. Keep in mind that anything you learn there can also be applied to universal rendering.

If you use something like [React Router](https://github.com/reactjs/react-router), you might also want to express your data fetching dependencies as static `fetchData()` methods on your route handler components. They may return [async actions](../advanced/AsyncActions.md), so that your `handleRender` function can match the route to the route handler component classes, dispatch `fetchData()` result for each of them, and render only after the Promises have resolved. This way the specific API calls required for different routes are colocated with the route handler component definitions. You can also use the same technique on the client side to prevent the router from switching the page until its data has been loaded.
