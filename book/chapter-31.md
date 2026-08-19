# Часть VIII. JavaScript и искусственный интеллект

## Глава 31. Агентные приложения: автономные системы и JavaScript как верховный оркестратор

К 2026 году парадигма разработки искусственного интеллекта совершила фундаментальный скачок от статичных диалоговых интерфейсов к автономным агентным системам. В этой эволюционной модели **JavaScript** закрепил за собой статус оркестратора, управляющего не просто отрисовкой ответов, но и предоставлением языковым моделям безопасного доступа к реальному миру через системные ресурсы и браузерные API. Актуальный стек «агентской» разработки позволяет нейросетям самостоятельно исполнять код, оперировать файловыми структурами и координировать распределённые задачи.

---

## 31.1. Model Context Protocol (MCP): стандартизация контекста и инструментов

Ключевым системным сдвигом стало массовое принятие **Model Context Protocol (MCP)** в качестве универсального отраслевого стандарта обмена данными между языковыми моделями и внешними инструментами.

**Что такое MCP:**

MCP — это открытый протокол, разработанный Anthropic, который стандартизирует способ предоставления языковым моделям доступа к инструментам и контексту. Он определяет единый интерфейс для подключения любых LLM к любым источникам данных и функциям.

**Архитектура MCP:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Клиентское приложение                    │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  MCP Клиент                          │  │
│  │   (JavaScript/TypeScript реализация)                 │  │
│  └─────────────┬───────────────────┬────────────────────┘  │
│                │                   │                       │
│  ┌─────────────▼───────┐   ┌───────▼─────────────────┐    │
│  │   MCP Сервер 1      │   │   MCP Сервер 2          │    │
│  │   (Файловая система)│   │   (База данных)        │    │
│  └─────────────┬───────┘   └───────┬─────────────────┘    │
│                │                   │                       │
│  ┌─────────────▼───────────────────▼─────────────────┐    │
│  │              Языковая модель (LLM)                 │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Как работает MCP:**

1. Сервер инструментов (Tools Server) предоставляет список доступных функций.
2. Клиент (JavaScript-приложение) передаёт этот список модели.
3. Модель решает, какие функции вызвать.
4. Клиент выполняет вызовы и возвращает результаты модели.

**Пример MCP-сервера на JavaScript:**

```javascript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

// Создание MCP-сервера
const server = new Server({
  name: 'file-system-tools',
  version: '1.0.0'
}, {
  capabilities: {
    tools: {}
  }
});

// Регистрация инструмента
server.setRequestHandler('tools/list', async () => {
  return {
    tools: [
      {
        name: 'read_file',
        description: 'Read the contents of a file',
        inputSchema: {
          type: 'object',
          properties: {
            path: { type: 'string', description: 'Path to file' }
          },
          required: ['path']
        }
      },
      {
        name: 'write_file',
        description: 'Write content to a file',
        inputSchema: {
          type: 'object',
          properties: {
            path: { type: 'string', description: 'Path to file' },
            content: { type: 'string', description: 'Content to write' }
          },
          required: ['path', 'content']
        }
      }
    ]
  };
});

// Обработка вызова инструмента
server.setRequestHandler('tools/call', async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === 'read_file') {
    const content = await fs.readFile(args.path, 'utf-8');
    return { content: [{ type: 'text', text: content }] };
  }
  
  if (name === 'write_file') {
    await fs.writeFile(args.path, args.content);
    return { content: [{ type: 'text', text: 'File written' }] };
  }
});

// Запуск сервера
const transport = new StdioServerTransport();
await server.connect(transport);
```

**Преимущества MCP:**

- **Унификация доступа:** MCP предоставляет JavaScript-приложениям стандартизированный способ трансляции безопасного контекста (локальных баз данных, хранилищ, корпоративных API) в пространство рассуждений языковой модели.

- **JavaScript как поставщик возможностей:** Вместо написания уникальных кастомных шлюзов под каждый ИИ-сервис разработчики используют JavaScript для развертывания компактных «серверов инструментов» (Tools Servers), которые любые передовые модели (Claude, GPT) могут динамически задействовать на лету.

- **Безопасность:** MCP определяет чёткие границы доступа, предотвращая неконтролируемое выполнение операций.

---

## 31.2. Function Calling: интеллектуальное управление системным окружением

Механизм **Function Calling** стёр грань между логикой приложения и когнитивными способностями модели, превратив JavaScript-функции в исполнительные «манипуляторы» искусственного интеллекта.

**Что такое Function Calling:**

Function Calling — это механизм, позволяющий языковым моделям вызывать внешние функции, описанные в JavaScript-коде. Модель получает список функций с их схемами и возвращает структурированный JSON, который указывает, какую функцию вызвать и с какими аргументами.

**Важное замечание о синтаксисе API.** Исходные параметры `functions` и `function_call`, с которых в 2023 году начиналась эта возможность у OpenAI, были официально признаны устаревшими ещё в декабре 2023 года (версия API `2023-12-01-preview`) и заменены на `tools` и `tool_choice`. Новый формат поддерживает параллельные вызовы нескольких функций за один ответ модели и расширяемый набор типов инструментов (не только пользовательские функции, но и, например, встроенный веб-поиск или MCP-серверы). Использовать устаревший синтаксис в новом коде в 2026 году не следует.

**Актуальный пример Function Calling:**

```javascript
// 1. Описание функций для модели (актуальный формат tools)
const tools = [
  {
    type: 'function',
    function: {
      name: 'get_current_weather',
      description: 'Get the current weather in a location',
      parameters: {
        type: 'object',
        properties: {
          location: {
            type: 'string',
            description: 'City name'
          }
        },
        required: ['location']
      }
    }
  }
];

// 2. Запрос к модели с инструментами
const response = await fetch('https://api.openai.com/v1/chat/completions', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${apiKey}` },
  body: JSON.stringify({
    model: 'gpt-4',
    messages: [{ role: 'user', content: 'What\'s the weather in London?' }],
    tools: tools,
    tool_choice: 'auto'
  })
});

const data = await response.json();
const toolCalls = data.choices[0].message.tool_calls;

// 3. Выполнение функции (модель может запросить несколько параллельно)
if (toolCalls) {
  const toolResults = [];
  for (const toolCall of toolCalls) {
    if (toolCall.function.name === 'get_current_weather') {
      const args = JSON.parse(toolCall.function.arguments);
      const weather = await getWeather(args.location);
      toolResults.push({
        role: 'tool',
        tool_call_id: toolCall.id,
        content: JSON.stringify(weather)
      });
    }
  }

  // 4. Возврат результатов модели
  const secondResponse = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${apiKey}` },
    body: JSON.stringify({
      model: 'gpt-4',
      messages: [
        { role: 'user', content: 'What\'s the weather in London?' },
        data.choices[0].message,
        ...toolResults
      ]
    })
  });
}
```

Стоит также иметь в виду, что для новых проектов OpenAI рекомендует более современный **Responses API** (`POST /v1/responses`) как предпочтительный интерфейс, хотя Chat Completions API с параметрами `tools`/`tool_choice`, показанный выше, по-прежнему широко используется и поддерживается.

**Ключевые возможности Function Calling:**

- **Декларативные спецификации:** Описывая строгие схемы и типы аргументов (с использованием JSON Schema), разработчик задаёт жёсткие рамки, в которых модель возвращает структурированный JSON-пакет для вызова целевой бизнес-логики.

- **Синергия с Browser APIs:** Агенты используют вызовы функций для взаимодействия с аппаратными уровнями платформы — например, для фиксации данных через `File System Access API` или управления системным буфером обмена через `Clipboard API`.

- **Интеграция с Web Platform:** Функции могут вызывать любые браузерные API — Fetch, Storage, DOM-манипуляции, Web Workers.


**Архитектурная схема Function Calling:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Пользовательский запрос                  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Языковая модель (LLM)                   │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │  Анализ запроса и выбор функции              │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│                       ▼                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │       JSON-вызов функции                            │  │
│  │  { name: 'get_weather', args: { location: 'London' }}│  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│                       ▼                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │        JavaScript-функция (оркестратор)            │  │
│  │   ┌─────────────────────────────────────────────┐  │  │
│  │   │  API-вызовы, Browser APIs, вычисления      │  │  │
│  │   └─────────────────────────────────────────────┘  │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│                       ▼                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │       Результат (возврат в модель)                 │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Высокопроизводительные рантаймы:**

Использование ультрабыстрых сред выполнения, таких как **Bun**, стало отраслевым стандартом для агентных CLI-инструментов.

**Почему Bun для агентных приложений:**

- Мгновенный запуск (в 4-5 раз быстрее Node.js).
- Встроенная поддержка TypeScript.
- Нативная поддержка HTTP/WebSocket.
- Высокая производительность ввода-вывода.

**Пример агентного инструмента на Bun:**

```javascript
// agent.ts
import { OpenAI } from 'openai';
import { readFile, writeFile } from 'node:fs/promises';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// Описание доступных функций
const tools = [
  {
    type: 'function',
    function: {
      name: 'read_file',
      description: 'Read a file from the filesystem',
      parameters: {
        type: 'object',
        properties: {
          path: { type: 'string', description: 'File path' }
        },
        required: ['path']
      }
    }
  },
  {
    type: 'function',
    function: {
      name: 'write_file',
      description: 'Write content to a file',
      parameters: {
        type: 'object',
        properties: {
          path: { type: 'string', description: 'File path' },
          content: { type: 'string', description: 'Content' }
        },
        required: ['path', 'content']
      }
    }
  }
];

// Основной цикл агента
async function agentLoop(userMessage) {
  const messages = [{ role: 'user', content: userMessage }];
  
  while (true) {
    const response = await openai.chat.completions.create({
      model: 'gpt-4',
      messages,
      tools,
      tool_choice: 'auto'
    });
    
    const message = response.choices[0].message;
    messages.push(message);
    
    // Если модель вызвала функцию
    if (message.tool_calls) {
      for (const toolCall of message.tool_calls) {
        const { name, arguments: args } = toolCall.function;
        const parsed = JSON.parse(args);
        
        let result;
        if (name === 'read_file') {
          result = await readFile(parsed.path, 'utf-8');
        } else if (name === 'write_file') {
          await writeFile(parsed.path, parsed.content);
          result = 'File written successfully';
        }
        
        messages.push({
          role: 'tool',
          tool_call_id: toolCall.id,
          content: result
        });
      }
    } else {
      // Финальный ответ
      return message.content;
    }
  }
}

// Запуск агента
const result = await agentLoop('Create a file called hello.txt with "Hello, World!"');
console.log(result);
```

---

## 31.3. AI SDK и Streaming UI: реактивность в реальном времени

Современные комплексы разработки интерфейсов (`AI SDK` от Vercel, Anthropic и др.) глубоко интегрированы с реактивными конвейерами языка.

**AI SDK (Vercel):**

AI SDK — это библиотека для интеграции ИИ в JavaScript-приложения, поддерживающая потоковую передачу данных и унифицированный интерфейс для различных моделей (OpenAI, Anthropic, Google, и другие).

```javascript
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

// Потоковая генерация текста
const stream = await streamText({
  model: openai('gpt-4'),
  prompt: 'Write a short story about AI'
});

// Чтение потока
for await (const chunk of stream) {
  console.log(chunk.text); // Покомпонентный вывод
}
```

**Streaming UI: реактивные интерфейсы для ИИ:**

Опираясь на нативные `Streams API` и реактивные фреймворки (React, Vue, Svelte), пользовательские интерфейсы динамически перерисовываются по мере генерации токенов моделью.

```tsx
// React-компонент с потоковой передачей
import { useChat } from 'ai/react';

function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    streamMode: 'text',
    onResponse: (response) => {
      // Обновление UI по мере поступления токенов
      console.log('Streaming response');
    }
  });

  return (
    <div>
      {messages.map((message, index) => (
        <div key={index}>
          <strong>{message.role}:</strong>
          <p>{message.content}</p>
        </div>
      ))}
      
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Type your message..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

**Streams API в действии:**

```javascript
// Потоковая передача данных от модели
async function* streamAIResponse(prompt) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({ prompt })
  });
  
  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    yield decoder.decode(value);
  }
}

// Использование в React
const [response, setResponse] = useState('');

useEffect(() => {
  const stream = streamAIResponse('Hello');
  for await (const chunk of stream) {
    setResponse(prev => prev + chunk);
  }
}, []);
```

**Преимущества потоковой передачи:**

- **Иммерсивный опыт:** Пользователь видит ответ по мере его генерации.
- **Покомпонентный рендеринг:** UI обновляется постепенно, без ожидания полного ответа.
- **Мгновенный отклик:** Первые токены появляются через сотни миллисекунд.

**Прогрессивное раскрытие контекста:**

Архитектурные паттерны динамической подгрузки данных защищают лимитированное окно контекста LLM от перегрузки, передавая в активную зону только релевантную в текущий момент информацию.

```javascript
// Инкрементальная загрузка контекста
async function getRelevantContext(query) {
  // 1. Поиск релевантных документов в векторной базе
  const docs = await vectorSearch(query, { limit: 5 });
  
  // 2. Постепенное добавление в контекст
  let context = '';
  for (const doc of docs) {
    context += doc.content;
    // 3. Проверка, не превышен ли лимит
    if (countTokens(context) > MAX_CONTEXT_TOKENS) break;
  }
  return context;
}
```

---

## 31.4. A2A (Agent-to-Agent): распределённый мультиагентный интеллект

Концепция **Agent-to-Agent (A2A)** описывает создание кооперативных систем, где специализированные ИИ-агенты разделяют ответственность для достижения общей цели.

**Что такое A2A:**

A2A — это протокол взаимодействия между несколькими ИИ-агентами, позволяющий им обмениваться информацией, делегировать задачи и координировать действия для достижения общей цели. Каждый агент специализируется на определённой области.

**Архитектура мультиагентной системы:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Пользовательский запрос                  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          Оркестратор (JavaScript)                    │  │
│  │          (EventTarget / Custom Events)               │  │
│  └──────┬──────────────┬─────────────┬─────────────────┘  │
│         │              │             │                    │
│  ┌──────▼─────┐  ┌─────▼─────┐  ┌───▼────────┐          │
│  │  Агент 1   │  │ Агент 2   │  │ Агент 3    │          │
│  │ (Дизайн)   │  │ (Код)     │  │ (Тесты)    │          │
│  └────────────┘  └───────────┘  └────────────┘          │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Результат (объединённый)                     │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Пример мультиагентной системы:**

```javascript
import { EventTarget } from 'events';

// Базовый класс агента
class Agent extends EventTarget {
  constructor(name, role) {
    super();
    this.name = name;
    this.role = role;
  }
  
  async process(task) {
    console.log(`[${this.name}] Processing: ${task}`);
    const result = await this.execute(task);
    this.dispatchEvent(new CustomEvent('completed', { detail: { result } }));
    return result;
  }
  
  async execute(task) {
    // Реализуется в подклассах
    throw new Error('Not implemented');
  }
}

// Специализированные агенты
class DesignAgent extends Agent {
  async execute(task) {
    // Проектирование архитектуры
    return { design: 'architecture.md', components: ['Header', 'Footer'] };
  }
}

class CodeAgent extends Agent {
  async execute(task) {
    // Написание кода
    return { files: ['Header.jsx', 'Footer.jsx'] };
  }
}

class TestAgent extends Agent {
  async execute(task) {
    // Написание тестов
    return { tests: ['Header.test.jsx', 'Footer.test.jsx'] };
  }
}

// Оркестратор
class AgentOrchestrator {
  constructor() {
    this.agents = [];
    this.events = new EventTarget();
  }
  
  addAgent(agent) {
    this.agents.push(agent);
    agent.addEventListener('completed', (event) => {
      this.onAgentComplete(agent, event.detail.result);
    });
  }
  
  async orchestrate(task) {
    const results = {};
    
    for (const agent of this.agents) {
      results[agent.role] = await agent.process(task);
    }
    
    return results;
  }
  
  onAgentComplete(agent, result) {
    console.log(`[${agent.name}] Completed with:`, result);
    this.events.dispatchEvent(new CustomEvent('agent-done', {
      detail: { agent: agent.role, result }
    }));
  }
}

// Использование
const orchestrator = new AgentOrchestrator();
orchestrator.addAgent(new DesignAgent('Designer', 'design'));
orchestrator.addAgent(new CodeAgent('Coder', 'code'));
orchestrator.addAgent(new TestAgent('Tester', 'test'));

const result = await orchestrator.orchestrate('Build a React component library');
console.log('Final result:', result);
```

**Разделение ролей в A2A:**

| Агент | Роль | Примеры задач |
|-------|------|---------------|
| Дизайнер | Проектирование | Архитектура, структура, компоненты |
| Кодер | Разработка | Написание кода, реализация логики |
| Тестировщик | Валидация | Написание тестов, проверка качества |
| Сборщик | Инфраструктура | Сборка, деплой, конфигурация |

**Событийная координация:**

Взаимодействие между агентами строится на базе встроенных механизмов платформы (`Custom Events`, шины на базе `EventTarget`), обеспечивая слаженность работы без жёсткой привязки к проприетарной инфраструктуре конкретного ИИ-вендора.

**Преимущества A2A:**

- **Масштабируемость:** Каждый агент специализируется на своей задаче.
- **Отказоустойчивость:** Сбой одного агента не блокирует всю систему.
- **Гибкость:** Можно добавлять новых агентов без изменения существующих.
- **Параллелизм:** Агенты могут работать одновременно.

---

## Заключение

В 2026 году проектирование агентных систем на JavaScript сводится к архитектуре безопасных, строго типизированных интерфейсов, которыми оперирует автономный интеллект. JavaScript выступает незаменимым мостом между виртуальным «мозгом» (LLM) и материальным «телом» (Web Platform), гарантируя исполнение сложных комплексных задач в защищённом контуре.

**Model Context Protocol (MCP)** стандартизирует доступ к инструментам и контексту, позволяя JavaScript-приложениям транслировать безопасный контекст (локальные базы данных, хранилища, корпоративные API) в пространство рассуждений языковой модели через серверы инструментов.

**Function Calling** стёр грань между логикой приложения и когнитивными способностями модели. Модель выбирает функцию из описанных схем и возвращает структурированный JSON для вызова JavaScript-функций, которые могут взаимодействовать с Browser APIs — File System Access, Clipboard API, Storage, Fetch.

**AI SDK** и **Streaming UI** обеспечивают потоковую передачу данных через Streams API с покомпонентным рендерингом в реальном времени, формируя иммерсивный опыт мгновенного отклика.

**A2A (Agent-to-Agent)** описывает кооперативные системы, где специализированные ИИ-агенты разделяют ответственность. Взаимодействие строится на базе встроенных механизмов платформы (`Custom Events`, `EventTarget`), обеспечивая слаженность работы без привязки к проприетарной инфраструктуре.

Навык проектирования схем вызовов и потоковой доставки контента становится для инженера приоритетнее написания классической рутинной логики. JavaScript выполняет роль универсального оркестратора, связывающего языковые модели с аппаратными и системными возможностями Web Platform.
