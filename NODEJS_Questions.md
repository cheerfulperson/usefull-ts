## 1\. What can you do with `for await` on a `request: IncomingMessage` instance?

You can treat an HTTP `IncomingMessage` (the `request` object in a server callback) as an async iterable. That means:

*   Read the request body chunk by chunk without manually wiring `'data'` and `'end'` events.
    
*   Process large payloads (file uploads, streaming JSON, multipart forms) in a memory-efficient manner.
    
*   Apply backpressure automatically: the loop pauses if downstream processing is slower than incoming data.
    

Example:

```js
import http from 'node:http';

http.createServer(async (req, res) => {
  let body = '';
  for await (const chunk of req) {
    body += chunk;
  }
  res.end(`Received ${body.length} bytes`);
}).listen(3000);
```

## 2\. How does Node.js natively hash passwords and when do you need external dependencies?

Node.js’s built-in `crypto` module offers:

*   `crypto.pbkdf2()` / `crypto.pbkdf2Sync()` for PBKDF2.
    
*   `crypto.scrypt()` / `crypto.scryptSync()` for scrypt (since v10.5.0).
    

Use these when you need a robust, memory-hard algorithm without extra installs.

You need external packages when you want:

*   Bcrypt (`bcrypt`, `bcryptjs`) for compatibility with other ecosystems.
    
*   Argon2 (`argon2` native binding) for the current state-of-the-art memory-hard hashing.
    
*   Higher-level abstractions (e.g., password salting, hash versioning in one API).
    

## 3\. What API does `nodejs/undici` implement?

Undici is Node.js’s high-performance HTTP/1.1 client library that implements:

*   The WHATWG Fetch standard (`fetch()`, `Request`, `Response`, `Headers`).
    
*   A low-level dispatcher/connection pool API (`Pool`, `Agent`).
    
*   HTTP/1.1 pipelining, keep-alive, and streaming bodies.
    

It’s the upstream for Node’s built-in `fetch` and is tuned for low overhead and high concurrency.

## 4\. What is a modern replacement for the `node:domain` API?

Domains were Node’s attempt at catching errors across async boundaries but are now deprecated. Modern approaches include:

*   Relying on promise chains and `async/await` with `try/catch`.
    
*   Using `AsyncLocalStorage` (from `async_hooks`) to maintain context across async calls.
    
*   Attaching explicit error handlers on event emitters and streams.
    
*   Leveraging `AbortController` to cancel operations on failure.
    

This enforces predictable error flow instead of the implicit “domain catch-all.”

## 5\. When can we use synchronous versions of file operations from `node:fs` instead of asynchronous ones and what should we look for?

Use sync `fs` APIs only in:

*   Startup scripts or initialization code (config load, build scripts, CLI tools).
    
*   One-off maintenance utilities where simplicity outweighs blocking concerns.
    

Avoid in request/worker handlers because:

*   They block the event loop, freezing all other operations.
    
*   Latency spikes if I/O is slow or filesystem is overloaded.
    

Ask yourself:

*   Will this run on every request or just once at boot?
    
*   Is the operation time-sensitive relative to other tasks?
    
*   Can blocking here degrade overall throughput?
    

## 6\. Propose best practices for handling errors in asynchronous code.

1.  Always attach error handlers
    
    *   Use `try/catch` inside `async` functions.
        
    *   Add `.catch()` to every promise chain.
        
2.  Centralize logging and response formatting
    
    *   Funnel errors through a single logger or middleware.
        
    *   Convert exceptions to standardized API error objects.
        
3.  Use AbortController for cancellation
    
    *   Pass an `AbortSignal` to long-running operations.
        
    *   Listen for aborts and clean up resources.
        
4.  Avoid unhandled rejections
    
    *   Register a global `'unhandledRejection'` listener during startup.
        
    *   Exit or restart process gracefully if critical.
        
5.  Fail fast on critical errors
    
    *   Don’t swallow errors silently.
        
    *   Crash and restart (e.g., via a process manager) if the state is compromised.
        

## 7\. How can vulnerabilities appear in node projects? Explain one: SQL Injection and how to prevent it.

Vulnerabilities often stem from unsanitized user input combined with dynamic code or query construction.

SQL Injection occurs when you build queries by concatenating strings:

```js
// Vulnerable
const sql = `SELECT * FROM users WHERE name = '${req.query.name}'`;
db.query(sql, …);
```

An attacker can supply `name = "'; DROP TABLE users; --"` and wreak havoc.

Prevention:

*   Always use parameterized queries or prepared statements:
  
    
    ```js
    const sql = 'SELECT * FROM users WHERE name = ?';
    db.query(sql, [req.query.name], …);
    ```
    
*   Employ ORMs (Sequelize, TypeORM) that abstract parameter binding.
    
*   Validate and sanitize all external inputs against expected patterns.
    

## 8\. How is a race condition possible in asynchronous programming? And how to protect your code from it?

A race condition arises when two or more async tasks access and modify shared state without coordination:

```js
async function incCounter() {
  let c = await readCounterFromDb();
  c++;
  await writeCounterToDb(c);
}

// Two concurrent calls can read the same initial c
incCounter();
incCounter();
```

Both see the same base value, then overwrite each other.

Protection strategies:

*   Use database transactions with row-level locks or `SELECT … FOR UPDATE`.
    
*   Employ in-memory mutex/semaphore libraries (`async-mutex`, `semaphore-async-await`).
    
*   Queue operations sequentially (e.g., `p-queue`) for critical sections.
    
*   Leverage atomic filesystem ops (e.g., `fs.rename()` is atomic on many platforms).
    

## 9\. What are the pros and cons of splitting code into `.js` and separate `.d.ts` typings?

Pros:

*   Pure JavaScript workflow: no compilation step.
    
*   TS users still get full type information via declarations.
    
*   Libraries can ship lighter payloads (no transpiled JS).
    

Cons:

*   Typings must be hand-maintained or generated—it can drift from source.
    
*   You miss out on compiler-enforced correctness during development.
    
*   Extra build step if you generate `.d.ts` from TypeScript.
    

## 10\. Give several typical design patterns for Node.js with examples.

1.  Singleton
    
    *   Module caching ensures a single instance.
        
    
    ```js
    // config.js
    const cfg = { port: 3000 };
    module.exports = cfg; // Every require returns same object
    ```
    
2.  Observer
    
    *   Using `EventEmitter` for pub/sub.
        
    
    ```js
    import { EventEmitter } from 'node:events';
    const bus = new EventEmitter();
    bus.on('user:created', user => sendWelcomeEmail(user));
    ```
    
3.  Strategy
    
    *   Swap out algorithms at runtime.
    
    ```js
    function doHash(data, strategy) {
      return strategy(data);
    }
    // Pass in crypto.scrypt or bcrypt.hash
    ```
    
4.  Factory
    
    *   Create instances based on parameters.
      
    
    ```js
    function createDatabaseClient(type) {
      if (type === 'mysql') return new MySQLClient();
      if (type === 'postgres') return new PostgresClient();
    }
    ```
    
5.  Adapter
    
    *   Wrap callback APIs to return promises.
        
    
    ```js
    import { readFile } from 'fs';
    import { promisify } from 'util';
    const readFileAsync = promisify(readFile);
    ```

## 11\. В чем заключается проблема толстых контролеров? (с примерами на ноде)

Толстые контролеры смешивают маршрутизацию, валидацию, бизнес-логику и доступ к данным в одном файле. Это приводит к:

*   Сложности тестирования: приходится мокать HTTP-запрос и все зависимости внутри одного метода.
    
*   Плохой читаемости: десятки строк кода в одном обработчике трудно поддерживать.
    
*   Высокому зацеплению: любая правка формы данных или изменение хранилища затрагивают весь контролер.
    

Пример «толстого» Express-контролера:

```js
app.post('/users', async (req, res) => {
  const { name, email } = req.body;
  if (!email.includes('@')) {             // валидация
    return res.status(400).send('Invalid');
  }
  const conn = await db.getConnection();  // доступ к данным
  const [user] = await conn.query(
    `INSERT INTO users SET name='${name}', email='${email}'`
  );
  await sendWelcomeEmail(user.email);     // бизнес-логика
  res.json({ id: user.insertId });
});
```

## 12\. Примеры протекания абстракций (типичных для ноды)

1.  Stream Backpressure
    
    *   Readable-stream может «засориться» данными, если потребитель не успевает обрабатывать события `data`.
        
2.  EventEmitter и скрытые ошибки
    
    *   Многие модули (`http.Server`, `net.Socket`) требуют прослушивания `error`, иначе процесс аварийно упадёт.
        
3.  Buffer vs string
    
    *   Неправильное указание кодировки приводит к неожиданному Uint8Array вместо строки.
        
4.  Callback-ориентированные API
    
    *   Смешивание промисов и callback-функций (например, `fs.readFile` + `fs.promises`) может вызвать двойный вызов обработчиков.
        

## 13\. Как можно создать `Singleton` с помощью системы модульности в ноде?

Node.js кеширует результат `require`/`import` при первом загрузке. Достаточно экспортировать единственный экземпляр:

```js
// logger.js
class Logger {
  constructor() { this.level = 'info'; }
  log(msg) { console.log(`[${this.level}]`, msg); }
}
module.exports = new Logger();
```

При каждом `require('./logger')` вы получите один и тот же объект.

Если нужен ленивый Singleton:

js

Копировать

```js
// dbClient.js
let instance = null;
class DBClient { /* ... */ }
module.exports = () => {
  if (!instance) instance = new DBClient();
  return instance;
};
```

## 14\. Как проще всего реализовать паттерн Strategy на JavaScript (и где его использовать в ноде)?

Strategy в JS — передать функцию или класс с единым интерфейсом:

```js
// hashingStrategies.js
import crypto from 'node:crypto';
import bcrypt from 'bcrypt';

export const strategies = {
  pbkdf2: (pwd, salt) => crypto.pbkdf2Sync(pwd, salt, 100000, 64, 'sha512'),
  bcrypt:  (pwd, salt) => bcrypt.hashSync(pwd, salt)
};

// authService.js
import { strategies } from './hashingStrategies.js';
export function hashPassword(pwd, salt, method) {
  return strategies[method](pwd, salt);
}
```

Используйте для:

*   Выбора алгоритма хеширования.
    
*   Форматов логирования (console, file, remote).
    
*   Разных способов парсинга (JSON, XML, CSV).
    

## 15\. Пример паттерна `Adapter` из встроенных библиотек ноды

`util.promisify` превращает старый callback-style API в Promise:

```js
import { readFile } from 'fs';
import { promisify } from 'node:util';

const readFileAsync = promisify(readFile);

// раньше
readFile('data.txt', (err, data) => { /* ... */ });

// теперь
await readFileAsync('data.txt', 'utf8');
```

Это адаптер между двумя несовместимыми интерфейсами (callback ↔ Promise).

## 16\. Какой паттерн проектирования реализует `EventEmitter`?

EventEmitter — это классический **Observer (Publish–Subscribe)**. Издатель генерирует события через `emit()`, а подписчики слушают их через `on()` или `once()`.

## 17\. Как связаны контракты `EventEmitter` и `Readable`?

`Readable` наследует от `EventEmitter` и обязуется эмитировать набор событий:

*   `data` — новый кусок данных.
    
*   `end` — поток закончился.
    
*   `error` — ошибка чтения.
    

Таким образом Readable является специализированным наблюдаемым, расширяющим общий контракт EventEmitter.

## 18\. Антипаттерны и примеры плохого стиля в node.js

*   **Callback Hell**: вложенные колбэки вместо промисов/async–await.
    
*   **Sync I/O в критичных местах**: `fs.readFileSync` внутри обработчика HTTP.
    
*   **Silent failures**: игнорирование ошибок на промисах или потоках.
    
*   **Прототипная поллюция**: изменение глобальных объектов (`String.prototype`).
    
*   **Большие монолитные файлы**: отсутствие модульности и четких границ.
    
*   **Hard-coded конфиги**: без ENV-переменных или конфигурационных файлов.
    

## 19\. Зачем нам поля `Error: cause, code, message, stack`?

*   `message` — человекочитаемое описание ошибки.
    
*   `code` — машинно-обрабатываемый идентификатор (например, `ENOENT`).
    
*   `cause` — вложенная оригинальная ошибка (цепочка ошибок).
    
*   `stack` — трассировка вызовов для отладки.
    

Эти поля помогают классифицировать, логировать и анализировать ошибки.

## 20\. Как скопировать папку с вложенными файлами и папками с помощью `node:fs`?

### С Node.js ≥16.7.0


```js
import { cp } from 'node:fs/promises';

await cp('sourceDir', 'destDir', { recursive: true });
```

### Ручная реализация (все версии)


```js
import { mkdir, readdir, copyFile, stat } from 'node:fs/promises';
import { join } from 'node:path';

async function copyDir(src, dest) {
  await mkdir(dest, { recursive: true });
  for (const entry of await readdir(src, { withFileTypes: true })) {
    const srcPath  = join(src, entry.name);
    const destPath = join(dest, entry.name);
    if (entry.isDirectory()) {
      await copyDir(srcPath, destPath);
    } else {
      await copyFile(srcPath, destPath);
    }
  }
}
```

## 20\. Как скопировать папку с вложенными файлами и папками с помощью `node:fs`?

Вы можете воспользоваться встроенным методом `fs.cp` из модуля `node:fs/promises`, появившимся в Node.js 16.7 и выше:

```js
import { cp } from 'node:fs/promises';

await cp('sourceDir', 'targetDir', { recursive: true });
```

Если ваша версия Node.js старее или вам нужен более тонкий контроль, можно обойтись без сторонних зависимостей, реализовав функцию вручную:

*   Чтение содержимого директории через `readdir`.
    
*   Для каждого элемента определяем через `stat` файл или папку.
    
*   Файлы копируем через `copyFile`; папки создаем через `mkdir` и рекурсивно копируем их содержимое.
    
*   Используем `Promise.all` для параллельного копирования, но следим за нагрузкой.
    

## 21\. Можем ли мы делать real-time приложения на Node.js??

Node.js идеально подходит для real-time приложений благодаря неблокирующей модели ввода-вывода и встроенной поддержке событий.

*   WebSocket: через официальный `ws` или `socket.io` (хотя в 2023 лучше смотреть на более лёгкие реализации, например `uWebSockets.js`).
    
*   Server-Sent Events (SSE): чистый HTTP-стриминг, подходит для однонаправленных обновлений.
    
*   HTTP/2 Push и HTTP/3 (QUIC): для более надёжной многопоточности в браузерах, поддерживается в Node.js..
    

## 22\. Какие есть подходы к логированию? Их отличия, плюсы и минусы.

| Подход | Примеры | Плюсы | Минусы |
| --- | --- | --- | --- |
| Простое консольное | `console.log` | Ничего не подключать, удобно для отладки | Нет ротации, сложнее фильтровать, неструктурированное |
| Трейсинг через файл | `fs.createWriteStream` | Лёгкая настройка, поток в файл | Самописные реализации обычно без ротации и уровня логов |
| Специализированные | Winston, pino, Bunyan | Уровни логирования, структурированные логи, ротация, JSON | Дополнительная зависимость, возможны накладные расходы |
| Внешние сервисы | Datadog, Loggly | Централизация, поиск, оповещения | Платно, нужна сеть, риск утечки данных |

## 23\. Где хранить секреты? (ключи API, токены, пароли от БД)

*   Переменные окружения (`process.env`), загружаемые через `.env` и `dotenv`.
    
*   Специализированные хранилища: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault.
    
*   Защищённые конфиг-файлы зашифрованные GPG или KMS (Google Cloud KMS).
    
*   Kubernetes Secrets или Docker Swarm Secrets для контейнеров.
    
*   Важно: никогда не коммитить .env или конфиги в репозиторий; привязывайте доступ к секретам к ролям и политиками доступа.
    

## 24\. Почему нужно делать `return await` внутри асинхронных функций, а не просто `return` промис?

Первичная причина — корректная работа `try/catch`. Если вы пишете:

```js
async function foo() {
  return fetchSomething()
}
```

то любой `throw` внутри `fetchSomething` будет необработанным до вызова `foo()`. Вариант с `return await`:

```js
async function foo() {
  try {
    return await fetchSomething()
  } catch (err) {
    handleError(err)
  }
}
```

позволяет ловить ошибки прямо внутри `foo`, а не выше по стэку вызовов.

## 25\. Как не заблокировать обслуживание других пользователей при обработке одного запроса?

*   Никогда не используйте синхронные API (например `fs.readFileSync`) в основном потоке.
    
*   Для тяжёлых вычислений выносите работу в воркеры (`worker_threads`) или в кластерный режим (`cluster`).
    
*   Используйте очереди задач или внешние сервисы (RabbitMQ, Redis Streams) для фоновых задач.
    
*   Разгружайте CPU-bound логику в микросервисы на других языках или языковых рантаймах.
    

## 26\. Что делать, если обработка запроса привела к необходимости завершить процесс?

Грейсфул-шатдаун (graceful shutdown):

1.  Остановить приём новых соединений в HTTP/HTTPS сервере.
    
2.  Дождаться завершения текущих запросов (через `server.close()`).
    
3.  Очистить ресурсы (базы, очереди, кеши).
    
4.  Вызвать `process.exit(code)`.
    

При этом важно настроить таймауты, чтобы не вечно ждать «вечных» запросов.

## 27\. Какие стили и парадигмы программирования вы используете в Node.js приложениях? Почему?

*   Модульность: каждый файл — отдельная ответственность.
    
*   Функциональный стиль: чистые функции, иммутабельность, легко тестируется.
    
*   Событийно-ориентированная архитектура: события через `EventEmitter` или шина сообщений.
    
*   Onion/Hexagonal архитектура: ядро бизнес-логики отделено от инфраструктуры.
    
*   Dependency Injection: для лёгкой подмены реализаций на тестах и в продакшене.
    

## 28\. В чем слабые стороны Node.js? ? Что на ноде писать плохо или невозможно?

*   CPU-bound задачи (криптография, видео-/аудиообработка) тормозят весь поток.
    
*   Работа с точными временными расчётами (низкоуровневые тайминги) лучше на C/C++.
    
*   Гарантии консистентности транзакций сложнее организовать без внешних библиотек и брокеров.
    
*   Глубокие математические расчёты (бигинты, научные вычисления) не так оптимизированы.
    

## 29\. В чем разница между stateful и stateless подходами для Node.js приложений? Как выбрать?

Stateful хранит состояние в памяти процесса (сессии, кеши), а stateless обрабатывает каждый запрос независимо, полагаясь на внешние хранилища.

*   Stateful проще работать со сессиями, но трудно масштабировать—нужен sticky-session или распределённый кеш.
    
*   Stateless легко масштабируется горизонтально, упрощает CI/CD, но требует централизованных хранилищ (Redis, базы) для состояния.
    

Выбирайте stateless для веб-API и микросервисов, stateful для игровых и real-time систем с очень низкой задержкой.

## 30\. Как ограничить пропускную способность эндпоинта (кол-во запросов в единицу времени)?

*   Внешние прокси: Nginx, HAProxy с rate-limit модулями.
    
*   В приложении: middleware-rate-limiter (express-rate-limit), Bottleneck, rate-limiter-flexible.
    
*   Алгоритмы: token bucket, leaky bucket, fixed window, sliding window.
    
*   Хранилище состояния счётчиков: in-memory (для одного инстанса) или Redis/Cluster для распределённых систем.
    

## 31\. В чем опасность примесей (mixins) для прикладного кода?

*   Загрязнение прототипов встроенных объектов и неожиданные коллизии имён.
    
*   Скрытое поведение, которое трудно отследить и протестировать.
    
*   Упростить понимание модуля невозможно, когда он «внезапно» получает сторонние методы.
    
*   Нарушается принцип единственной ответственности и инкапсуляции.
    

## 32\. Как реализовать архитектурную границу в приложениях на Node.js??

*   Явное разделение слоёв: контроллер → сервис → репозиторий.
    
*   Использование интерфейсов (TypeScript) для контракта между слоями.
    
*   Пакетная структура: core-package, infra-package, api-package в моно-репозитории.
    
*   DTO и мапперы между внешними DTO и внутренними доменными объектами.
    

## 33\. Что такое DI (внедрение зависимостей) и как его реализовать на ноде? (несколько вариантов)

1.  Конструкторная инъекция: передаём зависимости через аргументы клаccа или функции.
    
2.  Property-инъекция: назначаем поля объекта после создания.
    
3.  Service Locator: глобальный контейнер, который выдаёт зависимости по ключу.
    
4.  IoC-контейнеры: InversifyJS, Awilix, tsyringe — автоматизируют регистрацию и разрешение зависимостей.
    

## 34\. Почему middleware является антипаттерном? И как писать без него?

Middleware скрывают поток управления за цепочкой функций и делают логику менее предсказуемой.

*   Альтернатива: явная композиция функций:
    

js

Копировать

```
const handler = compose(validateInput, authorize, businessLogic);
```

*   Использовать чистые функции без внешних побочных эффектов и цепочек, передавая контекст вручную.
    

## 35\. Как снизить зацепление кода в приложениях на Node.js??

*   Определять контракты через интерфейсы/типы (TypeScript).
    
*   Внедрение зависимостей вместо `require` внутри модулей.
    
*   Событийная шина (`EventEmitter`) для уведомлений вместо прямых вызовов.
    
*   Разделение на мелкие, переиспользуемые модули с одной ответственностью.
    

## 36\. Почему нужно добавлять префикс `node:` при загрузке встроенных модулей?

*   Гарантирует, что вы точно получите встроенный модуль, даже если в `node_modules` есть пакет с тем же именем.
    
*   Улучшается разрешение зависимостей и работа инструментов сборки/анализа кода.
    
*   Повышает безопасность, исключая подмену важных API.
    

## 37\. Зачем нужен `AbortController`? Приведите примеры API, где он используется.

`AbortController` позволяет отменять асинхронные операции по сигналу:

*   `fetch(request, { signal })` — отмена HTTP-запроса.
    
*   `stream.pipeline(streams…, { signal })` — досрочное завершение потока.
    
*   `fs.promises.open(path, { signal })` — отмена операций с файловой системой.
    
*   `setTimeout`/`setImmediate` (с Node.js 18+) поддерживают `AbortSignal` для очистки таймеров.
    

## 38\. JSON сериализация и десериализация может работать долго и заблокировать поток, что с этим делать?

*   Использовать стриминг через `JSONStream`, `stream-json` — парсить и генерировать данные по кусочкам.
    
*   Выносить тяжёлые операции в воркер-потоки (`worker_threads`).
    
*   При больших объёмах данных отдавать их частями (пагинация, chunked responses).
    

## 39\. Как могут утечь все соединения из пула конекшенов к базе данных и как это предотвратить?

*   Утечка обычно происходит при выбрасывании ошибки до вызова `release()`/`close()`.
    
*   Всегда освобождайте соединения в блоке `finally` или `.finally()` у промиса.
    
*   Настройте таймауты для неиспользуемых соединений и `pool.on('error')`.
    
*   Используйте обёртки/транзакции, которые сами управляют приобретением и освобождением соединений.
    

## 40\. Как вы организовываете слой доступа к данным?

*   Репозиторий (Repository) или Data Mapper: отдельный класс для каждой коллекции/таблицы.
    
*   Использование ORM/ODM (TypeORM, Sequelize, Mongoose) или query-builder’а (knex.js).
    
*   Внедрение через DI для простой замены реализации (тесты против моков).
    
*   Публичный API слоя: CRUD-методы и специфичные запросы, без SQL-строк в бизнес-логике.

## 40\. How do you organize the data access layer?

I isolate all direct calls to the database behind a set of repository or data-mapper classes. Each repository encapsulates CRUD operations and complex queries for a single entity or aggregate. Above that, service classes receive injected repositories via constructor injection (or a simple service-locator) and coordinate transactions or business logic.

Folder structure example:

Копировать

```
src/
  ├─ domain/
  │    └─ user.entity.ts
  ├─ repositories/
  │    └─ user.repository.ts
  ├─ services/
  │    └─ user.service.ts
  └─ controllers/
       └─ user.controller.ts
```

## 41\. What is the advantage of `async/await` and promises over callbacks in Node.js? ? Where is callback unavoidable?

Advantages

*   Linear, readable control flow that resembles synchronous code.
    
*   Centralized error handling with `try/catch` instead of scattered callback checks.
    
*   Seamless composition via `Promise.all`, `Promise.race`.
    
*   Avoids “callback hell” and pyramid indentation.
    

When callbacks are unavoidable

*   Low-level APIs that expose only callback interfaces (e.g., legacy addons).
    
*   High-performance, hot paths where allocating promise objects per call exceeds acceptable overhead.
    
*   Very simple event emitters or streams where callback registration is the idiomatic pattern.
    

## 42\. What to do if you need different versions of npm dependencies in two parts of one application?

*   **Monorepo with workspaces**: define separate package.json for each sub-app, pin its own version of the dependency.
    
*   **npm aliasing**: install `npm install lodash@4 lodash5@npm:lodash@5` and import via different names.
    
*   **Scoped packages**: split functionality into two internal packages (e.g., `@myorg/module-v1` and `@myorg/module-v2`).
    
*   **Containerization**: run each part in its own Docker image with its own `node_modules`.
    

## 43\. Which Web APIs have appeared recently in Node.js and why were they added?

*   `fetch`, `Headers`, `Request`, `Response`: aligns with browser API, removes need for external HTTP clients.
    
*   **Web Streams** (`ReadableStream`, `WritableStream`) for unified streaming across fetch, file I/O, crypto.
    
*   `AbortController`**/**`AbortSignal` to cancel any async operation.
    
*   **Web Crypto** (`crypto.subtle`) for standardized cryptography.
    
*   **URLPattern**, **EventTarget**, **FormData**, **TextEncoder/TextDecoder** to close the gap with client-side JS.
    

These additions let you share code between browser and server without polyfills.

## 44\. What can be used instead of outdated pm2 and forever in modern world?

*   **Container orchestrators**: deploy as Docker containers and let Kubernetes/ECS handle restarts, scaling, logging and health checks.
    
*   **Systemd** or **launchd**: native OS process managers with robust restart policies.
    
*   **Deno deploy or AWS Lambda**: serverless platforms that eliminate process-management burden entirely.
    
*   **Built-in** `--watch` **+** `worker_threads` **clustering**: for small projects, combined with shell scripts and healthchecks.
    

## 45\. How to make business logic independent of framework and protocol?

*   Extract core services into plain TypeScript/JavaScript modules with no Express/Koa imports.
    
*   Define interfaces for I/O (e.g., `HttpTransport`, `MessageBusTransport`) and implement them in adapter layers.
    
*   Use Dependency Injection to supply framework-specific clients at bootstrap.
    
*   Keep controllers or routers only responsible for mapping protocol formats (HTTP, gRPC) to DTOs and back.
    

## 46\. Why do we no longer need axios, request, node-fetch?

Because modern Node.js includes the standard Fetch API, Web Streams, AbortController, and built-in handling of redirects, timeouts, JSON parsing, and form data. These native features reduce bundle size, remove dependencies, and ensure consistent behavior with browsers.

## 47\. For what might we need queues inside an application and external MQ systems?

*   **In-process queues** (e.g., BullMQ, Bee-Queue) for delayed jobs, rate-limiting, debouncing heavy tasks.
    
*   **External message brokers** (RabbitMQ, Kafka, Redis Streams) for cross-service event distribution, guaranteed delivery, scalable fan-out, and decoupling microservices.
    
*   **Use cases**: email sending, video encoding, transactional workflows, real-time notifications, event sourcing.
    

## 48\. Why is it dangerous if a dependency patches global objects, classes, or prototypes?

*   **Prototype pollution**: malicious or buggy patches can break other modules or open security holes.
    
*   **Name collisions**: two libraries might overwrite the same method with incompatible behaviors.
    
*   **Maintenance nightmare**: tracing mysterious side-effects across the entire runtime becomes impossible.
    
*   **Violation of encapsulation**: breaks the assumption that built-ins behave predictably.
    

## 49\. What is Node.js LTS and what does it give us?

Long-Term Support (LTS) releases are stable versions maintained for 30 months. They guarantee:

*   **Security patches** and critical bug fixes.
    
*   **API stability**, minimizing breaking changes.
    
*   **Predictable upgrade schedule**, letting teams plan migrations.
    

Using LTS ensures production environments remain supported and secure.

## 50\. What is WebSocket, why in 2023 using socket.io is a bad choice, and what should you use instead?

WebSocket is a bidirectional, full-duplex TCP channel over a single handshake. Socket.io adds automatic fallbacks (XHR polling), custom protocol overhead, and bloats both client and server bundles. In 2023, prefer:

*   `ws`: minimal, standards-compliant, super-fast.
    
*   **uWebSockets.js**: extreme performance and low memory footprint for high-throughput scenarios.
    
*   **WebTransport (QUIC-based)** for next-gen transport where supported.
    

## 51\. What does the flag `--watch` give?

Automatically restarts the Node.js process when source files change. It’s ideal for development loops—no third-party tools needed. In Node.js 18+, you can also combine it with experimental loader flags to rebuild on import changes.

## 52\. What is the state of the native test runner in Node.js now?

*   Available out of the box since Node.js 18 and marked stable in Node.js 20.
    
*   Supports `test()` blocks, subtests, assertions API, concurrent runs, snapshot testing.
    
*   TAP-compatible reporters and custom reporters.
    
*   Still fewer community plugins than Jest or Mocha, but steadily growing.
    

## 53\. Is there a way in Node.js to deliver an application as one executable file, and why?

Yes. Tools like **pkg**, **nexe**, **@vercel/ncc**, or **zeit/pkg** bundle your code, assets, and the Node runtime into a single binary. Benefits:

*   Single artifact deploys without installing Node.js on the target host.
    
*   Source code obfuscation and faster startup.
    
*   Simplified release management, especially in constrained environments.
    

## 54\. What ways exist to track asynchronous contexts and are they needed at all?

*   **Async Hooks**: low-level API for hooking into the lifecycle of async resources.
    
*   **AsyncLocalStorage**: higher-level API to store and propagate context (e.g., request IDs) across async calls.
    
*   **Third-party**: Zone.js, continuation-local-storage (deprecated).
    

Use them sparingly—only when you need per-request context for logging, tracing, or transaction management.

## 55\. When and how should Node.js versions be updated in projects?

*   Follow the **LTS release schedule**: plan upgrades twice a year (when new LTS and maintenance phases begin).
    
*   Perform **canary/staging deployments** first and run full regression test suites.
    
*   Update major versions in tandem with dependency upgrades to catch breaking changes early.
    
*   Automate version checks in CI (e.g., Renovate, Dependabot) and schedule time for manual verification.

## 60 Interview Questions for System Node.js Backend Engineer

Системный (платформенный) программист пишет код, не связанный с предметной областью: фреймворки, сетевые протоколы, транслятор, компиляторы, интерпретаторы, библиотеки, занимается вещами, которые могут быть переиспользованы в сотнях и тысячах разных проектов. Это называется производство средств производства. Систем программисту нужно знать node.js гораздо глубже, не только, его возможности, концепции, преимущества и недостатки, но и недокументированные возможности и даже баги, особенности платформы, которые очень редко используются, потому, что он строит прослойку между node.js и прикладным кодом, а прослойка эта позволяет делать прикладной код более абстрактным и приближенным к предметной области.

## 1\. Чего не хватает в ESM, но есть (поддерживается) в CJS?

В чистом ESM-модуле отсутствуют некоторые “удобства” CommonJS:

*   `require`: синхронный загрузчик модулей, вместе с `require.cache`, `require.extensions` и `require.main`.
    
*   `__dirname` **и** `__filename`: в ESM нужно пользоваться `import.meta.url` и утилитами из модуля `node:url`.
    
*   **Динамическое условное подключение**: `if (cond) require(...)` не работает на верхнем уровне — вместо этого нужен `import()` (асинхронно).
    
*   **Изменяемые экспорты**: в CJS можно делать
    
    ```js
    exports.foo = 1;
    exports.foo = 2; // перезапишет
    ```
    
    В ESM экспорт — неизменяемая привязка.
    
*   **Обратная совместимость с файлами JSON и бинарями**: в старых версиях ESM нужно было вручную использовать `fs.readFile` или флаг `--experimental-json-modules`.
    

## 2\. Для чего используется new `Error.captureStackTrace`?

*   Позволяет вручную захватить стек вызовов в момент создания объекта `Error`.
    
*   Используют, когда нужно убрать лишние рамки (функции-обёртки) из трассировки.
    
*   Сигнатура:
    
    ```js
    Error.captureStackTrace(targetError, constructorOpt);
    ```
    
    где `constructorOpt` — функция, начиная с которой не включать кадры стека.
    
*   Пример:
    
    ```js
    function MyError(msg) {
      Error.captureStackTrace(this, MyError);
      this.message = msg;
    }
    ```
    

## 3\. Почему node.js не однопоточный? Докажите, что даже не был однопоточным.

*   **Пул потоков libuv**: помимо главного цикла событий, Node создаёт пул из рабочих потоков (по умолчанию 4) для операций I/O (файлы, DNS, crypto), таймеров и др.
    
*   `worker_threads`: с бэкпортом (Node >= 10.5) можно создавать полноценные потоки JS.
    
*   **Встроенные C++ доп. потоки**: DNS-over-UDP, async\_hooks, эпифреймворки на C++ могут создавать собственные фоновые потоки.
    
*   **Доказательство**:
    
    ```js
    # Запустим чтение 10 больших файлов параллельно
    for i in {1..10}; do node -e "require('fs').readFile('/path/large', ()=>console.log('done',$i))"; done
    ```
    
    Одновременный вывод `done` показывает, что I/O происходит параллельно в пуле потоков.
    

## 4\. Как связаны `node:async_hooks` и `AsyncLocalStorage`?

*   `async_hooks` — низкоуровневый API для отслеживания жизненного цикла асинхронных ресурсов (создание, связка, уничтожение).
    
*   `AsyncLocalStorage` строится поверх `async_hooks`, предоставляя **контекст** (storage) на протяжении асинхронных цепочек.
    
*   Схема работы:
    
    1.  При создании ресурса `async_hooks` генерирует событие `init`.
        
    2.  `AsyncLocalStorage` перехватывает его, клонирует или наследует текущее состояние.
        
    3.  В любом месте асинхронного контекста можно достать и изменить значение через `asyncLocalStorage.getStore()`.
        

## 5\. Какие в ноде встроенные средства сериализации аналогичны JSON только для бинарной сериализации?

*   `v8.serialize` **/** `v8.deserialize` (модуль `node:v8` — быстрые бинарные сериализатор и десериализатор, поддерживает циклы и специальные типы).
    
*   `MessagePort` **и передача через** `postMessage` (из `worker_threads`) используют внутреннюю бинарную сериализацию.
    
*   `Buffer` + ручная реализация через `struct`\-подобные пакеты (например, `buffer.writeUInt*`).
    

## 6\. Как следить за изменениями файлов и директорий на диске и какие с этим могут возникать проблемы?

*   `fs.watch(path[, options], listener)`
    
    *   Шустрый, но нестабилен:
        
        *   На Linux — через inotify, может не срабатывать при массовых изменениях.
            
        *   На macOS — FSEvents, эмитирует “rename” вместо подробных событий.
            
*   `fs.watchFile(path[, options], listener)`
    
    *   Поллинг: регулярно опрашивает атрибуты файла (mtime, size).
        
    *   Медленнее, но более предсказуем.
        
*   **Проблемы**:
    
    *   Шум (лишние срабатывания при перезаписи temp-файлов).
        
    *   Потеря событий при одновременных изменениях (inotify-очередь переполняется).
        
    *   Непоследовательное поведение на разных платформах.
        

## 7\. Чем заменить deprecated `fs.exists` и почему его выпиливают из ноды?

*   **Заменить** на комбинацию:
    
    ```js
    fs.stat(path, (err, stats) => {
      if (!err) { /* exists */ }
      else if (err.code === 'ENOENT') { /* не существует */ }
      else { /* другая ошибка */ }
    });
    ```
    
    или `fs.access(path, fs.constants.F_OK, callback)`.
    
*   **Причина депрекса**:
    
    *   `fs.exists` не следует одному-единому стандарту ошибок (callback не получает `err`, а просто `true/false`).
        
    *   Приводит к негласным “гонкам” (TOCTOU) — состояние могло измениться между проверкой наличия и фактическим доступом.
        

## 8\. Что такое back pressure для стримов и какая проблема была бы без него?

*   **Back pressure** — механизм, при котором writable-стрим сигнализирует readable-стриму “замедляться”, когда буфер заполнен.
    
*   Без него:
    
    *   Писатель (producer) бы заливал память всё большими chunk’ами, если потребитель (consumer) не успевает их обрабатывать.
        
    *   Приложение могло бы съесть всю RAM и упасть с OOM.
        
*   В Node:
    
    *   `stream.write(chunk)` возвращает `false`, если внутренний буфер полон. Тогда нужно приостановить чтение (`readable.pause()`) и дождаться события `drain` на writable.
        

## 9\. Как защитить `SharedArrayBuffer` от записи из разных `worker_threads`?

*   Сам по себе `SharedArrayBuffer` не синхронизирован. Для защиты используют:
    
    *   **Atomics**:
        
        ```js
        Atomics.store(shared, index, value);
        Atomics.compareExchange(shared, index, old, new);
        Atomics.wait()/Atomics.notify();
        ```
        
        — атомарные операции чтения/записи и ожидания.
        
    *   **Мьютексы** (пользовательская обёртка на Atomics — “locking” через Atomics).
        
*   Без этого могут возникнуть **гонки** и **коррупция данных**.
    

## 10\. Докажите, что любой модуль в ноде при загрузке оборачивается в функцию и создает замыкание?

*   Код модуля инкапсулируется так:
    
    ```js
    (function(exports, require, module, __filename, __dirname) {
      // ваш код
    });
    ```
    
*   **Доказательство**:
    
    *   В REPL или debugger-е можно поставить точку останова в C++-файле `node_module_register.cc`: там вызывается `Function::Compile` для этой обёртки.
        
    *   При запуске модуля через `--print-module-wrapper` (в новых версиях Node) можно увидеть шаблон-обёртку.
        

## 11\. Где в ноде используется паттерн Revealing constructor?

*   `EventEmitter`:
    
    
    ```js
    function EventEmitter() {
      // локальные state-переменные
      this._events = Object.create(null);
      // …  
    }
    EventEmitter.prototype.on = function(ev, fn) { /* … */ };
    ```
    
    Приватные данные хранятся через замыкание/свойства, а публичный API раскрывается через `prototype`.
    
*   `Readable`**,** `Writable` из `stream` — похожая схема: конструктор скрывает детали внедрения в `duplexify`.
    
*   **TLS/TCP-сокеты**: внутренние C++-хэнгл инициализируются в конструкторе, а наружу выходят только методы `.write()`, `.end()`, `.on()`.
    

## 12\. Как сделать переопределение `write` для экземпляра `Writable` без создания класса наследника?


```js
const { Writable } = require('stream');

const w = new Writable({
  write(chunk, encoding, callback) {
    // ваша реализация
    console.log('got chunk', chunk);
    callback();
  }
});
```

Если у вас есть готовый экземпляр `w` и вы хотите заменить метод:

```js
w.write = function (chunk, encoding, cb) {
  // новый код
  Writable.prototype.write.call(this, chunk, encoding, cb);
};
```

## 13\. В чем причина медленных вызовов из JavaScript кода к аддонам на C, C++ или подключенных через N-API?

*   **Переход через границу V8 ↔ C++**: каждый вызов требует упаковки аргументов в v8::Value и обратно, с проверкой типов.
    
*   **Вызов из N-API** добавляет обёртку для удержания ABI-совместимости, что влечёт дополнительные аллокации.
    
*   **Синхронные блокирующие операции** в аддоне мешают event loop, ухудшая общую отзывчивость.
    

## 14\. Что такое `MessagePort` и `BroadcastChannel`?

*   `MessagePort` (из модуля `worker_threads` или `node:worker_threads`):
    
    *   Канал связи между двумя концами порта (`port1`, `port2`).
        
    *   Поддерживает двунаправленную передачу сообщений и передачу объектов (`postMessage`).
        
*   `BroadcastChannel` (экспериментально доступен в Node 18+):
    
    *   Подобен DOM API.
        
    *   Все подписчики на один и тот же канал (по имени) получают сообщения от любого отправителя.
        

## 15\. Чем отличаются `fs.stat`, `fs.fstat`, `fs.lstat`?

| Функция | Что проверяет | Особенности |
| --- | --- | --- |
| `fs.stat` | Информация о файле/директории | Разрешает символические ссылки (резолвит их). |
| `fs.lstat` | Информация о символьном звене | Не резолвит ссылки: возвращает данные ссылки, а не целевого файла. |
| `fs.fstat` | Информация по открытому дескриптору | Работает только с уже открытым `fd` (через `fs.open`). |

Каждая функция возвращает статистику файловой системы, но с разным поведением по отношению к символьным ссылкам и открытому дескриптору.

| Функция | Обрабатывает символические ссылки? | Источник информации | Сценарии применения |
| --- | --- | --- | --- |
| `fs.stat` | Да  | По пути (разрешает ссылки) | Узнать размер, тип и права реального файла за ссылкой |
| `fs.lstat` | Нет | По ссылке (метаданные ссылки) | Проверить, что указанный путь является символьной ссылкой |
| `fs.fstat` | Не применимо | По открытому дескриптору `fd` | Когда файл уже открыт через `fs.open` и нужен `stat` |

Примеры:

```js
// stat: следит за файлом, на который указывает ссылка
fs.stat('shortcut', (err, stats) => {
  console.log(`Target is ${stats.isFile() ? 'file' : 'dir'}`);
});

// lstat: возвращает, что `shortcut` — это символьная ссылка
fs.lstat('shortcut', (err, stats) => {
  console.log(`Link? ${stats.isSymbolicLink()}`); 
});

// fstat: информация по уже открытому fd
fs.open('file.txt', 'r', (_, fd) => {
  fs.fstat(fd, (err, stats) => {
    console.log(`Opened file size: ${stats.size}`);
    fs.close(fd, ()=>{});
  });
});
```

## 16\. Зачем включать правило `eslint: consistent-return` для оптимизации V8

V8 строит оптимизированный машинный код, исходя из внутренней модели функции. Незаконченные или разнородные пути возврата меняют “форму” функции, что приводит к:

*   Потере инлайнинга: V8 вынужден откатиться к медленному Tier-0 или Ignition.
    
*   Увеличению числа точек выхода, что усложняет сборку JIT-компилятора.
    
*   Резкому снижению производительности при большом числе вызовов.
    

Consistent-return заставляет всегда либо возвращать значение, либо никогда, что:

*   Делает CFG (control-flow graph) однородным.
    
*   Упрощает оптимизацию и путевое предсказание.
    
*   Снижает фреймовые аллокации при возврате undefined.
    

Пример плохого кода:


```js
function findUser(id) {
  if (!id) return;            // путь 1: no return value
  if (cache.has(id)) return cache.get(id); // путь 2: return object
  return fetchFromDb(id);      // путь 3: return Promise
}
```

Лучше:

```js
function findUser(id) {
  if (!id) return null;
  if (cache.has(id)) return cache.get(id);
  return fetchFromDb(id);
}
```

## 17\. Зачем в Node есть WASI и что он даёт

WASI (WebAssembly System Interface) – это стандартный набор API, позволяющий WebAssembly-модулям безопасно взаимодействовать с окружением вне браузера.

Возможности в Node:

*   Манипуляция ФС: открытие, чтение и запись файлов через предопределённые дескрипторы.
    
*   Работа с аргументами командной строки и окружением (`args`, `env`).
    
*   Управляемая сеть (экспериментально).
    
*   Изоляция: модули не видят глобальные объекты Node, только те, что вы передали в конструктор WASI.
    
*   Переносимость: один и тот же WASM-модуль запускается в Node, Wasmtime, Wasmer и в браузере.
    

Простой пример:

```js
import { WASI } from 'wasi';
import fs from 'fs';
import { fileURLToPath } from 'url';

const wasi = new WASI({
  args: process.argv,
  env: process.env,
  preopens: { '/sandbox': './data' }
});
const wasmPath = fileURLToPath(new URL('./module.wasm', import.meta.url));
const wasm = await WebAssembly.compile(fs.readFileSync(wasmPath));
const instance = await WebAssembly.instantiate(wasm, { wasi_snapshot_preview1: wasi.wasiImport });
wasi.start(instance);
```

## 18\. Возможности модуля `node:vm`

Модуль `vm` предоставляет API для компиляции и запуска JavaScript-кода в отдельных контекстах.

1.  Создание изолированного глобального пространства:
    
    ```js
    import { createContext, runInContext } from 'vm';
    const sandbox = { x: 1 };
    const ctx = createContext(sandbox);
    runInContext('x += 40; globalThis.y = x + 1;', ctx);
    console.log(sandbox.y); // 42
    ```
    
2.  Повторное исполнение заранее скомпилированного кода:

    ```js
    import { Script } from 'vm';
    const script = new Script('value **= 2');
    for (let i of [2,3,4]) {
      const local = { value: i };
      script.runInNewContext(local);
      console.log(local.value); // 4, 9, 16
    }
    ```
    
3.  Работа с ECMAScript модулями:
    
    js
    
    Копировать
    
    ```
    import { SyntheticModule, SourceTextModule } from 'vm';
    // можно эмулировать загрузчик ESM, создавать virtual modules
    ```
    
4.  Безопасное исполнение плагинов:
    
    *   Ограничить доступ к `process`, `fs` и другим критичным API.
        
    *   Инструмент для онлайн-редакторов, где пользовательский код не должен сломать хост.
        

## 19\. Депрекейшн API и стратегия их вывода

### Распространённые deprecated API

*   `fs.exists` → Заменить на `fs.stat`/`fs.access`.
    
*   `new Buffer(size)` → `Buffer.alloc(size)` / `Buffer.from(str)`.
    
*   `domain` → `async_hooks` / `AsyncLocalStorage`.
    
*   `require.extensions` → ES модули/бандлеры.
    
*   `process.EventEmitter` → `events.EventEmitter` напрямую.
    
*   `crypto.createCredentials` → `tls.createSecureContext`.
    

### Стратегия удаления

1.  Мягкий депрекейшн
    
    *   Документация помечает методы как deprecated.
        
    *   При первом вызове выводится warning: `node --trace-deprecation`.
        
2.  Поддержка в течение минимум двух мажорных версий
    
    *   Дают время на миграцию.
        
3.  Полное удаление
    
    *   В следующем мажорном релизе API убирается, сборка падает при попытке импортировать.
        

## 20\. Известные проблемы, баги и узкие места в Node.js

1.  **Блокировки event loop**
    
    *   Синхронные вычисления (шифрование, парсинг JSON больших размеров) останавливают цикл.
        
2.  **Ограниченный пул libuv**
    
    *   По умолчанию 4 потока для I/O. Большие количества файлов могут ставить I/O в очередь.
        
3.  **Утечки памяти**
    
    *   Неочищенные слушатели, замыкания, кэшированные модули.
        
4.  **ESM-производительность**
    
    *   Динамический импорт и нормализация путей дороже, чем CJS-`require`.
        
5.  **Проблемы платформенной поддержки**
    
    *   В Windows inotify-эквиваленты менее надёжны, FSEvents на macOS сжаты событиями.
        
6.  **Нестабильные экспериментальные API**
    
    *   Могут ломаться между релизами (Streams web-interop, BroadcastChannel).
        

## 21\. Реализация `promisify` и `callbackify`

### Promisify

```js
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn.call(this, ...args, (err, ...results) => {
        if (err) return reject(err);
        return resolve(results.length > 1 ? results : results[0]);
      });
    });
  };
}
// Пример
const readFile = promisify(fs.readFile);
readFile('data.json', 'utf-8').then(JSON.parse);
```

### Callbackify


```js
function callbackify(fn) {
  return function (...args) {
    if (typeof args.at(-1) !== 'function') {
      throw new TypeError('Callback last argument required');
    }
    const cb = args.pop();
    fn.call(this, ...args)
      .then(result => cb(null, result))
      .catch(err => cb(err));
  };
}
// Пример
async function getData() { return 42; }
const getDataCb = callbackify(getData);
getDataCb((err, val) => console.log(val)); // 42
```

## 22\. Зачем event loop разбит на фазы

Event loop выполняет разные типы задач в изолированных очередях, чтобы:

*   Гарантировать порядок: таймеры обрабатываются перед I/O callbacks.
    
*   Избежать голодания задач одного типа: macrotasks разделены по категориям.
    
*   Вставить микротаски между фазами, позволяя завершить все мелкие задачи до перехода дальше.
    

Фазы в libuv:

1.  timers
    
2.  pending callbacks
    
3.  idle, prepare
    
4.  poll
    
5.  check
    
6.  close callbacks
    

Механизм предотвращает ситуации, когда все `setTimeout` задерживаются из-за непрерывного чтения I/O.

## 23\. Микротаски vs макротаски

| Характеристика | Макротаски | Микротаски |
| --- | --- | --- |
| Примеры | `setTimeout`, `setImmediate`, I/O | `Promise.then`, `queueMicrotask` |
| Очередь | По фазам event loop | Специальная очередь, выполняется сразу после текущего стека вызовов |
| Приоритет | Ниже | Выполняются до перехода к макротаскам |
| Гарантия окончания | Нет | Вычищают всю очередь перед следующей фазой |

Макротаски группируют крупные операции, микротаски дают возможность “дожать” все мелкие изменения перед UI/слоем I/O.

## 24\. Обработка `uncaughtException` в Node.js

*   Если исключение выходит наружу без catch, Node генерирует событие `uncaughtException`.
    
*   По умолчанию это приводит к аварийному завершению процесса.
    
*   Ловить его можно так:

    ```js
    process.on('uncaughtException', (err) => {
      console.error('Crash:', err);
      process.exit(1);
    });
    ```
    
*   Рекомендуется не продолжать работу после uncaughtException, а корректно завершать ресурсы и перезапускать процесс (PM2, systemd).
    
*   Для промисов аналогично: `unhandledRejection` → логировать, следить за промисами.
    

## 25\. `nextTick`, `setImmediate` и `setTimeout`

| Функция | Очередь вызова | Использование |
| --- | --- | --- |
| `process.nextTick()` | Очередь next-tick, выполняется до любых задач I/O | Для микрозадач внутри текущего event loop-тактирования |
| `setImmediate(callback)` | Фаза **check**, после `poll` | Для исполнения сразу после завершения текущих I/O-операций |
| `setTimeout(cb, 0)` | Фаза **timers** следующего цикла event loop | Планирование через минимальный таймер |

Пример порядка вывода:

js

Копировать

```
console.log('start');
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
console.log('end');
// Вывод: start, end, nextTick, timeout, immediate
```

## 26\. Смысл `ref()` и `unref()`

Каждый сетевой сокет, таймер или сервер внутри libuv представлен handle, который по умолчанию «референсится», то есть блокирует exit процесса, пока жив.

*   `unref()` — снимает блокировку, позволяя процессу завершиться, даже если handle открыт.
    
*   `ref()` — добавляет обратно.
    

Сценарий: вы создали таймер для периодического логирования, но хотите, чтобы приложение могло закончить работу, если больше нет референсов на другие ресурсы:


```js
const t = setInterval(()=>console.log('tick'), 1000);
t.unref();
```

## 27\. Депрекейт `server.connections` и получение текущего числа подключений

Свойство `server.connections` было ненадёжно синхронизировано и зачастую возвращало устаревшее значение.

### Как считать сейчас

1.  Встроенный callback:
    
    
    ```js
    server.getConnections((err, count) => console.log(count));
    ```
    
2.  Ручной счётчик:

    ```js
    let active = 0;
    server.on('connection', () => active++);
    server.on('close', sock => active--);
    ```
    

## 28\. Основные причины утечек памяти и способы борьбы

1.  Неудалённые слушатели событий
    
2.  Глобальные структуры данных без очистки (`Map`, `Set`, кэш модулей)
    
3.  Долгие таймеры и интервалы без `clearTimeout`/`clearInterval`
    
4.  Замыкания, захватывающие большие объекты
    
5.  Неправильное использование `Buffer` (slice вместо copy)
    

### Инструменты борьбы

*   Профилировщик памяти (`--inspect`, DevTools).
    
*   Дамп heap (`heapdump`, `node --expose-gc` + `global.gc()`).
    
*   Анализ heap snapshots: искать объекты с большим Retained Size.
    
*   Использовать `weak` ссылки (`WeakMap`) для кэширования.
    

## 29\. Различия `node:cluster` vs `node:child_process`

| Характеристика | `cluster` | `child_process` |
| --- | --- | --- |
| Цель | Горизонтальный форк HTTP-сервера | Общий запуск дочерних процессов |
| Балансировка порта | Мастер автоматически делит соединения | Каждый процесс сам слушает порт |
| IPC | Встроен (по умолчанию) | Нужно заводить вручную (`fork`) |
| Точка отказа | Мастер — SPOF | Отдельные процессы без общего мастера |

### Когда `cluster` может быть узким местом

*   При экстремальном количестве соединений мастер, который распределяет TCP-сокеты, становится hot-spot’ом.
    
*   Решение: разделять нагрузку через внешний балансировщик (HAProxy, nginx) или несколько мастеров на разных портах.
    

## 30\. Когда вручную управлять сборкой мусора

Node не позволяет полностью отключить GC, но можно:

*   Запустить с `--expose-gc`.
    
*   В ключевых местах приложения вызывать `global.gc()`.
    

### Сценарии

*   **Перед началом пиковых нагрузок**: гарантировать чистую память.
    
*   **После крупного батча обработки**: освободить объём до следующей фазы.
    
*   **Замер производительности**: настраивать интервалы GC для уменьшения пауз в latency-critical системах.
    

При этом важно не злоупотреблять, чтобы не увеличить общую нагрузку на CPU.

## 31\. Способы отладки приложений и сценарии их применения

Node.js предоставляет несколько подходов для отладки, которые можно комбинировать в зависимости от сложности задачи и окружения:

*   Консольное логирование
    
    *   Быстрое введение «зацепок» при простых ошибках.
        
    *   Подходит для одноразовых проверок, но неэффективно для асинхронных и многопоточных сценариев.
        
*   Встроенный отладчик (`node inspect`)
    
    *   Позволяет ставить точки останова, шагать по коду и просматривать замыкания.
        
    *   Удобен в терминале, но имеет ограниченный интерфейс.
        
*   Chrome DevTools / VS Code Debugger (`--inspect[–brk]`)
    
    *   Графический интерфейс с вкладками Sources, Network и Memory.
        
    *   Идеален для сложных цепочек колбэков, кластерной отладки и анализа утечек памяти.
        
*   Профилирование через `node:profiler` или внешние инструменты (Clinic.js, 0x)
    
    *   Строит flame graph’ы для выявления «узких мест» CPU и долгих синхронных операций.
        
    *   Используют при проблемах с производительностью.
        
*   Снимки кучи и GC-трейсы (`--inspect`, `v8.getHeapSnapshot()`)
    
    *   Диагностика утечек памяти, анализ точек удержания объектов.
        
    *   Активно применяют при подозрении на накопление объектов в замыканиях или кэше.
        

## 32\. Сброс кеша `require` для CJS и ESM

В CommonJS каждый модуль кешируется по полному пути, поэтому для перезагрузки модуля:

```js
delete require.cache[require.resolve('имя-модуля')];
const fresh = require('имя-модуля');
```

*   Необходимо чистить зависимости и ссылки на них.
    
*   Работает только для CJS-модулей.
    

В ESM кеширование жёстко контролируется движком, прямого API нет. Возможные обходы:

1.  Динамический импорт с параметром-квери:
    
    ```js
    const mod = await import(`./module.js?update=${Date.now()}`);
    ```
    
2.  Создание отдельного `loader`\-хука:
    
    *   Использовать флаг `--experimental-loader` и перехватывать `resolve`/`load`.
        
3.  Перезапуск процесса:
    
    *   Надёжный, но тяжеловесный вариант.
        

## 33\. Источник специальных идентификаторов

Node.js инжектирует или предоставляет через глобальный объект следующие сущности:

*   `__dirname`, `__filename`, `require`, `module`, `exports`
    
    *   Автоматически добавляются в тело каждого CJS-модуля через обёртку-функцию.
        
    *   Node генерирует:

        ```js
        (function(exports, require, module, __filename, __dirname) { /* код модуля */ });
        ```
        
*   `import`, `import.meta`
    
    *   Статические конструкции под управлением ESM-лоадера, нет глобальной обёртки.
        
    *   `import.meta.url` заменяет `__filename`/`__dirname`.
        
*   `fetch`
    
    *   Современный Web API, доходящий в Node.js на уровне глобального объекта.
        
*   `Array`, `Object`, `Promise` и другие
    
    *   Определены внутренним движком V8, доступны через глобальный контекст.
        

## 34\. Отказ от библиотеки `node:url`

Раньше в Node.js существовал собственный модуль `url` (legacy API):

*   Функции `url.parse`, `url.format` устарели в пользу WHATWG-совместимого интерфейса.
    

Современный подход:

*   Использовать глобальный класс `URL` и модуль `node:internal/url` (через `import { URL } from 'url'` или глобально).
    
*   `node:url` как алиас избыточен и создает путаницу между legacy и WHATWG API.
    

## 35\. Стратегии масштабирования приложений на Node.js

| Стратегия | Описание | Преимущества | Ограничения |
| --- | --- | --- | --- |
| Горизонтальный кластер | Модуль `cluster` (форк процессов, общий порт) | Простая реализация, все CPU задействованы | Единый узел master, межпроцессный обмен |
| worker\_threads | Потоки общей памяти, обмен через `MessagePort` | Низкая задержка обмена, разделение памяти | Сложнее синхронизация, риски гонок |
| Микросервисы | Отдельные сервисы, общение по HTTP/GRPC/MessageQueue | Чёткая граница ответственности, масштаб по сервисам | Оверхед сети, сложнее отладка |
| Серверлесс (FaaS) | AWS Lambda, Azure Functions | Автоматическое масштабирование, оплата за вызов | Холодный старт, ограничение времени |
| Виртуальные машины | Docker + оркестратор (Kubernetes) | Гибкий контроль окружения, отказоустойчивость | Сложность инфраструктуры, задержки деплоя |

## 36\. CPU-, RAM- и I/O-интенсивные задачи

| Тип задачи | Описание | Примеры |
| --- | --- | --- |
| CPU-интенсивные | Задачи с большим числом синхронных вычислений, блокирующих event loop | Шифрование, обработка изображений, научные расчёты |
| RAM-интенсивные | Требуют большого объёма оперативной памяти для хранения данных | Кэширование, in-memory базы, большие массивы в JS |
| I/O-интенсивные | Задачи, зависящие от скорости ввода/вывода (диск, сеть) | Чтение/запись больших файлов, сетевые API, БД-запросы |

## 37\. Почему не стоит использовать `process.on('multipleResolves', handler)`

Событие `multipleResolves` появляется при повторном резолве/реджекте одного Promise. Назначение:

*   Уведомить о логической ошибке в коде (двойной `resolve`, `reject` и т.п.).
    
*   Если поставить глобальный хендлер, можно скрыть баги, и приложение продолжит работать в неконсистентном состоянии.
    

Рекомендация: исправить причину двойного разрешения, а не подавлять предупреждения.

## 38\. Опция `noDelay` у TCP-сокетов

*   Включает или отключает алгоритм Нэгла (Nagle’s algorithm).
    
*   При выключенном `noDelay` (`Nagle`\=вкл) мелкие пакеты буферизуются и отправляются объединённо, чтобы снизить сетевой overhead.
    
*   При `socket.setNoDelay(true)` пакеты отправляются сразу, уменьшая задержки в интерактивных приложениях (игры, голосовые сервисы).
    
*   Параметр `noDelay` у сервера (`new net.Server({ noDelay: true })`) применяется ко всем входящим соединениям по умолчанию.
    

## 39\. Работа `keep-alive` в HTTP и управление из Node.js

При HTTP/1.1 соединение по умолчанию длится несколько запросов:

*   Агент (`http.Agent`) реиспользует TCP-сокеты для последующих запросов к тому же хосту.
    
*   Параметры управления:
    
    *   `agent.keepAlive` и `agent.keepAliveMsecs` в `http.request`/`https.request`.
        
    *   Заголовки `Connection: keep-alive` / `Connection: close`.
        
    *   `socket.setKeepAlive(true, initialDelay)` для TCP-keep-alive на уровне ОС.
        
*   Контролируйте максимальное число свободных сокетов (`agent.maxFreeSockets`) и таймаут ожидания (`agent.timeout`).
    

## 40\. Модуль `node:perf_hooks` и работа с воркерами

Модуль `perf_hooks` служит для точного измерения производительности:

*   `performance.now()`, `performance.mark()`, `performance.measure()`
    
*   Отслеживание задержностей event loop (`monitorEventLoopDelay`)
    
*   `PerformanceObserver` может слушать события в основном потоке и воркерах.
    
*   В воркерах API доступен без ограничений, но объекты наблюдения создаются в каждом контексте отдельно.
    

## 41\. Экспериментальные итерируемые методы у стримов

Node 20+ предлагает поддержку методов `filter`, `map`, `reduce` прямо на ReadableStream:

*   Позволяет применять декларативные фильтрации и трансформации через `for await` и цепочки.
    
*   Плюсы:
    
    *   Меньше шаблонного кода с созданием Transform-классов.
        
    *   Интеграция с другими асинхронными итераторами.
        
*   Минусы:
    
    *   Экспериментальный статус, нестабильный API.
        
    *   Потенциальные накладные расходы при большом потоке данных.
        

Рекомендация: использовать в прототипах и тестировать на производительность перед продакшен-внедрением.

## 42\. Способы интернационализации приложений

*   Встроенный API `Intl`:
    
    *   `Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.Collator`
        
    *   Поддержка локалей без сторонних зависимостей.
        
*   Библиотеки на основе CLDR:
    
    *   formatjs (`react-intl`, `@formatjs/intl`), Globalize.js, Polyglot.js..
        
*   Менеджмент переводов:
    
    *   Файлы JSON/YAML с message-ключами.
        
    *   Инструменты для извлечения строк: i18next-scanner, gettext.
        
*   Форматирование шаблонов:
    
    *   ICU MessageFormat, `@formatjs/intl-messageformat`.
        

## 43\. Встроенный Test Runner и библиотека `node:assert`

Начиная с Node 18 появился встроенный тестовый раннер:

*   Запуск через `node --test` или `npm test` (при `"test": "node --test"`).
    
*   API:
    
    ```js
    import { test } from 'node:test';
    import assert from 'node:assert';
    
    test('пример', t => {
      assert.strictEqual(1 + 1, 2);
    });
    ```
    
*   Преимущества:
    
    *   Нет внешних зависимостей, быстрый стартап.
        
    *   Поддержка ESM, флаги `--test-only`, `--test-reporter`.
        

Лично я применяю его для модульных тестов без сложной инфраструктуры, используя `node:assert` для лёгких проверок.

## 44\. Основные ключи при запуске Node.js

Часто используемые CLI-флаги:

*   `--inspect[=<порт>]` и `--inspect-brk` — включение отладки с остановкой на первой строке.
    
*   `--experimental-worker` — активация `worker_threads` (до Node 12).
    
*   `--max-old-space-size=<MB>` — лимит памяти V8 для крупных приложений.
    
*   `--enable-source-maps` — автоматическая подгрузка sourcemap-файлов.
    
*   `--trace-warnings` / `--trace-deprecation` — вывод трассировок при генерации предупреждений.
    
*   `--loader <path>` — подключение пользовательского ESM-загрузчика.
    

## 45\. Снятие дампа кучи (heap snapshot) и дальнейший анализ

1.  Установка модуля:
    
    ```bash
    npm install heapdump
    ```
    
2.  Снятие снимка:
    
    
    ```js
    import heapdump from 'heapdump';
    heapdump.writeSnapshot('/tmp/heap-' + Date.now() + '.heapsnapshot');
    ```
    
3.  Альтернативно через инспектор:
    
    
    ```bash
    node --inspect app.js
    ```
    
    В Chrome DevTools: Memory → Take heap snapshot.
    
4.  Анализ:
    
    *   Открыть `.heapsnapshot` в панели Memory DevTools.
        
    *   Искать «detached DOM trees», утечки через глобальные объекты, массивы и замыкания.
        
    *   Сравнить несколько снимков, найти нарастающие цепочки ссылок.

## 46\. Как построить flame graph?

Для создания flame graph потребуются данные о времени выполнения функций и инструмент для визуализации.

1.  Сбор профиля
    
    *   Запустите Node.js с профайлером V8:
        
        
        ```
        node --prof app.js
        ```
        
    *   После завершения работы получите файл `isolate-*.log`.
        
2.  Конвертация лога
    
    *   Используйте скрипт из дистрибутива Node.js:
        
        
        ```
        node --prof-process isolate-*.log > processed.txt
        ```
        
3.  Генерация flame graph
    
    *   Установите утилиты Brendan Gregg’s FlameGraph:
        
        
        ```
        git clone https://github.com/brendangregg/FlameGraph.git
        ```
        
    *   Сформируйте входной стектрейс:
        
        
        ```
        node --prof-process isolate-*.log | FlameGraph/stackcollapse-v8.pl > out.folded
        ```
        
    *   Постройте SVG:
        
        
        ```
        FlameGraph/flamegraph.pl out.folded > flamegraph.svg
        ```
        
4.  Анализ
    
    *   Откройте `flamegraph.svg` в браузере.
        
    *   Узкие места отображаются широкими блоками вверху графа.
        

## 47\. Про ALPN и SNI и их поддержку в Node.js

Application-Layer Protocol Negotiation и Server Name Indication важны для современных HTTPS-соединений:

*   SNI (Server Name Indication)
    
    *   Позволяет клиенту указать доменное имя на этапе TLS-рукопожатия.
        
    *   Node.js поддерживает SNI по умолчанию через опцию `servername` в `tls.connect` или конфигурацию HTTPS-сервера.
        
*   ALPN (Application-Layer Protocol Negotiation)
    
    *   Клиент и сервер договариваются о протоколе прикладного уровня (HTTP/1.1, HTTP/2).
        
    *   В Node.js для сервера:
        
        
        ```js
        const server = http2.createSecureServer({
          allowHTTP1: true,
          key: fs.readFileSync('key.pem'),
          cert: fs.readFileSync('cert.pem')
        });
        ```
        
    *   Для клиента:
  
        
        ```js
        const client = http2.connect('https://example.com', {
          ALPNProtocols: ['h2', 'http/1.1']
        });
        ```
        

## 48\. Автоматическая перезагрузка процесса при изменении кода

Нативных средств в Node.js нет, но можно собрать простой watcher:

1.  Использовать модуль `fs.watch`:
    
    
    ```js
    const { spawn } = require('child_process');
    const { watch } = require('fs');
    
    let proc = spawn('node', ['app.js'], { stdio: 'inherit' });
    
    watch('./src', { recursive: true }, () => {
      proc.kill();
      proc = spawn('node', ['app.js'], { stdio: 'inherit' });
    });
    ```
    
2.  Или использовать флаг экспериментального наблюдения:
    
    
    ```
    node --watch app.js
    ```
    
3.  Альтернативы
    
    *   Пакеты `nodemon`, `pm2 --watch` для более гибкой настройки.
        

## 49\. Для чего нужен модуль `node:module`?

Модуль `node:module` предоставляет низкоуровневый API загрузчика:

*   Доступ к классу `Module` и его статическим методам
    
*   Позволяет:
    
    *   Создавать пользовательские резолверы и трансформеры
        
    *   Манипулировать кэшем через `Module._cache`
        
    *   Регистрировать дополнительные расширения с `Module._extensions`
        

Часто используется в инструментах для сборки и тестирования, где нужно перехватывать `require`/`import`.

## 50\. Работа с самоподписанными SSL-сертификатами и их ограничения

Для использования самоподписного сертификата:

1.  Генерация:
    
    
    ```
    openssl genrsa -out key.pem 2048
    openssl req -new -key key.pem -out csr.pem
    openssl x509 -req -in csr.pem -signkey key.pem -out cert.pem
    ```
    
2.  Настройка HTTPS-сервера:

    ```js
    https.createServer({
      key: fs.readFileSync('key.pem'),
      cert: fs.readFileSync('cert.pem')
    }, handler);
    ```
    

Ограничения:

*   Клиент браузера или HTTP-клиент будет жаловаться на небезопасность.
    
*   Нельзя использовать в продуктивных системах без доверенного CA.
    
*   При масштабировании нужно вручную устанавливать доверенные сертификаты на каждом клиенте или в промежуточных прокси.
    

## 51\. Web Crypto API vs `node:crypto`

| Параметр | Web Crypto API | node:crypto |
| --- | --- | --- |
| Доступность | Глобальный `crypto.subtle` | Требует `import { webcrypto } from 'crypto'` |
| Поддерживаемые алгоритмы | Ограниченный набор стандартных алгоритмов | Широкий выбор (OpenSSL-алгоритмы) |
| Асинхронность | Promise-базированное API | Синхронные и колбэк-функции |
| Сценарии использования | Веб-стандарты, единая логика на клиенте и сервере | Низкоуровневое шифрование, потоки, хэши |

## 52\. Web Streams API vs `node:stream`

| Характеристика | Web Streams API | node:stream |
| --- | --- | --- |
| Интерфейс | Promise-базированный, единый для Web и Node | Callback/событийный, оптимизирован для Node |
| Обратная совместимость | Нужны полифилы или флаги | Богатый набор методов `pipe`, `unpipe` |
| Сценарии | Совместимость между браузером и сервером | Высокопроизводительные файловые и сетевые I/O |

## 53\. Классы `Blob` и `File` из `node:buffer`

*   `Blob`
    
    *   Хранит бинарные данные как набор частей (`ArrayBuffer`, `Buffer`, `string`).
        
    *   Используется для удобной передачи и сериализации данных.
        
*   `File extends Blob`
    
    *   Дополнительно хранит имя, дату изменения и тип контента.
        
    *   Удобен для эмуляции работы с файлами в браузерных API и при приёме multipart в Node.js..
        

Применяются в Web Streams API и при взаимодействии с `fetch` в Node.js..

## 54\. Модели прав доступа module-based и process-based

*   Module-based
    
    *   Ограничение импорта через разрешённые пути или виртуальную файловую систему.
        
    *   Полезно для плагинов, sandboxing и повышенной безопасности.
        
*   Process-based
    
    *   Разделение прав на уровне пользовательских групп и прав ОС.
        
    *   Каждый процесс запускается от имени отдельного пользователя или с разным набором привилегий.
        

Часто комбинируются: главный процесс запускает Workers с ограниченными правами, а внутри ещё применяется модульное ограничение.

## 55\. Чего было deprecated в `node:async_hooks`?

*   `async_hooks.createHook` с коллбэками `init`, `before`, `after`, `destroy` осталось, а `Promise`\-специфичные события (`promiseResolve`) ранее были экспериментальными и меняли семантику.
    
*   Некоторые внутренние API, например `executionAsyncId()` без контекста, были помечены устаревшими в пользу `AsyncLocalStorage`.
    

## 56\. Класс `AsyncResource` и его использование

`AsyncResource` позволяет корректно сохранять контекст асинхронного выполнения:

```js
const { AsyncResource } = require('async_hooks');

class MyResource extends AsyncResource {
  constructor() {
    super('MY_RESOURCE');
  }

  run(callback) {
    this.runInAsyncScope(callback);
  }
}

const resource = new MyResource();
resource.run(() => {
  // контекст асинхронных хуков сохранён
});
```

Используется в библиотеках для прозрачной работы с `AsyncLocalStorage`, трассировкой и метриками.

## 57\. Как найти вызовы всех deprecated API в приложении?

1.  Запустить Node.js с флагом трассировки:
    
    ```
    node --trace-deprecation app.js
    ```
    
2.  Отфильтровать вывод по ключевым словам `DeprecationWarning`.
    
3.  Использовать ESLint-плагин `eslint-plugin-node` с правилом `no-deprecated-api`.
    
4.  Автоматизировать отчёт через CI, запрещая депрецированные вызовы как ошибки.
    

## 58\. Работа с зависимостями в single executable

*   Упаковщики (`pkg`, `nexe`) встраивают модули внутрь двоичного файла.
    
*   Для динамических `require` необходимо указать дополнительные параметры конфигурации, чтобы включить файлы в бандл.
    
*   Ограничение: бинарник не может динамически подхватить новый файл без пересборки.
    
*   В качестве альтернативы можно использовать виртуальную файловую систему в памяти или загружать плагины через сеть.
    

## 59\. Проблемы с нативным test runner в Node.js

*   Ограниченная экосистема плагинов и малое количество реализаций репортеров.
    
*   Нет поддержки snapshot-тестирования из коробки.
    
*   Отсутствует живой watch-режим, как у Jest или Mocha.
    
*   Миграция сложных проектов с параметризованными тестами или BDD-стилем требует дополнительных усилий.
    

## 60\. Новые возможности JavaScript в Node.js 18 и 20

| Версия | Фичи | Описание |
| --- | --- | --- |
| 18  | Global fetch и Web Streams | Убрана необходимость в `node-fetch`, унификация API |
|     | Test Runner | Встроенный фреймворк для юнит-тестов |
|     | `AbortSignal.timeout` | Таймауты через `fetch` и другие асинхронные операции |
|     | `Undici` как HTTP-клиент | Высокопроизводительный клиент вместо `http`/`https` |
| 20  | Experimental Web Crypto API (Stable) | Расширены алгоритмы и синхронная поддержка |
|     | Experimental iterable stream methods | Цепочки `map`, `filter`, `reduce` для потоков |
|     | `Object.hasOwn` | Статичный метод безопасности вместо `hasOwnProperty` |
|     | V8 10.0 | Новые оптимизации в JavaScript-движке |

## 60. Какие новые возможности JavaScript появились в node.js при обновлении до версий 18 и 20, 22?

## Node.js 18

*   Global fetch / Request / Response / Headers Позволяют выполнять HTTP-запросы по стандарту WHATWG без внешних библиотек:
    
    
    ```js
    const res = await fetch('https://api.example.com/data');
    const data = await res.json();
    ```
    
*   Web Streams API (ReadableStream, WritableStream, TransformStream) Единый асинхронный API потоков в стиле браузеров:

    
    ```js
    const { readable, writable } = new TransformStream();
    ```
    
*   Timers Promises API Promise-версия таймеров вместо колбэков:
    
    
    ```js
    import { setTimeout } from 'timers/promises';
    await setTimeout(100);
    ```
    
*   AbortSignal.timeout Быстрый способ задать таймаут для fetch, таймеров и других API:
    
    ```js
    const controller = new AbortController();
    AbortSignal.timeout(5000).pipeTo(controller.signal);
    await fetch(url, { signal: controller.signal });
    ```
    
*   Встроенный тестовый раннер (`node:test`) Простые юнит-тесты без сторонних фреймворков:

    
    ```js
    import test from 'node:test';
    import assert from 'node:assert';
    
    test('1 + 1 = 2', () => {
      assert.strictEqual(1 + 1, 2);
    });
    ```
    
*   Встроенный HTTP-клиент Undici Высокопроизводительный fetch-совместимый клиент вместо `http` / `https`:
    
    
    ```js
    import { request } from 'undici';
    const { body } = await request('https://example.com');
    ```
    

## Node.js 20

*   Stable Web Crypto API (`crypto.subtle`) Асинхронные криптовычисления по стандарту Web Crypto:
    
    ```js
    import { webcrypto } from 'crypto';
    const key = await webcrypto.subtle.generateKey({ name: 'AES-GCM', length: 256 }, true, ['encrypt', 'decrypt']);
    ```
    
*   Experimental iterable stream methods Возможность писать декларативные трансформации над `ReadableStream`:
    
    ```js
    const numbers = Readable.from([1,2,3,4,5]);
    for await (const x of numbers.filter(n => n % 2 === 1).map(n => n * 2)) {
      console.log(x);
    }
    ```
    
*   Object.hasOwn Удобный и безопасный метод проверки собственных свойств:
    
    
    ```js
    Object.hasOwn(obj, 'foo');
    ```
    
*   Top-level await в REPL и модулях CommonJS Можно использовать `await` без обёртки в интерактивном режиме.
    
*   V8 11 → новые языковые фичи
    
    *   `Array.prototype.at()` для доступа по отрицательным индексам
        
    *   `String.prototype.replaceAll()`
        
    *   `Promise.any()` + `AggregateError`
        

## Node.js 22

*   Полная стабилизация Web Streams API Отработаны все краевые случаи, единообразная работа в разных контекстах.
    
*   globalThis.structuredClone Встроенный глубокий клон любых JS-структур без JSON-ограничений:
    
    ```js
    const deep = structuredClone(complexObj);
    ```
    
*   globalThis.URLPattern Новый стандартный API для сопоставления и разбора URL:

    ```js
    const pattern = new URLPattern({ pathname: '/users/:id' });
    pattern.test('/users/42'); // true
    ```
    
*   Расширенный Intl
    
    *   `Intl.Locale` для работы с региональными настройками
        
    *   `Intl.DisplayNames` для человекочитаемых названий языков, регионов, валют
        
    *   `Intl.ListFormat` для форматирования списков:
        
        
        ```js
        new Intl.ListFormat('ru').format(['яблоко','груша','слива']);
        ```
        
*   Обновление V8 12 Включает последние оптимизации, поддержку новых синтаксических фич и более производительный GC.
    