
# Часть IV. Современная работа с данными

# Глава 15. URL API: адресация ресурсов и управление состоянием приложения

URL является одной из фундаментальных концепций Web Platform. Практически все браузерные API работают с URL: загрузка HTML-документов, импорт ES-модулей, сетевые запросы через Fetch API, навигация, работа с изображениями, файлами и мультимедийными ресурсами.

Современный JavaScript больше не рассматривает URL как обычную строку. Вместо ручной обработки адресов платформа предоставляет специализированные объекты, которые гарантируют корректный разбор, изменение и сериализацию адресов в соответствии со спецификацией WHATWG URL Standard.

Благодаря этому один и тот же код одинаково работает в браузерах, Node.js, Deno и Bun, что делает URL API одним из базовых инструментов современной кроссплатформенной разработки.

---

## 15.1. Объект `URL`

Класс `URL` представляет интернет-адрес в виде объекта с отдельными свойствами для каждой его части.

```javascript
const url = new URL("https://example.com:8080/users?id=42#profile");

console.log(url.protocol);   // "https:"
console.log(url.hostname);   // "example.com"
console.log(url.port);       // "8080"
console.log(url.pathname);   // "/users"
console.log(url.search);     // "?id=42"
console.log(url.hash);       // "#profile"
console.log(url.host);       // "example.com:8080"
console.log(url.origin);     // "https://example.com:8080"
console.log(url.href);       // "https://example.com:8080/users?id=42#profile"
```

Главное преимущество такого подхода заключается в том, что каждая часть адреса изменяется независимо. После изменения любого свойства объект автоматически формирует новый корректный URL.

```javascript
url.pathname = "/articles";
url.search = "?page=2";
console.log(url.href); // "https://example.com:8080/articles?page=2#profile"
```

Конструктор `new URL()` проверяет корректность входных данных по алгоритмам **WHATWG URL Standard**. Если строка не является допустимым URL, генерируется исключение `TypeError`.

```javascript
new URL("invalid url"); // TypeError: invalid url
```

**Важно:** Браузеры реализуют именно WHATWG URL Standard. Этот стандарт отличается от RFC 3986, который описывает общую грамматику URI. Поэтому некоторые адреса могут интерпретироваться браузером иначе, чем сетевыми библиотеками других языков программирования. Например, WHATWG URL Standard требует, чтобы протокол оканчивался на `:` и правильно обрабатывает символы Unicode в доменных именах (путем преобразования в Punycode).

**Дополнительные свойства и методы:**

- `url.toString()` — возвращает полный URL (аналог `url.href`).
- `url.toJSON()` — возвращает полный URL (используется при сериализации).
- `url.username` / `url.password` — логин и пароль (для аутентификации).

```javascript
const url = new URL("https://user:pass@example.com");
console.log(url.username); // "user"
console.log(url.password); // "pass"
```

---

## 15.2. Работа с относительными адресами

Во многих случаях известен не полный URL, а только относительный путь. Для его вычисления конструктор принимает второй параметр — базовый адрес.

```javascript
const image = new URL("images/logo.png", "https://example.com/app/");
console.log(image.href); // "https://example.com/app/images/logo.png"
```

**Алгоритм разрешения относительного пути:**

1. Базовый URL разбирается на компоненты.
2. Относительный путь объединяется с базовым в соответствии с правилами разрешения пути.
3. Результат нормализуется (удаляются `..` и `.`).

```javascript
new URL("../style.css", "https://example.com/app/page/");
// "https://example.com/app/style.css"

new URL("./style.css", "https://example.com/app/page/");
// "https://example.com/app/page/style.css"

new URL("//cdn.com/script.js", "https://example.com");
// "https://cdn.com/script.js" (протокол наследуется)
```

**Где используется разрешение относительных URL:**

- **Загрузка ES Modules** — спецификаторы модулей разрешаются относительно текущего URL модуля.

```javascript
import { helper } from "./helpers.js"; // Разрешается относительно текущего модуля
```

- **Выполнение `fetch()`** — относительные пути разрешаются относительно текущего URL страницы.

```javascript
fetch("/api/users"); // Относительно origin
fetch("./data.json"); // Относительно текущего пути
```

- **Работа Service Workers** — область действия (scope) определяется через URL.
- **Загрузка изображений, шрифтов и стилей** — относительные пути в CSS разрешаются относительно URL стиля.
- **Формирование ссылок в SPA** — навигация с относительными путями.

Использование объекта `URL` избавляет от ручной конкатенации строк и автоматически корректно разрешает относительные пути, учитывая все тонкости URL-спецификации.

---

## 15.3. `URLSearchParams`

Строка запроса (query string) является частью URL и представляется объектом `URLSearchParams`. Этот объект предоставляет удобный интерфейс для работы с параметрами запроса без ручного парсинга и формирования строк.

```javascript
const url = new URL("https://example.com");
url.searchParams.set("page", 2);
url.searchParams.set("sort", "name");
console.log(url.href); // "https://example.com/?page=2&sort=name"
```

**Полный набор операций:**

- `set(name, value)` — устанавливает параметр (удаляет все существующие значения и заменяет одним).
- `append(name, value)` — добавляет ещё одно значение (может быть несколько одинаковых ключей).
- `get(name)` — возвращает первое значение параметра.
- `getAll(name)` — возвращает массив всех значений параметра.
- `has(name)` — проверяет существование параметра.
- `delete(name)` — удаляет все значения параметра.
- `entries()` — итератор по парам [ключ, значение].
- `keys()` — итератор по ключам.
- `values()` — итератор по значениям.
- `toString()` — возвращает строку запроса (без `?`).

```javascript
const params = new URLSearchParams("?page=2&sort=name&filter=active");
console.log(params.get("page")); // "2"
console.log(params.get("sort")); // "name"
console.log(params.has("filter")); // true
params.set("page", "3");
console.log(params.toString()); // "page=3&sort=name&filter=active"
params.append("filter", "pending");
console.log(params.getAll("filter")); // ["active", "pending"]
```

**Кодирование специальных символов:**

Все специальные символы автоматически кодируются согласно правилам URL:

```javascript
const params = new URLSearchParams();
params.set("name", "John Smith & Sons");
params.set("query", "a=b&c=d");
console.log(params.toString());
// "name=John+Smith+%26+Sons&query=a%3Db%26c%3Dd"
```

- Пробел кодируется как `+` или `%20`.
- Символ `&` кодируется как `%26`.
- Символ `=` кодируется как `%3D`.

Это избавляет от необходимости самостоятельно использовать `encodeURIComponent()` и `decodeURIComponent()`.

**Использование с Fetch API:**

Объект `URLSearchParams` напрямую поддерживается Fetch API в качестве тела POST-запроса:

```javascript
const body = new URLSearchParams({
  login: "admin",
  password: "123456"
});

await fetch("/login", {
  method: "POST",
  body: body,
  headers: {
    "Content-Type": "application/x-www-form-urlencoded"
  }
});
```

Браузер автоматически устанавливает правильный заголовок `Content-Type: application/x-www-form-urlencoded;charset=UTF-8` и кодирует тело запроса.

---

## 15.4. Blob URL и временные ресурсы

Не все данные имеют постоянный интернет-адрес. Изображения, видео, PDF-файлы и другие бинарные данные могут существовать только в памяти браузера. Для таких объектов используется механизм **Blob URL** (также известный как Object URL).

```javascript
const blob = new Blob(["Hello, World!"], { type: "text/plain" });
const url = URL.createObjectURL(blob);
console.log(url); // "blob:https://example.com/550e8400-e29b-41d4-a716-446655440000"
```

**Структура Blob URL:**

```
blob:origin/UUID
```

- `blob:` — схема, указывающая на объект в памяти браузера.
- `origin` — протокол и домен страницы, создавшей URL.
- `UUID` — уникальный идентификатор объекта.

**Использование Blob URL:**

```javascript
// Отображение изображения
const imageBlob = await fetch("/api/avatar").then(r => r.blob());
const imageUrl = URL.createObjectURL(imageBlob);
document.getElementById("avatar").src = imageUrl;

// Скачивание файла
const fileBlob = new Blob(["Содержимое файла"], { type: "text/plain" });
const downloadUrl = URL.createObjectURL(fileBlob);
const link = document.createElement("a");
link.href = downloadUrl;
link.download = "file.txt";
link.click();

// Видео
const videoBlob = await fetch("/video").then(r => r.blob());
const videoUrl = URL.createObjectURL(videoBlob);
document.getElementById("video").src = videoUrl;
```

**Важные особенности:**

- Object URL существует только внутри текущего документа.
- Предоставляет доступ исключительно к связанному объекту `Blob` или `File`.
- Действует до тех пор, пока не будет явно освобождён или пока не выгрузится страница.

**Освобождение памяти:**

```javascript
URL.revokeObjectURL(url);
```

После вызова `revokeObjectURL()` адрес становится недействительным, а браузер получает возможность освободить связанные ресурсы. Если этого не сделать, объект продолжит удерживаться в памяти до выгрузки страницы, что может привести к утечкам памяти.

**Особенности использования в SPA:**

- Для долгоживущих SPA-приложений явный вызов `URL.revokeObjectURL()` является обязательной практикой.
- Для однократного использования (например, скачивание файла) можно освободить URL сразу после завершения операции.
- При обновлении изображения необходимо сначала отозвать старый URL, затем создать новый.

```javascript
function loadImage(blob, element) {
  if (element.src && element.src.startsWith('blob:')) {
    URL.revokeObjectURL(element.src);
  }
  const url = URL.createObjectURL(blob);
  element.src = url;
  return url;
}
```

---

## 15.5. URL как основа современной Web Platform

URL API используется практически всеми современными браузерными технологиями, делая URL универсальным механизмом адресации ресурсов независимо от их происхождения.

| Технология | Использование URL |
| ---------- | ----------------- |
| **ES Modules** | Разрешение путей импортируемых модулей через объектную модель URL. `import` использует URL для загрузки модулей. |
| **Fetch API** | URL как основной идентификатор сетевого ресурса. Первый аргумент `fetch()` может быть объектом URL. |
| **History API** | Изменение URL без полной перезагрузки страницы через `pushState()` и `replaceState()`. |
| **Service Workers** | Перехват запросов по URL для кеширования и офлайн-режима. |
| **Cache API** | Индексация ресурсов по URL. `cache.match()` и `cache.addAll()` используют URL. |
| **Blob URL** | Работа с объектами, существующими только в памяти браузера (изображения, файлы, видео). |
| **Navigation API** | Управление навигацией через URL-адреса. |
| **Import Maps** | Отображение логических имён модулей на реальные URL. |

**Пример единства подхода:**

```javascript
// Один и тот же URL может использоваться разными API
const url = new URL("/api/users", window.location.origin);

// ES Module import (динамический)
await import(url.href);

// Fetch запрос
await fetch(url);

// Service Worker регистрация
navigator.serviceWorker.register(url);

// Cache API
await caches.open('v1').then(cache => cache.add(url));
```

Благодаря единой модели URL, разработчики могут использовать один и тот же адрес для разных целей, что упрощает управление ресурсами и делает код более согласованным.

---

## Заключение главы

URL API давно перестал быть средством для обработки строк. Сегодня это фундаментальный компонент Web Platform, обеспечивающий единый способ адресации сетевых и локальных ресурсов.

Объект `URL` предоставляет доступ к каждой части адреса через отдельные свойства: `protocol`, `hostname`, `port`, `pathname`, `search`, `hash`. Конструктор `new URL()` автоматически проверяет корректность и разрешает относительные адреса относительно базового URL. `URLSearchParams` позволяет удобно работать с параметрами запроса без ручного парсинга — `get()`, `set()`, `append()`, `delete()`, автоматически кодируя специальные символы. Blob URL через `URL.createObjectURL()` и `URL.revokeObjectURL()` обеспечивают доступ к временным бинарным данным в памяти браузера, с обязательным освобождением памяти для предотвращения утечек.

Использование объектов `URL` и `URLSearchParams` делает код более надёжным, избавляет от ручного разбора строк и автоматически учитывает требования спецификации WHATWG URL Standard. Механизм Blob URL позволяет безопасно работать с временными бинарными данными, а единая модель URL используется большинством современных браузерных API — от ES Modules и Fetch до Service Workers и Cache API.

Для разработчика Modern JavaScript понимание URL API означает понимание того, как взаимодействуют между собой модули, сеть, файловые ресурсы, навигация и состояние приложения. Именно поэтому URL является одной из ключевых технологий современной Web Platform.