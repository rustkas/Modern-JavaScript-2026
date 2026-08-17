# Часть II. Современный язык JavaScript

## Глава 5. Функции как основа современной архитектуры JavaScript

Функции всегда были одной из сильнейших сторон JavaScript, однако к 2026 году их роль значительно расширилась. Сегодня именно функции лежат в основе асинхронного программирования, реактивных моделей, обработки потоков данных, модульной композиции и большинства API Web Platform.

В современном JavaScript функция — это не просто исполняемый блок кода, а самостоятельный объект языка, способный хранить состояние, участвовать в композиции вычислений и выступать универсальным интерфейсом между различными уровнями приложения.

---

## 5.1. Функции как объекты первого класса

JavaScript остаётся одним из немногих массовых языков, где функции являются полноценными объектами первого класса (*First-Class Functions*). Это означает, что функции могут использоваться точно так же, как и любые другие значения — числа, строки, объекты.

**Что можно делать с функциями как с объектами первого класса:**

- **Передавать функции в качестве параметров** — функция может принимать другую функцию как аргумент. Это основа callback-стиля и higher-order функций.

```javascript
function processData(data, callback) {
  const result = data.map(item => item * 2);
  callback(result);
}
```

- **Возвращать функции из других функций** — функция может создавать и возвращать новую функцию. Это основа фабрик функций и замыканий.

```javascript
function createMultiplier(factor) {
  return function(value) {
    return value * factor;
  };
}
const double = createMultiplier(2);
```

- **Хранить функции в структурах данных** — функции можно добавлять в массивы, объекты, Map, Set.

```javascript
const handlers = {
  success: (data) => console.log('Success:', data),
  error: (err) => console.error('Error:', err),
};
```

- **Создавать фабрики функций** — функции, генерирующие другие функции с заданным поведением.

```javascript
function createLogger(prefix) {
  return function(message) {
    console.log(`[${prefix}] ${message}`);
  };
}
const infoLogger = createLogger('INFO');
```

- **Композировать поведение во время выполнения** — динамическое объединение функций для создания нового поведения.

```javascript
function compose(f, g) {
  return function(x) {
    return f(g(x));
  };
}
```

Именно благодаря этим свойствам выросла вся современная архитектура языка — от Promise API (где функции передаются в `.then()`) до React (компоненты как функции), Vue (composition API), Node.js (middleware на основе функций) и большинства современных библиотек.

---

## 5.2. Лексическое окружение и замыкания

Замыкания (*Closures*) являются фундаментальным механизмом JavaScript. Каждая функция сохраняет ссылку на своё лексическое окружение — набор переменных, доступных в области видимости, где функция была объявлена.

**Как работает замыкание:**

Когда функция объявляется, движок JavaScript создаёт внутреннюю ссылку на лексическое окружение (Lexical Environment), в котором она была создана. Эта ссылка сохраняется внутри функции и называется внутренним слотом `[[Environment]]`. Даже после того, как внешняя функция завершила выполнение и её переменные, казалось бы, должны быть уничтожены, внутренняя функция продолжает хранить ссылку на них через своё лексическое окружение.

```javascript
function createCounter() {
  let count = 0;
  return function() {
    count++;
    return count;
  };
}
const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
// Переменная count продолжает существовать благодаря замыканию
```

**Где используются замыкания:**

- **Фабрики объектов** — создание объектов с приватным состоянием.

```javascript
function createUser(name) {
  let _name = name;
  return {
    getName: () => _name,
    setName: (newName) => { _name = newName; }
  };
}
```

- **Приватное состояние** — инкапсуляция данных, недоступных извне.

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance;
  return {
    deposit: (amount) => { balance += amount; },
    withdraw: (amount) => {
      if (amount <= balance) { balance -= amount; }
    },
    getBalance: () => balance
  };
}
```

- **Мемоизация** — кеширование результатов функций.

```javascript
function memoize(fn) {
  const cache = new Map();
  return function(arg) {
    if (cache.has(arg)) return cache.get(arg);
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}
```

- **Middleware** — цепочки обработчиков в веб-фреймворках.

```javascript
function logger(req, next) {
  console.log('Request:', req.url);
  next(req);
}
```

- **Обработчики событий** — сохранение состояния в колбэках.

```javascript
function setupButton(button) {
  let clicks = 0;
  button.addEventListener('click', () => {
    clicks++;
    console.log(`Clicked ${clicks} times`);
  });
}
```

- **Функциональная композиция** — объединение функций для создания новых.

Замыкания реализованы через Environment Records — структуры, которые хранят привязки между именами переменных и их значениями. Каждый раз, когда создаётся функция, она получает ссылку на текущий Environment Record через внутренний слот `[[Environment]]`. Когда функция вызывается, создаётся новый Execution Context, и его Lexical Environment связывается с сохранённым `[[Environment]]`.

---

## 5.3. Стрелочные функции и современная модель `this`

Стрелочные функции стали стандартом современной разработки. Они были введены в ES6 и значительно упростили работу с функциями в JavaScript.

**Ключевые особенности стрелочных функций:**

- **Лексическое наследование `this`** — стрелочная функция не имеет собственного `this`. Вместо этого она использует `this` из окружающей области видимости (лексическое окружение). Это устраняет необходимость в `bind()`, `self = this` или `that = this`.

```javascript
// С обычной функцией
function Timer() {
  this.seconds = 0;
  setInterval(function() {
    this.seconds++; // this — глобальный объект или undefined
  }, 1000);
}

// Со стрелочной функцией
function Timer() {
  this.seconds = 0;
  setInterval(() => {
    this.seconds++; // this — экземпляр Timer
  }, 1000);
}
```

- **Отсутствие собственного `arguments`** — стрелочная функция не имеет объекта `arguments`. Если нужен доступ к аргументам, следует использовать rest-параметры: `(...args) => {}`.

- **Отсутствие `prototype`** — стрелочные функции не имеют свойства `prototype`, поэтому они не могут быть использованы как конструкторы.

- **Невозможность использования через `new`** — стрелочная функция не может быть вызвана с оператором `new`, так как у неё нет внутреннего метода `[[Construct]]`.

```javascript
const Arrow = () => {};
new Arrow(); // TypeError: Arrow is not a constructor
```

**Когда обычная функция остаётся предпочтительным выбором:**

- **Методы объектов** — когда требуется динамический `this`, указывающий на объект.

```javascript
const obj = {
  name: 'Object',
  getName: function() { return this.name; } // Обычная функция
};
```

- **Конструкторы** — для создания объектов через `new`.

```javascript
function Person(name) { this.name = name; } // Обычная функция
```

- **Пользовательские классы** — методы классов по умолчанию используют обычные функции.

```javascript
class MyClass {
  method() { return this; } // Обычная функция
}
```

- **API, ожидающие динамический `this`** — например, методы массивов, где `this` может быть передан через второй аргумент.

```javascript
const result = [1, 2, 3].map(function(item) {
  return this.multiplier * item;
}, { multiplier: 2 }); // this привязан ко второму аргументу
```

---

## 5.4. Итераторы, генераторы и ленивые вычисления

Современный JavaScript всё чаще работает не с массивами, а с потоками данных. В основе такого подхода лежат итераторы, генераторы и протоколы Iterator и Iterable.

**Итераторы** — объекты, реализующие метод `next()`, который возвращает объект с полями `value` и `done`. Итераторы позволяют обходить коллекции последовательно, без загрузки всех элементов в память.

```javascript
function createRange(start, end) {
  let current = start;
  return {
    next() {
      if (current <= end) {
        return { value: current++, done: false };
      }
      return { done: true };
    }
  };
}
const rangeIterator = createRange(1, 5);
console.log(rangeIterator.next().value); // 1
console.log(rangeIterator.next().value); // 2
```

**Генераторы (`function*`)** — специальные функции, которые могут приостанавливать своё выполнение и возобновлять его. Они возвращают объект-итератор и используются для создания последовательностей.

```javascript
function* generateSequence(start, end) {
  for (let i = start; i <= end; i++) {
    yield i; // Возвращает значение и приостанавливается
  }
}
const generator = generateSequence(1, 5);
console.log(generator.next().value); // 1
console.log(generator.next().value); // 2
```

**Асинхронные генераторы** (`async function*`) — генераторы, которые могут использовать `await` и возвращать промисы.

```javascript
async function* fetchPages(urls) {
  for (const url of urls) {
    const response = await fetch(url);
    yield response.json();
  }
}
```

**Протоколы Iterator и Iterable** — объект является итерируемым (`Iterable`), если имеет метод `[Symbol.iterator]`, возвращающий итератор. Встроенные структуры данных (Array, Set, Map, String) реализуют этот протокол.

```javascript
const iterableArray = [1, 2, 3];
const iterator = iterableArray[Symbol.iterator]();
console.log(iterator.next()); // { value: 1, done: false }
```

**Iterator Helpers** — набор методов, превративших итераторы в полноценный инструмент обработки данных. Они позволяют выполнять операции `.map()`, `.filter()`, `.take()`, `.drop()`, `.flatMap()` непосредственно над итераторами, не создавая промежуточных коллекций и не загружая все данные в память.

```javascript
function* numbers() {
  yield 1; yield 2; yield 3; yield 4; yield 5;
}
const result = numbers()
  .filter(n => n % 2 === 0)  // Только чётные
  .map(n => n * 10)          // Умножить на 10
  .take(2)                   // Взять первые 2
  .toArray();                // Преобразовать в массив
// result = [20, 40] — без создания промежуточных массивов
```

Iterator Helpers делают обработку больших наборов данных значительно эффективнее, так как операции выполняются лениво — значения вычисляются только по мере необходимости.

---

## 5.5. Асинхронные функции и потоки выполнения

Ключевой особенностью современного JavaScript стала асинхронность. Функции перестали быть исключительно синхронными вычислениями.

**Promise** — объект, представляющий результат асинхронной операции. Может находиться в состояниях pending, fulfilled или rejected.

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => resolve('Done'), 1000);
});
promise.then(result => console.log(result));
```

**async/await** — синтаксический сахар над Promise, позволяющий писать асинхронный код в синхронном стиле.

```javascript
async function fetchData() {
  const response = await fetch('/api/data');
  const data = await response.json();
  return data;
}
```

**Асинхронные итераторы** — итераторы, использующие Promise для получения следующего значения. Реализуют протокол AsyncIterable.

```javascript
async function* asyncGenerator() {
  yield await fetchData();
  yield await fetchMoreData();
}
```

**`for await...of`** — цикл для обхода асинхронных итераторов.

```javascript
for await (const item of asyncGenerator()) {
  console.log(item);
}
```

**AbortController** — механизм для отмены асинхронных операций.

```javascript
const controller = new AbortController();
const signal = controller.signal;
fetch('/api/data', { signal })
  .then(response => response.json())
  .catch(err => {
    if (err.name === 'AbortError') {
      console.log('Запрос отменён');
    }
  });
controller.abort();
```

Большинство API Web Platform строится вокруг этой модели — функции принимают и возвращают Promise, используют async/await и поддерживают отмену операций через AbortController.

---

## 5.6. Композиция функций

Одной из важнейших идей современного JavaScript является композиция небольших независимых функций. Вместо написания больших монолитных функций код разбивается на маленькие, каждая из которых решает одну задачу.

**Higher-Order Functions** — функции, которые принимают другие функции как аргументы или возвращают функции.

```javascript
function map(array, transform) {
  const result = [];
  for (const item of array) {
    result.push(transform(item));
  }
  return result;
}
```

**Function Composition** — объединение нескольких функций в одну, где результат каждой передаётся следующей.

```javascript
const compose = (f, g) => (x) => f(g(x));
const double = x => x * 2;
const square = x => x * x;
const doubleThenSquare = compose(square, double);
console.log(doubleThenSquare(5)); // 100
```

**Pipeline-подход** — последовательное применение функций к данным.

```javascript
const pipeline = (initial, ...fns) => fns.reduce((value, fn) => fn(value), initial);
const result = pipeline(
  5,
  x => x * 2,   // 10
  x => x + 3,   // 13
  x => x * x    // 169
);
```

**Фабрики функций** — функции, создающие другие функции с заданными параметрами.

```javascript
function createValidator(rule, errorMessage) {
  return function(value) {
    if (!rule(value)) {
      throw new Error(errorMessage);
    }
    return true;
  };
}
const isEmail = createValidator(
  v => v.includes('@'),
  'Invalid email'
);
```

**Middleware** — последовательность функций-обработчиков, каждая из которых может выполнять действие и передавать управление следующей.

```javascript
function createMiddlewareChain(...middlewares) {
  return function(req, res) {
    let index = 0;
    function next() {
      if (index < middlewares.length) {
        const middleware = middlewares[index++];
        middleware(req, res, next);
      }
    }
    next();
  };
}
```

**Decorators** — специальный синтаксис для добавления поведения к функциям и классам. На момент написания книги предложение находится на стадии **Stage 3** процесса TC39: спецификация стабилизирована, но нативной поддержки в браузерных движках и Node.js пока нет — decorators работают только через транспиляцию (TypeScript 5+ или Babel).

```javascript
@logged
function process(data) {
  return data * 2;
}
// Эквивалентно: process = logged(process);
```

Важная оговорка для этого примера: синтаксис декораторов **над обычными функциями** (как в примере выше) не входит в текущее предложение Stage 3 — там определены декораторы только для классов, их методов, полей и аксессоров. Декораторы для отдельно стоящих функций рассматриваются отдельным предложением (Function Decorators), которое находится на более ранней стадии (Stage 1). На практике декораторы сегодня применяются именно к классам:

```javascript
function logged(originalMethod, context) {
  const methodName = context.name;
  return function(...args) {
    console.log(`Calling ${String(methodName)}`);
    return originalMethod.call(this, ...args);
  };
}

class UserService {
  @logged
  async getUser(id) {
    return await fetch(`/api/users/${id}`).then(r => r.json());
  }
}
```

Композиция функций позволяет строить приложения из небольших переиспользуемых компонентов вместо крупных монолитных классов, что улучшает тестируемость и поддерживаемость кода.


---

## 5.7. Иммутабельность и современные методы коллекций

Современная разработка всё чаще использует неизменяемые структуры данных. Вместо изменения существующих объектов и массивов создаются новые копии с обновлёнными данными. Это упрощает реактивные системы, управление состоянием и тестирование.

**Новые методы массивов, создающие новые массивы вместо мутации:**

- **`toSorted()`** — возвращает отсортированную копию массива, не изменяя оригинал. Замена для `sort()`.

```javascript
const arr = [3, 1, 4, 1, 5];
const sorted = arr.toSorted((a, b) => a - b);
console.log(sorted); // [1, 1, 3, 4, 5]
console.log(arr); // [3, 1, 4, 1, 5] — не изменился
```

- **`toReversed()`** — возвращает перевёрнутую копию массива. Замена для `reverse()`.

```javascript
const arr = [1, 2, 3];
const reversed = arr.toReversed();
console.log(reversed); // [3, 2, 1]
console.log(arr); // [1, 2, 3]
```

- **`toSpliced()`** — возвращает копию массива с удалёнными/добавленными элементами. Замена для `splice()`.

```javascript
const arr = [1, 2, 3, 4];
const spliced = arr.toSpliced(1, 2, 10, 20);
console.log(spliced); // [1, 10, 20, 4]
console.log(arr); // [1, 2, 3, 4]
```

- **`with()`** — возвращает копию массива с заменённым элементом по индексу.

```javascript
const arr = [1, 2, 3];
const updated = arr.with(1, 10);
console.log(updated); // [1, 10, 3]
console.log(arr); // [1, 2, 3]
```

**Преимущества иммутабельных операций:**

- **Реактивные системы** — иммутабельность позволяет определять изменения путём сравнения ссылок, что упрощает обнаружение изменений.
- **Управление состоянием** — состояние можно безопасно обновлять, не опасаясь побочных эффектов.
- **Тестирование** — иммутабельные функции легче тестировать, поскольку они не имеют побочных эффектов.
- **Параллельная обработка** — иммутабельные данные безопасно передавать между потоками.

---

## 5.8. Функции как универсальный интерфейс платформы

Практически вся Web Platform взаимодействует с приложением посредством функций. Функции стали универсальным механизмом интеграции JavaScript с возможностями платформы.

**Где используются функции как интерфейс:**

- **Обработчики событий** — функции, вызываемые при наступлении событий.

```javascript
element.addEventListener('click', (event) => {
  console.log('Clicked', event.target);
});
```

- **Fetch API** — функции-колбэки в `.then()` и `async/await`.

```javascript
fetch('/api/data')
  .then(response => response.json())
  .then(data => console.log(data));
```

- **Streams API** — обработчики данных в потоках.

```javascript
const stream = new ReadableStream({
  start(controller) { /* ... */ },
  pull(controller) { /* ... */ }
});
```

- **Web Workers** — функции-обработчики сообщений.

```javascript
worker.onmessage = (event) => {
  console.log('Received:', event.data);
};
```

- **Scheduler API** — задачи как функции.

```javascript
scheduler.postTask(() => {
  console.log('Task executed');
});
```

- **Таймеры** — колбэки в `setTimeout`, `setInterval`.

```javascript
setTimeout(() => console.log('Timer'), 1000);
```

- **Animation API** — функции-анимации.

```javascript
requestAnimationFrame((timestamp) => {
  updateAnimation(timestamp);
});
```

- **Web Components** — методы жизненного цикла как функции.

```javascript
class MyComponent extends HTMLElement {
  connectedCallback() { /* ... */ }
  disconnectedCallback() { /* ... */ }
}
```

Функции являются универсальным контрактом между JavaScript и Web Platform. Любая функция, независимо от того, является она синхронной, асинхронной, стрелочной или обычной, может быть передана в любой API, ожидающий обработчик или колбэк.

---

## Заключение главы

Современный JavaScript рассматривает функции не просто как средство организации кода, а как основу архитектуры языка. Функции как объекты первого класса позволяют передавать их в качестве значений, возвращать из других функций и хранить в структурах данных. Замыкания обеспечивают инкапсуляцию состояния и приватность данных. Стрелочные функции упрощают работу с `this` и делают код более читаемым. Итераторы и генераторы открывают возможности для ленивых вычислений и работы с потоками данных. Асинхронные функции интегрируются с Promise и async/await, создавая единую модель выполнения. Композиция функций позволяет строить сложную логику из небольших независимых компонентов. Иммутабельные методы коллекций упрощают работу с данными без побочных эффектов. Вся Web Platform использует функции как универсальный интерфейс для взаимодействия с приложением. Именно функции объединяют синхронные и асинхронные вычисления, модульную композицию, обработку потоков данных и интеграцию с платформой. В сочетании с итераторами, генераторами, асинхронными функциями и неизменяемыми методами коллекций они формируют стиль разработки, ориентированный на небольшие независимые компоненты, высокую читаемость и предсказуемое поведение приложений.
