# Руководство для начинающих

## Цель данного руководства

Это руководство предназначено для введения в redux-saga в доступной(надеюсь, что это так) форме.

Первым шагом в изучении этого руководства, мы рассмотрим пример простого счетчика из репозитория Redux. Это приложение довольно простое, но оно хорошо илюстрирует основную мысль redux-saga и начинающий не потеряет сути в излишнем обилии деталей.

### Установка примера

Сначала нам нужно склонировать [репозиторий](https://github.com/yelouafi/redux-saga-beginner-tutorial).

```sh
$ git clone https://github.com/yelouafi/redux-saga-beginner-tutorial.git
```

> Финальный результат кода после выполнения руководства находится в ветке `sagas`.

Затем переходим в корневую директорию репозитория и устанавливаем зависимости:

```sh
$ cd redux-saga-beginner-tutorial
$ npm install
```

Запускаем приложение:

```sh
$ npm start
```
Открываем в браузере: http://192.168.1.65:9966/

Мы запустили элементарный счетчик состоящий из 2 кнопок `Increment` *(увеличить на 1) и `Decrement` *(уменьшить на 1) и поля вывода текущего значения счетчика. В дальнейшим мы познакомимся с асинхронными вызовами.

> Если у вас возникли трудности и не удалось запустить приложение. Вы можете посмотреть возникшие трудности и их решения или же если ваш случай еще не встречался создать свой issue в оригинальном [репозитории с руководством](https://github.com/yelouafi/redux-saga-beginner-tutorial/issues).

## Hello Sagas!

Давайте создадим нашу первую Сагу. Не будем нарушать традиций, и напишем реализацию 'Hello, world' на Сагах.

Создадим файл `sagas.js`, затем добавим в него следующий код:

```javascript
export function* helloSaga() {
  console.log('Hello Sagas!')
}
```

Попрошу отставить панику, это обычная функция (если забыть про `*`). Она выводит сообщение в консоль.

Для того чтобы запустить нашу Сагу, нам нужно:

- создать `redux-saga` middleware, в которую мы передадим список Саг - которые хотим запустить (у нас пока будет всего одна `helloSaga`)
- подключить `redux-saga` middleware в Redux Store

Теперь изменим `main.js`:

```javascript
// ...
import createSagaMiddleware from 'redux-saga'

// ...
import { helloSaga } from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
sagaMiddleware.run(helloSaga)

// остальное без изменений
```

Сначала, мы импортируем наш модуль Саги из `./sagas`. Затем создаем middleware используя [функцию-фабрику](https://ru.wikipedia.org/wiki/Фабричный_метод) `createSagaMiddleware` которая экспортируется для нас из `redux-saga` библиотеки.

Перед запуском нашей `helloSaga`, мы должны подключить созданный middleware в Store используя `applyMiddleware` *(мы его импортируем из Redux). Дальше мы можем использовать `sagaMiddleware.run(helloSaga)` для запуска работы нашей Саги.

Пока наша Сага не делает ничего особенного. Только выводит информационное сообщение.

## Создаем Асинхронные вызовы

Теперь настало время сделать что-то более похожее на то, что мы видели в примере счетчика. Рассмотрим работу асинхронных вызовов, сделаем еще одну кнопку, которая будет увеличивать значение счетчика спустя 1 секунду после нажатия.

Модифицируем наш компонент Counter в `Counter.js` - добавим ему новую кнопку, которая при клике будет вызывать [callback-функцию](https://ru.wikipedia.org/wiki/Callback_(%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)) `onIncrementAsync`

```javascript
const Counter = ({ value, onIncrement, onDecrement, onIncrementAsync }) =>
// ...
  <div>
    {' '}
    <button onClick={onIncrementAsync}>Increment after 1 second</button>
    <hr />
    <div>Clicked: {value} times</div>
  </div>
// ...
```

Дальше реализуем функцию `onIncrementAsync` чтобы она отправляла action в Store.

`main.js` модифицируется следующим образом:

```javascript
function render() {
  ReactDOM.render(
    <Counter
      onIncrementAsync={() => action('INCREMENT_ASYNC')}
    />,
    document.getElementById('root')
  )
}
```

Примечание: в отличие от redux-thunk, наш компонент **отправляет** в action **объект**, а не функцию возвращающую объект.

Сейчас мы создадим еще одну Сагу которая будет выполнять асинхронный вызов. В нашем случае она будет:

> на каждый `INCREMENT_ASYNC` action - запускать функцию, которая:

> - будет ждать 1 секунду, и после этого увиличивать значение счетчика.

Add the following code to the `sagas.js` module:
Добавим следующий код в `sagas.js`:

```javascript
import { takeEvery, delay } from 'redux-saga'
import { put } from 'redux-saga/effects'

// наша Сага-рабочий: выполняет асинхронную работу
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// наша Сага-надзиратель: просматривает поток actions и на каждый action типа: INCREMENT_ASYNC заставляет работать Сагу-рабочего
// *(если точнее: создает новый экземпляр Саги-рабочего, который умирает выполнив свою работу.)
// При takeEvery могут одновременно жить несколько экземпляров Саг-рабочих(близнецов), выполняющих параллельно одну и ту же работу.
export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

Самое время объяснить, что тут происходит.

Мы импортируем `delay`, вспомогательную функцию `redux-saga` возвращающую [Promise](https://learn.javascript.ru/promise) который перейдет в состояние resolve после ожидания переданного значения миллисекунд. Мы будем использовать эту функцию чтобы *блокировать* генератор.

Работа Саг основана на [функциях-генераторах](https://learn.javascript.ru/generator), используя которые `redux-sage middleware` с помощью ключевого слова *yield* приостанавливает выполнение инструкций последовательности действий Саг до тех пор пока не закончится выполнение асинхронных операций и promise(переданный в yield, в котором и происходят асинхронные операции) не перейдет в состояние resolve или reject.В примере выше, Сага `incrementAsync` остановит своё выполнение до тех пор пока promise возвращаемый функцией `delay`, не перейдет в состоянии resolve, в нашем примере это произойдет через 1 секунду.

Когда `promise` перейдет в состояние `resolve`, `middleware` продолжит выполнение саги, код будет продолжать выполняться до тех пор, пока не встретит следующий `yield`.
В этом примере,следующая остановка выполнения кода произойдет до получения результата вызова: `put({type: 'INCREMENT'})`, который информирует `middleware` о отравке `INCREMENT` action.

`put` это один из примеров того, как мы можем вызывать *эффекты*. Эффекты - это простые JavaScript объекты, которые содержат инструкции, которые нужно выполнить middleware. Когда middleware получает эффект - она приостанавливает дальнейшее выполнение Саги, до тех пор, пока эфект не будет выполнен.

Подведем итог, `incrementAsync` сага ждет 1 секунду с помощью вызова `delay(1000)`, затем отправляет  `INCREMENT` action.

Создадим другую Сагу `watchIncrementAsync`. Мы использовали `takeEvery`, вспомогательную функцию предоставляемую `redux-saga`, чтобы начать прослушивать
отправляемые `INCREMENT_ASYNC` actions и запускает на каждый action этого типа `incrementAsync`.

Сейчас у нас есть 2 Саги, и нам нужно чтобы они работали обе одновременно. Чтобы это сделать , нам необходимо добавить в `rootSaga` запуск еще одной Саги.
В файл `sagas.js` добавим следующий код:

```javascript
// можно сделать одну точку запуска всех саг
export default function* rootSaga() {
  yield [
    helloSaga(),
    watchIncrementAsync()
  ]
}
```

Эта в этой Сага мы можем увидеть, как можно передать в `yield` массив который вернет результат вызовов наших двух Саг,`helloSaga` и `watchIncrementAsync`.
Таким образом мы можем осуществлять параллельное выполнение с ожиданием завершения всех параллельных действий перед дальнейшим выполнением кода. Сейчас нам всего-лишь запустить `sagaMiddleware.run` и передать в неё `rootSaga` в файле `main.js`.

```javascript
// ...
import rootSaga from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = ...
sagaMiddleware.run(rootSaga)

// ...
```

## Making our code testable

We want to test our `incrementAsync` Saga to make sure it performs the desired task.

Create another file `sagas.spec.js`:

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

Since `incrementAsync` is a Generator function, when we run it outside the middleware,
Each time you invoke `next` on the generator, you get an object of the following shape

```javascript
gen.next() // => { done: boolean, value: any }
```

The `value` field contains the yielded expression, i.e. the result of the expression after
the `yield`. The `done` field indicates if the generator has terminated or if there are still
more 'yield' expressions.

In the case of `incrementAsync`, the generator yields 2 values consecutively:

1. `yield delay(1000)`
2. `yield put({type: 'INCREMENT'})`

So if we invoke the next method of the generator 3 times consecutively we get the following
results:

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

The first 2 invocations return the results of the yield expressions. On the 3rd invocation
since there is no more yield the `done` field is set to true. And since the `incrementAsync`
Generator doesn't return anything (no `return` statement), the `value` field is set to
`undefined`.

So now, in order to test the logic inside `incrementAsync`, we'll simply have to iterate
over the returned Generator and check the values yielded by the generator.

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next(),
    { done: false, value: ??? },
    'incrementAsync should return a Promise that will resolve after 1 second'
  )
});
```

The issue is how do we test the return value of `delay`? We can't do a simple equality test
on Promises. If `delay` returned a *normal* value, things would've been be easier to test.

Well, `redux-saga` provides a way to make the above statement possible. Instead of calling
`delay(1000)` directly inside `incrementAsync`, we'll call it *indirectly*:

```javascript
// ...
import { put, call } from 'redux-saga/effects'
import { delay } from 'redux-saga'

export function* incrementAsync() {
  // use the call Effect
  yield call(delay, 1000)
  yield put({ type: 'INCREMENT' })
}
```

Instead of doing `yield delay(1000)`, we're now doing `yield call(delay, 1000)`. What's the difference?

In the first case, the yield expression `delay(1000)` is evaluated before it gets passed to the caller of `next` (the caller could be the middleware when running our code. It could also be our test code which runs the Generator function and iterates over the returned Generator). So what the caller gets is a Promise, like in the test code above.

In the second case, the yield expression `call(delay, 1000)` is what gets passed to the caller of `next`. `call` just like `put`, returns an Effect which instructs the middleware to call a given function with the given arguments. In fact, neither `put` nor `call` performs any dispatch or asynchronous call by themselves, they simply return plain JavaScript objects.

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

What happens is that the middleware examines the type of each yielded Effect then decides how to fulfill that Effect. If the Effect type is a `PUT` then it will dispatch an action to the Store. If the Effect is a `CALL` then it'll call the given function.

This separation between Effect creation and Effect execution makes it possible to test our Generator in a surprisingly easy way:

```javascript
import test from 'tape';

import { put, call } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    call(delay, 1000),
    'incrementAsync Saga must call delay(1000)'
  )

  assert.deepEqual(
    gen.next().value,
    put({type: 'INCREMENT'}),
    'incrementAsync Saga must dispatch an INCREMENT action'
  )

  assert.deepEqual(
    gen.next(),
    { done: true, value: undefined },
    'incrementAsync Saga must be done'
  )

  assert.end()
});
```

Since `put` and `call` return plain objects, we can reuse the same functions in our test code. And to test the logic of `incrementAsync`, we simply iterate over the generator and do `deepEqual` tests on its values.

In order to run the above test, run:

```sh
$ npm test
```

which should report the results on the console.
