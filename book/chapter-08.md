# Часть III. Работа с Web Platform

# Глава 8. DOM: объектная модель документа и современная архитектура пользовательского интерфейса

Несмотря на стремительное развитие Web Platform, **Document Object Model (DOM)** остаётся центральной моделью представления пользовательского интерфейса в браузере. Именно через DOM браузер связывает HTML-документ, CSS и JavaScript в единую исполняемую систему.

Однако роль JavaScript существенно изменилась. Если раньше код непосредственно создавал, удалял и перестраивал элементы страницы, то современные приложения стремятся минимизировать количество прямых операций с DOM. JavaScript всё чаще управляет состоянием приложения, а платформа, браузер или UI-фреймворк самостоятельно синхронизируют это состояние с документом.

Понимание устройства DOM остаётся необходимым даже при использовании React, Vue, Angular, Svelte, Lit или Web Components, поскольку все они в конечном итоге работают поверх одной и той же объектной модели документа.

---

## 8.1. DOM как объектная модель Web Platform

DOM представляет HTML-документ не как текст, а как взаимосвязанную систему объектов.

Каждый HTML-элемент становится экземпляром определённого интерфейса платформы.

Основу этой модели образуют несколько ключевых типов:

- **Node** — базовый интерфейс для всех узлов дерева документа. Предоставляет основные методы навигации: `parentNode`, `childNodes`, `firstChild`, `nextSibling`. Все остальные типы узлов наследуют от Node.

- **Document** — корневой объект страницы и точка входа для большинства браузерных API. Через `document` доступны методы создания элементов (`createElement`), поиска по селекторам (`querySelector`), работа с cookies (`document.cookie`) и многое другое.

- **Element** — базовый тип HTML- и SVG-элементов. Добавляет методы работы с атрибутами (`getAttribute`, `setAttribute`, `hasAttribute`), поиска вложенных элементов (`querySelector`, `getElementsByTagName`) и управления классами (`classList`).

- **HTMLElement** — специализированная ветвь HTML-интерфейсов (`HTMLDivElement`, `HTMLButtonElement`, `HTMLInputElement` и др.). Каждый HTML-тег имеет свой интерфейс с дополнительными свойствами. Например, `HTMLInputElement` добавляет свойства `value`, `checked`, `type`, а `HTMLAnchorElement` — `href`, `target`, `rel`.

- **Text** — текстовые узлы, содержащие текстовое содержимое элементов. Не могут иметь дочерних узлов.

- **Comment** — узлы комментариев HTML (`<!-- ... -->`). Не влияют на отображение, но присутствуют в DOM.

- **DocumentFragment** — лёгкий контейнер для временного хранения узлов. Не является частью основного документа и не влияет на отображение, пока не будет вставлен.

Интерфейсы Web IDL реализуются через прототипную модель JavaScript. Каждый объект DOM наследует методы от своих прототипов в цепочке: `HTMLDivElement` → `HTMLElement` → `Element` → `Node` → `EventTarget` → `Object`.

**Пример цепочки прототипов:**

```javascript
const div = document.createElement('div');
console.log(div instanceof HTMLDivElement); // true
console.log(div instanceof HTMLElement);    // true
console.log(div instanceof Element);        // true
console.log(div instanceof Node);           // true
console.log(div instanceof EventTarget);    // true
```

Это означает, что `div` имеет доступ ко всем методам, определённым на каждом из этих интерфейсов: от `addEventListener` (EventTarget) до `appendChild` (Node) и `getAttribute` (Element).

---

## 8.2. DOM как живое дерево объектов

DOM не является снимком HTML-файла. После загрузки страницы браузер поддерживает постоянно изменяющееся дерево объектов, которое отражает текущее состояние документа.

Любое изменение структуры документа, атрибутов, текстового содержимого, пользовательского ввода или результатов работы JavaScript немедленно отражается в DOM. Это означает, что DOM всегда актуален — он не требует синхронизации с каким-либо внутренним состоянием, он сам является этим состоянием.

**Навигация по дереву DOM:**

- `parentNode` / `parentElement` — доступ к родительскому узлу или элементу.
- `childNodes` — коллекция всех дочерних узлов (включая текстовые узлы и комментарии).
- `children` — коллекция только дочерних элементов (без текстовых узлов).
- `firstChild` / `lastChild` — первый и последний дочерний узел.
- `firstElementChild` / `lastElementChild` — первый и последний дочерний элемент.
- `nextSibling` / `previousSibling` — следующий и предыдущий узел-сосед.
- `nextElementSibling` / `previousElementSibling` — следующий и предыдущий элемент-сосед.

**Создание и удаление узлов:**

- `document.createElement(tagName)` — создание нового элемента.
- `document.createTextNode(text)` — создание текстового узла.
- `parent.appendChild(child)` — добавление дочернего узла в конец.
- `parent.removeChild(child)` — удаление дочернего узла.
- `element.remove()` — удаление элемента (современный метод, без обращения к родителю).
- `element.cloneNode(deep)` — клонирование узла. Если `deep === true`, копируются все дочерние узлы.

**DocumentFragment как инструмент пакетных изменений:**

DocumentFragment — это контейнер для узлов, который не является частью основного документа. Изменения внутри DocumentFragment не вызывают перерасчёт Layout до тех пор, пока фрагмент не будет вставлен в документ.

**Пример использования DocumentFragment:**

```javascript
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const item = document.createElement('li');
  item.textContent = `Item ${i}`;
  fragment.appendChild(item);
}
document.getElementById('list').appendChild(fragment);
```

Без DocumentFragment каждое добавление элемента вызывало бы перерасчёт Layout. С DocumentFragment все 1000 элементов добавляются за одну операцию, вызывая только один перерасчёт Layout. Это значительно повышает производительность при массовых изменениях.

---

## 8.3. Современная работа с DOM

За последние годы стиль работы с DOM значительно изменился. Вместо постоянного поиска элементов через универсальные селекторы современные приложения стремятся минимизировать обращения к DOM, кэшировать ссылки на элементы, использовать делегирование событий, изменять состояние пакетно и избегать лишних перерасчётов Layout.

**Современные методы работы с DOM:**

- **`querySelector(selector)` / `querySelectorAll(selector)`** — поиск элементов по CSS-селектору. Заменяют устаревшие `getElementById`, `getElementsByClassName`, `getElementsByTagName`. Возвращают первый найденный элемент или статическую коллекцию всех подходящих элементов.

```javascript
const button = document.querySelector('#submit-btn');
const allItems = document.querySelectorAll('.list-item');
```

- **`element.closest(selector)`** — поиск ближайшего родительского элемента (включая сам элемент), соответствующего селектору. Полезен для делегирования событий.

```javascript
const card = event.target.closest('.product-card');
if (card) {
  const id = card.dataset.productId;
}
```

- **`element.matches(selector)`** — проверка, соответствует ли элемент указанному CSS-селектору. Возвращает `true` или `false`.

```javascript
if (element.matches('.active.highlight')) {
  // Элемент имеет оба класса
}
```

- **`element.replaceChildren(...nodes)`** — атомарная замена всех дочерних элементов. Удаляет всех детей и добавляет новые за одну операцию.

```javascript
list.replaceChildren(...newItems); // Быстро заменяет все элементы
```

- **`element.append(...nodes)` / `element.prepend(...nodes)`** — добавление элементов в конец или начало списка дочерних элементов. Принимает несколько узлов или строк.

```javascript
list.append('Text', newElement, 'More text');
```

- **`element.before(...nodes)` / `element.after(...nodes)`** — вставка элементов до или после текущего элемента.

```javascript
element.before('<p>Перед элементом</p>');
element.after(newElement);
```

- **`element.replaceWith(...nodes)`** — замена текущего элемента новыми узлами.

```javascript
oldElement.replaceWith(newElement);
```

- **`element.classList`** — объект для работы с классами. Содержит методы `add()`, `remove()`, `toggle()`, `contains()`, `replace()`.

```javascript
element.classList.add('active');
element.classList.toggle('expanded');
element.classList.remove('hidden');
```

- **`element.dataset`** — объект, предоставляющий доступ к `data-*` атрибутам. Имена атрибутов преобразуются из kebab-case в camelCase: `data-user-id` становится `dataset.userId`.

```javascript
element.dataset.userId = '123'; // Устанавливает data-user-id="123"
const id = element.dataset.userId; // Читает data-user-id
```

**Пример современного подхода:**

```javascript
// Устаревший подход
const elem = document.getElementById('myId');
elem.className += ' active';
elem.setAttribute('data-count', '5');

// Современный подход
const elem = document.querySelector('#myId');
elem.classList.add('active');
elem.dataset.count = '5';
```

Именно эти методы сегодня составляют основу повседневной работы с DOM. Они более читаемы, безопасны и производительны.

---

## 8.4. Shadow DOM и инкапсуляция компонентов

Одним из важнейших этапов развития Web Platform стало появление Shadow DOM. Он позволяет создавать полностью инкапсулированные компоненты, внутреннее устройство которых скрыто от внешнего документа.

**Открытый и закрытый Shadow Root:**

- `attachShadow({ mode: 'open' })` — создаёт открытый Shadow Root. Доступ к нему можно получить через `element.shadowRoot`.
- `attachShadow({ mode: 'closed' })` — создаёт закрытый Shadow Root. `element.shadowRoot` возвращает `null`. Используется для полной инкапсуляции.

```javascript
// Открытый Shadow DOM
const openShadow = element.attachShadow({ mode: 'open' });
console.log(element.shadowRoot); // ShadowRoot

// Закрытый Shadow DOM
const closedShadow = element.attachShadow({ mode: 'closed' });
console.log(element.shadowRoot); // null
```

**Дерево теневого DOM** — это изолированное дерево узлов, которое не влияет на основной документ. Стили, определённые в Shadow DOM, не проникают наружу, а стили снаружи не проникают внутрь (за исключением наследуемых свойств).

**Распределение содержимого через `<slot>`** — элементы `<slot>` определяют точки вставки для содержимого, переданного в компонент.

```html
<my-component>
  <span slot="header">Мой заголовок</span>
  <p>Основное содержимое</p>
</my-component>
```

Внутри Shadow DOM:

```html
<div>
  <slot name="header">Заголовок по умолчанию</slot>
  <slot>Основное содержимое по умолчанию</slot>
</div>
```

**Наследование стилей** — стили из основного документа не проникают в Shadow DOM, если явно не разрешено с помощью CSS-свойств (например, `color`, `font` наследуются). Для явного наследования используется CSS-свойство `all: inherit` или наследуемые свойства.

**Взаимодействие с Custom Elements** — Shadow DOM часто используется вместе с Custom Elements для создания полноценных Web Components.

**Пример создания Shadow DOM:**

```javascript
class MyComponent extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        :host { display: block; border: 1px solid #ccc; padding: 10px; }
        :host(.fancy) { border-color: gold; }
        ::slotted(.highlight) { color: red; }
      </style>
      <div part="content">
        <slot name="header">Default header</slot>
        <slot>Default content</slot>
      </div>
    `;
  }
}
customElements.define('my-component', MyComponent);
```

Web Components стали фундаментом многих современных UI-библиотек, поскольку они обеспечивают настоящую изоляцию, недостижимую при использовании только CSS-классов или BEM. Компоненты, созданные с помощью Shadow DOM, не конфликтуют с остальным кодом страницы.

---

## 8.5. Наблюдение за изменениями документа

Современный JavaScript редко использует периодический опрос состояния страницы. Вместо этого платформа предоставляет специализированные механизмы наблюдения, которые эффективно отслеживают изменения и уведомляют приложение.

**`MutationObserver`** — наблюдение за изменениями DOM. Отслеживает добавление и удаление узлов, изменение атрибутов, изменение текстового содержимого.

```javascript
const observer = new MutationObserver((mutations) => {
  mutations.forEach((mutation) => {
    if (mutation.type === 'attributes') {
      console.log(`Атрибут ${mutation.attributeName} изменён`);
    }
    if (mutation.type === 'childList') {
      console.log('Добавлены или удалены дочерние узлы');
    }
  });
});
observer.observe(targetElement, {
  attributes: true,
  childList: true,
  subtree: true
});
```

`MutationObserver` используется в библиотеках для отслеживания изменений в DOM, например, для автоматической перекомпиляции шаблонов.

**`ResizeObserver`** — наблюдение за изменением размеров элементов. Более эффективен, чем `window.onresize` с вычислениями, поскольку срабатывает только при изменении размера конкретных элементов.

```javascript
const resizeObserver = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    console.log(`Размер элемента: ${width}x${height}`);
  }
});
resizeObserver.observe(element);
```

**`IntersectionObserver`** — наблюдение за видимостью элементов в области просмотра. Используется для ленивой загрузки изображений, бесконечной прокрутки, анимаций при появлении.

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      loadImage(entry.target);
      observer.unobserve(entry.target);
    }
  });
}, { threshold: 0.1 });
document.querySelectorAll('img[data-src]').forEach(img => observer.observe(img));
```

**`PerformanceObserver`** — наблюдение за метриками производительности. Используется для сбора данных о Core Web Vitals и других показателях.

```javascript
const perfObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === 'largest-contentful-paint') {
      console.log(`LCP: ${entry.startTime}`);
    }
  }
});
perfObserver.observe({ entryTypes: ['largest-contentful-paint', 'layout-shift', 'first-input'] });
```

Каждый из этих API решает свою задачу:

- `MutationObserver` — изменение структуры и атрибутов DOM.
- `ResizeObserver` — изменение размеров элементов.
- `IntersectionObserver` — появление элемента в области просмотра.
- `PerformanceObserver` — анализ производительности приложения.

Браузер самостоятельно сообщает приложению об изменениях вместо постоянного polling, что экономит ресурсы и повышает производительность.

---

## 8.6. DOM и Rendering Pipeline

Любая модификация DOM потенциально влияет на работу графического конвейера браузера. Графический конвейер (Rendering Pipeline) — это последовательность этапов, которые браузер выполняет для отображения страницы.

**Последовательность этапов:**

```
JavaScript
  ↓
DOM
  ↓
Style
  ↓
Layout
  ↓
Paint
  ↓
Composite
```

**Style** — вычисление стилей для каждого элемента. Браузер применяет CSS-правила к элементам DOM с учётом каскада, специфичности и наследования.

**Layout (или Reflow)** — вычисление геометрии элементов. Браузер определяет размеры и позиции каждого элемента на странице. Это самый дорогой этап, поскольку затрагивает всё дерево документа.

**Paint** — отрисовка пикселей. Браузер создаёт слои и заполняет их цветами, изображениями и текстом.

**Composite** — объединение слоёв. Браузер комбинирует все слои в финальное изображение для отображения на экране. Самый быстрый этап.

**Операции, вызывающие Layout:**

```javascript
// Чтение геометрических свойств
const height = element.offsetHeight;
const width = element.offsetWidth;
const rect = element.getBoundingClientRect();
const scrollTop = element.scrollTop;

// Изменение геометрических свойств
element.style.height = '100px';
element.style.width = '50%';
element.style.margin = '10px';
element.style.display = 'none'; // Изменение display на block/none
element.style.position = 'absolute';

// Добавление или удаление элементов
element.appendChild(newChild);
element.removeChild(oldChild);
```

**Операции, вызывающие только Paint (без Layout):**

```javascript
// Изменение визуальных свойств, не влияющих на геометрию
element.style.backgroundColor = 'red';
element.style.color = 'blue';
element.style.opacity = '0.5';
element.style.boxShadow = '0 0 10px rgba(0,0,0,0.5)';
```

**Операции, выполняющиеся только на этапе Composite (без Layout и Paint):**

```javascript
// CSS-трансформации и opacity
element.style.transform = 'translateX(10px)';
element.style.opacity = '0.5'; // Если элемент уже отрисован
element.style.transform = 'scale(0.8)';
```

**Layout Thrashing** — проблема, возникающая при чередовании чтения и записи геометрических свойств. Каждое чтение после записи вызывает принудительный перерасчёт Layout, чтобы гарантировать актуальность данных.

```javascript
// Плохо — layout thrashing
for (let i = 0; i < elements.length; i++) {
  const height = elements[i].offsetHeight; // Чтение → принудительный Layout
  elements[i].style.height = height + 10 + 'px'; // Запись → планирует новый Layout
}
// На каждой итерации происходит перерасчёт

// Хорошо — пакетные операции
const heights = elements.map(el => el.offsetHeight); // Все чтения (один Layout)
elements.forEach((el, i) => {
  el.style.height = heights[i] + 10 + 'px'; // Все записи (один Layout)
});
// Всего два перерасчёта вместо тысячи
```

Минимизация количества изменений DOM остаётся одним из важнейших факторов производительности. Каждый перерасчёт Layout может занимать миллисекунды, а при 60 кадрах в секунду (16.7 мс на кадр) даже несколько лишних перерасчётов могут привести к пропуску кадров и видимым задержкам.

---

## 8.7. DOM в эпоху современных UI-фреймворков

Современные приложения редко работают с DOM напрямую. React, Vue, Angular, Svelte, Solid, Lit и Web Components используют различные модели синхронизации состояния, однако все они в конечном счёте взаимодействуют с одной и той же объектной моделью документа.

**Основные подходы к обновлению DOM:**

**Virtual DOM (React, Vue)** — создание виртуального дерева, отражающего желаемое состояние интерфейса. После каждого изменения состояния создаётся новое виртуальное дерево, которое сравнивается с предыдущим (диффинг). Затем вычисляется минимальный набор изменений для применения к реальному DOM.

```javascript
// Упрощённый пример Virtual DOM
const oldVNode = h('div', { class: 'container' }, 'Hello');
const newVNode = h('div', { class: 'container active' }, 'Hello World');
const patches = diff(oldVNode, newVNode);
patch(domElement, patches);
```

**Fine-Grained Reactivity (Solid, Svelte)** — отслеживание изменений на уровне отдельных переменных. Система реактивности знает, какие части DOM зависят от каждой переменной, и обновляет только их при изменении. Не требуется диффинг всего дерева.

```javascript
// Упрощённый пример Fine-Grained Reactivity
const name = signal('World');
createEffect(() => {
  element.textContent = `Hello ${name()}`; // Зависит от name
});
name.set('JavaScript'); // Обновляется только textContent элемента
```

**Signals (Angular, Qwik, Preact)** — реактивные примитивы, представляющие собой наблюдаемые значения. Компоненты подписываются на сигналы и обновляются автоматически при их изменении.

```javascript
// Пример с Signals
const count = signal(0);
const doubled = computed(() => count() * 2);
createEffect(() => {
  console.log(`Count: ${count()}, Double: ${doubled()}`);
});
count.set(5); // Выведет: Count: 5, Double: 10
```

**Компиляция шаблонов (Svelte, Vue)** — анализ шаблонов на этапе сборки и генерация оптимального кода обновления DOM. Компилятор анализирует, какие переменные используются в каждом месте шаблона, и генерирует код, который обновляет только конкретные узлы.

**Нативные Web Components (Lit, Stencil)** — использование встроенных механизмов платформы: Custom Elements, Shadow DOM, HTML Templates. Обновления выполняются через стандартные методы, такие как `attributeChangedCallback` и `requestUpdate`.

Различия между современными фреймворками заключаются прежде всего в стратегии обновления DOM, а не в самой модели документа. Все они в конечном счёте вызывают одни и те же методы DOM API, такие как `setAttribute`, `textContent`, `appendChild`, `removeChild`. Выбор фреймворка определяет, как часто и как именно происходят эти вызовы.

---

## 8.8. JavaScript как координатор Web Platform

В современной архитектуре JavaScript выступает не непосредственным «рисовальщиком» интерфейса, а координатором взаимодействия многочисленных API платформы.

DOM становится лишь одним из участников общей системы, включающей:

- **CSSOM** — объектная модель CSS. JavaScript может управлять стилями через `element.style`, `CSSStyleSheet`, `CSS Typed OM`. Современные приложения используют CSS-переменные и CSS-классы для управления стилями, а не прямой доступ к стилям элементов.

- **Rendering Pipeline** — графический конвейер. JavaScript инициирует изменения, но браузер самостоятельно выполняет Style, Layout, Paint и Composite.

- **Fetch API** — сетевое взаимодействие. JavaScript отправляет запросы, получает данные и обновляет состояние приложения, что приводит к изменениям в DOM.

- **Navigation API** — управление навигацией. JavaScript может перехватывать навигацию, изменять URL и управлять историей без перезагрузки страницы.

- **Web Components** — компонентная архитектура. JavaScript создаёт кастомные элементы, управляет их жизненным циклом и взаимодействием.

- **Web Workers** — многопоточность. JavaScript переносит тяжёлые вычисления в отдельные потоки, не блокируя основной поток и DOM.

- **Streams API** — потоки данных. JavaScript обрабатывает большие объёмы данных по частям, постепенно обновляя DOM.

- **Storage API** — хранение данных. JavaScript сохраняет состояние приложения в localStorage, sessionStorage или IndexedDB.

Основная задача JavaScript — управлять состоянием приложения и координировать работу платформы, а не выполнять все операции самостоятельно. Это означает, что код становится более декларативным: разработчик описывает, каким должно быть состояние, а платформа определяет, как именно его отобразить.

---

## Заключение главы

DOM остаётся фундаментальной частью Web Platform, однако его роль изменилась. Если раньше JavaScript напрямую управлял каждым элементом страницы, то современные приложения стремятся минимизировать количество непосредственных изменений документа, передавая большую часть работы браузеру или специализированным механизмам платформы. Понимание устройства DOM, жизненного цикла узлов, принципов инкапсуляции через Shadow DOM и влияния изменений на Rendering Pipeline позволяет создавать интерфейсы, которые одновременно остаются масштабируемыми, производительными и совместимыми с любой современной средой разработки.