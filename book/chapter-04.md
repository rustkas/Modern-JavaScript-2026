# Часть II. Современный язык JavaScript

# Глава 4. Современный синтаксис: выразительность, композиция и безопасность

Современный JavaScript отличается не количеством новых ключевых слов, а изменением стиля разработки. Язык постепенно избавился от большинства шаблонного кода, сделав типичные операции более декларативными и безопасными.

Сегодня синтаксис языка ориентирован на три ключевых принципа:

- неизменяемость данных;
- композицию небольших функций;
- явное описание намерений разработчика.

В этой главе рассматриваются конструкции, которые стали основой современного JavaScript-кода независимо от используемого фреймворка или среды выполнения.

---

## 4.1. Неизменяемые привязки и блочная область видимости

Современный JavaScript практически отказался от использования `var`. Основными средствами объявления переменных стали `const` и `let`.

**`const`** — объявляет неизменяемую привязку. Значение, присвоенное `const`, не может быть переопределено в той же области видимости.

```javascript
const MAX_SIZE = 100;
MAX_SIZE = 200; // TypeError: Assignment to constant variable
```

**`let`** — объявляет изменяемую переменную с блочной областью видимости.

```javascript
let counter = 0;
counter = 1; // OK
```

**Блочная область видимости** означает, что переменные, объявленные с `let` и `const`, существуют только внутри блока `{}`, в котором они объявлены. Это отличается от `var`, который имеет функциональную область видимости и может быть доступен за пределами блока.

```javascript
if (true) {
  var varVariable = 'var';
  let letVariable = 'let';
  const constVariable = 'const';
}
console.log(varVariable); // 'var' — доступно
console.log(letVariable); // ReferenceError: letVariable is not defined
console.log(constVariable); // ReferenceError: constVariable is not defined
```

**Temporal Dead Zone (TDZ)** — период между входом в блок и фактическим объявлением переменной, когда переменная существует, но недоступна. Доступ к переменной в TDZ вызывает `ReferenceError`.

```javascript
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 10;
```

**Различие между неизменяемой привязкой и изменяемым объектом:** `const` гарантирует, что привязка не может быть переназначена, но не делает объект или массив неизменяемыми. Свойства объекта, объявленного через `const`, могут быть изменены.

```javascript
const user = { name: 'John' };
user.name = 'Jane'; // OK — объект изменён
user = { name: 'Jane' }; // TypeError — переназначение невозможно
```

**Рекомендации по использованию `const` по умолчанию:** всегда используйте `const`, если только переменная не должна быть переназначена. Это делает код более предсказуемым и защищает от случайных изменений. Используйте `let` только тогда, когда переназначение действительно необходимо.

---

## 4.2. Деструктуризация как язык работы с данными

Деструктуризация позволяет извлекать значения из объектов и массивов и присваивать их переменным с использованием синтаксиса, напоминающего структуру данных.

**Object Destructuring** — извлечение свойств объекта в переменные.

```javascript
const user = { id: 1, name: 'Alice', age: 30 };
const { id, name, age } = user;
console.log(id, name, age); // 1 'Alice' 30
```

**Array Destructuring** — извлечение элементов массива в переменные.

```javascript
const colors = ['red', 'green', 'blue'];
const [first, second, third] = colors;
console.log(first, second, third); // 'red' 'green' 'blue'
```

**Вложенные структуры** — деструктуризация вложенных объектов и массивов.

```javascript
const user = {
  id: 1,
  address: { city: 'Moscow', street: 'Tverskaya' }
};
const { address: { city, street } } = user;
console.log(city, street); // 'Moscow' 'Tverskaya'

const matrix = [[1, 2], [3, 4]];
const [[a, b], [c, d]] = matrix;
console.log(a, b, c, d); // 1 2 3 4
```

**Значения по умолчанию** — присваивание значений, если свойство отсутствует.

```javascript
const { username = 'Guest', role = 'user' } = { username: 'Alice' };
console.log(username); // 'Alice'
console.log(role); // 'user'
```

**Переименование свойств** — присвоение свойства объекта переменной с другим именем.

```javascript
const user = { id: 1, name: 'Alice' };
const { name: userName, id: userId } = user;
console.log(userName, userId); // 'Alice' 1
```

**Параметры функций** — деструктуризация в параметрах функции.

```javascript
function createUser({ name, email, age = 18 }) {
  console.log(name, email, age);
}
createUser({ name: 'Bob', email: 'bob@example.com' }); // 'Bob' 'bob@example.com' 18
```

Деструктуризация делает код более декларативным — она явно показывает, какие данные извлекаются из объекта, и облегчает рефакторинг, поскольку изменение структуры данных требует изменения только в месте деструктуризации, а не в нескольких строках кода.

---

## 4.3. Spread и Rest как основа неизменяемости

Оператор `...` (spread/rest) стал одним из главных инструментов современной разработки.

**Spread (`...`) для копирования объектов** — создание нового объекта, содержащего все свойства исходного.

```javascript
const user = { name: 'Alice', age: 30 };
const userCopy = { ...user };
console.log(userCopy); // { name: 'Alice', age: 30 }
```

**Spread для копирования массивов** — создание нового массива с теми же элементами.

```javascript
const numbers = [1, 2, 3];
const numbersCopy = [...numbers];
console.log(numbersCopy); // [1, 2, 3]
```

**Spread для объединения коллекций** — создание нового объекта или массива из нескольких источников.

```javascript
const user = { name: 'Alice', age: 30 };
const userDetails = { email: 'alice@example.com', age: 31 };
const merged = { ...user, ...userDetails };
console.log(merged); // { name: 'Alice', age: 31, email: 'alice@example.com' }

const arr1 = [1, 2];
const arr2 = [3, 4];
const combined = [...arr1, ...arr2];
console.log(combined); // [1, 2, 3, 4]
```

**Обновление вложенных структур** — создание нового объекта с изменённым вложенным свойством.

```javascript
const user = { name: 'Alice', address: { city: 'Moscow', street: 'Tverskaya' } };
const updatedUser = {
  ...user,
  address: { ...user.address, city: 'Saint Petersburg' }
};
console.log(updatedUser.address.city); // 'Saint Petersburg'
```

**Rest-параметры функций** — сбор оставшихся аргументов в массив.

```javascript
function logAll(first, ...rest) {
  console.log(first); // 1
  console.log(rest); // [2, 3, 4]
}
logAll(1, 2, 3, 4);
```

**Rest в деструктуризации** — сбор оставшихся свойств или элементов в переменную.

```javascript
const user = { id: 1, name: 'Alice', age: 30, role: 'admin' };
const { id, name, ...rest } = user;
console.log(id, name); // 1 'Alice'
console.log(rest); // { age: 30, role: 'admin' }

const colors = ['red', 'green', 'blue', 'yellow'];
const [first, second, ...others] = colors;
console.log(first, second); // 'red' 'green'
console.log(others); // ['blue', 'yellow']
```

**Ограничения поверхностного копирования (shallow copy):** Spread создаёт только поверхностную копию. Вложенные объекты и массивы копируются по ссылке, поэтому изменения во вложенных объектах затронут и исходный, и новый объект.

```javascript
const original = { name: 'Alice', address: { city: 'Moscow' } };
const copy = { ...original };
copy.address.city = 'Saint Petersburg';
console.log(original.address.city); // 'Saint Petersburg' — изменился
```

Для глубокого копирования используется `structuredClone()`:

```javascript
const deepCopy = structuredClone(original);
deepCopy.address.city = 'New York';
console.log(original.address.city); // 'Saint Petersburg' — не изменился
```

---

## 4.4. Безопасная работа с отсутствующими значениями

Современный JavaScript предлагает встроенные механизмы для работы с неполными данными, заменяющие длинные цепочки проверок.

**Optional Chaining (`?.`)** — безопасный доступ к свойствам вложенных объектов. Если значение слева от `?.` равно `null` или `undefined`, выражение возвращает `undefined` без ошибки.

```javascript
const user = { profile: { name: 'Alice' } };
console.log(user.profile?.name); // 'Alice'
console.log(user.settings?.theme); // undefined (без ошибки)

// С вызовом методов
const result = user.getSettings?.(); // undefined, если метод отсутствует

// С доступом к массиву
const firstItem = items?.[0]; // undefined, если items = null
```

**Nullish Coalescing (`??`)** — возвращает правый операнд только если левый равен `null` или `undefined`. Отличается от `||`, который возвращает правый операнд для любого ложного значения (`0`, `''`, `false`, `NaN`).

```javascript
const value1 = null ?? 'default'; // 'default'
const value2 = undefined ?? 'default'; // 'default'
const value3 = 0 ?? 'default'; // 0 (число 0 не является null или undefined)
const value4 = '' ?? 'default'; // '' (пустая строка не является null или undefined)

// Сравнение с ||
const value5 = 0 || 'default'; // 'default' (0 — ложное значение)
const value6 = '' || 'default'; // 'default' (пустая строка — ложное значение)
```

**Logical Assignment (`&&=`, `||=`, `??=`)** — комбинируют логические операции с присваиванием.

- `&&=` — присваивает значение, только если текущее значение истинно.
- `||=` — присваивает значение, только если текущее значение ложно.
- `??=` — присваивает значение, только если текущее значение равно `null` или `undefined`.

```javascript
let a = true;
a &&= false; // a = false (т.к. true && false = false)

let b = 0;
b ||= 10; // b = 10 (т.к. 0 || 10 = 10)

let c = null;
c ??= 'default'; // c = 'default' (т.к. null ?? 'default' = 'default')
let d = 0;
d ??= 100; // d = 0 (т.к. 0 не является null или undefined)
```

**Типичные сценарии использования:**

- Работа с API — безопасный доступ к данным ответа:

```javascript
const user = await fetchUser();
const city = user?.address?.city ?? 'Unknown';
```

- Конфигурационные объекты — использование значений по умолчанию:

```javascript
const config = loadConfig();
const port = config?.server?.port ?? 3000;
```

- Пользовательские настройки — проверка наличия параметров:

```javascript
const userSettings = getUserSettings();
const theme = userSettings?.preferences?.theme ?? 'light';
```

- Безопасный вызов методов — когда метод может отсутствовать:

```javascript
const handler = {
  process: (data) => console.log(data)
};
handler?.process?.(data) ?? console.log('No handler');
```

---

## 4.5. Современная работа со строками

Шаблонные литералы превратили строки в полноценный инструмент построения декларативного кода.

**Интерполяция** — вставка значений в строку с помощью `${expression}`:

```javascript
const name = 'Alice';
const age = 30;
const greeting = `Hello, ${name}! You are ${age} years old.`;
console.log(greeting); // 'Hello, Alice! You are 30 years old.'
```

**Многострочные строки** — строки, содержащие переносы без использования `\n`:

```javascript
const multiline = `
  This is a
  multiline
  string
`;
console.log(multiline);
```

**Tagged Templates** — функции, которые обрабатывают шаблонные литералы. Функция получает массив строковых фрагментов и значения интерполяций.

```javascript
function highlight(strings, ...values) {
  let result = '';
  for (let i = 0; i < strings.length; i++) {
    result += strings[i];
    if (i < values.length) {
      result += `<strong>${values[i]}</strong>`;
    }
  }
  return result;
}
const name = 'Alice';
const age = 30;
const html = highlight`User: ${name}, Age: ${age}`;
// 'User: <strong>Alice</strong>, Age: <strong>30</strong>'
```

Tagged Templates используются для создания предметно-ориентированных языков (DSL) и безопасного построения SQL-, HTML- и CSS-шаблонов. Например, библиотеки для работы с CSS-in-JS используют Tagged Templates для создания стилизованных компонентов:

```javascript
const Button = styled.button`
  background: blue;
  color: white;
  padding: 10px;
  &:hover {
    background: darkblue;
  }
`;
```

Tagged Templates остаются важным механизмом несмотря на развитие JSX и шаблонных систем фреймворков, поскольку они встроены в язык и не требуют дополнительных инструментов сборки.

---

## 4.6. Современные возможности объектов

JavaScript значительно расширил средства описания и работы с объектами.

**Сокращённая запись свойств** — когда имя свойства совпадает с именем переменной:

```javascript
const name = 'Alice';
const age = 30;
const user = { name, age };
console.log(user); // { name: 'Alice', age: 30 }
```

**Вычисляемые имена свойств** — динамические имена свойств в квадратных скобках:

```javascript
const propName = 'email';
const user = {
  name: 'Alice',
  [propName]: 'alice@example.com'
};
console.log(user.email); // 'alice@example.com'
```

**Методы объектов** — сокращённая запись функций в объектах:

```javascript
const user = {
  name: 'Alice',
  greet() {
    console.log(`Hello, ${this.name}`);
  }
};
user.greet(); // 'Hello, Alice'
```

**Object.hasOwn()** — безопасная проверка наличия собственного свойства (без учёта прототипов). Замена `Object.prototype.hasOwnProperty.call()`.

```javascript
const user = { name: 'Alice', age: 30 };
console.log(Object.hasOwn(user, 'name')); // true
console.log(Object.hasOwn(user, 'toString')); // false
```

**Object.groupBy()** — группировка элементов массива по ключу, возвращающемуся из функции-колбэка.

```javascript
const users = [
  { name: 'Alice', age: 30, role: 'admin' },
  { name: 'Bob', age: 25, role: 'user' },
  { name: 'Charlie', age: 30, role: 'user' }
];
const groupedByAge = Object.groupBy(users, (user) => user.age);
console.log(groupedByAge);
// {
//   30: [{ name: 'Alice', age: 30, role: 'admin' }, { name: 'Charlie', age: 30, role: 'user' }],
//   25: [{ name: 'Bob', age: 25, role: 'user' }]
// }
```

**Map.groupBy()** — аналогично `Object.groupBy()`, но возвращает `Map`, что позволяет использовать любые типы ключей.

```javascript
const groupedByRole = Map.groupBy(users, (user) => user.role);
// Map(2) { 'admin' => [...], 'user' => [...] }
```

**Object.fromEntries()** — преобразование массива пар [ключ, значение] в объект. Обратная операция к `Object.entries()`.

```javascript
const entries = [['name', 'Alice'], ['age', 30]];
const user = Object.fromEntries(entries);
console.log(user); // { name: 'Alice', age: 30 }
```

**Object.entries()** — преобразование объекта в массив пар [ключ, значение].

```javascript
const user = { name: 'Alice', age: 30 };
const entries = Object.entries(user);
console.log(entries); // [['name', 'Alice'], ['age', 30]]
```

Эти возможности упрощают преобразование данных и уменьшают количество вспомогательного кода, делая работу с объектами более декларативной и безопасной.

---

## 4.7. Современные классы

Несмотря на популярность функционального программирования, классы продолжают играть важную роль в организации кода.

**Public Fields** — объявление свойств непосредственно в теле класса:

```javascript
class User {
  name = 'Guest'; // Public field
  age = 18;
}
const user = new User();
console.log(user.name); // 'Guest'
```

**Private Fields** — приватные свойства, доступные только внутри класса. Обозначаются символом `#`.

```javascript
class BankAccount {
  #balance = 0;
  deposit(amount) {
    this.#balance += amount;
  }
  getBalance() {
    return this.#balance;
  }
}
const account = new BankAccount();
account.deposit(100);
console.log(account.getBalance()); // 100
console.log(account.#balance); // SyntaxError: Private field '#balance' is not accessible
```

**Private Methods** — приватные методы, доступные только внутри класса.

```javascript
class Calculator {
  #validate(value) {
    if (typeof value !== 'number') {
      throw new Error('Value must be a number');
    }
    return value;
  }
  add(a, b) {
    return this.#validate(a) + this.#validate(b);
  }
}
```

**Static Fields** — статические свойства, принадлежащие классу, а не экземпляру.

```javascript
class AppConfig {
  static API_URL = 'https://api.example.com';
  static TIMEOUT = 5000;
}
console.log(AppConfig.API_URL); // 'https://api.example.com'
```

**Static Blocks** — статические блоки инициализации, выполняемые при создании класса.

```javascript
class Database {
  static connection;
  static {
    this.connection = this.initializeConnection();
  }
  static initializeConnection() {
    // Сложная инициализация
    return { host: 'localhost', port: 5432 };
  }
}
```

**Наследование** — создание подклассов с помощью `extends`.

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    console.log(`${this.name} makes a sound`);
  }
}
class Dog extends Animal {
  constructor(name, breed) {
    super(name);
    this.breed = breed;
  }
  speak() {
    console.log(`${this.name} barks`);
  }
}
```

**Приватные элементы как механизм настоящей инкапсуляции** — в отличие от соглашений с использованием `_` (например, `_private`), приватные поля `#` действительно недоступны извне, что обеспечивает настоящую инкапсуляцию.

**Когда классы оправданы:**

- Когда требуется создание множества однотипных объектов с общим поведением.
- Когда необходима иерархия наследования.
- Когда нужна инкапсуляция состояния с приватными полями.
- Когда интеграция с существующим кодом, использующим классы.

**Когда предпочтительнее композиция функций:**

- Когда поведение должно быть переиспользовано независимо от структуры данных.
- Когда требуется гибкость и динамическое изменение поведения.
- Когда код не требует сложного состояния или иерархии.
- В функциональном стиле программирования.

---

## 4.8. Современный стиль написания кода

Синтаксис сам по себе не делает программу современной. Важно то, как эти конструкции используются в совокупности.

**Основные рекомендации современного стиля:**

1. **Использовать `const` по умолчанию** — объявлять переменные через `const`, если только они не требуют переназначения.

```javascript
// Плохо
let name = 'Alice';
let age = 30;

// Хорошо
const name = 'Alice';
let age = 30; // только если изменяется
```

2. **Отдавать предпочтение деструктуризации** — извлекать данные из объектов и массивов декларативно.

```javascript
// Плохо
const name = user.name;
const age = user.age;

// Хорошо
const { name, age } = user;
```

3. **Избегать мутации данных** — использовать spread и методы, возвращающие новые копии.

```javascript
// Плохо
const arr = [1, 2, 3];
arr.push(4);

// Хорошо
const arr = [1, 2, 3];
const newArr = [...arr, 4];
```

4. **Применять Optional Chaining вместо каскадов проверок** — упрощать доступ к вложенным данным.

```javascript
// Плохо
const city = user && user.address && user.address.city;

// Хорошо
const city = user?.address?.city;
```

5. **Использовать Spread вместо ручного копирования** — создавать копии объектов и массивов лаконично.

```javascript
// Плохо
const copy = Object.assign({}, obj);

// Хорошо
const copy = { ...obj };
```

6. **Писать небольшие композиционные функции** — каждая функция должна решать одну задачу.

```javascript
// Плохо
function processUser(user) {
  // 20 строк кода для валидации, преобразования, логирования
}

// Хорошо
const validateUser = (user) => { /* ... */ };
const transformUser = (user) => { /* ... */ };
const logUser = (user) => { /* ... */ };
const processUser = (user) => pipe(validateUser, transformUser, logUser)(user);
```

7. **Минимизировать шаблонный код** — использовать современные конструкции для сокращения повторяющегося кода.

```javascript
// Плохо
const fullName = user.firstName + ' ' + user.lastName;

// Хорошо
const fullName = `${user.firstName} ${user.lastName}`;
```

Совокупность этих практик делает код легче для чтения, тестирования и сопровождения. Код становится более предсказуемым, поскольку явно описывает намерения разработчика, а не только шаги выполнения.

---

## Заключение главы

Современный синтаксис JavaScript — это не набор отдельных языковых нововведений, а система взаимосвязанных средств, направленных на повышение выразительности, безопасности и предсказуемости кода. `const` и `let` заменили `var`, обеспечивая блочную область видимости и защиту от случайных переопределений. Деструктуризация упрощает извлечение данных и делает код более декларативным. Spread и Rest позволяют работать с данными без мутации, что критично для реактивных систем. Optional Chaining и Nullish Coalescing делают работу с неполными данными безопасной и лаконичной. Шаблонные литералы и Tagged Templates открывают возможности для создания DSL. Новые методы объектов упрощают преобразование данных. Современные классы обеспечивают настоящую инкапсуляцию. Все эти конструкции, используемые в совокупности, позволяют описывать намерения разработчика напрямую, сокращая объём шаблонного кода и снижая вероятность ошибок. Освоение этих возможностей формирует основу современного стиля программирования, который одинаково эффективно применяется как в браузерных приложениях, так и в серверных средах выполнения.