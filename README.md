# redux-saga-in-russian

Original [Redux-saga](https://yelouafi.github.io/redux-saga/index.html) documentation with a translation into Russian

[![Join the chat at https://gitter.im/yelouafi/redux-saga](https://badges.gitter.im/yelouafi/redux-saga.svg)](https://gitter.im/yelouafi/redux-saga?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![npm version](https://img.shields.io/npm/v/redux-saga.svg?style=flat-square)](https://www.npmjs.com/package/redux-saga) [![CDNJS](https://img.shields.io/cdnjs/v/redux-saga.svg?style=flat-square)](https://cdnjs.com/libraries/redux-saga)

`redux-saga` это библиотека,цель которой сделать работу с [побочными эффектами](https://ru.wikipedia.org/wiki/Побочный_эффект_(программирование) (таких как: асинхронные операции, работа с "[грязными](https://ru.wikipedia.org/wiki/%D0%A7%D0%B8%D1%81%D1%82%D0%BE%D1%82%D0%B0_%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8)" функциями) в React/Redux приложениях проще и удобнее. 

Идея `redux-saga` направлена на создание отдельного дополнительного звена в [потокe](https://rajdee.gitbooks.io/redux-in-russian/content/docs/basics/DataFlow.html) в вашем приложении, в котором будет осуществляться работа с побочными эфектами.
`redux-saga` это [middleware](https://rajdee.gitbooks.io/redux-in-russian/content/docs/advanced/Middleware.html) для [redux](https://github.com/rajdee/redux-in-russian), которая предоставляет возможность запустить, приостановить или отменить поток из основного приложения с обычными redux [actions](https://rajdee.gitbooks.io/redux-in-russian/content/docs/basics/Actions.html), сага имеет доступ к redux состоянию приложения и может сама инициировать отправление redux actions.

Сага использует возможности синтаксиса ES6(ES2015) - [генераторы](https://learn.javascript.ru/generator) *(если вы еще не владеете генераторами - чтобы разобраться с ними нужно знать как работает [promise](https://learn.javascript.ru/promise)), чтобы сделать поток асинхронных операций простым для чтения, написания и тестирования. Таким образом поток асинхронного кода будет выглядеть как обычный синхронный JavaScript код. Что-то вроде [ES2017-async/await](https://github.com/tc39/ecmascript-asyncawait) - только генераторы имеют несколько большую гибкость, в обмен на большую сложность исполнения *(но в нашем случае она скрыта внутри реализации саги)

Возможно раньше вы пользовались `redux-thunk` для обработки данных после ваших запросов. В отличие от `redux-thunk` работая с сагами ваш код со временем не превратится в [callback hell](http://i.imgur.com/EGGwaXP.png), вы можете легко тестировать ваш асинхронный код и ваши actions будут лаконичными.

# Быстрый старт!

## Установка

```sh
$ npm install --save redux-saga
```

Alternatively, you may use the provided UMD builds directly in the `<script>` tag of an HTML page. See [this section](#using-umd-build-in-the-browser).
Так же вы можете подключить с помощью UMD([Universal Module Definition](https://github.com/umdjs/umd))  библиотеки сагу непосредственно в тег `<script>` в HTML странице. Подробнее можно посмотреть [ниже](#using-umd-build-in-the-browser).

## Пример использования

Предполагается что у нас реализован пользовательский интерфейс, запрашивающий некоторый данные с сервера при клике на кнопку. ( Рассмотрим action запускающий сагу - второстепенное оставляем за кадром)

```javascript
class UserComponent extends React.Component {
  ...
  onSomeButtonClicked() {
    const { userId, dispatch } = this.props
    dispatch({type: 'USER_FETCH_REQUESTED', payload: {userId}})
  }
  ...
}
```

Компонент отправляет action в [Store](https://rajdee.gitbooks.io/redux-in-russian/content/docs/basics/Store.html). Далее создаем Сагу, которая слушает все `USER_FETCH_REQUESTED` actions и вызывает функцию взаимодействующую с [API](https://ru.wikipedia.org/wiki/API) для получения пользовательских данных.

#### `sagas.js`

```javascript
import { takeEvery, takeLatest } from 'redux-saga'
import { call, put } from 'redux-saga/effects'
import Api from '...'

// Сага: её вызов произойдет тогда, когда сработает USER_FETCH_REQUESTED action
function* fetchUser(action) {
   try {
      const user = yield call(Api.fetchUser, action.payload.userId);
      yield put({type: "USER_FETCH_SUCCEEDED", user: user});
   } catch (e) {
      yield put({type: "USER_FETCH_FAILED", message: e.message});
   }
}

/*
  Начинается вызов fetchUser на каждый `USER_FETCH_REQUESTED` action.
  takeEvery - позволяет параллельное выполнение запросов, если новый action пришел до выполнения саги от предыдущего
*/
function* mySaga() {
  yield* takeEvery("USER_FETCH_REQUESTED", fetchUser);
}

/*
  Есть альтернатива: использовать takeLatest.

  takeLatest - не позволяет параллельного выпольнения запроса. Если пришел новый action
  в то время как сага от предыдущего находится в состоянии ожидания ответа от сервера,
  то она будет отменена и будет получена только самая актуальная информация(от последнего запроса)
*/
function* mySaga() {
  yield* takeLatest("USER_FETCH_REQUESTED", fetchUser);
}

export default mySaga;
```

Чтобы включить работу саг, необходимо подключить её в Redux Store используя `redux-saga` middleware.

#### `main.js`

```javascript
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

import reducer from './reducers'
import mySaga from './sagas'

// создаем сагу - middleware
const sagaMiddleware = createSagaMiddleware()
// подключаем её в Store
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)

// Затем, запускаем работу саги
sagaMiddleware.run(mySaga)

// [render](https://ru.wikipedia.org/wiki/%D0%A0%D0%B5%D0%BD%D0%B4%D0%B5%D1%80%D0%B8%D0%BD%D0%B3) приложения
```

# Документация

- [Введение](http://yelouafi.github.io/redux-saga/docs/introduction/BeginnerTutorial.html)
- [Основы](http://yelouafi.github.io/redux-saga/docs/basics/index.html)
- [Продвинутое использование](http://yelouafi.github.io/redux-saga/docs/advanced/index.html)
- [Рецепты](http://yelouafi.github.io/redux-saga/docs/recipes/index.html)
- [Внешние ресурсы](http://yelouafi.github.io/redux-saga/docs/ExternalResources.html)
- [Возможные трудности](http://yelouafi.github.io/redux-saga/docs/Troubleshooting.html)
- [Словарь терминов](http://yelouafi.github.io/redux-saga/docs/Glossary.html)
- [API справочник](http://yelouafi.github.io/redux-saga/docs/api/index.html)

# Переводы

- [Chinese](https://github.com/superRaytin/redux-saga-in-chinese)
- [Chinese Traditional](https://github.com/neighborhood999/redux-saga)

# Использование UMD библиотеки для браузеров.

Так же возможна **umd** библиотека `redux-saga` доступная по пути `dist/`. Когда используется umd библиотека `redux-saga` то она будет доступна как `ReduxSaga` в глобальном объектe window.

Использование umd версии может пригодиться если вы не используете сборщики проектов (Webpack/Gulp/Grunt/Browserify и др.)

Доступны Следующие типы:

- [https://unpkg.com/redux-saga/dist/redux-saga.js](https://unpkg.com/redux-saga/dist/redux-saga.js)  
- [https://unpkg.com/redux-saga/dist/redux-saga.min.js](https://unpkg.com/redux-saga/dist/redux-saga.min.js)


**Important!** If the browser you are targeting doesn't support *ES2015 generators*, you must provide a valid polyfill, such as [the one provided by `babel`](). The polyfill must be imported before **redux-saga**:
**Важно!** Если ваша целевая аудитория использует браузеры не поддерживающие *ES2015 генераторы*, тогда вам нужно использовать [полифилл](https://learn.javascript.ru/dom-polyfill), например:
['babel'](https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.25/browser-polyfill.min.js). Полифилл должен находиться **до** `redux-saga`

```javascript
import 'babel-polyfill'
// затем
import sagaMiddleware from 'redux-saga'
```

# Сборка примеров из исходников

```sh
$ git clone https://github.com/yelouafi/redux-saga.git
$ cd redux-saga
$ npm install
$ npm test
```

Примеры ниже делались на основе примеров репозитория Redux.

> Примеры: `npm run ...` запускать из корневой директории проекта

### Примеры счетчиков

Ниже находятся 3 примера счетчиков.

#### counter-vanilla

Демонстрация использования нативного JavaScript и UMD библиотеки. Все находится в `index.html`.

To launch the example, just open `index.html` in your browser.
Чтобы запустить пример, нужно просто открыть `index.html` в вашем браузере.

> Важно: ваш браузер должен поддерживать генераторы. Последние версии Chrome/Firefox/Edge подойдут.

#### counter

Демонстрация использует `webpack` и высокоуровневый API `takeEvery`.

```sh
$ npm run counter

# пример теста для генератора
$ npm run test-counter
```

#### cancellable-counter

Демонстрация Использования низкоуровнего API для демонстрации отмены задач.

```sh
$ npm run cancellable-counter
```

### Shopping Cart example

Демонстрация исполнения тележки покупателя интернет магазина.

```sh
$ npm run shop

# пример теста для генератора
$ npm run test-shop
```

### async example

Демонстрация исполнения асинхронного запроса для получения данных с удаленного сервера.

```sh
$ npm run async

# пример теста для генератора
$ npm run test-async
```

### real-world example (with webpack hot reloading)

Демонстрация поиска аккаунта на GitHub с выводом аватара указанного пользователя и список репозиториев, которым пользователь поставил звезду + мониторинг состояния приложения, возвращение на предыдущее состояние и его обнуление.

```sh
$ npm run real-world

# извините, тестов нет
```
