# Техническое интервью: Lesta Games — Frontend Developer (React)
### Вопросы + развёрнутые ответы

> Разбито по темам. Каждый вопрос — с полным ответом, который можно адаптировать под разговор. В конце — задачи на живое кодирование.

---

## 1. React — Основы и внутреннее устройство

---

**Q: Как работает Virtual DOM и зачем он нужен?**

React хранит в памяти лёгкое представление DOM-дерева — Virtual DOM. При изменении состояния React строит новое виртуальное дерево и сравнивает его со старым через алгоритм reconciliation (diffing). Находит минимальный набор изменений и применяет их к реальному DOM — это дешевле, чем перестраивать весь DOM напрямую.

Важные детали алгоритма: React сравнивает деревья по уровням, а не рекурсивно через всё дерево. Если типы элементов различаются — React уничтожает старое поддерево и строит новое. Поэтому `key` на списках критичен: без него React не знает, какой элемент переместился, а какой — новый.

Начиная с React 16, reconciler называется Fiber. Он разбивает рендеринг на единицы работы и умеет прерываться между ними — это основа для Concurrent Mode, Suspense и Transitions.

---

**Q: Что такое React Fiber и что он изменил по сравнению со старым reconciler?**

Старый reconciler (Stack) был синхронным: начав обходить дерево, он не мог остановиться до завершения. При большом дереве это блокировало main thread и вызывало подвисания UI.

Fiber переписал reconciler как список связанных объектов — fiber nodes. Каждый fiber — это единица работы. Reconciliation теперь прерываемый: React может остановиться, отдать управление браузеру (например, для обработки ввода пользователя), и продолжить позже.

Это дало возможность:
- **Concurrent Mode** — рендерить несколько версий дерева одновременно
- **Suspense** — приостанавливать рендер до готовности данных
- **useTransition / startTransition** — помечать обновления как «не срочные», чтобы React мог приоритизировать важные изменения (например, ввод в поле) над менее важными (обновление списка)

---

**Q: В чём разница между контролируемым и неконтролируемым компонентом?**

**Контролируемый**: значение поля хранится в state React, каждый keystroke вызывает `onChange` и обновляет state. React полностью владеет значением. Плюсы: синхронизация с другими частями UI, валидация в реальном времени, простое тестирование. Минусы: при каждом вводе — ре-рендер компонента.

```tsx
const [value, setValue] = useState('');
<input value={value} onChange={e => setValue(e.target.value)} />
```

**Неконтролируемый**: значение живёт в DOM, React обращается к нему через `ref` только когда нужно (submit, blur). Плюсы: меньше ре-рендеров, ближе к нативному поведению. Минусы: сложнее синхронизировать с другими частями UI.

```tsx
const ref = useRef<HTMLInputElement>(null);
<input ref={ref} defaultValue="initial" />
// читаем: ref.current?.value
```

React Hook Form по умолчанию работает с неконтролируемыми компонентами — именно поэтому он такой быстрый на больших формах.

---

**Q: Что такое reconciliation и как key влияет на него?**

Reconciliation — процесс сравнения старого и нового виртуального дерева для определения минимального набора изменений в реальном DOM.

Правила алгоритма:
1. Если изменился тип элемента (`div` → `span`) — старое поддерево уничтожается, монтируется новое
2. Если тип не изменился — React обновляет пропсы существующего узла
3. При рендере списков React использует `key` для отслеживания идентичности элементов

Без `key` (или с `key={index}`): если список переупорядочивается, React не понимает, что элемент сдвинулся, и перестраивает его с нуля вместо перемещения. Это дорого и приводит к потере локального состояния компонентов в списке.

Правильный `key` — стабильный уникальный идентификатор из данных (id из БД), не индекс массива.

---

**Q: Объясни разницу между useMemo и useCallback. Когда использовать каждый?**

`useMemo` мемоизирует **результат вычисления**. Пересчитывает только при изменении зависимостей.

```tsx
const sorted = useMemo(() => [...items].sort(compareFn), [items]);
```

`useCallback` мемоизирует **саму функцию**. Возвращает ту же ссылку на функцию между рендерами, пока зависимости не изменились.

```tsx
const handleClick = useCallback(() => doSomething(id), [id]);
```

**Когда использовать:**
- `useMemo` — для дорогих вычислений (сортировка/фильтрация большого массива, сложные трансформации данных)
- `useCallback` — когда функцию передаёшь в `React.memo`-обёрнутый дочерний компонент или в зависимости другого хука

**Когда НЕ использовать:** на каждой функции и переменной подряд — это антипаттерн. Мемоизация не бесплатна: React всё равно запускает хук, сравнивает зависимости, хранит кэш. Для простых примитивных значений это только накладные расходы.

---

**Q: Что такое React.memo и когда он реально помогает?**

`React.memo` — HOC, который оборачивает компонент и пропускает ре-рендер, если пропсы не изменились (поверхностное сравнение по ссылке).

```tsx
const UserCard = React.memo(({ user }: { user: User }) => {
  return <div>{user.name}</div>;
});
```

**Помогает**, когда:
- Компонент рендерится часто из-за ре-рендера родителя
- Вычисления внутри компонента нетривиальны
- Пропсы — примитивы или стабильные объекты (через useMemo/useCallback)

**Не помогает**, когда:
- В пропсах передаются новые объекты/функции на каждый рендер — сравнение всегда false
- Компонент и так рендерится редко
- Компонент простой — overhead от сравнения сопоставим с ценой рендера

Для кастомного сравнения — второй аргумент `areEqual(prevProps, nextProps)`.

---

**Q: Как устроен хук useEffect? Что такое cleanup и зачем он нужен?**

`useEffect` запускает side effect после рендера. Принимает функцию и массив зависимостей.

```tsx
useEffect(() => {
  const subscription = dataStream.subscribe(handler);
  return () => subscription.unsubscribe(); // cleanup
}, [dataStream]);
```

**Фазы:**
1. React рендерит компонент
2. Обновляет DOM
3. Запускает cleanup предыдущего эффекта (если был)
4. Запускает новый эффект

**Cleanup** нужен для:
- Отписки от событий/стримов (иначе memory leak)
- Отмены fetch-запросов (иначе setState на unmounted компонент)
- Очистки таймеров и интервалов

**Зависимости:**
- `[]` — только при mount/unmount
- `[dep]` — при изменении dep
- без массива — при каждом рендере (почти всегда ошибка)

В React 18 Strict Mode эффекты намеренно запускаются дважды в dev-режиме — чтобы поймать ошибки с отсутствующим cleanup.

---

**Q: Что такое useLayoutEffect и чем отличается от useEffect?**

`useLayoutEffect` запускается **синхронно** после DOM-мутаций, но **до** того как браузер отрисует пиксели. `useEffect` — асинхронно после отрисовки.

```
Рендер → DOM update → useLayoutEffect → [браузер рисует] → useEffect
```

**Когда нужен useLayoutEffect:**
- Измерение DOM-элементов (getBoundingClientRect) и немедленное изменение их позиции/размера
- Предотвращение визуального мерцания при изменении layout

**Риски:** блокирует отрисовку браузера — если там тяжёлые вычисления, пользователь увидит подвисание. Используй только когда `useEffect` даёт видимое мерцание.

На сервере (SSR) `useLayoutEffect` не работает — React выбрасывает warning. Для универсального кода используй `useIsomorphicLayoutEffect` — кастомный хук, который на сервере подставляет `useEffect`.

---

**Q: Что такое useTransition и useDeferredValue? Чем они различаются?**

Оба появились в React 18 и решают одну задачу: отделить «срочные» обновления от «несрочных», чтобы не блокировать ввод пользователя.

**`useTransition`** — оборачивает сам вызов setState и помечает его как низкоприоритетный. React выполнит его, когда main thread свободен.

```tsx
const [isPending, startTransition] = useTransition();

function handleSearch(query: string) {
  setInputValue(query);           // срочно — обновить поле ввода сразу
  startTransition(() => {
    setFilteredList(filter(list, query)); // несрочно — дорогая фильтрация
  });
}

return (
  <>
    <input value={inputValue} onChange={e => handleSearch(e.target.value)} />
    {isPending ? <Spinner /> : <List items={filteredList} />}
  </>
);
```

**`useDeferredValue`** — «откладывает» уже существующее значение. Полезно, когда не контролируешь, где вызывается setState (например, значение приходит из пропсов).

```tsx
const deferredQuery = useDeferredValue(query); // query обновляется сразу, deferredQuery — с задержкой

const filtered = useMemo(() => filter(list, deferredQuery), [list, deferredQuery]);
```

**Разница:**
- `useTransition` — ты контролируешь что именно «несрочно» и получаешь `isPending`
- `useDeferredValue` — откладываешь значение которое уже есть, без `isPending`

**Подводный камень:** оба не гарантируют конкретный таймаут. React откладывает настолько, насколько нужно. Если список маленький — разницы нет вообще. Используй только при реальной проблеме с производительностью, подтверждённой профилировщиком.

---

**Q: Что нового в React 18 и как Automatic Batching изменил поведение?**

До React 18 batching (объединение нескольких setState в один ре-рендер) работал только внутри синтетических обработчиков событий React. В async коде каждый setState вызывал отдельный ре-рендер.

```tsx
// До React 18 — два ре-рендера в setTimeout
setTimeout(() => {
  setCount(c => c + 1); // ре-рендер
  setFlag(f => !f);     // ещё ре-рендер
}, 1000);

// React 18 — один ре-рендер (Automatic Batching)
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);     // оба объединяются → один ре-рендер
}, 1000);
```

То же самое теперь работает в fetch callbacks, Promise.then, нативных обработчиках событий.

**Если нужно отключить batching** (редко): `flushSync` из `react-dom`:
```tsx
import { flushSync } from 'react-dom';

flushSync(() => setCount(c => c + 1)); // немедленный ре-рендер
flushSync(() => setFlag(f => !f));     // ещё один немедленный ре-рендер
```

**Другие изменения React 18:**
- `createRoot` вместо `ReactDOM.render` — обязательно для Concurrent features
- `useId` — стабильные уникальные ID для SSR без гидрации-гонок
- `useSyncExternalStore` — для подписки на внешние сторы совместимо с Concurrent Mode
- Strict Mode двойной вызов эффектов — намеренно, чтобы поймать non-idempotent cleanup

---

**Q: Что такое React Server Components (RSC)? Чем отличаются от SSR?**

RSC — компоненты, которые рендерятся **только на сервере** и никогда не попадают в клиентский bundle. Они могут напрямую обращаться к БД, файловой системе, внутренним API — без лишнего round-trip.

```tsx
// app/users/page.tsx — Server Component по умолчанию в Next.js 14+
async function UsersPage() {
  const users = await db.user.findMany(); // прямой запрос к БД, нет API слоя
  return <UserList users={users} />;
}
```

**RSC vs SSR:**

| | SSR | RSC |
|--|--|--|
| Где рендерится | Сервер (при каждом запросе) | Сервер (может кэшироваться) |
| JS в bundle | Да — компонент hydrate-ится на клиенте | Нет — в bundle не попадает |
| Интерактивность | Есть после гидрации | Нет (только статический вывод) |
| Доступ к данным | Через API | Напрямую (БД, fs, secrets) |

**Когда НЕ использовать RSC:**
- Нужны useState, useEffect, обработчики событий → `'use client'`
- Нужен доступ к браузерным API (window, localStorage)

**Главный подводный камень:** граница `'use client'` создаёт отдельный «бандл». Всё что импортируется из Client Component — тоже попадает в bundle. Серверные компоненты нельзя импортировать внутри клиентских — только передавать через пропсы/children.

---

**Q: Что такое Suspense и как он работает с data fetching?**

`Suspense` — механизм, позволяющий компоненту «приостановить» рендеринг, пока не будут готовы данные, и показать fallback.

```tsx
<Suspense fallback={<Skeleton />}>
  <UserProfile userId={id} /> {/* может «приостановиться» }
</Suspense>
```

Компонент приостанавливается, бросая Promise. React ловит его, рендерит fallback, и когда Promise резолвится — повторяет рендер компонента.

**С React 18 + Next.js App Router:**
```tsx
// Серверный компонент — await встроен
async function UserProfile({ userId }: { userId: string }) {
  const user = await fetchUser(userId); // Suspense boundary выше покажет fallback пока ждём
  return <div>{user.name}</div>;
}
```

**С TanStack Query (клиентский):**
```tsx
const { data } = useSuspenseQuery({
  queryKey: ['user', id],
  queryFn: () => fetchUser(id),
});
// не нужен if (isLoading) — Suspense boundary выше обрабатывает
```

**Подводные камни:**
- Waterfall-запросы: если несколько Suspense-компонентов вложены, они ждут по очереди. Решение: `Promise.all` или parallel данные через один fetch.
- Suspense не ловит ошибки — для этого нужен `ErrorBoundary` рядом.
- На клиенте без специальной поддержки (useSuspenseQuery, `use(promise)`) работать не будет.

---

**Q: Что такое хук use() в React 19 и как он изменяет работу с промисами?**

`use()` — новый хук в React 19, который позволяет читать значение из Promise или Context внутри компонента.

```tsx
// React 19 — читаем Promise прямо в компоненте
import { use } from 'react';

function UserCard({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // приостанавливает компонент если Promise не resolved
  return <div>{user.name}</div>;
}

// Родитель создаёт Promise и передаёт вниз
function Page() {
  const userPromise = fetchUser(id); // создаётся вне компонента, чтобы не пересоздаваться
  return (
    <Suspense fallback={<Skeleton />}>
      <UserCard userPromise={userPromise} />
    </Suspense>
  );
}
```

**Отличие от await в Server Components:**
- `use()` работает на клиенте внутри обычных компонентов
- Можно использовать условно (в отличие от хуков — это не «хук» в привычном смысле)
- Также работает с Context: `const theme = use(ThemeContext)` — аналог useContext

**Подводный камень:** Promise нужно создавать вне компонента или мемоизировать. Если создавать при каждом рендере — будет бесконечный цикл (новый Promise = новое ожидание = ре-рендер = новый Promise).

---

**Q: Что такое useOptimistic в React 19?**

`useOptimistic` — хук для оптимистичных обновлений: показывает пользователю результат действия немедленно, не дожидаясь ответа сервера. Если запрос упадёт — откатывает к реальному значению.

```tsx
function LikeButton({ postId, initialLikes }: { postId: string; initialLikes: number }) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    initialLikes,
    (current, increment: number) => current + increment
  );

  async function handleLike() {
    addOptimisticLike(1); // UI обновляется немедленно
    await likePost(postId); // реальный запрос
    // если упадёт — React автоматически вернёт initialLikes
  }

  return (
    <button onClick={handleLike}>
      ❤️ {optimisticLikes}
    </button>
  );
}
```

До React 19 это делали вручную: обновляли локальный state, затем инвалидировали кэш TanStack Query при ошибке. Теперь это встроено в React.

---

**Q: Что такое React Actions и как они работают в React 19?**

Actions — паттерн для обработки мутаций с встроенной поддержкой pending/error state. Используется вместе с `useActionState` и `<form action={...}>`.

```tsx
// React 19
async function createUser(prevState: State, formData: FormData) {
  const name = formData.get('name') as string;
  try {
    await db.user.create({ data: { name } });
    return { success: true, error: null };
  } catch {
    return { success: false, error: 'Failed to create user' };
  }
}

function UserForm() {
  const [state, formAction, isPending] = useActionState(createUser, {
    success: false,
    error: null,
  });

  return (
    <form action={formAction}>
      <input name="name" />
      <button disabled={isPending}>
        {isPending ? 'Creating...' : 'Create'}
      </button>
      {state.error && <p>{state.error}</p>}
    </form>
  );
}
```

Это убирает необходимость в ручном `useState` для loading/error на каждую форму.

---

**Q: Подводные камни замыканий в хуках — stale closure**

Одна из самых частых ошибок. Хук захватывает значение переменной в момент создания, и если зависимости не указаны — работает со старым значением.

```tsx
// ОШИБКА — stale closure
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // всегда читает count = 0 из замыкания
    }, 1000);
    return () => clearInterval(id);
  }, []); // пустой массив — захвачено начальное значение

  return <div>{count}</div>; // будет прыгать между 0 и 1
}

// ПРАВИЛЬНО — функциональное обновление или добавить count в deps
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1); // читаем актуальное значение через updater function
  }, 1000);
  return () => clearInterval(id);
}, []);
```

**Другой вариант — useRef для «живого» значения:**
```tsx
const countRef = useRef(count);
useEffect(() => { countRef.current = count; }, [count]);

useEffect(() => {
  const id = setInterval(() => {
    console.log(countRef.current); // всегда актуально
  }, 1000);
  return () => clearInterval(id);
}, []);
```

---

**Q: Почему нельзя вызывать хуки условно и что происходит если нарушить это правило?**

React идентифицирует хуки по порядку вызова, а не по имени. Внутри компонента React хранит «список хуков» — при каждом рендере хуки должны вызываться в том же порядке.

```tsx
// НЕЛЬЗЯ
function Component({ showExtra }: { showExtra: boolean }) {
  const [a, setA] = useState(0);
  if (showExtra) {
    const [b, setB] = useState(0); // ❌ нарушает порядок
  }
  const [c, setC] = useState(0);
}

// Что произойдёт: при первом рендере (showExtra=true) хуки: [a, b, c]
// При следующем (showExtra=false): [a, c] — React попытается сопоставить
// стейт для 'c' со слотом, где был 'b' → неправильное значение или краш
```

**Правило:** всегда вызывай хуки на верхнем уровне компонента, никогда внутри условий, циклов, вложенных функций. Если нужна условная логика — выноси в отдельный компонент.

---

**Q: Подводные камни Object identity в зависимостях useEffect**

```tsx
// ПРОБЛЕМА — бесконечный цикл
function Component({ userId }: { userId: string }) {
  const options = { includeDeleted: false }; // новый объект при каждом рендере

  useEffect(() => {
    fetchUser(userId, options); // options — новая ссылка каждый раз
  }, [userId, options]); // React видит изменение → запускает эффект → ре-рендер → ...
}

// РЕШЕНИЕ 1 — вынести объект за пределы компонента (если не зависит от props/state)
const DEFAULT_OPTIONS = { includeDeleted: false };

// РЕШЕНИЕ 2 — useMemo
const options = useMemo(() => ({ includeDeleted: false }), []);

// РЕШЕНИЕ 3 — примитивные зависимости вместо объекта
useEffect(() => {
  fetchUser(userId, { includeDeleted: false });
}, [userId]); // только примитив
```

То же самое с функциями — каждый рендер создаёт новую ссылку на инлайн-функцию. Если передаёшь функцию в deps — используй `useCallback`.

---

**Q: Как правильно типизировать ref в TypeScript и какие бывают виды ref?**

```tsx
// 1. Ref на DOM-элемент — начальное значение null, тип элемента
const inputRef = useRef<HTMLInputElement>(null);
// inputRef.current — HTMLInputElement | null

// 2. Mutable ref для хранения значения (не вызывает ре-рендер)
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
timerRef.current = setTimeout(() => {}, 1000);

// 3. forwardRef — пробрасываем ref из родителя в дочерний компонент
const FancyInput = forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  return <input ref={ref} {...props} />;
});

// 4. useImperativeHandle — контролируем что видит родитель через ref
const VideoPlayer = forwardRef<VideoAPI, VideoProps>((props, ref) => {
  const videoRef = useRef<HTMLVideoElement>(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
    // родитель не видит сам videoRef.current — только этот API
  }));

  return <video ref={videoRef} {...props} />;
});
```

**Подводный камень:** `useRef<T>(null)` и `useRef<T | null>(null)` — разные типы. Первый говорит TypeScript «это ref для DOM, будет заполнен React'ом» → `current` readonly. Второй — mutable ref → `current` можно менять.

---

**Q: Чем React отличается от Vue? Сравни подходы к реактивности и шаблонизации.**

Ключевое различие — **как обнаруживаются изменения**:

**React** — pull-based с явным обновлением. `setState` инициирует пересчёт поддерева через Virtual DOM diffing. React не знает что именно изменилось — просто перезапускает функцию компонента и сравнивает вывод.

**Vue** — push-based с реактивной системой. Vue 3 использует `Proxy` для отслеживания зависимостей: при обращении к `ref.value` внутри computed/watch Vue запоминает подписку. При изменении — точечно уведомляет только тех, кто зависит от этого значения.

```tsx
// React — явный стейт, вся функция перезапускается
function Counter() {
  const [count, setCount] = useState(0);
  // при setCount вся функция выполняется заново
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

```vue
<!-- Vue 3 — реактивный ref, обновляется только то, что читает count.value -->
<script setup>
const count = ref(0);
</script>
<template>
  <button @click="count++">{{ count }}</button>
</template>
```

**Шаблонизация:**
| | React (JSX) | Vue (SFC) |
|---|---|---|
| Синтаксис | JS + HTML в одном файле | Отдельные `<template>`, `<script>`, `<style>` |
| Выражения | Полный JS в `{}` | Ограниченные выражения в `{{ }}`, директивы `v-if`, `v-for` |
| Условная логика | `&&`, тернар, if вне JSX | `v-if` / `v-show` |
| Стили | CSS-in-JS, CSS Modules | `<style scoped>` встроен |
| Compile-time | React Compiler (опционально) | Шаблон компилируется в render-функцию всегда |

**Что лучше зависит от контекста:**
- Vue — меньше boilerplate, понятнее для новичков, встроенная реактивность из коробки
- React — больше экосистема, гибче архитектура, JSX = полный JS без ограничений директив

---

**Q: Как работает SolidJS? Почему он быстрее React и чем принципиально отличается?**

SolidJS отказался от Virtual DOM полностью. Вместо диффинга — **точечные DOM-обновления** на основе реактивных примитивов.

**Ключевые отличия:**

**1. Компоненты выполняются один раз**
```tsx
// React — функция вызывается при каждом рендере
function ReactCounter() {
  const [count, setCount] = useState(0);
  console.log('render'); // выводится при каждом обновлении
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// SolidJS — функция выполняется ОДИН раз, только count() в JSX — реактивный
function SolidCounter() {
  const [count, setCount] = createSignal(0);
  console.log('setup'); // выводится ОДИН раз при монтировании
  return <button onClick={() => setCount(c => c + 1)}>{count()}</button>;
  //                                                    ^— вызов как функция, не значение
}
```

**2. Fine-grained reactivity — обновляется только то, что изменилось**
```tsx
// SolidJS — при изменении count обновляется только текстовый узел кнопки
// React — перезапускается вся функция компонента и diffing через VDOM
```

**3. Нет stale closure проблем** — в SolidJS сигналы читаются в момент рендера DOM, не в момент создания функции. `count()` всегда возвращает актуальное значение.

**Почему быстрее:**
- Нет создания/сравнения Virtual DOM объектов
- DOM-операции — только там, где реально изменились данные
- Нет overhead от reconciliation алгоритма
- Компоненты = инициализация, не «рендер-функции»

**Когда выбирать SolidJS:** максимальная производительность важнее зрелости экосистемы. Для игровых дашбордов, real-time визуализации — SolidJS может быть оправдан. Для большинства продуктов экосистема React перевешивает разницу в скорости.

---

**Q: Как работает Angular? Чем его подход к архитектуре отличается от React?**

Angular — **opinionated full-framework** в отличие от React (библиотека). Включает: маршрутизацию, формы, HTTP-клиент, DI, анимации — всё встроено и стандартизировано.

**Ключевые архитектурные отличия:**

| Аспект | React | Angular |
|---|---|---|
| Тип | Библиотека для UI | Полноценный фреймворк |
| Реактивность | State + Virtual DOM | Change Detection (Zone.js → Signals в v17+) |
| Шаблоны | JSX (JS) | HTML-шаблоны с директивами, двустороннее связывание |
| DI | React Context / сторонние | Встроенный Dependency Injection |
| Состояние | Внешние либы | NgRx (Redux-like), Signals, Сервисы |
| Язык | JS / TS (опционально) | TypeScript обязательно |
| Обучение | Постепенно | Крутая кривая — много концепций сразу |

**Change Detection в Angular:**

До Angular 17: Zone.js патчил все async операции (setTimeout, Promise, DOM events) и при любом событии запускал проверку всего дерева компонентов.

```typescript
// Angular компонент
@Component({
  selector: 'app-counter',
  template: `<button (click)="increment()">{{ count }}</button>`,
  changeDetection: ChangeDetectionStrategy.OnPush, // аналог React.memo
})
export class CounterComponent {
  count = 0;
  increment() { this.count++; }
}
```

**Angular 17+ Signals** — аналог SolidJS-сигналов, избавляет от Zone.js:
```typescript
count = signal(0);
// шаблон автоматически обновляется только при изменении count
```

**DI — главное, что стоит понять:**
```typescript
@Injectable({ providedIn: 'root' }) // singleton на всё приложение
class PlayerService {
  constructor(private http: HttpClient) {} // DI инжектирует HttpClient
  getPlayer(id: string) { return this.http.get<Player>(`/api/players/${id}`); }
}

@Component(...)
class PlayerComponent {
  constructor(private playerService: PlayerService) {} // DI инжектирует сервис
}
```

React не имеет встроенного DI — его роль выполняют Context + кастомные хуки, но без явного контейнера зависимостей.

**Когда Angular выигрывает у React:** крупные enterprise-команды с жёсткими стандартами, где важна консистентность кода без договорённостей — Angular диктует структуру. Для игровой индустрии (Lesta, Wargaming) чаще встречается React из-за гибкости.

---

**Q: Сравни React, Vue, SolidJS, Angular по механизму реактивности — итоговая таблица.**

| | React | Vue 3 | SolidJS | Angular 17+ |
|---|---|---|---|---|
| **Модель реактивности** | Virtual DOM + diffing | Proxy-based signals | Fine-grained signals | Signals / Zone.js |
| **Компонент выполняется** | При каждом ре-рендере | При изменении зависимостей | Один раз | При CD-цикле |
| **Обновление DOM** | Batch через VDOM diff | Точечное через dep tracking | Напрямую, без VDOM | Через CD tree |
| **Стейт** | `useState` / `useReducer` | `ref`, `reactive` | `createSignal` | `signal` / сервисы |
| **Derived state** | `useMemo` | `computed` | `createMemo` | `computed` |
| **Side effects** | `useEffect` | `watchEffect` / `watch` | `createEffect` | `effect` |
| **Условный рендер** | JSX: `&&`, тернар | `v-if` / `v-show` | `<Show>` компонент | `*ngIf` / `@if` |
| **Списки** | `.map()` в JSX | `v-for` | `<For>` компонент | `*ngFor` / `@for` |
| **Размер бандла** | ~42 кБ | ~22 кБ | ~7 кБ | ~130 кБ+ |
| **DI** | Context + хуки | `provide`/`inject` | Context | Встроенный DI |
| **TypeScript** | Опциональный | Опциональный | Опциональный | Обязательный |
| **Экосистема** | Огромная | Большая | Маленькая | Средняя |

**Главный вывод для интервью:** React выбирают не за то, что он быстрее или проще — а за зрелую экосистему, огромное сообщество и предсказуемость однонаправленного потока данных. SolidJS быстрее теоретически, Vue понятнее для старта, Angular — для enterprise с командными стандартами. Для Lesta (высоконагруженные веб-порталы, live-данные) React с Concurrent features — оправданный выбор.

---

## 2. Производительность

---

**Q: Как профилировать React-приложение? Что смотришь первым?**

**Инструменты:**
1. **React DevTools Profiler** — записывает рендеры компонентов: что рендерилось, почему, сколько времени. Ищу компоненты с высоким самостоятельным временем рендера и компоненты, которые рендерятся без изменения пропсов.
2. **Chrome Performance tab** — показывает весь main thread: JS execution, layout, paint, compositing. Ищу долгие задачи (>50ms), layout thrashing (forced reflow в цикле), частые repaint.
3. **Chrome Memory tab** — heap snapshots для поиска memory leaks. Записываю два снапшота с паузой и сравниваю — что осталось в памяти и не было собрано GC.

**Первым смотрю:** Flame graph в Profiler на предмет «толстых» компонентов и ненужных ре-рендеров. Потом Performance tab — есть ли задачи >50ms, которые блокируют ввод пользователя.

---

**Q: Что такое windowing / виртуализация списков и когда её применять?**

Windowing — рендеринг только тех элементов списка, которые видны во viewport. Остальные — не в DOM.

Без виртуализации список из 10 000 элементов создаёт 10 000 DOM-узлов. Это долгая начальная отрисовка, большой memory footprint и медленный скролл.

С виртуализацией в DOM всегда ~20–50 элементов, остальные симулируются высотой контейнера.

**Когда применять:**
- Списки > 100–200 элементов с ожиданием прокрутки
- Таблицы с большим числом строк
- Ленты контента (социальные, история событий)

**Библиотеки:** `react-window` (лёгкий, только виртуализация), `react-virtual` / `@tanstack/virtual` (хук-подход, гибкий), `react-virtuoso` (динамические высоты, sticky headers).

**Важно:** виртуализация усложняет доступность (a11y) и поиск по странице Ctrl+F — учитывай это при принятии решения.

---

**Q: Что такое code splitting и как реализуется в React?**

Code splitting — разбивка бандла на части, которые загружаются по требованию, а не все сразу при открытии страницы.

```tsx
// Без code splitting — всё в одном бандле
import HeavyComponent from './HeavyComponent';

// С code splitting — отдельный chunk, загружается лениво
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

**Стратегии разделения:**
- По маршрутам (самое эффективное — пользователь не загружает код страниц, которые не открывал)
- По компонентам (модальные окна, тяжёлые виджеты, редко используемый функционал)
- По провайдерам/библиотекам (charts, editor, map — часто большие зависимости)

В Next.js код роутов разделяется автоматически. В Vite/Webpack — через dynamic `import()`.

---

**Q: Как предотвратить лишние ре-рендеры в большом дереве компонентов?**

Несколько подходов в связке:

1. **Структура стейта ближе к потребителю** — не поднимай стейт выше, чем нужно. Если только один компонент использует значение — держи его там.

2. **React.memo на листовых компонентах** — оборачивай компоненты, которые часто ре-рендерятся без реальных изменений пропсов.

3. **Стабильные ссылки** — useCallback для функций, useMemo для объектов/массивов, передаваемых вниз.

4. **Разделение Context** — один Context с большим объектом = ре-рендер всех подписчиков при изменении любого поля. Разделяй на несколько контекстов по зонам ответственности (AuthContext, ThemeContext, UIStateContext).

5. **Оптимистичные паттерны с useTransition** — помечай обновления как низкоприоритетные, чтобы React не блокировал ввод пользователя.

6. **Zustand/Jotai вместо Context для частообновляемых данных** — они подписывают компонент только на нужный slice стора, а не на весь объект.

---

**Q: Что такое layout thrashing и как его избежать в React?**

Layout thrashing (forced reflow) — когда JS многократно чередует чтение и запись DOM-свойств в одном синхронном блоке. Каждое чтение после записи заставляет браузер немедленно пересчитать layout.

```js
// Плохо — forced reflow 3 раза
element1.style.width = element2.offsetWidth + 'px'; // read + write
element3.style.width = element4.offsetWidth + 'px'; // read + write
element5.style.width = element6.offsetWidth + 'px'; // read + write

// Хорошо — batch reads, then batch writes
const w2 = element2.offsetWidth;
const w4 = element4.offsetWidth;
const w6 = element6.offsetWidth;
element1.style.width = w2 + 'px';
element3.style.width = w4 + 'px';
element5.style.width = w6 + 'px';
```

В React это обычно возникает в `useLayoutEffect` или в обработчиках событий с прямыми DOM-манипуляциями. Решение: собирать все чтения до записей, или использовать `requestAnimationFrame` для разделения.

---

## 3. Стейт-менеджмент

---

**Q: Когда использовать useState, useReducer, Context, и внешний стор (Zustand/Redux)?**

**useState** — локальное состояние компонента. Простые значения, не нужны другим компонентам.

**useReducer** — сложная логика переходов состояния, несколько связанных значений, которые обновляются вместе. Читаемее, чем цепочка useState, легче тестировать редьюсер отдельно.

**Context** — данные, нужные многим компонентам на разных уровнях (тема, язык, данные пользователя). Не подходит для часто обновляемых данных — все подписчики ре-рендерятся при любом изменении.

**Zustand / Jotai** — глобальный стейт с гранулярными подписками. Компонент подписывается только на нужный slice — нет лишних ре-рендеров. Проще в setup, чем Redux.

**Redux / RTK** — крупные приложения с предсказуемым потоком данных, необходимостью time-travel debugging, сложными selector'ами (reselect), middleware.

**Правило:** начинай с useState. Когда нужно шарить вверх — поднимай. Когда Context даёт проблему с производительностью или сложностью — переходи на Zustand.

---

**Q: Сравни все основные решения для стейт-менеджмента в React — плюсы, минусы, когда что выбирать.**

**Полная таблица сравнения:**

| Решение | Scope | Гранулярность ре-рендеров | DevTools | Bundle | Лучше всего для |
|---|---|---|---|---|---|
| `useState` | Компонент | — (только сам компонент) | React DevTools | 0 кБ | Локальный UI-стейт |
| `useReducer` | Компонент | — | React DevTools | 0 кБ | Сложные переходы состояний |
| `Context` | Поддерево | Плохая — все consumers ре-рендерятся | React DevTools | 0 кБ | Редко меняющиеся глобальные данные |
| **Zustand** | Глобальный | Отличная — по selector'у | Zustand DevTools (Redux DevTools) | ~3 кБ | Глобальный UI-стейт, простые apps |
| **Jotai** | Атом | Отличная — каждый атом отдельно | Jotai DevTools | ~3 кБ | Независимые атомарные куски стейта |
| **Redux RTK** | Глобальный | Хорошая — через reselect | Redux DevTools (полный, time-travel) | ~12 кБ | Большие apps, сложный граф данных |
| **TanStack Query** | Глобальный кэш | Хорошая — по queryKey | TanStack Query DevTools | ~13 кБ | Server state: кэш, loading, sync |
| **XState** | Машина состояний | Отличная | XState Inspector | ~10 кБ | Сложные флоу с явными переходами |

**Дерево решений:**

```
Стейт нужен только одному компоненту?
  └── ДА → useState / useReducer

Стейт нужен нескольким компонентам, редко меняется?
  └── ДА → Context (тема, i18n, данные сессии)

Стейт нужен часто и в разных частях дерева?
  ├── Данные с сервера (загрузка, кэш) → TanStack Query / RTK Query
  ├── UI-стейт (фильтры, модалки) → Zustand
  ├── Много независимых атомарных кусков → Jotai
  ├── Уже есть Redux в проекте → RTK
  └── Сложные флоу (wizard, onboarding) → XState
```

**Главные антипаттерны:**

```tsx
// 1. Server state в Redux — не нужно, для этого TanStack Query
dispatch(fetchUsers()); // ❌ если только данные с API
useQuery({ queryKey: ['users'], queryFn: fetchUsers }); // ✅

// 2. Вычисляемые данные в стейте — хранить только источник
const [items, setItems] = useState([]);
const [count, setCount] = useState(0); // ❌ дублирует items.length

const count = items.length; // ✅ вычисляй на лету

// 3. Context для часто меняющихся данных — все потребители ре-рендерятся
const MouseContext = createContext({ x: 0, y: 0 }); // ❌ 60fps ре-рендеров
// Лучше → Zustand с selector'ом или вообще useRef для данных не для UI

// 4. Один огромный Redux store вместо feature slices
// ❌ store: { app: { auth: {...}, profile: {...}, feed: {...}, settings: {...} } }
// ✅ playerSlice, authSlice, matchSlice — отдельные, независимые
```

**Что я использую в реальных проектах:**
- TanStack Query — всегда, для всего что с сервера
- Zustand — глобальный UI-стейт (opened panels, filters, user preferences)
- useReducer — внутри сложных форм или wizard-флоу внутри одного экрана
- Context — только для инжекции (ThemeProvider, AuthContext с редкими обновлениями)

---

**Q: Как работает useContext? Какие у него проблемы с производительностью?**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

// Провайдер
<ThemeContext.Provider value={theme}>
  {children}
</ThemeContext.Provider>

// Потребитель
const theme = useContext(ThemeContext);
```

**Проблема производительности:** при изменении value все компоненты, вызывающие `useContext(ThemeContext)`, ре-рендерятся — независимо от того, изменилась ли нужная им часть.

Если в Context хранится объект `{ user, settings, notifications }` и меняется только `notifications` — ре-рендерятся и компоненты, которым нужен только `user`.

**Решения:**
1. Разделить Context на независимые контексты (UserContext, SettingsContext, NotificationsContext)
2. Мемоизировать value через useMemo
3. Для часто обновляемых данных — Zustand с селекторами вместо Context

---

**Q: Объясни паттерн «lifting state up» и его ограничения.**

Lifting state up — перемещение общего состояния в ближайшего общего предка компонентов, которым оно нужно.

```
        Parent (state здесь)
       /        \
  Child A      Child B
 (читает)    (изменяет)
```

**Ограничения:**
- **Prop drilling**: если предок далеко вверх, стейт нужно пробрасывать через промежуточные компоненты, которым он не нужен
- **Лишние ре-рендеры**: при изменении стейта ре-рендерится весь поддерево предка
- **Связанность**: компоненты-потребители становятся зависимы от структуры дерева

Для решения prop drilling — Context или внешний стор. Для гранулярного управления ре-рендерами — Zustand.

---

**Q: Чем отличается TanStack Query от Redux RTK Query? Когда выбирать одно, а когда другое?**

Оба управляют server state — кэшируют ответы сервера, управляют состоянием загрузки, дедуплицируют запросы.

**TanStack Query:**
- Стек-независимый (React, Vue, Solid)
- Более богатая функциональность: infinite queries, optimistic updates, background refetch, focus/network-aware refetch
- Легче настроить, меньше boilerplate
- Не привязан к Redux — можно использовать с любым стором

**RTK Query:**
- Встроен в Redux Toolkit — нулевая конфигурация если уже используешь Redux
- Генерирует actions, reducers, hooks автоматически из endpoint-описания
- Имеет доступ к Redux store — можно в cache invalidation логике читать другой стор

**Выбор:**
- Новый проект без Redux → **TanStack Query**
- Уже есть Redux → **RTK Query** — консистентность дороже
- Нужны сложные optimistic updates с rollback → **TanStack Query** более гибкий

---

**Q: Как работает Zustand под капотом? Почему он не вызывает лишних ре-рендеров?**

Zustand хранит стейт вне React (в обычном JS-объекте с замыканием). Компоненты подписываются на него через хук `useStore` и передают **selector** — функцию, которая извлекает нужный кусок стейта.

```tsx
const useStore = create<State>((set) => ({
  count: 0,
  user: null,
  increment: () => set(state => ({ count: state.count + 1 })),
}));

// Компонент подписывается только на count — ре-рендерится только при изменении count
function Counter() {
  const count = useStore(state => state.count); // selector
  const increment = useStore(state => state.increment);
  return <button onClick={increment}>{count}</button>;
}

// Этот компонент НЕ ре-рендерится при изменении count — только при изменении user
function UserDisplay() {
  const user = useStore(state => state.user);
  return <div>{user?.name}</div>;
}
```

Внутри: Zustand слушает изменения стейта и сравнивает результат selector'а до и после. Если результат (по ссылке) не изменился — ре-рендер не происходит. Это кардинально отличается от Context, где ре-рендерятся все подписчики при любом изменении провайдера.

**Подводный камень — selector возвращает объект:**
```tsx
// ПЛОХО — новый объект при каждом обновлении стора = лишние ре-рендеры
const { count, user } = useStore(state => ({ count: state.count, user: state.user }));

// ХОРОШО — отдельные селекторы
const count = useStore(state => state.count);
const user = useStore(state => state.user);

// ИЛИ — shallow comparison через zustand/shallow
import { shallow } from 'zustand/shallow';
const { count, user } = useStore(
  state => ({ count: state.count, user: state.user }),
  shallow // сравнивает поля объекта, а не ссылку
);
```

---

**Q: Объясни разницу между client state и server state. Почему это важно?**

**Client state** — данные, которые существуют только в браузере: открыто ли модальное окно, выбранная вкладка, введённый текст в форме, тема интерфейса. Источник правды — браузер.

**Server state** — данные, живущие на сервере: список пользователей, профиль игрока, история матчей. Браузер только кэширует их копию. Источник правды — сервер.

```tsx
// Client state — Zustand/useState
const isModalOpen = useUIStore(state => state.isModalOpen);

// Server state — TanStack Query
const { data: user } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  staleTime: 5 * 60 * 1000, // считать данные свежими 5 минут
});
```

**Почему важно разделять:**
- Server state требует отдельной логики: кэширование, инвалидация, refetch, pagination, optimistic updates
- Смешивать их в одном Redux-сторе — типичная причина усложнения кода без пользы
- TanStack Query/RTK Query устраняют целый класс багов: гонки запросов, дублированные fetch, устаревший кэш

---

**Q: Как реализовать optimistic update в TanStack Query?**

Оптимистичное обновление — изменить UI немедленно, не дожидаясь сервера. Если запрос падает — откатить.

```tsx
const queryClient = useQueryClient();

const mutation = useMutation({
  mutationFn: (newTodo: Todo) => api.createTodo(newTodo),

  onMutate: async (newTodo) => {
    // 1. Отменяем текущие refetch для todos чтобы не затёрли наш оптимистичный апдейт
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    // 2. Сохраняем текущее состояние кэша для отката
    const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

    // 3. Оптимистично добавляем в кэш
    queryClient.setQueryData<Todo[]>(['todos'], old => [
      ...(old ?? []),
      { ...newTodo, id: 'temp-id', isOptimistic: true },
    ]);

    return { previousTodos }; // контекст для onError
  },

  onError: (err, newTodo, context) => {
    // 4. При ошибке — откатываем к сохранённому состоянию
    queryClient.setQueryData(['todos'], context?.previousTodos);
  },

  onSettled: () => {
    // 5. В любом случае — инвалидируем кэш чтобы получить актуальные данные
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

---

**Q: Как Jotai отличается от Zustand? Когда выбирать атомарный подход?**

**Zustand** — один стор с объектом, компоненты подписываются через селекторы. Хорошо для связанного состояния, которое обновляется вместе.

**Jotai** — атомарный подход: мельчайшие единицы стейта (`atom`), компоненты подписываются на конкретные атомы. Производный стейт — через `derived atoms`.

```tsx
// Jotai
const countAtom = atom(0);
const doubledAtom = atom(get => get(countAtom) * 2); // derived

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const doubled = useAtomValue(doubledAtom); // read-only
  return <button onClick={() => setCount(c => c + 1)}>{count} (x2: {doubled})</button>;
}
```

**Когда Jotai лучше Zustand:**
- Много независимых кусков стейта, которые редко обновляются вместе
- Нужен производный стейт с автоматическим пересчётом (как computed в Vue)
- Атомы можно создавать динамически (например, один атом на строку таблицы)

**Когда Zustand лучше Jotai:**
- Связанное состояние: несколько полей, которые обновляются как транзакция
- Нужны actions (методы) рядом со стейтом
- Привычная flat структура без «паутины атомов»

---

**Q: Как правильно структурировать Redux store в большом приложении?**

Стандарт сейчас — Redux Toolkit (RTK) со slice-паттерном:

```tsx
// features/player/playerSlice.ts
const playerSlice = createSlice({
  name: 'player',
  initialState: {
    profile: null as PlayerProfile | null,
    stats: null as PlayerStats | null,
    status: 'idle' as 'idle' | 'loading' | 'error',
  },
  reducers: {
    setProfile(state, action: PayloadAction<PlayerProfile>) {
      state.profile = action.payload; // Immer позволяет мутировать
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchPlayerStats.pending, state => { state.status = 'loading'; })
      .addCase(fetchPlayerStats.fulfilled, (state, action) => {
        state.stats = action.payload;
        state.status = 'idle';
      })
      .addCase(fetchPlayerStats.rejected, state => { state.status = 'error'; });
  },
});
```

**Принципы структуры:**
- **Feature-based slices**: каждая фича — свой slice, не один гигантский стор
- **Normalize server data**: для сущностей с ID использовать `createEntityAdapter` → `{ ids: [], entities: {} }`
- **Selectors отдельно**: не делать логику в компонентах — только `useSelector(selectPlayerStats)`
- **RTK Query для server state**: не класть ответы API в Redux вручную

**Подводный камень — мутации в extraReducers без Immer:**
```tsx
// Это работает — Immer под капотом позволяет мутировать draft
state.profile = action.payload; ✅

// Это НЕ работает — возвращаешь новый объект И мутируешь state одновременно
state.profile = action.payload;
return state; // ❌ нельзя делать оба
```

---

**Q: Что такое нормализация стейта и зачем она нужна?**

Нормализация — хранить данные по ID, как в БД, а не в виде вложенных массивов.

```tsx
// Ненормализованный (плохо для обновлений)
const state = {
  battles: [
    { id: '1', player: { id: 'p1', name: 'Alpha', score: 100 }, ... },
    { id: '2', player: { id: 'p1', name: 'Alpha', score: 100 }, ... },
    // player p1 дублирован в каждом battle — при обновлении имени нужно пройти весь массив
  ]
};

// Нормализованный (быстрые точечные обновления)
const state = {
  battles: { ids: ['1', '2'], entities: { '1': { id: '1', playerId: 'p1' }, ... } },
  players: { ids: ['p1'], entities: { 'p1': { id: 'p1', name: 'Alpha', score: 100 } } },
};

// Обновить имя игрока — O(1), не O(n)
state.players.entities['p1'].name = 'Beta';
```

В RTK — `createEntityAdapter`:
```tsx
const playersAdapter = createEntityAdapter<Player>();
const initialState = playersAdapter.getInitialState();
// даёт: addOne, addMany, updateOne, removeOne, selectAll, selectById
```

Нужно когда: одни и те же сущности встречаются в разных местах стора, нужны быстрые lookups по ID, часто делаются точечные обновления одной записи.

---

## 4. TypeScript

---

**Q: Что такое generics и где применяешь в React?**

Generics — параметрический полиморфизм: функция/тип работает с разными типами, сохраняя type safety.

```tsx
// Generic компонент списка
function List<T extends { id: string }>({
  items,
  renderItem,
}: {
  items: T[];
  renderItem: (item: T) => ReactNode;
}) {
  return <ul>{items.map(item => <li key={item.id}>{renderItem(item)}</li>)}</ul>;
}

// Generic хук
function useLocalStorage<T>(key: string, initial: T): [T, (value: T) => void] {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initial;
  });
  // ...
}
```

Применяю для: переиспользуемых компонентов (Table, Select, List), кастомных хуков, утилит для работы с данными.

---

**Q: Объясни разницу между type и interface. Когда что использовать?**

`interface` — описывает форму объекта, поддерживает declaration merging (расширение в разных местах кода), extends для наследования.

`type` — более широкий инструмент: может быть объединением (union), пересечением (intersection), mapped type, conditional type, примитивом.

```ts
// interface — только объекты
interface User {
  id: string;
  name: string;
}
interface AdminUser extends User {
  role: 'admin';
}

// type — объединения, кортежи, mapped types
type Status = 'idle' | 'loading' | 'error' | 'success';
type Nullable<T> = T | null;
type Readonly<T> = { readonly [K in keyof T]: T[K] };
```

**Практика:** я использую `interface` для описания форм объектов (пропсы компонентов, DTO), `type` для всего остального — union types, utility types, conditional types.

---

**Q: Что такое discriminated unions и как применять в React?**

Discriminated union — тип-объединение с общим «дискриминирующим» полем, которое TypeScript использует для сужения типа.

```ts
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function DataDisplay<T>({ state }: { state: AsyncState<T> }) {
  switch (state.status) {
    case 'idle': return <Placeholder />;
    case 'loading': return <Spinner />;
    case 'success': return <Content data={state.data} />; // TypeScript знает, что data есть
    case 'error': return <ErrorMessage error={state.error} />; // и что error есть
  }
}
```

Применяю для: состояний загрузки/ошибки, action types в reducer, вариантов компонента (button variant, notification type). Это лучше, чем `isLoading: boolean` + `error?: Error` + `data?: T` — такая комбинация позволяет невалидные состояния (isLoading=true и data одновременно).

---

**Q: Что такое conditional types и infer? Приведи практический пример.**

```ts
// Conditional type: T extends U ? X : Y
type IsArray<T> = T extends any[] ? true : false;
type A = IsArray<string[]>; // true
type B = IsArray<string>;   // false

// infer — извлечение типа изнутри
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;
type Resolved = UnpackPromise<Promise<User>>; // User

// Практический пример — тип возврата async функции
type AsyncReturnType<T extends (...args: any) => Promise<any>> =
  T extends (...args: any) => Promise<infer R> ? R : never;

async function fetchUser(): Promise<User> { ... }
type FetchResult = AsyncReturnType<typeof fetchUser>; // User
```

Использую для типизации утилит, wrapping функций, и когда нужно извлечь тип из сложной структуры без дублирования.

---

**Q: Что такое mapped types? Построй несколько полезных utility types с нуля.**

Mapped types — создают новый тип путём итерации по ключам существующего.

```ts
// Синтаксис: { [K in keyof T]: ... }

// Встроенный Partial — делает все поля опциональными
type MyPartial<T> = { [K in keyof T]?: T[K] };

// Встроенный Readonly
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };

// Pick — выбрать подмножество ключей
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };

// Record — создать объект с заданными ключами и типом значений
type MyRecord<K extends keyof any, V> = { [P in K]: V };

// Omit — исключить ключи
type MyOmit<T, K extends keyof T> = MyPick<T, Exclude<keyof T, K>>;

// Required — убрать опциональность
type MyRequired<T> = { [K in keyof T]-?: T[K] }; // -? убирает ?

// DeepPartial — рекурсивный Partial
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Mutable — убирает readonly
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
```

**Продвинутый пример — ремапинг ключей через `as`:**
```ts
// Геттеры для всех полей объекта
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User { name: string; age: number }
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

---

**Q: Что такое template literal types? Приведи практические примеры.**

Template literal types (TS 4.1+) — создают строковые типы через шаблонный синтаксис, аналогично template literals в JS.

```ts
type EventName = 'click' | 'focus' | 'blur';
type Handler = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur'

// Типизированные CSS-свойства
type CSSUnit = 'px' | 'em' | 'rem' | '%';
type CSSValue = `${number}${CSSUnit}`;
// "16px", "1.5em", "100%" — всё валидно; "16abc" — ошибка

// Типизированные пути к вложенным свойствам объекта
type DotPaths<T, Prefix extends string = ''> = {
  [K in keyof T]: T[K] extends object
    ? DotPaths<T[K], `${Prefix}${string & K}.`>
    : `${Prefix}${string & K}`;
}[keyof T];

interface Config { server: { host: string; port: number }; debug: boolean }
type ConfigPath = DotPaths<Config>; // 'server.host' | 'server.port' | 'debug'

// Практика — типизированная функция get по пути
function get<T, P extends DotPaths<T>>(obj: T, path: P): unknown {
  return path.split('.').reduce((acc: any, key) => acc[key], obj);
}
```

---

**Q: Объясни type narrowing в TypeScript — все способы и когда какой использовать.**

Type narrowing — сужение типа в определённой ветке кода на основе проверок.

```ts
type StringOrNumber = string | number;

// 1. typeof guard
function process(value: StringOrNumber) {
  if (typeof value === 'string') {
    value.toUpperCase(); // TypeScript знает: string
  } else {
    value.toFixed(2); // TypeScript знает: number
  }
}

// 2. instanceof guard
function handleError(err: Error | ApiError) {
  if (err instanceof ApiError) {
    console.log(err.statusCode); // ApiError
  }
}

// 3. in operator — проверка наличия свойства
type Circle = { kind: 'circle'; radius: number };
type Square = { kind: 'square'; side: number };
type Shape = Circle | Square;

function area(shape: Shape) {
  if ('radius' in shape) {
    return Math.PI * shape.radius ** 2; // Circle
  }
  return shape.side ** 2; // Square
}

// 4. Discriminated union — через общее поле (лучший паттерн)
function area2(shape: Shape) {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2;
    case 'square': return shape.side ** 2;
  }
}

// 5. Custom type guard — is predicat
function isApiError(err: unknown): err is ApiError {
  return err instanceof Error && 'statusCode' in err;
}
if (isApiError(err)) {
  console.log(err.statusCode); // TS знает, что ApiError
}

// 6. Assertion function — бросает если тип неверный
function assert(value: unknown, message: string): asserts value is string {
  if (typeof value !== 'string') throw new Error(message);
}
assert(input, 'Expected string');
input.toUpperCase(); // TS знает: string после assert

// 7. Exhaustive check — убеждаемся что все кейсы обработаны
function assertNever(value: never): never {
  throw new Error(`Unhandled case: ${value}`);
}

function processShape(shape: Shape) {
  switch (shape.kind) {
    case 'circle': return handleCircle(shape);
    case 'square': return handleSquare(shape);
    default: return assertNever(shape); // ошибка компиляции если добавить Triangle без кейса
  }
}
```

---

**Q: Что такое Variance в TypeScript? Covariance vs Contravariance.**

Variance определяет, как подтипирование одного типа влияет на подтипирование составного типа.

```ts
class Animal { name: string = '' }
class Dog extends Animal { breed: string = '' }
// Dog — подтип Animal (Dog extends Animal)
```

**Covariance (ковариантность)** — если `Dog extends Animal`, то `Producer<Dog> extends Producer<Animal>`. Типичен для возвращаемых значений.

```ts
type Producer<T> = () => T;
const produceDog: Producer<Dog> = () => new Dog();
const produceAnimal: Producer<Animal> = produceDog; // ✅ ковариантно
// Безопасно: если ожидаем Animal, получение Dog — OK (Dog является Animal)
```

**Contravariance (контравариантность)** — обратное: `Consumer<Animal> extends Consumer<Dog>`. Типичен для аргументов функций.

```ts
type Consumer<T> = (value: T) => void;
const handleAnimal: Consumer<Animal> = (a) => console.log(a.name);
const handleDog: Consumer<Dog> = handleAnimal; // ✅ контравариантно
// Безопасно: если функция работает с любым Animal — она точно справится с Dog
// ❌ обратное небезопасно: handleDog не может заменить handleAnimal — она ожидает breed
```

**В TypeScript функции в позиции методов — бивариантны (исторически, для совместимости):**
```ts
// --strictFunctionTypes включён по умолчанию в strict mode
// Функции как свойства (method shorthand) — бивариантны
interface Processor {
  process(value: Dog): void; // бивариантно — принимает и Dog и Animal
}

// Функции как поля — контравариантны (строже)
interface Processor2 {
  process: (value: Dog) => void; // контравариантно
}
```

**Практическое значение:**
```ts
// Почему это важно — безопасность при передаче коллбэков
function forEach<T>(arr: T[], callback: (item: T) => void) {}

const dogs: Dog[] = [];
const handleAnimalFn = (a: Animal) => console.log(a.name);
forEach(dogs, handleAnimalFn); // ✅ работает — контравариантность коллбэков
```

---

**Q: Что такое `satisfies` оператор (TS 4.9)? Зачем он нужен?**

`satisfies` проверяет, что значение соответствует типу, но **не сужает** тип до него. Сохраняет конкретный (более точный) тип выведенный из значения.

```ts
type Color = 'red' | 'green' | 'blue';
type Palette = Record<Color, string | [number, number, number]>;

// Без satisfies — тип слишком широкий
const palette: Palette = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255],
};
palette.red.map(x => x); // ❌ ошибка: string | number[] — TS не знает что red это number[]

// С satisfies — тип остаётся точным, проверка всё равно есть
const palette2 = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255],
} satisfies Palette;
palette2.red.map(x => x);   // ✅ TS знает: red — [number, number, number]
palette2.green.toUpperCase(); // ✅ TS знает: green — string
palette2.yellow;              // ❌ ошибка: yellow не в Palette — проверка работает
```

**Другой кейс — валидация конфиг-объектов:**
```ts
const routes = {
  home: { path: '/', auth: false },
  profile: { path: '/profile', auth: true },
  admin: { path: '/admin', auth: true },
} satisfies Record<string, { path: string; auth: boolean }>;

// routes.home.path — string (точный тип), не string | boolean
```

---

**Q: Как работают utility types Parameters, ReturnType, Awaited и как строить свои?**

```ts
// Встроенные utility types для функций
function createUser(name: string, age: number): Promise<User> { ... }

type Params = Parameters<typeof createUser>;    // [name: string, age: number]
type Return = ReturnType<typeof createUser>;    // Promise<User>
type Resolved = Awaited<Return>;               // User (TS 4.5+)

// ConstructorParameters и InstanceType
class EventEmitter<T> {
  constructor(private name: string, private handlers: Map<string, T[]>) {}
}
type EmitterArgs = ConstructorParameters<typeof EventEmitter>; // [string, Map<string, any[]>]
type EmitterInstance = InstanceType<typeof EventEmitter>;      // EventEmitter<unknown>

// Свой ExtractPromise (аналог Awaited)
type ExtractPromise<T> = T extends Promise<infer R>
  ? R extends Promise<any> ? ExtractPromise<R> : R  // рекурсивно для Promise<Promise<T>>
  : T;

type A = ExtractPromise<Promise<Promise<string>>>; // string

// Свой OverloadReturnType — тип последнего overload
function parse(value: string): number;
function parse(value: string[]): number[];
function parse(value: any): any { ... }

// ReturnType берёт последний overload — ограничение TS
type ParseReturn = ReturnType<typeof parse>; // any (реализация), не number | number[]
```

---

**Q: Как типизировать типобезопасный event emitter?**

Классическая задача для демонстрации generics + mapped types.

```ts
type EventMap = Record<string, any>;

type EventKey<T extends EventMap> = string & keyof T;
type EventHandler<T extends EventMap, K extends EventKey<T>> = (payload: T[K]) => void;

class TypedEventEmitter<T extends EventMap> {
  private listeners: Partial<{ [K in keyof T]: Array<EventHandler<T, K & string>> }> = {};

  on<K extends EventKey<T>>(event: K, handler: EventHandler<T, K>): this {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event]!.push(handler);
    return this;
  }

  off<K extends EventKey<T>>(event: K, handler: EventHandler<T, K>): this {
    this.listeners[event] = this.listeners[event]?.filter(h => h !== handler) as any;
    return this;
  }

  emit<K extends EventKey<T>>(event: K, payload: T[K]): void {
    this.listeners[event]?.forEach(handler => handler(payload));
  }
}

// Использование
interface GameEvents {
  playerJoined: { playerId: string; name: string };
  scoreUpdated: { playerId: string; score: number };
  gameOver: void;
}

const emitter = new TypedEventEmitter<GameEvents>();

emitter.on('playerJoined', ({ playerId, name }) => console.log(name)); // ✅ типизировано
emitter.on('scoreUpdated', ({ score }) => console.log(score));         // ✅
emitter.emit('playerJoined', { playerId: '1', name: 'Alpha' });        // ✅
emitter.emit('playerJoined', { score: 100 });                          // ❌ ошибка TS
```

---

**Q: Что такое recursive types и как их применять?**

Recursive types — типы, которые ссылаются на себя. Полезны для деревьев, вложенных структур, JSON-подобных данных.

```ts
// JSON-значение
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

const config: JSONValue = {
  name: 'app',
  settings: {
    debug: true,
    levels: [1, 2, 3],
    nested: { deep: 'value' },
  },
};

// Дерево категорий
interface Category {
  id: string;
  name: string;
  children?: Category[]; // рекурсия
}

// Рекурсивный Deep Readonly
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

interface State {
  user: { name: string; roles: string[] };
  settings: { theme: { color: string } };
}
type FrozenState = DeepReadonly<State>;
// Теперь даже state.user.roles — ReadonlyArray, state.settings.theme.color — readonly string
```

**Ограничение:** TypeScript может упасть с "Type instantiation is excessively deep" на очень глубоких рекурсиях. В TS 4.5 добавлена оптимизация для tail-recursive conditional types.

---

**Q: Что такое `infer` в conditional types — продвинутые паттерны?**

```ts
// 1. Извлечь тип первого аргумента функции
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;
type A = FirstArg<(name: string, age: number) => void>; // string

// 2. Unbox вложенных типов
type Unbox<T> = T extends Array<infer U>
  ? Unbox<U> // рекурсивно
  : T extends Promise<infer U>
  ? Unbox<U>
  : T;
type B = Unbox<Promise<Array<string>>>; // string

// 3. Получить тип ключа из Record
type ValueOf<T> = T[keyof T];
type Keys = ValueOf<{ a: string; b: number; c: boolean }>; // string | number | boolean

// 4. Распаковать Promise из возврата async функции
type AsyncResult<T extends (...args: any[]) => Promise<any>> =
  T extends (...args: any[]) => Promise<infer R> ? R : never;

async function loadUser(id: string): Promise<{ id: string; name: string }> { ... }
type UserResult = AsyncResult<typeof loadUser>; // { id: string; name: string }

// 5. Tuple to union
type TupleToUnion<T extends any[]> = T[number];
type Colors = TupleToUnion<['red', 'green', 'blue']>; // 'red' | 'green' | 'blue'

// 6. Тип-безопасный Object.entries
type Entries<T> = { [K in keyof T]: [K, T[K]] }[keyof T];
type UserEntries = Entries<{ name: string; age: number }>;
// ['name', string] | ['age', number]
```

---

**Q: Как работает Declaration Merging и Module Augmentation? Практические применения.**

**Declaration Merging** — объединение нескольких деклараций одного имени в одно определение.

```ts
// Interface merging — добавление методов к существующему интерфейсу
interface Window {
  myCustomFlag: boolean; // расширяем глобальный Window
}
window.myCustomFlag = true; // ✅ без ошибки

// Namespace merging — добавление вещей к пространству имён
namespace Express {
  interface Request {
    user?: AuthUser; // добавляем поле user к Request
  }
}
```

**Module Augmentation** — добавление типов к существующим модулям:

```ts
// Расширяем React Router типы
import 'react-router-dom';
declare module 'react-router-dom' {
  interface Register {
    router: typeof router; // типизируем useParams, useLoaderData и т.д.
  }
}

// Расширяем переменные окружения
declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
    NEXTAUTH_SECRET: string;
    NODE_ENV: 'development' | 'production' | 'test';
  }
}
// Теперь process.env.DATABASE_URL — string, не string | undefined

// Расширяем Zustand store для middleware
declare module 'zustand' {
  interface StoreMutators<S, A> {
    'zustand/logger': WithLogger<S>;
  }
}
```

**Главный кейс на интервью:** добавление `user` в `express.Request` — классический пример для JWT middleware:
```ts
// src/types/express.d.ts
declare global {
  namespace Express {
    interface Request {
      user?: { id: string; roles: string[] };
    }
  }
}
```

---

## 5. Real-time и браузерные API

---

**Q: Как работает WebSocket? Как правильно реализовать в React с cleanup?**

WebSocket — постоянное двунаправленное соединение между клиентом и сервером. В отличие от HTTP — сервер может сам отправлять данные без запроса от клиента.

```tsx
function useWebSocket<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [status, setStatus] = useState<'connecting' | 'open' | 'closed'>('connecting');
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => setStatus('open');
    ws.onmessage = (event) => setData(JSON.parse(event.data));
    ws.onclose = () => setStatus('closed');
    ws.onerror = (err) => console.error(err);

    return () => {
      ws.close(); // cleanup — закрываем соединение при unmount
    };
  }, [url]);

  const send = useCallback((message: unknown) => {
    wsRef.current?.send(JSON.stringify(message));
  }, []);

  return { data, status, send };
}
```

**Важные моменты:**
- Reconnect-логика при обрыве (exponential backoff)
- Heartbeat / ping-pong для поддержания соединения
- Обработка сообщений в очереди, если соединение не готово

---

**Q: Что такое SSE (Server-Sent Events)? Чем отличается от WebSocket?**

SSE — однонаправленный поток событий от сервера к клиенту через обычный HTTP. Клиент не отправляет данные через SSE — только слушает.

```ts
const eventSource = new EventSource('/api/stream');
eventSource.onmessage = (e) => console.log(JSON.parse(e.data));
eventSource.onerror = () => eventSource.close();
// cleanup
return () => eventSource.close();
```

**SSE vs WebSocket:**
| | SSE | WebSocket |
|--|--|--|
| Направление | Сервер → клиент | Двунаправленный |
| Протокол | HTTP | ws:// / wss:// |
| Reconnect | Автоматический (браузер) | Ручной |
| Прокси/firewall | Лучше совместимость | Могут блокировать |
| Использование | Стриминг LLM-ответов, лента событий | Чат, игра, real-time collaboration |

Я использовал SSE для стриминга LLM-ответов (AI-проект) — там идеально: сервер пишет токены по мере генерации, клиент отображает в реальном времени.

---

**Q: Как работает IntersectionObserver? Где применять в React?**

`IntersectionObserver` отслеживает пересечение элемента с viewport (или другим элементом) без polling и без scroll event listener — асинхронно и эффективно.

```tsx
function useIntersectionObserver(
  ref: RefObject<Element>,
  options?: IntersectionObserverInit
) {
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => setIsVisible(entry.isIntersecting),
      options
    );
    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref, options]);

  return isVisible;
}
```

**Применения:**
- Lazy loading изображений (загружать только когда видно)
- Infinite scroll (загрузить следующую страницу при достижении конца)
- Анимации появления (trigger при попадании в viewport)
- Аналитика (отслеживать, был ли элемент виден пользователю)

---

**Q: Что такое requestAnimationFrame и когда использовать в React?**

`rAF` запрашивает у браузера вызов функции перед следующей отрисовкой кадра (~60 раз в секунду). Это оптимальный момент для изменений DOM — браузер сам решает, когда рисовать.

```tsx
// Throttle частых событий через rAF
function useThrottledScroll() {
  const [scrollY, setScrollY] = useState(0);
  const rafId = useRef<number>(0);

  useEffect(() => {
    const handleScroll = () => {
      cancelAnimationFrame(rafId.current);
      rafId.current = requestAnimationFrame(() => {
        setScrollY(window.scrollY);
      });
    };
    window.addEventListener('scroll', handleScroll, { passive: true });
    return () => {
      window.removeEventListener('scroll', handleScroll);
      cancelAnimationFrame(rafId.current);
    };
  }, []);

  return scrollY;
}
```

**Когда нужен:**
- Анимации через JS (когда CSS недостаточно)
- Throttle scroll/resize обработчиков
- Любые DOM-измерения и манипуляции, которые должны быть синхронизированы с отрисовкой

---

## 6. Архитектура и паттерны

---

**Q: Как организовать большую кодовую базу React-приложения?**

Предпочитаю **feature-based структуру** над layer-based:

```
src/
  features/
    auth/
      components/
      hooks/
      api/
      store/
      index.ts       ← публичный API фичи
    profile/
    notifications/
  shared/
    components/      ← переиспользуемые UI-компоненты
    hooks/           ← переиспользуемые хуки
    utils/
    types/
  app/
    providers/
    router/
    store/
```

**Принципы:**
- **Явный публичный API**: каждая фича экспортирует только то, что нужно снаружи через `index.ts`. Внутренние детали — приватны.
- **Запрет cross-feature импортов**: `features/auth` не импортирует из `features/profile` напрямую — только через `shared`.
- **Коллокация**: тесты, типы, хуки рядом с компонентом, который их использует — не в отдельной папке на верхнем уровне.

---

**Q: Что такое compound components и когда их применять?**

Compound components — паттерн, где родительский компонент управляет состоянием, а дочерние — его части. Общаются через Context, а не пропсы.

```tsx
// Использование
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Trigger value="profile">Profile</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="profile"><ProfilePanel /></Tabs.Content>
  <Tabs.Content value="settings"><SettingsPanel /></Tabs.Content>
</Tabs>
```

**Когда применять:**
- Сложные UI-компоненты с несколькими связанными частями (Accordion, Dialog, Select, Tabs)
- Когда нужна гибкость в структуре без передачи кучи пропсов

---

**Q: Как реализовать глобальную обработку ошибок в React?**

Два уровня:

**1. Error Boundaries** — class-компоненты, ловят ошибки рендеринга в поддереве:
```tsx
class ErrorBoundary extends Component {
  state = { hasError: false, error: null };
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  componentDidCatch(error: Error, info: ErrorInfo) {
    Sentry.captureException(error, { extra: info });
  }
  render() {
    if (this.state.hasError) return <ErrorFallback error={this.state.error} />;
    return this.props.children;
  }
}
```

**2. Глобальный обработчик для async ошибок:**
```ts
window.addEventListener('unhandledrejection', (event) => {
  Sentry.captureException(event.reason);
});
```

Error Boundaries **не ловят**: ошибки в обработчиках событий, async функциях, SSR, ошибки самого boundary.

---

## 7. Тестирование

---

**Q: Какую стратегию тестирования используешь для React-приложений?**

Придерживаюсь **Testing Trophy** (Kent C. Dodds):

```
         E2E (Playwright/Cypress) — мало, но покрывают критические флоу
    Integration Tests (RTL) — основа: компоненты + хуки вместе
Unit Tests (Vitest/Jest) — утилиты, редьюсеры, изолированная логика
```

**Принцип RTL:** тестируй поведение, а не реализацию. Не проверяй внутренний state — проверяй, что пользователь видит и может сделать.

```tsx
// Плохо — тестирует реализацию
expect(component.state('isOpen')).toBe(true);

// Хорошо — тестирует поведение
await userEvent.click(screen.getByRole('button', { name: 'Open menu' }));
expect(screen.getByRole('menu')).toBeVisible();
```

В EtaCar переход с Cypress на Playwright дал: покрытие с 65 до 90%, CI с 30 до 8 минут, экономия ~$2K в год.

---

**Q: Как тестировать кастомные хуки?**

```tsx
import { renderHook, act } from '@testing-library/react';

test('useCounter increments value', () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

Для хуков с Context — оборачиваю в wrapper:
```tsx
const { result } = renderHook(() => useAuth(), {
  wrapper: ({ children }) => <AuthProvider>{children}</AuthProvider>,
});
```

---

## 8. CSS и стили

---

**Q: Как работает CSS-in-JS и какие у него трейдоффы?**

CSS-in-JS (styled-components, Emotion) — стили пишутся в JS/TS, генерируются уникальные классы в runtime.

**Плюсы:**
- Типизированные пропсы для стилей
- Автоматическая область видимости (нет глобальных конфликтов)
- Условные стили через props без лишних className
- Удаление неиспользуемых стилей (нет глобального CSS)

**Минусы:**
- Runtime overhead: генерация классов в браузере
- Больший bundle size
- Стили не кэшируются браузером отдельно от JS

**Альтернативы с нулевым runtime:**
- `CSS Modules` — область видимости через хэшированные имена, нет runtime
- `Tailwind CSS` — utility-first, стили в HTML, нет CSS файлов вообще
- `vanilla-extract` — CSS-in-TS, но компилируется в статические CSS файлы

Для performance-чувствительных продуктов предпочитаю CSS Modules или Tailwind.

---

## 9. Системный дизайн (Frontend System Design)

---

**Q: Как бы ты спроектировал live-дашборд статистики игрока с обновлениями в реальном времени?**

**Требования (уточнить):** частота обновлений, количество метрик, размер аудитории, нужна ли история.

**Архитектура:**

```
WebSocket / SSE
      ↓
Нормализованный стор (Zustand / RTK)
      ↓
Selector'ы с мемоизацией (reselect)
      ↓
Компоненты с React.memo
```

**Детали:**
1. **Транспорт:** WebSocket для двунаправленного (управление подпиской), SSE если только сервер → клиент
2. **Стор:** нормализованная структура `{ byId: {}, ids: [] }` — дешевле точечные обновления
3. **Батчинг:** React 18 автоматически батчит setState — несколько обновлений за один рендер
4. **Виртуализация:** если история событий длинная — react-virtual
5. **Throttle:** если сервер шлёт 60 событий/сек, а UI нужно обновлять раз в 100мс — throttle на стороне клиента
6. **Оптимистичные обновления:** для ставок/действий пользователя — обновить UI сразу, откатить при ошибке

---

**Q: Как организовать компонентную библиотеку для большой команды?**

**Структура:**
```
packages/
  ui/               ← примитивы (Button, Input, Modal)
  icons/            ← иконки
  tokens/           ← дизайн-токены (цвета, шрифты, spacing)
```

**Принципы:**
1. **Headless компоненты** для сложной логики (Radix UI, Headless UI) — поведение без стилей, стили сверху
2. **Compound components** для составных UI (Tabs, Accordion, Select)
3. **Polymorphic `as` prop** — `<Button as="a" href="/link">` рендерит `<a>`, сохраняя стили кнопки
4. **Storybook** — документация компонентов, visual regression тесты
5. **Версионирование и changelog** — Changesets для управления версиями в монорепозитории
6. **Экспорт токенов** — CSS custom properties + JS объект, чтобы работало в любом контексте

---

## 10. Живое кодирование — типичные задачи

---

### Задача 1: Debounce хук

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

// Использование
const debouncedSearch = useDebounce(searchQuery, 300);
useEffect(() => {
  if (debouncedSearch) fetchResults(debouncedSearch);
}, [debouncedSearch]);
```

---

### Задача 2: Infinite scroll хук

```tsx
function useInfiniteScroll(callback: () => void) {
  const observerRef = useRef<IntersectionObserver | null>(null);
  const triggerRef = useCallback((node: Element | null) => {
    if (observerRef.current) observerRef.current.disconnect();
    if (!node) return;

    observerRef.current = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) callback();
    });
    observerRef.current.observe(node);
  }, [callback]);

  return triggerRef;
}

// Использование
const { data, fetchNextPage } = useInfiniteQuery(...);
const triggerRef = useInfiniteScroll(fetchNextPage);

return (
  <ul>
    {data.pages.flat().map(item => <li key={item.id}>{item.name}</li>)}
    <li ref={triggerRef} /> {/* sentinel элемент */}
  </ul>
);
```

---

### Задача 3: Generic Table компонент

```tsx
interface Column<T> {
  key: keyof T;
  header: string;
  render?: (value: T[keyof T], row: T) => ReactNode;
}

function Table<T extends { id: string | number }>({
  data,
  columns,
}: {
  data: T[];
  columns: Column<T>[];
}) {
  return (
    <table>
      <thead>
        <tr>{columns.map(col => <th key={String(col.key)}>{col.header}</th>)}</tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={row.id}>
            {columns.map(col => (
              <td key={String(col.key)}>
                {col.render
                  ? col.render(row[col.key], row)
                  : String(row[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

---

### Задача 4: useFetch с отменой запроса

```tsx
function useFetch<T>(url: string) {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
  }>({ data: null, loading: true, error: null });

  useEffect(() => {
    const controller = new AbortController();

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json() as Promise<T>;
      })
      .then(data => setState({ data, loading: false, error: null }))
      .catch(err => {
        if (err.name === 'AbortError') return; // игнорируем отмену
        setState({ data: null, loading: false, error: err });
      });

    return () => controller.abort(); // cleanup — отменяем при unmount или смене url
  }, [url]);

  return state;
}
```

---

## Вопросы по стеку Lesta — что могут спросить специфично

- **Canvas / WebGL**: есть ли опыт? (игровые компании иногда используют для визуализации статистики)
- **Web Workers**: знаешь ли, как вынести тяжёлые вычисления из main thread?
- **WASM**: слышал ли, понимаешь ли зачем?
- **Internationalization (i18n)**: работал ли с react-i18next / i18next? (у Lesta несколько локалей)
- **A/B тестирование**: как на фронте реализуется feature flags?
- **Аналитика**: интеграция GTM, custom events, отслеживание пользовательских сценариев

---

## Чеклист перед интервью

- [ ] Знаю разницу между Fiber reconciler и старым Stack reconciler
- [ ] Могу объяснить, когда useMemo/useCallback помогает, а когда добавляет накладные расходы
- [ ] Знаю про layout thrashing и как его избежать
- [ ] Умею написать хук с WebSocket и правильным cleanup
- [ ] Понимаю разницу type vs interface в TypeScript
- [ ] Могу реализовать debounce, infinite scroll, generic table без подсказок
- [ ] Готов обсудить, как бы организовал large-scale компонентную библиотеку
- [ ] Знаю про виртуализацию списков и когда она нужна
