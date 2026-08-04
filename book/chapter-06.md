# Часть II. Современный язык JavaScript

## Глава 6. Асинхронность: архитектура современного выполнения

Асинхронность всегда была одной из ключевых особенностей JavaScript. Однако к 2026 году она превратилась из набора отдельных механизмов в единую архитектуру выполнения, охватывающую браузеры, Node.js, Deno, Bun и другие современные среды.

Сегодня практически все API платформы построены вокруг единой модели: задачи, промисы, асинхронные функции, потоки данных, отмена операций и кооперативное планирование. Благодаря этому разработчик может описывать сложные распределённые процессы линейным, понятным кодом, не теряя контроля над производительностью приложения.

---

## 6.1. Event Loop и модель выполнения

Асинхронность JavaScript основана на Event Loop — механизме, координирующем выполнение синхронного кода и обработку событий.

**Компоненты модели выполнения:**

- **Call Stack (стек вызовов)** — структура данных, хранящая информацию о текущих выполняемых функциях. Когда функция вызывается, она помещается на вершину стека. Когда функция завершается, она удаляется из стека. JavaScript выполняется синхронно, пока стек не становится пустым.

- **Task Queue (очередь задач, макрозадач)** — очередь, в которую попадают задачи, готовые к выполнению. Сюда попадают колбэки `setTimeout`, `setInterval`, события DOM, сетевые запросы. Event Loop забирает задачи из этой очереди по одной, когда стек вызовов пуст.

- **Microtask Queue (очередь микрозадач)** — очередь с более высоким приоритетом, чем Task Queue. Сюда попадают колбэки `Promise.then`, `Promise.catch`, `Promise.finally`, `queueMicrotask()`, `MutationObserver`. Вся очередь микрозадач полностью обрабатывается после завершения каждой макрозадачи, прежде чем Event Loop перейдёт к следующей задаче.

- **Rendering Pipeline (в браузере)** — обновление интерфейса происходит между макрозадачами, после полной обработки очереди микрозадач. Браузер стремится выполнять рендеринг с частотой 60 Гц (каждые 16.7 мс), но может делать это чаще или реже в зависимости от нагрузки.

**Порядок выполнения:**

1. Выполняется весь синхронный код — стек вызовов полностью очищается.
2. Обрабатывается вся очередь микрозадач — все `Promise.then`, `queueMicrotask` и т.д. выполняются до конца.
3. (В браузере) Выполняется рендеринг — обновление интерфейса, если необходимо.
4. Берётся следующая задача из Task Queue — одна макрозадача (например, один колбэк `setTimeout`).
5. Возврат к шагу 2.

**Пример, демонстрирующий порядок выполнения:**

```javascript
console.log('1. Синхронный код');

setTimeout(() => {
  console.log('4. Макрозадача (setTimeout)');
}, 0);

Promise.resolve().then(() => {
  console.log('3. Микрозадача (Promise)');
});

queueMicrotask(() => {
  console.log('2. Микрозадача (queueMicrotask)');
});

console.log('1.5 Синхронный код');
// Вывод:
// 1. Синхронный код
// 1.5 Синхронный код
// 2. Микрозадача (queueMicrotask)
// 3. Микрозадача (Promise)
// 4. Макрозадача (setTimeout)
```

`setTimeout` с задержкой 0 мс не выполняется немедленно — он помещается в Task Queue и ждёт, пока Event Loop освободится. Все микрозадачи выполняются перед ним, даже если он был запланирован раньше.

**Event Loop в Node.js, Deno и Bun** имеет аналогичную структуру, но с дополнительными фазами: таймеры, ожидание I/O, опрос, проверка, закрытие. Это связано с необходимостью обрабатывать файловую систему, сетевые сокеты и другие системные ресурсы. Однако принцип с макро- и микрозадачами остаётся тем же.

---

## 6.2. Promise как универсальный контракт асинхронности

Promise окончательно закрепился в роли универсального интерфейса асинхронных операций. Современные API Web Platform практически полностью отказались от callback-модели.

**Состояния Promise:**

- **Pending** — начальное состояние, операция ещё выполняется.
- **Fulfilled** — операция успешно завершена, доступен результат.
- **Rejected** — операция завершилась с ошибкой, доступна причина.

**Базовое использование:**

```javascript
const promise = new Promise((resolve, reject) => {
  // Асинхронная операция
  if (success) {
    resolve(result);
  } else {
    reject(error);
  }
});

promise
  .then(result => console.log(result))
  .catch(error => console.error(error))
  .finally(() => console.log('Завершено'));
```

**Promise используется в современных API:**

- Fetch API — `fetch()` возвращает Promise с Response.
- Clipboard API — `navigator.clipboard.readText()` возвращает Promise.
- File System API — `window.showDirectoryPicker()` возвращает Promise.
- Web Locks — `navigator.locks.request()` возвращает Promise.
- Web Bluetooth — `navigator.bluetooth.requestDevice()` возвращает Promise.
- WebUSB — `navigator.usb.requestDevice()` возвращает Promise.
- Navigation API — `navigation.navigate()` возвращает Promise.
- Scheduler API — `scheduler.postTask()` возвращает Promise.

**Современные методы композиции Promise:**

- `Promise.all([p1, p2, p3])` — ожидает выполнения всех промисов. Если любой завершается с ошибкой, весь Promise.all отклоняется с этой ошибкой.

```javascript
const [users, posts] = await Promise.all([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/posts').then(r => r.json())
]);
```

- `Promise.allSettled([p1, p2, p3])` — ожидает выполнения всех промисов, независимо от результата. Возвращает массив объектов с полями `status`, `value` или `reason`.

```javascript
const results = await Promise.allSettled([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/posts').then(r => r.json())
]);
results.forEach(result => {
  if (result.status === 'fulfilled') {
    console.log('Успешно:', result.value);
  } else {
    console.log('Ошибка:', result.reason);
  }
});
```

- `Promise.any([p1, p2, p3])` — ожидает выполнения первого успешного промиса. Если все промисы отклонены, возвращает ошибку `AggregateError`.

```javascript
const response = await Promise.any([
  fetch('/api/us-east').then(r => r.json()),
  fetch('/api/us-west').then(r => r.json())
]);
// Используется результат самого быстрого успешного ответа
```

- `Promise.race([p1, p2, p3])` — ожидает выполнения первого завершённого промиса (успешного или отклонённого).

```javascript
const result = await Promise.race([
  fetch('/api/data'),
  new Promise((_, reject) => 
    setTimeout(() => reject(new Error('Timeout')), 5000)
  )
]);
// Если запрос выполняется дольше 5 секунд — ошибка Timeout
```

- `Promise.withResolvers()` — создаёт Promise вместе с его функциями `resolve` и `reject`. Упрощает работу с промисами, когда функции разрешения нужны вне конструктора.

```javascript
// Традиционный подход
const promise = new Promise((resolve, reject) => {
  // resolve и reject доступны только здесь
});

// Современный подход
const { promise, resolve, reject } = Promise.withResolvers();
// resolve и reject доступны в любой области видимости
```

---

## 6.3. Async/Await как основной стиль разработки

Конструкция `async/await` стала стандартным способом описания асинхронной логики. Она позволяет писать асинхронный код в синхронном стиле, что улучшает читаемость и упрощает обработку ошибок.

**Базовое использование:**

```javascript
async function fetchData() {
  try {
    const response = await fetch('/api/data');
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Ошибка загрузки:', error);
    throw error;
  } finally {
    console.log('Запрос завершён');
  }
}
```

**Современный подход включает:**

- **Последовательное выполнение** — когда каждая операция зависит от предыдущей:

```javascript
const user = await fetchUser(id);
const posts = await fetchUserPosts(user.id);
const comments = await fetchPostsComments(posts[0].id);
```

- **Параллельный запуск независимых операций** — когда операции не зависят друг от друга:

```javascript
const [user, posts, settings] = await Promise.all([
  fetchUser(id),
  fetchUserPosts(id),
  fetchUserSettings(id)
]);
```

- **Централизованную обработку ошибок** — с помощью `try/catch` на уровне функции или модуля:

```javascript
try {
  const data = await riskyOperation();
} catch (error) {
  // Обработка ошибки на уровне компонента
}
```

- **Корректное распространение исключений** — ошибка, возникшая в `async` функции, автоматически превращается в отклонённый Promise, который можно обработать через `catch`:

```javascript
async function getData() {
  throw new Error('Ошибка');
}
getData().catch(err => console.error(err));
```

**Типичные архитектурные ошибки:**

1. **Случайная последовательная обработка независимых запросов** — когда запросы не зависят друг от друга, но выполняются последовательно:

```javascript
// ❌ Плохо — потеря конкурентности
const user = await fetchUser(id);
const posts = await fetchPosts(id);
// Второй запрос ждёт завершения первого

// ✅ Хорошо — параллельное выполнение
const [user, posts] = await Promise.all([
  fetchUser(id),
  fetchPosts(id)
]);
```

2. **Лишние `await`** — когда результат промиса не нужен или можно передать промис напрямую:

```javascript
// ❌ Плохо — лишний await
const user = await fetchUser(id);
return user;

// ✅ Хорошо
return fetchUser(id);
```

3. **Потеря конкурентности в циклах** — когда цикл с `await` выполняется последовательно вместо параллельного:

```javascript
// ❌ Плохо — последовательно в цикле
for (const id of ids) {
  const user = await fetchUser(id); // Ждёт каждого
}

// ✅ Хорошо — параллельный запуск
const users = await Promise.all(ids.map(id => fetchUser(id)));
```

---

## 6.4. Асинхронные потоки данных

Не вся асинхронность сводится к одному результату. Современные приложения работают с непрерывными потоками данных, которые поступают постепенно.

**Async Iterator и Async Generator** позволяют обрабатывать потоки данных по частям:

```javascript
// Асинхронный генератор
async function* generateData() {
  for (let i = 0; i < 10; i++) {
    await new Promise(resolve => setTimeout(resolve, 1000));
    yield i;
  }
}

// Использование async iterator
for await (const value of generateData()) {
  console.log(value); // 0, 1, 2, ... (с задержкой 1 секунда)
}
```

**Streams API** — более низкоуровневый механизм для работы с потоками данных:

- **ReadableStream** — источник данных, из которого можно читать.
- **WritableStream** — приёмник данных, в который можно записывать.
- **TransformStream** — преобразует данные из ReadableStream в WritableStream.

```javascript
// Чтение потока из Fetch API
const response = await fetch('/api/large-data');
const reader = response.body.getReader();
let chunks = [];
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  chunks.push(value);
}
// Можно обрабатывать данные по частям, не дожидаясь полной загрузки
```

**Асинхронные потоки позволяют эффективно обрабатывать:**

- **Сетевые ответы** — загрузка больших объёмов данных по частям (JSON Streaming, Server-Sent Events).
- **Большие файлы** — чтение и запись файлов без загрузки всего содержимого в память.
- **Потоковое видео** — получение и отображение видео по мере загрузки.
- **Журналы событий** — обработка логов в реальном времени.
- **AI-модели** — получение ответов от AI по токенам (постепенная генерация текста).
- **Непрерывные вычисления** — обработка больших наборов данных по мере их поступления.

---

## 6.5. Отмена операций и управление жизненным циклом

Одним из важнейших достижений современной платформы стала унификация механизма отмены операций через `AbortController` и `AbortSignal`.

**AbortController** — объект, который создаётся для управления отменой одной или нескольких операций.

```javascript
const controller = new AbortController();
const signal = controller.signal;
```

**AbortSignal** — объект, который передаётся в асинхронные API для отслеживания сигнала отмены. У сигнала есть свойства `aborted` (статус отмены) и `reason` (причина отмены).

**Использование AbortController с Fetch API:**

```javascript
const controller = new AbortController();
const signal = controller.signal;

// Передаём сигнал в fetch
const response = await fetch('/api/data', { signal });

// Отмена запроса через 5 секунд
setTimeout(() => controller.abort(), 5000);
```

**Отмена операции в пользовательском коде:**

```javascript
function fetchWithTimeout(url, timeout) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);
  
  return fetch(url, { signal: controller.signal })
    .finally(() => clearTimeout(timeoutId));
}
```

**AbortController используется во многих современных API:**

- Fetch API — отмена сетевых запросов.
- Streams API — отмена чтения или записи потоков.
- Web Locks — отмена запроса блокировки.
- Scheduler API — отмена запланированных задач.
- File System Access API — отмена операций с файловой системой.

**Корректная отмена операций позволяет избежать:**

- **Утечек памяти** — когда операция продолжает выполняться после уничтожения компонента.
- **Лишних сетевых запросов** — когда запрос больше не нужен, но продолжает выполняться.
- **Зависших Promise** — когда Promise никогда не разрешается, потому что компонент был уничтожен.
- **Работы с уже уничтоженными компонентами** — когда результат операции пытается обновить несуществующий интерфейс.

---

## 6.6. Конкурентность без многопоточности

JavaScript остаётся однопоточным языком исполнения в основном потоке. Однако современные приложения активно используют конкурентное выполнение задач, чтобы не блокировать основной поток.

**Web Workers** — изолированные потоки, выполняющие JavaScript в фоновом режиме. Они не имеют доступа к DOM и выполняются в отдельном контексте.

```javascript
// main.js
const worker = new Worker('worker.js');
worker.postMessage({ task: 'heavy-computation', data: largeArray });
worker.onmessage = (event) => {
  console.log('Результат:', event.data);
};

// worker.js
self.onmessage = (event) => {
  const result = heavyComputation(event.data);
  self.postMessage(result);
};
```

**Shared Workers** — разделяемые воркеры, которые могут использоваться несколькими окнами или вкладками.

```javascript
// main.js
const sharedWorker = new SharedWorker('shared-worker.js');
sharedWorker.port.postMessage('Hello from tab 1');
sharedWorker.port.onmessage = (event) => {
  console.log('Сообщение от shared worker:', event.data);
};
```

**Service Workers** — специальные воркеры, которые работают как прокси между приложением и сетью. Используются для кеширования, офлайн-режима, push-уведомлений.

```javascript
// service-worker.js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('v1').then((cache) => {
      return cache.addAll(['/index.html', '/styles.css']);
    })
  );
});
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request);
    })
  );
});
```

**Worklets** — лёгкие скрипты, выполняющиеся на определённых этапах рендеринга. Используются для Paint Worklet (кастомная отрисовка), Animation Worklet (кастомные анимации), Audio Worklet (обработка аудио).

**Worker Threads (Node.js)** — аналоги Web Workers для Node.js, позволяющие выполнять код в отдельных потоках.

Вместо блокировки главного потока вычисления выносятся в специализированные среды выполнения, а основной поток остаётся отзывчивым для взаимодействия с пользователем.

---

## 6.7. Планирование задач

Современная Web Platform предоставляет множество средств управления приоритетами выполнения задач.

**Классические механизмы:**

- `setTimeout(callback, delay)` — планирование выполнения задачи с задержкой.
- `requestAnimationFrame(callback)` — планирование выполнения перед следующим рендерингом.
- `requestIdleCallback(callback)` — планирование выполнения, когда браузер простаивает.

**Scheduler API** — более современный механизм для управления приоритетами задач. Позволяет задавать приоритеты и управлять выполнением сложных задач.

```javascript
// Пост задач с разными приоритетами
scheduler.postTask(() => {
  // Критичная для UI задача
  updateUI();
}, { priority: 'user-blocking' });

scheduler.postTask(() => {
  // Фоновая задача, не критичная для UI
  syncData();
}, { priority: 'background' });
```

**Приоритеты задач в Scheduler API:**

- **`user-blocking`** — задачи, критичные для взаимодействия с пользователем. Выполняются с наивысшим приоритетом.
- **`user-visible`** — задачи, которые влияют на отображение, но не критичны. Выполняются после `user-blocking`.
- **`background`** — фоновые задачи, которые могут выполняться в любое время. Выполняются с низшим приоритетом.

**Использование Signal для управления задачами:**

```javascript
const controller = new TaskController({ priority: 'user-visible' });
scheduler.postTask(() => {
  // Задача
}, { signal: controller.signal });

// Изменение приоритета
controller.setPriority('background');

// Отмена задачи
controller.abort();
```

Планирование задач делает выполнение приложения более предсказуемым и помогает браузеру эффективнее распределять вычислительные ресурсы между критичными и фоновыми операциями.

---

## 6.8. Асинхронность как единая модель Web Platform

Практически вся современная платформа использует единые принципы асинхронности. Именно через Promise и Async Iterators работают:

- **Fetch API** — сетевые запросы.
- **Streams API** — потоковая обработка данных.
- **Navigation API** — управление навигацией.
- **File System Access API** — работа с файловой системой.
- **Clipboard API** — работа с буфером обмена.
- **Web Bluetooth** — взаимодействие с Bluetooth-устройствами.
- **WebUSB** — взаимодействие с USB-устройствами.
- **Web Serial** — взаимодействие с последовательными портами.
- **WebTransport** — низкоуровневая сетевая передача данных.
- **AI API браузера** — взаимодействие с локальными AI-моделями.

Единство модели асинхронности делает знания универсальными — один и тот же подход применяется практически ко всем современным интерфейсам платформы. Если разработчик понимает Promise и async/await, он может работать с любым современным API без необходимости изучения уникальных паттернов асинхронности для каждого интерфейса.

---

## Заключение главы

Современная асинхронность JavaScript — это не отдельная возможность языка, а фундаментальная архитектура выполнения приложений. Event Loop определяет порядок выполнения синхронного кода, макро- и микрозадач. Promise стал универсальным контрактом для асинхронных операций, а async/await — основным стилем разработки. Асинхронные итераторы и Streams API позволяют работать с непрерывными потоками данных. AbortController обеспечивает единый механизм отмены операций. Web Workers и Worklets выносят вычисления в отдельные потоки, сохраняя основной поток отзывчивым. Scheduler API позволяет управлять приоритетами задач. Все эти механизмы образуют единую модель, одинаково работающую в браузерах, серверных рантаймах и облачных средах. Понимание этой модели позволяет создавать масштабируемые, отзывчивые и предсказуемые приложения, эффективно использующие возможности современной Web Platform.