# Часть IV. Современная работа с данными

# Глава 14. Streams API: потоковая обработка данных и эффективное использование памяти

К 2026 году **Streams API** стал одним из ключевых компонентов Web Platform. Он позволяет приложениям обрабатывать данные по мере их поступления, не дожидаясь завершения загрузки всего файла или сетевого ответа. Такой подход уменьшает потребление памяти, ускоряет отображение информации и позволяет создавать приложения, способные эффективно работать с большими объёмами данных, мультимедиа, искусственным интеллектом и сетевыми потоками.

Streams API используется во многих современных API платформы, включая **Fetch API**, **Compression Streams**, **File System Access API**, **WebTransport** и другие. Вместо передачи больших массивов данных целиком платформа передаёт последовательность небольших блоков (chunks), которые могут сразу поступать на обработку.

---

## 14.1. `ReadableStream`: чтение данных по мере поступления

`ReadableStream` представляет источник потоковых данных. Вместо того чтобы полностью загружать содержимое в память, поток последовательно выдаёт небольшие блоки данных. Это позволяет приложению начинать обработку практически сразу после получения первых байтов.

**Наиболее распространённые источники потоков:**

- ответы `fetch()` через `response.body`;
- файлы, открытые через File System Access API;
- сетевые соединения (WebSocket, WebTransport);
- видеопотоки и медиа-данные;
- генераторы данных (пользовательские источники).

```javascript
// Чтение потока из Fetch API
const response = await fetch('/api/large-data');
const reader = response.body.getReader();
const decoder = new TextDecoder();

let receivedData = '';
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  receivedData += decoder.decode(value, { stream: true });
}
```

**Преимущества потокового чтения:**

- **Уменьшение использования оперативной памяти** — данные обрабатываются по частям, а не целиком.
- **Сокращение количества крупных выделений памяти** — снижается нагрузка на сборщик мусора.
- **Более быстрый отклик интерфейса** — пользователь получает результаты значительно раньше окончания загрузки.

Потоки особенно эффективны при работе с файлами размером в сотни мегабайт или гигабайты, когда загрузка всего содержимого в память становится невозможной или экономически нецелесообразной.

**Создание собственного `ReadableStream`:**

```javascript
const stream = new ReadableStream({
  start(controller) {
    // Инициализация источника
    controller.enqueue('первый блок');
    controller.enqueue('второй блок');
    controller.close();
  },
  pull(controller) {
    // Запрос дополнительных данных (при необходимости)
  },
  cancel(reason) {
    // Обработка отмены чтения
  }
});
```

---

## 14.2. `WritableStream`: запись данных в различные назначения

`WritableStream` представляет поток, принимающий данные. Получателем данных может быть файл, сетевое соединение, объект WebSocket, браузерный API или пользовательский обработчик. Каждый поступающий блок записывается сразу после получения, поэтому нет необходимости сначала собирать весь объём информации в памяти.

```javascript
const writableStream = new WritableStream({
  write(chunk) {
    // Обработка полученного блока
    console.log('Получен блок:', chunk);
  },
  close() {
    console.log('Поток закрыт');
  },
  abort(reason) {
    console.log('Поток прерван:', reason);
  }
});
```

**Использование `pipeTo()` для соединения потоков:**

```javascript
// Чтение из источника и запись в файл
const response = await fetch('/api/data');
const fileHandle = await window.showSaveFilePicker();
const writable = await fileHandle.createWritable();

await response.body.pipeTo(writable);
// Данные передаются напрямую из сети в файл без промежуточного хранения
```

**Пример записи в WebSocket:**

```javascript
const ws = new WebSocket('wss://example.com/stream');
const writable = new WritableStream({
  write(chunk) {
    ws.send(chunk);
  }
});

await readableStream.pipeTo(writable);
```

**Особенности браузеров:**

- **Chromium (Chrome, Edge, Opera)** — поддерживают запись непосредственно в файл пользователя через File System Access API (`createWritable()`).
- **Firefox и Safari** — предоставляют только внутреннее файловое хранилище Origin Private File System (OPFS), поэтому прямой доступ к файловой системе пользователя остаётся ограниченным. Для таких случаев рекомендуется использовать `showSaveFilePicker` (в Chromium) или альтернативные подходы (загрузка через `<a download>`).

---

## 14.3. `TransformStream`: преобразование данных во время передачи

`TransformStream` соединяет входной и выходной потоки. Каждый поступающий блок может быть изменён перед передачей следующему потребителю. Таким образом можно построить цепочку последовательных преобразований данных.

```javascript
const transformStream = new TransformStream({
  transform(chunk, controller) {
    // Преобразование данных
    const transformed = chunk.toUpperCase();
    controller.enqueue(transformed);
  },
  flush(controller) {
    // Завершающие операции
  }
});

// Использование в конвейере
await readableStream
  .pipeThrough(transformStream)
  .pipeTo(writableStream);
```

**Встроенные TransformStream:**

- `TextEncoderStream` — преобразует текст в байты (UTF-8).
- `TextDecoderStream` — преобразует байты в текст (UTF-8).
- `CompressionStream` — сжимает данные (gzip, deflate).
- `DecompressionStream` — распаковывает сжатые данные.

**Пример цепочки преобразований:**

```javascript
const response = await fetch('/api/data');
const textDecoder = new TextDecoderStream();
const upperCaseTransform = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk.toUpperCase());
  }
});

await response.body
  .pipeThrough(textDecoder)     // Байты → текст
  .pipeThrough(upperCaseTransform) // Текст → текст с uppercase
  .pipeTo(writableStream);
```

**Типичные задачи TransformStream:**

- кодирование и декодирование текста;
- сжатие данных (`CompressionStream`) и распаковка (`DecompressionStream`);
- фильтрация данных (удаление ненужных блоков);
- изменение формата (например, преобразование XML в JSON);
- потоковый разбор файлов (CSV, логов);
- обработка бинарных протоколов.

Каждый этап работает независимо и получает данные только тогда, когда предыдущий этап их подготовил. В результате приложение может строить длинные конвейеры обработки без создания промежуточных массивов в памяти.

---

## 14.4. Конвейеры потоков и автоматическое управление скоростью передачи

Одной из сильнейших сторон Streams API является возможность объединять несколько потоков в единый конвейер.

**Основные методы:**

- `pipeTo(destination)` — соединяет ReadableStream с WritableStream.
- `pipeThrough(transform)` — соединяет ReadableStream с TransformStream и возвращает новый ReadableStream.

```javascript
// Пример конвейера
const pipeline = await fetch('/api/compressed-data')
  .then(response => response.body)
  .then(stream => stream.pipeThrough(new DecompressionStream('gzip')))
  .then(stream => stream.pipeThrough(new TextDecoderStream()))
  .then(stream => stream.pipeThrough(parseJSONStream()));
```

**Структура конвейера:**

```
ReadableStream (источник)
      ↓
TransformStream (преобразование 1)
      ↓
TransformStream (преобразование 2)
      ↓
WritableStream (приёмник)
```

Каждый компонент выполняет только одну задачу. Платформа самостоятельно управляет передачей данных между этапами конвейера.

**Backpressure (обратное давление):** Если потребитель начинает работать медленнее источника данных, поток автоматически снижает скорость передачи новых блоков. Это предотвращает переполнение памяти и потерю данных.

**Как работает Backpressure:**

1. Приёмник (WritableStream) сигнализирует, что не может принимать новые данные (буфер заполнен).
2. Поток приостанавливает передачу данных.
3. Когда приёмник освобождается, передача возобновляется.

Благодаря этому:

- не происходит переполнение памяти (буферы не растут бесконтрольно);
- не требуется вручную управлять размерами буферов;
- источник и получатель работают с согласованной скоростью.

Backpressure является одной из важнейших особенностей Streams API и делает потоковую архитектуру устойчивой даже при больших объёмах данных и неравномерной скорости обработки.

---

## 14.5. Потоковая обработка текста и JSON

Одной из наиболее востребованных задач является обработка текстовых данных, поступающих по сети. Для преобразования последовательности байтов в текст используется стандартный объект `TextDecoderStream`.

```javascript
const response = await fetch('/api/data');
const textStream = response.body.pipeThrough(new TextDecoderStream());

const reader = textStream.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log('Получен текст:', value);
}
```

**Важная особенность JSON:** Стандартный JSON представляет собой единую структуру данных. Поэтому такой документ нельзя корректно разбирать до получения его целиком. Невозможно безопасно вызвать `JSON.parse()` для первых нескольких килобайт большого JSON-файла, поскольку синтаксис JSON требует завершённости всей структуры.

**NDJSON (Newline Delimited JSON):** Для потоковой передачи используется формат, где каждая строка представляет отдельный законченный JSON-объект.

```json
{"id":1,"name":"Alice"}
{"id":2,"name":"Bob"}
{"id":3,"name":"Carol"}
```

**Пример парсинга NDJSON:**

```javascript
async function parseNDJSON(response) {
  const reader = response.body
    .pipeThrough(new TextDecoderStream())
    .getReader();
  
  let buffer = '';
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    
    buffer += value;
    const lines = buffer.split('\n');
    buffer = lines.pop() || '';
    
    for (const line of lines) {
      if (line.trim()) {
        const data = JSON.parse(line);
        processData(data);
      }
    }
  }
}
```

**Пример потоковой обработки с использованием TransformStream:**

```javascript
class NDJSONParser extends TransformStream {
  constructor() {
    super({
      start() {
        this.buffer = '';
      },
      transform(chunk, controller) {
        this.buffer += chunk;
        const lines = this.buffer.split('\n');
        this.buffer = lines.pop() || '';
        
        for (const line of lines) {
          if (line.trim()) {
            try {
              controller.enqueue(JSON.parse(line));
            } catch (e) {
              controller.error(e);
            }
          }
        }
      },
      flush(controller) {
        if (this.buffer.trim()) {
          controller.enqueue(JSON.parse(this.buffer));
        }
      }
    });
  }
}

// Использование
const response = await fetch('/api/events');
const parsedStream = response.body
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(new NDJSONParser());
```

**Применение потоковой обработки NDJSON:**

- **Потоковые API искусственного интеллекта** — получение ответов от LLM по токенам.
- **Обработка журналов событий** — чтение и анализ логов в реальном времени.
- **Системы мониторинга** — потоковая передача метрик.
- **Передача больших массивов событий в реальном времени** — например, данные с датчиков или пользовательских действий.

**Важно понимать:** Streams API не содержит встроенного потокового JSON-парсера. Потоковая обработка JSON достигается либо использованием NDJSON, либо специализированными библиотеками, поддерживающими инкрементальный разбор обычного JSON (например, `jsonparse` или `oboe.js`).

---

## 14.6. Streaming HTML и постепенная отрисовка страницы

Streams API позволяет серверу отправлять HTML не одним большим документом, а небольшими частями. Браузер начинает разбирать и отображать страницу сразу после получения первых фрагментов.

**Пример потоковой отправки HTML:**

```javascript
// Серверная часть (Node.js)
async function streamHTML(response) {
  response.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
  
  // Отправка заголовка сразу
  response.write('<!DOCTYPE html><html><head><title>Потоковая страница</title></head><body>');
  
  // Отправка основного контента
  for (let i = 0; i < 100; i++) {
    response.write(`<div>Блок ${i}</div>`);
    await new Promise(resolve => setTimeout(resolve, 100));
  }
  
  response.write('</body></html>');
  response.end();
}
```

**Браузерная часть (клиент):**

```javascript
// Клиент может принимать потоковый HTML через fetch
const response = await fetch('/streaming-page');
const reader = response.body.pipeThrough(new TextDecoderStream()).getReader();

// Постепенное добавление контента
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  // Добавление HTML по мере поступления
  document.getElementById('content').innerHTML += value;
}
```

**Преимущества Streaming HTML:**

- **Уменьшение времени первого отображения страницы** — пользователь видит интерфейс раньше.
- **Улучшение показателей Core Web Vitals** — особенно First Contentful Paint (FCP) и Largest Contentful Paint (LCP).
- **Более отзывчивый интерфейс** — контент появляется постепенно, а не все сразу после завершения загрузки.

**Пример потоковой передачи с приоритетами:** Сначала могут быть отправлены заголовок страницы, меню, основные стили и каркас интерфейса. Пока пользователь уже видит интерфейс, сервер продолжает формировать и передавать оставшуюся часть документа.

Streaming HTML широко используется современными серверными фреймворками (например, React Server Components, Astro, Qwik) и особенно эффективен при серверном рендеринге (SSR).

---

## 14.7. Streams API как основа современных Web API

Сегодня Streams API используется значительно шире, чем только в Fetch API. Он лежит в основе множества современных технологий Web Platform:

- **Fetch API** — потоковое чтение ответов через `response.body`. Baseline Widely Available.
- **Compression Streams API** — сжатие и распаковка данных.
- **WebTransport** — потоковая передача данных по сети. Долгое время эта технология оставалась Chromium-only (с поддержкой в Firefox с 2023 года), но достигла полноценного статуса Baseline только в марте 2026 года, когда поддержку добавил Safari 26.4 — это очень недавнее изменение, актуальное для книги, ориентированной на 2026 год.
- **File System Access API** — потоковое чтение и запись файлов. Как отмечалось в предыдущих главах, остаётся Chromium-only технологией.
- **Media API** — обработка аудио и видео потоков.
- **WebCodecs** — кодирование и декодирование медиа. Также прошёл путь от Chromium-only решения к кросс-браузерному: Firefox добавил полную поддержку на десктопе с версии 130 (2024), а Safari завершил переход только с версией 26 (сентябрь 2025) — до этого браузер поддерживал лишь видеочасть API, без работы со звуком. Стоит учитывать, что на Android поддержка WebCodecs в Firefox по-прежнему неполная.
- **Обмен данными между Web Workers** — передача потоков между потоками выполнения.

**Важное уточнение к примеру ниже:** в WebCodecs API нет встроенных классов `VideoDecoderStream` и `VideoEncoderStream` — платформа предоставляет только низкоуровневые `VideoDecoder` и `VideoEncoder`, работающие с колбэками, а не как готовые `TransformStream`. Обёртки со Streams-совместимым интерфейсом (как в примере ниже) нужно писать самостоятельно — это общепринятый паттерн в сообществе, но не часть спецификации:

```javascript
// Пример объединения различных API через Streams.
// VideoDecoderStream и VideoEncoderStream — НЕ встроенные классы платформы,
// а пользовательские обёртки над нативными VideoDecoder/VideoEncoder,
// оформленные как TransformStream. Их нужно реализовать самостоятельно.

class VideoDecoderStream extends TransformStream {
  constructor(config) {
    let decoder;
    super({
      start(controller) {
        decoder = new VideoDecoder({
          output: (frame) => controller.enqueue(frame),
          error: (e) => controller.error(e)
        });
        decoder.configure(config);
      },
      transform(chunk) {
        decoder.decode(chunk);
      },
      async flush() {
        await decoder.flush();
        decoder.close();
      }
    });
  }
}

async function processVideo(videoFile, decoderConfig, encoderConfig) {
  // Чтение файла как потока
  const fileStream = await videoFile.stream();

  // Декодирование видео через собственную обёртку
  const decodedStream = fileStream.pipeThrough(new VideoDecoderStream(decoderConfig));

  // Обработка кадров
  const processedStream = decodedStream.pipeThrough(new TransformStream({
    transform(frame, controller) {
      const processed = processFrame(frame);
      controller.enqueue(processed);
    }
  }));

  // Кодирование и запись в файл потребует аналогичной обёртки VideoEncoderStream
  await processedStream.pipeTo(writableStream);
}
```

Благодаря единой модели потоков разработчик может объединять различные API в общие конвейеры обработки без преобразования данных между несовместимыми форматами. Это делает код проще, быстрее и значительно экономичнее с точки зрения использования памяти — при условии, что разработчик понимает, какие звенья конвейера предоставлены платформой напрямую, а какие приходится реализовывать самому.


---

## Заключение главы

Streams API стал фундаментом современной архитектуры обработки данных в JavaScript. Вместо загрузки больших объёмов информации целиком приложение получает последовательность небольших блоков, которые сразу поступают на обработку. `ReadableStream` обеспечивает чтение данных по мере поступления, `WritableStream` — запись в различные назначения, `TransformStream` — преобразование данных во время передачи. Конвейеры потоков с использованием `pipeTo()` и `pipeThrough()` позволяют строить сложные цепочки обработки. Механизм Backpressure автоматически регулирует скорость передачи данных, предотвращая переполнение памяти. Текстовая обработка через `TextDecoderStream` и NDJSON позволяет потоково работать с текстовыми и JSON-данными. Streaming HTML улучшает производительность и восприятие страниц.

Современные приложения строят цепочки обработки, объединяя `ReadableStream`, `TransformStream` и `WritableStream` в единые конвейеры. Платформа автоматически управляет скоростью передачи данных, предотвращает переполнение памяти с помощью механизма **Backpressure** и обеспечивает эффективное взаимодействие между различными компонентами Web Platform. Благодаря этому Streams API стал одним из важнейших инструментов для разработки высокопроизводительных веб-приложений, работающих с большими объёмами данных и потоковыми сервисами в реальном времени.
