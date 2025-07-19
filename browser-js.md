# Browser JS 

## 1\. Что такое DOM?

DOM (Document Object Model) — это программный интерфейс для HTML и XML документов. Он представляет страницу в виде дерева узлов, где каждый узел соответствует элементу, атрибуту или тексту в документе.

С помощью DOM можно динамически изменять структуру, содержимое и стили страницы через JavaScript.

## 2\. Типы узлов DOM-дерева

В DOM-дереве различают несколько типов узлов:

*   Element (`Node.ELEMENT_NODE`, код 1)
    
*   Text (`Node.TEXT_NODE`, код 3)
    
*   Comment (`Node.COMMENT_NODE`, код 8)
    
*   Document (`Node.DOCUMENT_NODE`, код 9)
    
*   DocumentFragment (`Node.DOCUMENT_FRAGMENT_NODE`, код 11)
    
*   Attr (`Node.ATTRIBUTE_NODE`, код 2) — устаревший, атрибуты удобнее получать через свойства элемента
    

Каждый узел имеет свойство `nodeType` и `nodeName` для определения типа.

## 3\. Методы поиска элементов в DOM

Самые распространённые методы:

*   `document.getElementById(id)` — элемент по уникальному идентификатору
    
*   `document.getElementsByClassName(className)` — HTMLCollection по классу
    
*   `document.getElementsByTagName(tagName)` — HTMLCollection по имени тега
    
*   `document.getElementsByName(name)` — NodeList по атрибуту name
    
*   `document.querySelector(selector)` — первый элемент по CSS-селекторам
    
*   `document.querySelectorAll(selector)` — NodeList всех элементов по CSS-селекторам
    

## 4\. Свойства для перемещения по DOM-дереву

Для навигации между узлами используют:

*   Родительские узлы:
    
    *   `node.parentNode`
        
    *   `node.parentElement`
        
*   Дочерние узлы:
    
    *   `node.childNodes` (включает текстовые узлы)
        
    *   `node.children` (только элементы)
        
    *   `node.firstChild` / `node.firstElementChild`
        
    *   `node.lastChild` / `node.lastElementChild`
        
*   Соседние узлы:
    
    *   `node.previousSibling` / `node.previousElementSibling`
        
    *   `node.nextSibling` / `node.nextElementSibling`
        

## 5\. Разница между attribute и property у DOM-элементов

Атрибут (attribute) — значение, заданное в HTML-разметке. Свойство (property) — свойство DOM-объекта в JavaScript.

*   Атрибуты всегда строки и отражают начальное состояние элемента.
    
*   Свойства могут иметь другие типы (boolean, object) и изменяются динамически.
    

Пример: `<input id="chk" checked>`

*   `element.getAttribute('checked')` вернёт `"checked"` или `null`.
    
*   `element.checked` вернёт `true` или `false` в зависимости от текущего состояния.
    

## 6\. Что такое BOM?

BOM (Browser Object Model) — набор объектов, предоставляемых браузером за пределами DOM. Основные компоненты:

*   `window` — глобальный объект
    
*   `navigator` — информация о браузере и платформе
    
*   `location` — работа с URL и навигацией
    
*   `history` — управление историей переходов
    
*   `screen` — данные о дисплее
    
*   `console`, `setTimeout`, `fetch` и другие глобальные API
    

BOM позволяет управлять поведением окна и средой выполнения.

## 7\. Виды событий в JavaScript

События можно условно разделить на категории:

*   UI-события: `load`, `resize`, `scroll`, `error`
    
*   Мышиные события: `click`, `dblclick`, `mousedown`, `mouseup`, `mousemove`
    
*   Клавиатурные события: `keydown`, `keyup`, `keypress`
    
*   События фокуса: `focus`, `blur`, `focusin`, `focusout`
    
*   События форм: `submit`, `change`, `input`, `invalid`
    
*   События касания и Pointer: `touchstart`, `touchmove`, `pointerdown` и др.
    
*   Clipboard-события: `copy`, `paste`, `cut`
    
*   Drag & Drop: `dragstart`, `dragover`, `drop`
    
*   Custom Events: пользовательские через `new CustomEvent()`
    

## 8\. Как добавить обработчик события на DOM-элемент

Самый гибкий способ — `addEventListener`:

```js
const button = document.querySelector('button');
function onClick(event) {
  console.log('Нажали кнопку', event);
}
button.addEventListener('click', onClick, { capture: false, passive: true });
```

Параметры:

*   Тип события (`'click'`)
    
*   Функция-обработчик
    
*   Опции или булево значение (capture, once, passive)
    

Можно также использовать свойство:

```js
button.onclick = onClick;
```

## 9\. Как удалить обработчик события с DOM-элемента

Чтобы удалить добавленный через `addEventListener` обработчик, нужно передать те же аргументы:

```js
button.removeEventListener('click', onClick, { capture: false });
```

Удаление через `onclick`:

```js
button.onclick = null;
```

Важно, чтобы функция-обработчик была тем же объектом (не анонимная).

## 10\. Что такое распространение события (Event Propagation)

Когда событие срабатывает на вложенном элементе, оно проходит три фазы:

1.  Фаза захвата (capturing): от корня документа к целевому элементу
    
2.  Фаза цели (target): обработка на самом целевом элементе
    
3.  Фаза всплытия (bubbling): от целевого элемента обратно к корню
    

Параметр `capture` в `addEventListener` позволяет слушать события на этапе захвата. По умолчанию обработчики работают на этапе всплытия.

## 11\. Что такое делегирование событий (Event Delegation)

Это приём, когда вместо навешивания обработчика на каждый вложенный элемент, один слушатель вешают на общий контейнер. Внутри обработчика проверяют источник события через `event.target` или `event.currentTarget`:

```js
document.querySelector('#list').addEventListener('click', event => {
  if (event.target.matches('li')) {
    console.log('Нажали:', event.target.textContent);
  }
});
```

Преимущества:

*   Меньше обработчиков в памяти
    
*   Автоматически работает для динамически добавленных дочерних элементов
    

## 12\. Как использовать media-выражения в JavaScript

Для реакции на изменение размера или ориентации экрана используют `window.matchMedia`:

```js
const mq = window.matchMedia('(max-width: 768px)');

function handleChange(e) {
  if (e.matches) {
    console.log('Экран ≤ 768px');
  } else {
    console.log('Экран > 768px');
  }
}

// Проверка сразу
handleChange(mq);

// Обработка изменений
mq.addEventListener('change', handleChange);
```

Метод возвращает объект `MediaQueryList` с булевым свойством `matches` и событием `change`.

## 13\. Расскажите про координаты в браузере?

Координаты в браузере отражают положение указателя или элемента относительно разных отсчётных систем:

*   `clientX` / `clientY` Показывают позицию относительно видимой области окна (viewport), без учёта прокрутки.
    
*   `pageX` / `pageY` Позиция относительно левого верхнего угла всего документа, включая прокрученный участок.
    
*   `screenX` / `screenY` Координаты относительно левого верхнего угла экрана пользователя.
    

Эти свойства доступны, например, в событиях мыши (`MouseEvent`) или касания (`TouchEvent`).

## 14\. Разница между HTMLCollection и NodeList

*   HTMLCollection
    
    *   Живой (live) список элементов, обновляется при изменении DOM.
        
    *   Содержит только узлы типа `Element`.
        
    *   Не поддерживает методы массива (`forEach` в старых браузерах).
        
*   NodeList
    
    *   Может быть статическим (например, результат `querySelectorAll`) или живым (например, `childNodes`).
        
    *   Содержит любые узлы (`Element`, `Text`, `Comment`).
        
    *   Поддерживает `forEach`, а в современных реализациях и другие итерационные методы.
        

## 15\. Как динамически добавить элемент на HTML-страницу?

1.  Создать элемент:
    
    js
    
    Копировать
    
    ```
    const div = document.createElement('div');
    div.textContent = 'Новый блок';
    div.className = 'new-item';
    ```
    
2.  Вставить в нужное место:
    
    *   `parent.appendChild(div)` — добавит в конец родителя.
        
    *   `parent.insertBefore(div, referenceNode)` — вставит перед `referenceNode`.
        
    *   `parent.append(div)` — аналог `appendChild`, но принимает строки.
        
    *   `element.insertAdjacentHTML(position, htmlString)` — вставит HTML рядом с `element`.
        

## 16\. Разница между feature detection, feature inference и анализом строки user-agent

*   Feature detection Надёжно проверяет наличие API или метода в рантайме, например:
    
    ```js
    if ('fetch' in window) { /* можно использовать fetch */ }
    ```
    
*   Feature inference Догадка о наличии фичи на основании поведения других методов, например, тестирование таймаутов для определения точности таймеров.
    
*   User-Agent sniffing Парсинг `navigator.userAgent` для определения браузера/платформы. Ненадёжно из-за возможного спуфинга и различий в строках UA.
    

Рекомендуется всегда применять feature detection.

## 17\. Разница между e.preventDefault() и e.stopPropagation()

*   `e.preventDefault()` Отменяет стандартное поведение браузера для данного события (переход по ссылке, отправка формы и т.д.).
    
*   `e.stopPropagation()` Прекращает дальнейшую всплытие или захват события по дереву DOM, но не влияет на действие по умолчанию.
    

Обе функции можно вызывать в одном обработчике, но они решают различные задачи.

## 18\. Разница между event.target и event.currentTarget

*   `event.target` Указывает на самый вложенный элемент, на котором событие фактически произошло (например, клик по `<span>` внутри `<button>`).
    
*   `event.currentTarget` Элемент, на котором висит текущий обработчик (тот, к которому был применён `addEventListener`).
    

При делегировании событий важно ориентироваться на `event.target`.

## 19\. Разница между .stopPropagation() и .stopImmediatePropagation()

*   `stopPropagation()` Останавливает дальнейшее распространение события вверх или вниз по дереву, но не мешает выполнению остальных обработчиков на том же элементе.
    
*   `stopImmediatePropagation()` Делает то же самое, что и `stopPropagation()`, и дополнительно предотвращает вызов остальных обработчиков того же типа на текущем элементе.
    

## 20\. Разница между событиями load и DOMContentLoaded

*   `DOMContentLoaded` Срабатывает, когда весь HTML загружен и распарсен, а скрипты, подключённые с `defer`, выполнены. Не ждёт картинок, стилей и фреймов.
    
*   `load` Дождётся полной загрузки всех ресурсов страницы: стилей, изображений, шрифтов и вложенных фреймов.
    

## 21\. Сколько аргументов принимает addEventListener?

Метод `addEventListener` принимает три аргумента:

1.  `type` (string) — тип события (`'click'`, `'keydown'` и т.д.).
    
2.  `listener` (function) — функция-обработчик.
    
3.  `options` (boolean или object, необязательно) — режим захвата (`capture`), одноразовый вызов (`once`), пассивный слушатель (`passive`) и т.д.
    

## 22\. Разница между innerHTML и outerHTML

*   `element.innerHTML` Возвращает или устанавливает HTML-содержимое **внутри** элемента, не затрагивая сам элемент.
    
*   `element.outerHTML` Возвращает или устанавливает **весь** элемент вместе с его содержимым. При присвоении элемент будет полностью заменён новым HTML.
    

## 23\. Разница между JSON и XML

*   JSON Лёгкий формат обмена данными, основан на синтаксисе JavaScript-объектов. Поддерживает объекты, массивы, строки, числа, `true`/`false`, `null`.
    
*   XML Маркированный формат с четкой иерархией тегов, атрибутами, поддержкой пространств имён и схем валидации. Более многословный, но гибкий для описания документов.
    

JSON предпочтителен в веб-приложениях из-за компактности и простоты парсинга в JavaScript.

## 24\. Как узнать об использовании метода event.preventDefault()?

У объекта события есть свойство `defaultPrevented`. Если `e.preventDefault()` был вызван, то:

```js
element.addEventListener('click', e => {
  if (e.defaultPrevented) {
    console.log('Поведение по умолчанию отменено');
  }
});
```

Это булево значение позволяет проверить, блокируется ли стандартное действие.

## 25\. Для чего используется свойство `window.navigator`

`window.navigator` предоставляет информацию о браузере и среде выполнения. Он включает свойства:

*   `navigator.userAgent` — строка с данными о браузере и ОС.
    
*   `navigator.platform` — платформа (например, "Win32", "Linux x86\_64").
    
*   `navigator.language` и `navigator.languages` — предпочтительные языки пользователя.
    
*   `navigator.onLine` — булево значение, показывает, онлайн ли пользователь.
    
*   Методы геолокации (`navigator.geolocation`) и данные о батарее (`navigator.getBattery()`).
    

## 26\. Для чего используется метод `.focus()`

Метод `element.focus()` устанавливает фокус ввода на указанный элемент, делая его `document.activeElement`.

*   Включает элемент в последовательность клавиатурной навигации.
    
*   Генерирует событие `focus`.
    
*   При необходимости браузер автоматически прокручивает страницу, чтобы элемент был виден.
    

Пример:

js

Копировать

```
const input = document.querySelector('input');
input.focus();
```

## 27\. Для чего используется свойство `document.forms`

`document.forms` возвращает живую коллекцию (`HTMLCollection`) всех элементов `<form>` на странице.

*   Доступ по индексу или имени формы: `document.forms[0]`, `document.forms.loginForm`.
    
*   Позволяет быстро перебирать и управлять всеми формами:
    

js

Копировать

```
for (const form of document.forms) {
  console.log(form.name, form.action);
}
```

## 28\. Для чего используется метод `.scrollIntoView()`

`element.scrollIntoView(options)` прокручивает контейнер (или окно) так, чтобы элемент оказался в зоне видимости.

Параметры `options`:

*   `behavior`: `"auto"` (по умолчанию) или `"smooth"` для плавного скролла.
    
*   `block`: `"start" | "center" | "end" | "nearest"` — вертикальное выравнивание.
    
*   `inline`: `"start" | "center" | "end" | "nearest"` — горизонтальное выравнивание.
    

js

Копировать

```
document.getElementById('section2').scrollIntoView({ 
  behavior: 'smooth', 
  block: 'start' 
});
```

## 29\. Разница между методами `.submit()` и `.requestSubmit()`

| Метод | Поведение |
| --- | --- |
| `form.submit()` | Немедленно отправляет форму без выполнения валидации HTML5 и без генерации события `submit`. |
| `form.requestSubmit()` | Запускает встроенную валидацию, генерирует событие `submit`, позволяет указать кнопку-инициатор. |

`requestSubmit()` лучше подходит для программного сабмита, когда важно учесть валидность полей и отработку слушателей `submit`.

## 30\. Расскажите о `IntersectionObserver`

`IntersectionObserver` — API для асинхронного отслеживания видимости (пересечения) элемента с корнем (viewport или другим элементом).

Ключевые моменты:

1.  Создание наблюдателя:
    
    ```js
    const observer = new IntersectionObserver(callback, {
      root: null,           // null = viewport
      rootMargin: '0px',
      threshold: [0, 0.5, 1]
    });
    ```
    
2.  Подписка на элементы:

    ```js
    observer.observe(document.querySelector('.item'));
    ```
    
3.  Колбэк получает массив `entries` с полями:
    
    *   `entry.isIntersecting` — булево: элемент в зоне пересечения.
        
    *   `entry.intersectionRatio` — доля площади элемента, видимая пользователю.
        
4.  Отключение:

    ```js
    observer.unobserve(elem);
    observer.disconnect();
    ```
    

Используется для ленивой загрузки изображений, бесконечного скролла и анимаций при появлении в зоне видимости.

## 31\. Расскажите о `URLSearchParams`

`URLSearchParams` упрощает работу с параметрами запроса в URL:

*   Создание:
    
    js
    
    Копировать
    
    ```
    const params = new URLSearchParams(window.location.search);
    ```
    
*   Основные методы:
    
    *   `params.get(name)` — первое значение параметра.
        
    *   `params.getAll(name)` — все значения параметра в виде массива.
        
    *   `params.has(name)` — булево, существует ли параметр.
        
    *   `params.set(name, value)` — установить значение (заменить все предыдущие).
        
    *   `params.append(name, value)` — добавить ещё одно значение.
        
    *   `params.delete(name)` — удалить все значения параметра.
        
    *   `params.toString()` — получить закодированную строку запроса.
        

Пример обновления URL без перезагрузки:

```js
params.set('page', '2');
history.replaceState(null, '', '?' + params.toString());
```

## 32\. Какие есть ограничения у `window.close()`

*   **Можно закрывать только окна, открытые скриптом** через `window.open`.
    
*   **Основное окно (открытое пользователем)** браузер не позволит закрыть программно, чтобы предотвратить злоупотребления.
    
*   **Некоторые браузеры** игнорируют `window.close()` без пользовательского действия (например, при событии клика).
    

## 33\. Как создавать пользовательское событие (Custom Events)

1.  Создать событие с помощью конструктора `CustomEvent`:
    
    ```js
    const event = new CustomEvent('my-event', {
      detail: { foo: 'bar' },
      bubbles: true,
      cancelable: true,
      composed: false
    });
    ```
    
2.  Отправить событие на элемент:
    
    ```js
    elem.dispatchEvent(event);
    ```
    
3.  Обработать событие:
    
    ```js
    elem.addEventListener('my-event', e => {
      console.log(e.detail.foo);
    });
    ```
    

`detail` позволяет передать произвольные данные слушателям.

## 34\. Что такое IndexedDB? Как работает IndexedDB

IndexedDB — встроенная в браузер асинхронная NoSQL-база данных для хранения больших объёмов структурированных данных, включая файлы и Blobs.

Основные этапы работы:

1.  Открытие/создание базы:
    
    ```js
    const request = indexedDB.open('MyDB', 1);
    request.onupgradeneeded = e => {
      const db = e.target.result;
      db.createObjectStore('users', { keyPath: 'id', autoIncrement: true });
    };
    ```
    
2.  Успешное открытие:
    
    
    ```js
    request.onsuccess = e => {
      const db = e.target.result;
      // Работа с транзакциями
    };
    ```
    
3.  Транзакции и операции:
    
    ```js
    const tx = db.transaction('users', 'readwrite');
    const store = tx.objectStore('users');
    store.add({ name: 'Alice' });
    store.get(1).onsuccess = e => console.log(e.target.result);
    tx.oncomplete = () => console.log('Транзакция завершена');
    ```
    
4.  Асинхронность реализована через события (`onsuccess`, `onerror`).
    

Для упрощения можно применять обёртки-промисификаторы (например, библиотека `idb`).

## 35\. Расскажите о методе `requestAnimationFrame()`

`window.requestAnimationFrame(callback)` планирует вызов `callback` перед следующим перерисовыванием (vsync) браузера.

*   `callback` получает аргумент — высокоточный временной штамп.
    
*   Возвращает числовой идентификатор, который можно передать в `cancelAnimationFrame(id)` для отмены.
    
*   Предпочтительнее `setTimeout`/`setInterval` для анимации: обеспечивает плавность и экономит ресурсы при невидимом элементе.
    

Пример анимации:

```js
let start = null;
function step(timestamp) {
  if (!start) start = timestamp;
  const progress = timestamp - start;
  elem.style.transform = `translateX(${Math.min(progress / 10, 200)}px)`;
  if (progress < 2000) {
    requestAnimationFrame(step);
  }
}
requestAnimationFrame(step);
```

## 36\. Разница между LocalStorage, SessionStorage и Cookies

Ниже разбираем три основных механизма хранения данных в браузере и их ключевые отличия.

## LocalStorage

*   Объём Обычно 5–10 МБ на домен (в разных браузерах значение может отличаться).
    
*   Жизненный цикл Данные хранятся **пока не удалены вручную** (через API или очистку браузера).
    
*   Доступ Доступен только на стороне клиента через `window.localStorage`. Не отправляется автоматически в HTTP-запросах.
    
*   Интерфейс
    
    ```js
    localStorage.setItem('key', 'value');
    const val = localStorage.getItem('key');
    localStorage.removeItem('key');
    localStorage.clear();
    ```
    
*   Безопасность Доступен любому JavaScript на странице (подвержен XSS). Нельзя задать флаги Secure/HttpOnly.
    
*   Случаи использования Кеширование настроек, UI-состояния, данных, не чувствительных к безопасности и не требующих отправки на сервер.
    

## SessionStorage

*   Объём Тот же порядок, что и LocalStorage (около 5–10 МБ на домен).
    
*   Жизненный цикл Хранится **до закрытия вкладки или окна**. Данные не разделяются между разными вкладками.
    
*   Доступ Доступен только в той же вкладке через `window.sessionStorage`. Не отправляется в HTTP-запросах.
    
*   Интерфейс

    ```js
    sessionStorage.setItem('sessionKey', 'value');
    const v = sessionStorage.getItem('sessionKey');
    sessionStorage.removeItem('sessionKey');
    sessionStorage.clear();
    ```
    
*   Безопасность Аналогично LocalStorage: доступен JS, уязвим к XSS, без флагов Secure/HttpOnly.
    
*   Случаи использования Хранение временных данных для одной пользовательской сессии в конкретной вкладке (напр., состояние формы, шаги в мастере).
    

## Cookies

*   Объём Ограничены ~4 КБ на каждую куку, обычно не более 20–50 кук на домен.
    
*   Жизненный цикл Можно задать явно через атрибут `expires` или `max-age`. Без атрибута живут до закрытия браузера (session cookies).
    
*   Доступ По умолчанию передаются вместе с каждым HTTP-запросом (заголовок `Cookie`). Доступны в JS через `document.cookie`, но можно сделать `HttpOnly`, чтобы закрыть доступ для JS.
    
*   Интерфейс
    
    ```js
    // Установка
    document.cookie = 'name=value; path=/; max-age=3600; Secure; SameSite=Lax';
    
    // Чтение
    const all = document.cookie; // строка "name=value; other=othervalue"
    
    // Удаление
    document.cookie = 'name=; max-age=0; path=/';
    ```
    
*   Безопасность Можно задать `Secure`, `HttpOnly`, `SameSite` для защиты от XSS/CSRF. Тем не менее низкий объём и постоянная передача в запросах повышает нагрузку на сеть.
    
*   Случаи использования Аутентификация (токены, сессии), трекинг, небольшие настройки, которые должны быть доступны на сервере.
    

## Сравнительная таблица

| Характеристика | LocalStorage | SessionStorage | Cookies |
| --- | --- | --- | --- |
| Объём | ~5–10 МБ | ~5–10 МБ | ~4 КБ на куку, 20–50 кук на домен |
| Жизненный цикл | Постоянно | До закрытия вкладки | Зависит от `expires`/`max-age` |
| Доступ в HTTP-запросах | Нет | Нет | Да  |
| Доступ из JavaScript | `localStorage` | `sessionStorage` | `document.cookie` (если не HttpOnly) |
| API | Простое KV-хранилище | Простое KV-хранилище | Строковый парсинг и формирование |
| Возможность скрыть от JS | Нет | Нет | Да, через `HttpOnly` |
| Подверженность XSS | Да  | Да  | Нет (если `HttpOnly`) |
| Использование | Параметры UI, кеш | Сессионное состояние | Сессии, аутентификация, трекинг |

### Рекомендации по выбору

*   Выбирайте **Cookies**, если данные нужно отправлять на сервер автоматически или хранить токены аутентификации с флагами безопасности.
    
*   Используйте **SessionStorage**, когда нужно сохранить данные только на время одной вкладки (шаги формы, фильтры).
    
*   Применяйте **LocalStorage** для кэша, настроек и любых данных, не требующих защиты `HttpOnly` и не привязанных к конкретной сессии.
    
