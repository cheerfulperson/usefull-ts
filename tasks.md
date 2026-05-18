Ниже — разбор **типовых задач на собеседовании Middle/Senior React Frontend Developer 3–5 лет**. Я разделю их на категории: **JavaScript**, **React**, **TypeScript**, **асинхронность**, **алгоритмы**, **архитектура**, **performance**, **forms**, **API**, **testing**, **CSS**, **system design**.

---

# 1. JavaScript-задачи

## 1.1. Что выведет код: event loop

### Задача

```js
console.log('A');

setTimeout(() => {
  console.log('B');
}, 0);

Promise.resolve().then(() => {
  console.log('C');
});

console.log('D');
```

### Ответ

```txt
A
D
C
B
```

### Что проверяют

Понимание порядка:

```txt
sync code → microtasks → macrotasks
```

`Promise.then` — microtask.
`setTimeout` — macrotask.

---

## 1.2. Более сложный event loop

### Задача

```js
console.log('1');

setTimeout(() => {
  console.log('2');

  Promise.resolve().then(() => {
    console.log('3');
  });
}, 0);

Promise.resolve().then(() => {
  console.log('4');

  setTimeout(() => {
    console.log('5');
  }, 0);
});

console.log('6');
```

### Ответ

```txt
1
6
4
2
3
5
```

### Разбор

Сначала sync:

```txt
1
6
```

Потом microtask:

```txt
4
```

Внутри неё создаётся `setTimeout 5`.

Очередь macrotasks:

```txt
timeout 2
timeout 5
```

Выполняется `timeout 2`, внутри него создаётся microtask `3`.

После каждой macrotask очищаются microtasks:

```txt
3
```

Потом следующая macrotask:

```txt
5
```

---

## 1.3. Promise executor

### Задача

```js
console.log('A');

new Promise((resolve) => {
  console.log('B');
  resolve();
}).then(() => {
  console.log('C');
});

console.log('D');
```

### Ответ

```txt
A
B
D
C
```

### Что проверяют

Executor внутри `new Promise` выполняется **синхронно**.

---

## 1.4. Async/await

### Задача

```js
async function run() {
  console.log('A');

  await null;

  console.log('B');
}

console.log('C');

run();

console.log('D');
```

### Ответ

```txt
C
A
D
B
```

### Почему

До `await` код выполняется синхронно.
После `await` продолжение функции уходит в microtask queue.

---

## 1.5. Closures

### Задача

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

### Ответ

```txt
3
3
3
```

### Почему

`var` имеет function scope. Все callbacks ссылаются на одну и ту же переменную `i`.

### Как исправить

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

Ответ:

```txt
0
1
2
```

---

## 1.6. Написать debounce

### Задача

Реализовать функцию `debounce`.

```js
function debounce(fn, delay) {
  let timerId;

  return function (...args) {
    clearTimeout(timerId);

    timerId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}
```

### Пример использования

```js
const onSearch = debounce((value) => {
  console.log('search:', value);
}, 300);

onSearch('r');
onSearch('re');
onSearch('rea');
onSearch('react');
```

Выполнится только последний вызов.

### Что проверяют

Понимание:

```txt
closures
timers
this
arguments
```

---

## 1.7. Написать throttle

### Задача

```js
function throttle(fn, delay) {
  let lastCall = 0;

  return function (...args) {
    const now = Date.now();

    if (now - lastCall >= delay) {
      lastCall = now;
      fn.apply(this, args);
    }
  };
}
```

### Где используется

```txt
scroll
resize
mousemove
drag
```

---

## 1.8. Deep clone

### Простая версия

```js
function deepClone(value) {
  if (value === null || typeof value !== 'object') {
    return value;
  }

  if (Array.isArray(value)) {
    return value.map(deepClone);
  }

  const result = {};

  for (const key in value) {
    result[key] = deepClone(value[key]);
  }

  return result;
}
```

### Подводные камни

Эта версия не покрывает:

```txt
Date
Map
Set
RegExp
Function
Symbol
циклические ссылки
prototype
non-enumerable properties
```

### Более сильный ответ

```js
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') {
    return value;
  }

  if (seen.has(value)) {
    return seen.get(value);
  }

  if (value instanceof Date) {
    return new Date(value);
  }

  if (value instanceof RegExp) {
    return new RegExp(value);
  }

  if (Array.isArray(value)) {
    const result = [];
    seen.set(value, result);

    for (const item of value) {
      result.push(deepClone(item, seen));
    }

    return result;
  }

  const result = {};
  seen.set(value, result);

  for (const key of Object.keys(value)) {
    result[key] = deepClone(value[key], seen);
  }

  return result;
}
```

### Что проверяют

```txt
рекурсия
типы данных
WeakMap
циклические ссылки
понимание ограничений
```

---

# 2. React-задачи

## 2.1. Что будет с count

### Задача

```tsx
const [count, setCount] = useState(0);

useEffect(() => {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
}, []);
```

### Ответ

```txt
count = 1
```

### Почему

Внутри эффекта `count` равен `0`.

Все три вызова эквивалентны:

```tsx
setCount(1);
setCount(1);
setCount(1);
```

---

## 2.2. Как получить count = 3

```tsx
useEffect(() => {
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
}, []);
```

### Почему

React применяет updates последовательно:

```txt
0 → 1 → 2 → 3
```

---

## 2.3. Stale closure

### Задача

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);
    }, 1000);

    return () => clearInterval(id);
  }, []);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

### Что не так

`setInterval` всегда будет видеть `count` из первого render.

То есть:

```txt
count = 0
```

### Вариант исправления через dependencies

```tsx
useEffect(() => {
  const id = setInterval(() => {
    console.log(count);
  }, 1000);

  return () => clearInterval(id);
}, [count]);
```

### Вариант через ref

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;
  }, [count]);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(countRef.current);
    }, 1000);

    return () => clearInterval(id);
  }, []);

  return (
    <button onClick={() => setCount(prev => prev + 1)}>
      {count}
    </button>
  );
}
```

### Что проверяют

```txt
closures
useEffect dependencies
useRef
lifecycle effects
```

---

## 2.4. Найти ошибку в useEffect

### Задача

```tsx
useEffect(() => {
  fetchUser(userId);
}, []);
```

### Проблема

Если `userId` изменится, эффект не перезапустится.

### Исправление

```tsx
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

### Более сильный вариант

```tsx
useEffect(() => {
  const controller = new AbortController();

  async function loadUser() {
    try {
      const response = await fetch(`/api/users/${userId}`, {
        signal: controller.signal,
      });

      const data = await response.json();
      setUser(data);
    } catch (error) {
      if (error.name !== 'AbortError') {
        setError(error);
      }
    }
  }

  loadUser();

  return () => {
    controller.abort();
  };
}, [userId]);
```

---

## 2.5. Реализовать `useDebounce`

### Задача

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const id = window.setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      window.clearTimeout(id);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

### Использование

```tsx
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (!debouncedQuery) return;

    fetch(`/api/search?q=${debouncedQuery}`);
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={event => setQuery(event.target.value)}
    />
  );
}
```

### Что проверяют

```txt
custom hooks
useEffect cleanup
generic types
controlled input
API calls
```

---

## 2.6. Реализовать `usePrevious`

### Задача

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}
```

### Использование

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const previousCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {previousCount}</p>
      <button onClick={() => setCount(prev => prev + 1)}>
        +
      </button>
    </div>
  );
}
```

### Важный момент

`useRef` не вызывает re-render при изменении `.current`.

---

## 2.7. Реализовать `useToggle`

```tsx
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => {
    setValue(prev => !prev);
  }, []);

  const setTrue = useCallback(() => {
    setValue(true);
  }, []);

  const setFalse = useCallback(() => {
    setValue(false);
  }, []);

  return {
    value,
    toggle,
    setTrue,
    setFalse,
  };
}
```

### Что проверяют

```txt
custom hooks
useState
useCallback
API хука
```

---

## 2.8. Реализовать Modal через Portal

### Задача

```tsx
import { createPortal } from 'react-dom';

type ModalProps = {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
};

function Modal({ isOpen, onClose, children }: ModalProps) {
  if (!isOpen) return null;

  return createPortal(
    <div
      role="dialog"
      aria-modal="true"
      className="modal-backdrop"
      onClick={onClose}
    >
      <div
        className="modal-content"
        onClick={event => event.stopPropagation()}
      >
        <button
          type="button"
          aria-label="Close modal"
          onClick={onClose}
        >
          ×
        </button>

        {children}
      </div>
    </div>,
    document.body
  );
}
```

### Что можно добавить для Senior-уровня

```txt
закрытие по Escape
focus trap
restore focus после закрытия
scroll lock
aria-labelledby
aria-describedby
animation
```

---

## 2.9. Закрытие modal по Escape

```tsx
useEffect(() => {
  if (!isOpen) return;

  function handleKeyDown(event: KeyboardEvent) {
    if (event.key === 'Escape') {
      onClose();
    }
  }

  document.addEventListener('keydown', handleKeyDown);

  return () => {
    document.removeEventListener('keydown', handleKeyDown);
  };
}, [isOpen, onClose]);
```

---

## 2.10. Реализовать Error Boundary

### Важно

Error Boundary в React пока классически реализуется через class component.

```tsx
type Props = {
  children: React.ReactNode;
};

type State = {
  hasError: boolean;
};

class ErrorBoundary extends React.Component<Props, State> {
  state: State = {
    hasError: false,
  };

  static getDerivedStateFromError(): State {
    return {
      hasError: true,
    };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error(error, info);
  }

  render() {
    if (this.state.hasError) {
      return <div>Something went wrong</div>;
    }

    return this.props.children;
  }
}
```

### Что проверяют

```txt
render errors
fallback UI
logging
React lifecycle
```

### Важно сказать

Error Boundary не ловит:

```txt
ошибки внутри event handlers
async errors
setTimeout errors
server-side rendering errors
ошибки внутри самого Error Boundary
```

---

# 3. Задачи на React rendering и performance

## 3.1. Почему компонент перерендеривается

### Задача

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  const user = {
    name: 'Alex',
  };

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child user={user} />
    </>
  );
}

const Child = React.memo(function Child({ user }) {
  console.log('Child render');

  return <div>{user.name}</div>;
});
```

### Вопрос

Почему `Child` перерендеривается, хотя используется `React.memo`?

### Ответ

Потому что при каждом render создаётся новый объект:

```tsx
const user = {
  name: 'Alex',
};
```

Для shallow comparison это новый prop.

### Исправление

```tsx
const user = useMemo(() => {
  return {
    name: 'Alex',
  };
}, []);
```

---

## 3.2. Почему useCallback может быть нужен

### Задача

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  function handleClick() {
    console.log('clicked');
  }

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child onClick={handleClick} />
    </>
  );
}

const Child = React.memo(function Child({ onClick }) {
  console.log('Child render');

  return <button onClick={onClick}>Click</button>;
});
```

### Проблема

`handleClick` создаётся заново при каждом render.

### Исправление

```tsx
const handleClick = useCallback(() => {
  console.log('clicked');
}, []);
```

### Но Senior-ответ

Не надо использовать `useCallback` везде. Он полезен, когда:

```txt
функция передаётся в memoized child
функция является dependency другого hook
создание функции реально влияет на performance
```

---

## 3.3. Большой список

### Задача

Есть список из 10 000 элементов. UI тормозит. Что делать?

### Хороший ответ

```txt
virtualization
pagination
infinite scroll
memoization row components
stable keys
avoid inline heavy calculations
move filtering/sorting to server or web worker
debounce search
```

### Пример с react-window

```tsx
import { FixedSizeList } from 'react-window';

function UsersList({ users }) {
  return (
    <FixedSizeList
      height={500}
      width={400}
      itemCount={users.length}
      itemSize={40}
    >
      {({ index, style }) => (
        <div style={style}>
          {users[index].name}
        </div>
      )}
    </FixedSizeList>
  );
}
```

---

# 4. TypeScript-задачи

## 4.1. Типизировать props

```tsx
type ButtonProps = {
  variant: 'primary' | 'secondary';
  disabled?: boolean;
  children: React.ReactNode;
  onClick: () => void;
};

function Button({
  variant,
  disabled = false,
  children,
  onClick,
}: ButtonProps) {
  return (
    <button
      className={`button button_${variant}`}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}
```

---

## 4.2. Discriminated union

### Задача

Описать состояние загрузки.

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };
```

### Использование

```tsx
function UsersView({ state }: { state: RequestState<User[]> }) {
  switch (state.status) {
    case 'idle':
      return <div>Idle</div>;

    case 'loading':
      return <div>Loading...</div>;

    case 'success':
      return <UsersList users={state.data} />;

    case 'error':
      return <div>{state.error}</div>;

    default:
      return null;
  }
}
```

### Что проверяют

```txt
union types
type narrowing
safe state modeling
```

---

## 4.3. Generic fetcher

### Задача

```ts
async function request<T>(url: string): Promise<T> {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }

  return response.json() as Promise<T>;
}
```

### Использование

```ts
type User = {
  id: string;
  name: string;
};

const user = await request<User>('/api/user');
```

### Senior-комментарий

`as Promise<T>` не валидирует данные runtime. Для реальной безопасности лучше использовать schema validation:

```txt
zod
valibot
io-ts
```

---

## 4.4. Типизировать polymorphic component

Частая senior-задача для design system.

### Пример

```tsx
type ButtonProps<T extends React.ElementType> = {
  as?: T;
  children: React.ReactNode;
} & React.ComponentPropsWithoutRef<T>;

function Button<T extends React.ElementType = 'button'>({
  as,
  children,
  ...props
}: ButtonProps<T>) {
  const Component = as || 'button';

  return (
    <Component {...props}>
      {children}
    </Component>
  );
}
```

### Использование

```tsx
<Button onClick={() => console.log('click')}>
  Button
</Button>

<Button as="a" href="/profile">
  Link
</Button>
```

### Что проверяют

```txt
generics
React.ElementType
ComponentPropsWithoutRef
design system thinking
```

---

# 5. Задачи на API и async

## 5.1. Search input с debounce и отменой запроса

### Требование

Сделать поиск пользователей:

```txt
input
debounce 300 ms
loading state
error state
отмена старого запроса
защита от race condition
```

### Решение

```tsx
type User = {
  id: string;
  name: string;
};

function UserSearch() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  const [users, setUsers] = useState<User[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!debouncedQuery.trim()) {
      setUsers([]);
      return;
    }

    const controller = new AbortController();

    async function loadUsers() {
      try {
        setIsLoading(true);
        setError(null);

        const response = await fetch(
          `/api/users?search=${encodeURIComponent(debouncedQuery)}`,
          {
            signal: controller.signal,
          }
        );

        if (!response.ok) {
          throw new Error('Failed to load users');
        }

        const data = await response.json();
        setUsers(data);
      } catch (error) {
        if (error instanceof Error && error.name !== 'AbortError') {
          setError(error.message);
        }
      } finally {
        setIsLoading(false);
      }
    }

    loadUsers();

    return () => {
      controller.abort();
    };
  }, [debouncedQuery]);

  return (
    <div>
      <input
        value={query}
        onChange={event => setQuery(event.target.value)}
        placeholder="Search users"
      />

      {isLoading && <div>Loading...</div>}
      {error && <div role="alert">{error}</div>}

      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Что проверяют

```txt
controlled input
debounce
useEffect
AbortController
loading/error state
encoding query params
race conditions
```

### Подводный камень

В `finally` может быть проблема, если request aborted. Иногда нужно аккуратно не делать `setIsLoading(false)` для отменённого запроса, если новый запрос уже стартовал. Более продвинутый вариант — request id.

---

## 5.2. Race condition без AbortController

```tsx
useEffect(() => {
  let ignore = false;

  async function loadUsers() {
    const response = await fetch(`/api/users?search=${query}`);
    const data = await response.json();

    if (!ignore) {
      setUsers(data);
    }
  }

  loadUsers();

  return () => {
    ignore = true;
  };
}, [query]);
```

### Идея

Старый запрос может завершиться позже нового. Флаг `ignore` не даёт старому ответу обновить state.

---

# 6. Задачи на формы

## 6.1. Controlled form

### Задача

Сделать форму логина.

```tsx
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  function handleSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();

    console.log({
      email,
      password,
    });
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Email
        <input
          type="email"
          value={email}
          onChange={event => setEmail(event.target.value)}
        />
      </label>

      <label>
        Password
        <input
          type="password"
          value={password}
          onChange={event => setPassword(event.target.value)}
        />
      </label>

      <button type="submit">Login</button>
    </form>
  );
}
```

### Что проверяют

```txt
controlled components
event typing
preventDefault
accessibility labels
```

---

## 6.2. Dynamic fields

### Задача

Добавлять и удалять email-поля.

```tsx
type EmailField = {
  id: string;
  value: string;
};

function EmailsForm() {
  const [emails, setEmails] = useState<EmailField[]>([
    {
      id: crypto.randomUUID(),
      value: '',
    },
  ]);

  function addEmail() {
    setEmails(prev => [
      ...prev,
      {
        id: crypto.randomUUID(),
        value: '',
      },
    ]);
  }

  function removeEmail(id: string) {
    setEmails(prev => prev.filter(email => email.id !== id));
  }

  function updateEmail(id: string, value: string) {
    setEmails(prev =>
      prev.map(email =>
        email.id === id
          ? { ...email, value }
          : email
      )
    );
  }

  return (
    <div>
      {emails.map(email => (
        <div key={email.id}>
          <input
            value={email.value}
            onChange={event => updateEmail(email.id, event.target.value)}
          />

          <button
            type="button"
            onClick={() => removeEmail(email.id)}
          >
            Remove
          </button>
        </div>
      ))}

      <button type="button" onClick={addEmail}>
        Add email
      </button>
    </div>
  );
}
```

### Важный момент

Не использовать индекс массива как `key`, если список изменяется.

---

# 7. Алгоритмические задачи

На frontend-интервью обычно дают не hardcore LeetCode, а практичные задачи.

## 7.1. Удалить дубликаты из массива

```js
function unique(array) {
  return [...new Set(array)];
}
```

### Для объектов по id

```js
function uniqueById(items) {
  const map = new Map();

  for (const item of items) {
    map.set(item.id, item);
  }

  return [...map.values()];
}
```

---

## 7.2. Сгруппировать массив по ключу

### Задача

```js
const users = [
  { id: 1, role: 'admin', name: 'Alex' },
  { id: 2, role: 'user', name: 'Bob' },
  { id: 3, role: 'admin', name: 'Kate' },
];
```

Нужно получить:

```js
{
  admin: [
    { id: 1, role: 'admin', name: 'Alex' },
    { id: 3, role: 'admin', name: 'Kate' },
  ],
  user: [
    { id: 2, role: 'user', name: 'Bob' },
  ],
}
```

### Решение

```js
function groupBy(array, key) {
  return array.reduce((acc, item) => {
    const groupKey = item[key];

    if (!acc[groupKey]) {
      acc[groupKey] = [];
    }

    acc[groupKey].push(item);

    return acc;
  }, {});
}
```

### TypeScript-версия

```ts
function groupBy<T, K extends keyof T>(
  array: T[],
  key: K
): Record<string, T[]> {
  return array.reduce<Record<string, T[]>>((acc, item) => {
    const groupKey = String(item[key]);

    if (!acc[groupKey]) {
      acc[groupKey] = [];
    }

    acc[groupKey].push(item);

    return acc;
  }, {});
}
```

---

## 7.3. Flatten array

### Задача

```js
flatten([1, [2, [3, 4]], 5]);
// [1, 2, 3, 4, 5]
```

### Решение

```js
function flatten(array) {
  const result = [];

  for (const item of array) {
    if (Array.isArray(item)) {
      result.push(...flatten(item));
    } else {
      result.push(item);
    }
  }

  return result;
}
```

---

## 7.4. Проверить палиндром

```js
function isPalindrome(value) {
  const normalized = value
    .toLowerCase()
    .replace(/[^a-z0-9а-яё]/gi, '');

  return normalized === normalized.split('').reverse().join('');
}
```

---

## 7.5. Найти самый частый элемент

```js
function mostFrequent(array) {
  const counts = new Map();

  for (const item of array) {
    counts.set(item, (counts.get(item) || 0) + 1);
  }

  let result;
  let maxCount = 0;

  for (const [item, count] of counts) {
    if (count > maxCount) {
      result = item;
      maxCount = count;
    }
  }

  return result;
}
```

---

## 7.6. Реализовать memoize

```js
function memoize(fn) {
  const cache = new Map();

  return function (...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);

    return result;
  };
}
```

### Подводные камни

```txt
JSON.stringify не подходит для всех случаев
порядок ключей объектов может влиять
функции/undefined/Symbol сериализуются плохо
cache может расти бесконечно
```

---

# 8. Задачи на data transformation

Это очень частый формат для frontend.

## 8.1. Построить дерево из плоского списка

### Дано

```js
const items = [
  { id: 1, parentId: null, name: 'Root' },
  { id: 2, parentId: 1, name: 'Child 1' },
  { id: 3, parentId: 1, name: 'Child 2' },
  { id: 4, parentId: 2, name: 'Nested child' },
];
```

### Нужно

```js
[
  {
    id: 1,
    parentId: null,
    name: 'Root',
    children: [
      {
        id: 2,
        parentId: 1,
        name: 'Child 1',
        children: [
          {
            id: 4,
            parentId: 2,
            name: 'Nested child',
            children: [],
          },
        ],
      },
      {
        id: 3,
        parentId: 1,
        name: 'Child 2',
        children: [],
      },
    ],
  },
]
```

### Решение

```js
function buildTree(items) {
  const map = new Map();
  const roots = [];

  for (const item of items) {
    map.set(item.id, {
      ...item,
      children: [],
    });
  }

  for (const item of map.values()) {
    if (item.parentId === null) {
      roots.push(item);
    } else {
      const parent = map.get(item.parentId);

      if (parent) {
        parent.children.push(item);
      }
    }
  }

  return roots;
}
```

### Что проверяют

```txt
Map
O(n)
мутация контролируемых структур
tree data
edge cases
```

### Edge cases

```txt
parentId не найден
циклическая структура
несколько root nodes
пустой список
дубликаты id
```

---

## 8.2. Нормализация данных

### Дано

```js
const users = [
  { id: '1', name: 'Alex' },
  { id: '2', name: 'Bob' },
];
```

### Нужно

```js
{
  byId: {
    '1': { id: '1', name: 'Alex' },
    '2': { id: '2', name: 'Bob' },
  },
  allIds: ['1', '2'],
}
```

### Решение

```js
function normalizeUsers(users) {
  return users.reduce(
    (acc, user) => {
      acc.byId[user.id] = user;
      acc.allIds.push(user.id);

      return acc;
    },
    {
      byId: {},
      allIds: [],
    }
  );
}
```

### Зачем это нужно

```txt
быстрый доступ по id
удобное обновление entities
Redux-style state
избежание дублирования данных
```

---

# 9. Redux / State management задачи

## 9.1. Написать reducer

### Задача

```ts
type Todo = {
  id: string;
  text: string;
  completed: boolean;
};

type State = {
  todos: Todo[];
};

type Action =
  | { type: 'todo/add'; payload: { text: string } }
  | { type: 'todo/toggle'; payload: { id: string } }
  | { type: 'todo/remove'; payload: { id: string } };

function todosReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'todo/add':
      return {
        ...state,
        todos: [
          ...state.todos,
          {
            id: crypto.randomUUID(),
            text: action.payload.text,
            completed: false,
          },
        ],
      };

    case 'todo/toggle':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed }
            : todo
        ),
      };

    case 'todo/remove':
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload.id),
      };

    default:
      return state;
  }
}
```

### Важный Senior-комментарий

Генерация `id` внутри reducer делает reducer не совсем pure. Лучше генерировать `id` до dispatch:

```ts
dispatch({
  type: 'todo/add',
  payload: {
    id: crypto.randomUUID(),
    text,
  },
});
```

---

## 9.2. Когда React Query, когда Redux

### Хороший ответ

```txt
React Query / TanStack Query:
server state, кеш API, refetch, stale data, pagination, retries

Redux / Zustand:
client state, сложная бизнес-логика, wizard state, editor state, complex UI state

Context:
theme, locale, auth user, простые глобальные значения

Local state:
то, что нужно одному компоненту или маленькой ветке
```

---

# 10. CSS / Layout задачи

## 10.1. Центрировать элемент

### Flexbox

```css
.parent {
  display: flex;
  justify-content: center;
  align-items: center;
}
```

### Grid

```css
.parent {
  display: grid;
  place-items: center;
}
```

---

## 10.2. Сделать responsive grid

```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 16px;
}
```

### Что хорошо сказать

`auto-fit` и `minmax` позволяют сетке адаптироваться без большого количества media queries.

---

## 10.3. Truncate text

```css
.text {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}
```

---

## 10.4. Sticky header

```css
.header {
  position: sticky;
  top: 0;
  z-index: 10;
}
```

### Подводные камни

`position: sticky` может не работать, если у родителя есть:

```css
overflow: hidden;
overflow: auto;
overflow: scroll;
```

---

## 10.5. CSS specificity

### Задача

Какой цвет будет?

```css
button {
  color: blue;
}

.primary {
  color: red;
}

#submit {
  color: green;
}
```

```html
<button id="submit" class="primary">
  Submit
</button>
```

### Ответ

```txt
green
```

`id` имеет большую specificity, чем class и tag selector.

---

# 11. Accessibility-задачи

## 11.1. Accessible button

Плохо:

```tsx
<div onClick={onClick}>Click</div>
```

Лучше:

```tsx
<button type="button" onClick={onClick}>
  Click
</button>
```

### Почему

`button` из коробки поддерживает:

```txt
keyboard navigation
Enter/Space
focus
role
disabled
screen readers
```

---

## 11.2. Accessible input

```tsx
<label htmlFor="email">
  Email
</label>

<input
  id="email"
  type="email"
  value={email}
  onChange={event => setEmail(event.target.value)}
/>
```

---

## 11.3. Accessible modal

Что нужно упомянуть:

```txt
role="dialog"
aria-modal="true"
aria-labelledby
focus trap
restore focus
close on Escape
keyboard navigation
scroll lock
```

---

# 12. Testing задачи

## 12.1. Протестировать кнопку

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(prev => prev + 1)}>
      Count: {count}
    </button>
  );
}
```

### Тест

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('increments count after click', async () => {
  const user = userEvent.setup();

  render(<Counter />);

  const button = screen.getByRole('button', {
    name: /count: 0/i,
  });

  await user.click(button);

  expect(
    screen.getByRole('button', {
      name: /count: 1/i,
    })
  ).toBeInTheDocument();
});
```

### Что проверяют

Тестировать поведение, а не внутреннюю реализацию.

---

## 12.2. Тест async loading

```tsx
test('loads users', async () => {
  render(<UsersPage />);

  expect(screen.getByText(/loading/i)).toBeInTheDocument();

  expect(await screen.findByText('Alex')).toBeInTheDocument();
});
```

### Хорошо сказать

Для API mocking часто используют:

```txt
MSW
jest mocks
vi mocks
```

---

# 13. System design frontend tasks

Это самые важные задачи для Senior.

---

## 13.1. Спроектировать Autocomplete

### Что нужно покрыть

```txt
controlled input
debounce
API request
loading state
error state
empty state
keyboard navigation
highlight selected option
ARIA combobox
cache
race conditions
AbortController
virtualization для длинных списков
click outside
mobile behavior
```

### Компонентная структура

```txt
Autocomplete
  Input
  SuggestionsList
  SuggestionItem
  EmptyState
  LoadingState
  ErrorState
```

### State

```ts
type AutocompleteState<T> = {
  query: string;
  options: T[];
  selectedIndex: number;
  isOpen: boolean;
  isLoading: boolean;
  error: string | null;
};
```

### Основные события

```txt
onChange input
onFocus
onBlur / click outside
ArrowDown
ArrowUp
Enter
Escape
select option
```

### Подводные камни

```txt
старый request пришёл позже нового
слишком много запросов
не работает клавиатура
нет aria-атрибутов
список слишком большой
focus теряется при клике
```

---

## 13.2. Спроектировать Data Table

### Требования

```txt
columns config
sorting
filtering
pagination
row selection
loading state
empty state
error state
server-side mode
client-side mode
column resizing
sticky header
virtualization
URL sync
accessibility
```

### Возможный API

```tsx
<DataTable
  columns={columns}
  data={users}
  rowKey={user => user.id}
  sorting={sorting}
  onSortingChange={setSorting}
  pagination={pagination}
  onPaginationChange={setPagination}
/>
```

### Columns

```ts
type Column<T> = {
  id: string;
  header: string;
  accessor: (row: T) => React.ReactNode;
  sortable?: boolean;
  width?: number;
};
```

### Senior-ответ

Нужно разделить:

```txt
headless logic
UI rendering
state management
server integration
accessibility
performance
```

Хороший подход — использовать headless table engine или написать похожую архитектуру.

---

## 13.3. Спроектировать Notification system

### Требования

```txt
show success/error/warning/info notification
auto-dismiss
manual close
queue
max visible count
position
animations
accessibility
global API
```

### State

```ts
type Notification = {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  title?: string;
  message: string;
  duration?: number;
};
```

### Context API

```tsx
type NotificationsContextValue = {
  notify: (notification: Omit<Notification, 'id'>) => void;
  close: (id: string) => void;
};

const NotificationsContext =
  React.createContext<NotificationsContextValue | null>(null);
```

### Provider

```tsx
function NotificationsProvider({ children }: { children: React.ReactNode }) {
  const [notifications, setNotifications] = useState<Notification[]>([]);

  const notify = useCallback((notification: Omit<Notification, 'id'>) => {
    setNotifications(prev => [
      ...prev,
      {
        ...notification,
        id: crypto.randomUUID(),
      },
    ]);
  }, []);

  const close = useCallback((id: string) => {
    setNotifications(prev =>
      prev.filter(notification => notification.id !== id)
    );
  }, []);

  return (
    <NotificationsContext.Provider value={{ notify, close }}>
      {children}

      <div className="notifications">
        {notifications.map(notification => (
          <NotificationItem
            key={notification.id}
            notification={notification}
            onClose={() => close(notification.id)}
          />
        ))}
      </div>
    </NotificationsContext.Provider>
  );
}
```

---

## 13.4. Спроектировать Auth flow

### Что обсудить

```txt
login
logout
refresh token
access token
protected routes
role-based access
401 handling
403 handling
token expiration
silent refresh
storage strategy
CSRF/XSS risks
```

### Хороший ответ про хранение токенов

```txt
localStorage удобен, но уязвим к XSS
httpOnly cookie защищает от доступа JS, но требует CSRF-защиты
memory storage безопаснее от persistence, но теряется при refresh
```

### Protected route

```tsx
function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { user, isLoading } = useAuth();

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" replace />;
  }

  return children;
}
```

---

# 14. Архитектурные задачи

## 14.1. Как организовать структуру React-приложения

### Простая feature-based структура

```txt
src/
  app/
    providers/
    router/
    store/
  features/
    auth/
    user-search/
    notifications/
  shared/
    ui/
    api/
    lib/
    hooks/
    types/
```

### Что хорошо объяснить

```txt
shared не должен зависеть от features
features могут использовать shared
app собирает providers/router/config
не делать огромную папку components
не смешивать бизнес-логику и UI
```

---

## 14.2. Как работать с legacy codebase

Хороший план:

```txt
1. Найти критичные user flows.
2. Покрыть их тестами.
3. Включить linting/formatting.
4. Постепенно добавить TypeScript.
5. Выделить API layer.
6. Разделить большие компоненты.
7. Удалить dead code.
8. Улучшить error handling.
9. Внедрять изменения постепенно.
```

### Важная мысль

Не переписывать всё сразу без бизнес-причины.

---

# 15. Practical React coding tasks

## 15.1. Todo List

### Требования

```txt
добавить todo
удалить todo
toggle completed
filter all/active/completed
счётчик оставшихся
```

### Решение

```tsx
type Todo = {
  id: string;
  text: string;
  completed: boolean;
};

type Filter = 'all' | 'active' | 'completed';

function TodoApp() {
  const [text, setText] = useState('');
  const [todos, setTodos] = useState<Todo[]>([]);
  const [filter, setFilter] = useState<Filter>('all');

  function addTodo(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();

    const trimmedText = text.trim();

    if (!trimmedText) return;

    setTodos(prev => [
      ...prev,
      {
        id: crypto.randomUUID(),
        text: trimmedText,
        completed: false,
      },
    ]);

    setText('');
  }

  function toggleTodo(id: string) {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  }

  function removeTodo(id: string) {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  }

  const filteredTodos = todos.filter(todo => {
    if (filter === 'active') {
      return !todo.completed;
    }

    if (filter === 'completed') {
      return todo.completed;
    }

    return true;
  });

  const activeCount = todos.filter(todo => !todo.completed).length;

  return (
    <div>
      <form onSubmit={addTodo}>
        <input
          value={text}
          onChange={event => setText(event.target.value)}
          placeholder="New todo"
        />

        <button type="submit">Add</button>
      </form>

      <div>
        <button onClick={() => setFilter('all')}>All</button>
        <button onClick={() => setFilter('active')}>Active</button>
        <button onClick={() => setFilter('completed')}>Completed</button>
      </div>

      <ul>
        {filteredTodos.map(todo => (
          <li key={todo.id}>
            <label>
              <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => toggleTodo(todo.id)}
              />

              {todo.text}
            </label>

            <button onClick={() => removeTodo(todo.id)}>
              Remove
            </button>
          </li>
        ))}
      </ul>

      <p>Active: {activeCount}</p>
    </div>
  );
}
```

### Что проверяют

```txt
state updates
immutability
controlled form
derived state
keys
basic architecture
```

---

## 15.2. Tabs component

```tsx
type Tab = {
  id: string;
  label: string;
  content: React.ReactNode;
};

type TabsProps = {
  tabs: Tab[];
  defaultTabId?: string;
};

function Tabs({ tabs, defaultTabId }: TabsProps) {
  const [activeTabId, setActiveTabId] = useState(
    defaultTabId ?? tabs[0]?.id
  );

  const activeTab = tabs.find(tab => tab.id === activeTabId);

  return (
    <div>
      <div role="tablist">
        {tabs.map(tab => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={tab.id === activeTabId}
            onClick={() => setActiveTabId(tab.id)}
          >
            {tab.label}
          </button>
        ))}
      </div>

      <div role="tabpanel">
        {activeTab?.content}
      </div>
    </div>
  );
}
```

### Senior-улучшения

```txt
keyboard navigation
controlled/uncontrolled API
URL sync
lazy mount panels
aria-controls
id links
```

---

## 15.3. Accordion

```tsx
type AccordionItem = {
  id: string;
  title: string;
  content: React.ReactNode;
};

function Accordion({ items }: { items: AccordionItem[] }) {
  const [openedId, setOpenedId] = useState<string | null>(null);

  function toggle(id: string) {
    setOpenedId(prev => prev === id ? null : id);
  }

  return (
    <div>
      {items.map(item => {
        const isOpen = openedId === item.id;

        return (
          <section key={item.id}>
            <button
              type="button"
              aria-expanded={isOpen}
              onClick={() => toggle(item.id)}
            >
              {item.title}
            </button>

            {isOpen && (
              <div>
                {item.content}
              </div>
            )}
          </section>
        );
      })}
    </div>
  );
}
```

---

# 16. Security задачи

## 16.1. XSS

### Вопрос

Что опасного здесь?

```tsx
<div dangerouslySetInnerHTML={{ __html: userContent }} />
```

### Ответ

Если `userContent` содержит вредоносный HTML/JS, можно получить XSS.

### Защита

```txt
не использовать dangerouslySetInnerHTML без необходимости
sanitize HTML
escape user content
CSP
не хранить sensitive tokens в localStorage при высоком XSS risk
```

---

## 16.2. CSRF

### Что сказать

CSRF актуален, когда auth основан на cookies, автоматически отправляемых браузером.

Защита:

```txt
SameSite cookies
CSRF token
Origin/Referer validation
double submit cookie pattern
```

---

# 17. Browser API задачи

## 17.1. IntersectionObserver для infinite scroll

```tsx
function useInfiniteScroll(
  callback: () => void,
  options?: IntersectionObserverInit
) {
  const ref = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    const element = ref.current;

    if (!element) return;

    const observer = new IntersectionObserver(entries => {
      const firstEntry = entries[0];

      if (firstEntry.isIntersecting) {
        callback();
      }
    }, options);

    observer.observe(element);

    return () => {
      observer.disconnect();
    };
  }, [callback, options]);

  return ref;
}
```

### Подводный камень

`callback` и `options` должны быть стабильными, иначе observer будет пересоздаваться.

---

## 17.2. Click outside hook

```tsx
function useClickOutside<T extends HTMLElement>(
  onClickOutside: () => void
) {
  const ref = useRef<T | null>(null);

  useEffect(() => {
    function handleMouseDown(event: MouseEvent) {
      const element = ref.current;

      if (!element) return;

      if (!element.contains(event.target as Node)) {
        onClickOutside();
      }
    }

    document.addEventListener('mousedown', handleMouseDown);

    return () => {
      document.removeEventListener('mousedown', handleMouseDown);
    };
  }, [onClickOutside]);

  return ref;
}
```

---

# 18. Частые live coding задачи по сложности

## Junior+/Middle

```txt
Counter
Todo List
Tabs
Accordion
Modal
Dropdown
Search with debounce
Form validation
Pagination
Sort/filter list
```

## Strong Middle

```txt
Autocomplete
Infinite scroll
Data table
useDebounce
usePrevious
useClickOutside
useLocalStorage
Race condition handling
React.memo optimization
Context provider
```

## Senior

```txt
Design system component API
Polymorphic components
Virtualized list
Complex form builder
Frontend auth architecture
Notification system
Permission-based rendering
Error boundary strategy
Microfrontend trade-offs
Performance audit
Legacy refactoring strategy
```

---

# 19. Что особенно важно показать на собеседовании

Не просто написать код, а проговаривать:

```txt
какие edge cases есть
какая сложность решения
как избежать race condition
как чистить side effects
как обеспечить accessibility
как типизировать API
как тестировать
как улучшить performance
какие trade-offs у решения
```

---

# 20. Самые вероятные задачи для React FE 3–5 лет

Если приоритизировать, я бы готовил в таком порядке:

## 1. JavaScript output questions

```txt
event loop
closures
this
promises
async/await
hoisting
immutability
```

## 2. React hooks

```txt
useEffect dependencies
stale closure
functional setState
custom hooks
useRef vs useState
useMemo/useCallback
```

## 3. Практические компоненты

```txt
Modal
Dropdown
Tabs
Accordion
Todo List
Autocomplete
Search input
Data table
```

## 4. API state

```txt
loading/error/data
AbortController
race conditions
debounce
pagination
infinite scroll
```

## 5. TypeScript

```txt
props typing
generics
union types
discriminated unions
utility types
polymorphic components
```

## 6. Performance

```txt
React.memo
useMemo
useCallback
virtualization
bundle splitting
render profiling
```

## 7. Architecture

```txt
project structure
state management choice
server state vs client state
legacy refactoring
design system
testing strategy
```

---

# 21. Мини-набор задач, который стоит уметь написать с нуля

Вот что я бы обязательно натренировал перед интервью:

```txt
1. debounce function
2. throttle function
3. useDebounce
4. usePrevious
5. useClickOutside
6. Modal with portal
7. Search with debounce + AbortController
8. Todo list with filters
9. Autocomplete with keyboard navigation
10. Build tree from flat array
11. groupBy
12. deepClone
13. React.memo optimization example
14. ErrorBoundary
15. Data table design
```

---

# 22. Как отвечать, чтобы звучать как Middle/Senior

Плохой ответ:

> Я бы сделал useEffect и fetch.

Хороший ответ:

> Я бы сделал controlled input, добавил debounce, чтобы не стрелять запросом на каждый символ. Для запроса использовал бы AbortController или request id, чтобы избежать race condition. Отдельно обработал бы loading, error и empty states. Для accessibility добавил бы label, keyboard navigation и aria-атрибуты. Если список большой — добавил бы virtualization. API-вызовы я бы вынес в отдельный слой или использовал React Query.

Именно такой уровень детализации обычно отличает сильного Middle от Senior.
