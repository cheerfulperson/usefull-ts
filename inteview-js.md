## Table of Content

- [Node.js](#Node.js)
- [Browser](#Browser)
- [JavaScript](#JavaScript)

## Node.js

 - Multithreading on Node.js. Event Loop [link](https://habr.com/ru/articles/479062/)
 - Node.js event loop, timers and process.nextTick() [link](https://medium.com/devschacht/event-loop-timers-and-nexttick-18579cd122e0) | [en](https://nodejs.org/en/guides/event-loop-timers-and-nexttick)
 - Know your tool: Event Loop in libuv [link](https://habr.com/ru/articles/336498/)
 - Node.js: promise, async/await and process.nextTick [link](https://dev-gang.ru/article/nodejs-vizualizacija-promise-asyncawait-and-processnexttick-qb8eepuukb/)
 - What is `module`? [link](https://nodejs.org/api/modules.html#modules_the_module_object)
 - Difference between module.exports and exports in Node.js [link](https://www.geeksforgeeks.org/difference-between-module-exports-and-exports-in-node-js/)
 - Node Event Emitters [link](https://medium.com/developers-arena/nodejs-event-emitters-for-beginners-and-for-experts-591e3368fdd2)
 - What is a `Cluster` module? [link](https://nodejsdev.ru/api/cluster/#clustersetupprimarysettings)
 - Implementing Node.js Cluster for Improved Performance [link](https://medium.com/@mjdrehman/implementing-node-js-cluster-for-improved-performance-f800146e58e1)
 - Clustering [link](https://nodejsdev.ru/guides/webdraftt/cluster/) | [link](https://habr.com/ru/articles/713420/)
 - Node.js REPL [ru](https://nodejsdev.ru/api/repl/) | [en](https://nodejs.org/api/repl.html#repl)
 - Advantages of single threaded web server over multi-threaded web server? [link_ru](https://habr.com/ru/companies/timeweb/articles/592017/)

## Browser

 - What is Device Pixel Ratio? [link](https://habr.com/ru/sandbox/128978/)
 - How browser work? [ru](https://developer.mozilla.org/ru/docs/Web/Performance/How_browsers_work)

## JavaScript:

- #### Core:

  - JavaScript Internals Basics [link](https://habr.com/ru/companies/alfa/articles/651005/)

- #### Observers:

  - Watch an object for changes in Vanilla JavaScript [link](https://medium.com/@suvechhyabanerjee92/watch-an-object-for-changes-in-vanilla-javascript-a5f1322a4ca5)

- #### Advanced Expressions

  - `Object.is` `(optional)` [link](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is)
  - let var const - differences [link](https://www.freecodecamp.org/news/var-let-and-const-whats-the-difference/)
  - Temporal dead zone
  - Hoisting
  - what is polyfills?

- #### Functions

  - arrow func/ func expression/ func declaration

- #### Objects Built-in methods.

  - Object `keys/values`
  - Know how to use built-in methods

- #### Arrays Built-in methods

  - Know how to copy array
  - Know how to copy a part of array
  - Know how to modify array
  - Know how to flatten nested array

- #### Arrays Iterating, Sorting, Filtering

  - Know how to sort Array
  - Know several method how to iterate Array elements
  - Be able to custom sorting for Array
  - Be able to filter Array elements

### JavaScript in Browser:

- #### Global object window

  - Document (DOM)

- #### Events Basics

  - DOM Events
  - Know basic Event types
  - Mouse / Keyboard Events
  - Form / Input Events
  - Event Listeners
  - Event Phases (difference between them)
  - Custom events `(optional)`

- #### Events Propagation / Preventing

  - Know Event propagation cycle
  - Know how to stop Event propagation (`stopPropagation() / stopImmediatePropagation()`)
  - Know how to prevent Event default browser behavior (`event.preventDefault()`)
  - Delegating
  - Understand Event delegating benefits and drawbacks

- #### Timers

  - setTimeout / setInterval
  - clearTimeout / clearInterval

- #### Web Storage API & cookies

  - LocalStorage
  - SessionStorage

- #### Date & time `(optional)`

  - Date object
  - Date methods, props
  - Timezones `(optional)`
  - Internationalization js (Intl) `(optional)`

### Design patterns:

- #### Intermediate knowledge of patterns and best practices:

  - KISS, DRY, YAGNI

- #### Objects Built-in methods.

  - Know static Object methods
  - Property flags & descriptors (student is able to set property via Object. defineProperty)
  - Know how to create iterable objects, Symbol.iterator usage `(optional)`

- #### ECMAScript Data Types & Expressions

  - Object computed props
  - Be able to loop through Object keys

- #### Functional Scope

  - Know global scope and functional scope
  - Know variables visibility areas
  - Understand nested scopes and able work with them

- #### Functions Parameters / Arguments

  - Know how to define Function parameters
  - Know difference between parameters passing by value and by reference
  - Know how to handle dynamic amount of Function parameters

- #### Closures Advanced

  - Context (lexical environment)
  - Understand function creation context (lexical environment)
  - Be able to explain difference between scope and context
  - Inner/outer lexical environment
  - Understand lexical environment traversing mechanism
  - Understand connection between function and lexical environment

- #### ECMAScript Intermediate

  - Function default parameters
  - Know how to use spread operator for Function arguments
  - Be able to compare `arguments` and `rest parameters`
  - Spread operator for Array
  - Understand and able to use spread operator for Array concatenation
  - Destructuring assignment
  - Be able to discover destructuring assignment concept
  - Understand variables and Function arguments destructuring assignment
  - Know how `for..of` loop works `(optional)`

  #### Modules in JavaScript

  - What is module / module pattern? For what purposes they were created?
  - Modules types (AMD, ES6, CommonJS, UMD).
  - Modules syntax.
  - Common modules features (export default, named exports, exports as, etc).
  - Dynamic imports.

- #### Advanced Functions

  - `this` in functions
  - Reference Type & losing `this`
  - Understand difference between function and method
  - Understand how `this` works, realize `this` possible issues
  - Manage `this`
  - Be able to replace `this` value
  - Be able to use `call` and `apply` Function built-in methods
  - Know how to bind `this` scope to function
  - Binding, binding one function twice

- #### Functional Patterns

  - Callback (Function as argument)
  - Know callback pattern
  - Know IIFE pattern `(optional)`
  - Understand callback limitations (callback hell) `(optional)`
  - Carrying and partial functions

- #### Object Oriented Programming

  - `new` keyword
  - Understand how `new` keyword works
  - Function constructor
  - Know function constructor concept
  - Able to create constructor functions
  - Public, private, static members
  - Know how to create public/static/private members
  - Understand OOP emulation patterns and conventions `(optional)`

- #### ECMAScript Classes

  - Class declaration
  - Know `class` declaration syntax
  - Understand difference between `class` and `constructor function`
  - Getter/setter
  - What does `super()` do and where we have to use it?

- #### Prototypal Inheritance Basics

  - `__proto__` property
  - Understand `__proto__` object property
  - Able to use [Object.create] and define `__proto__` explicitly
  - `prototype` property
  - Know function `prototype` property
  - Understand dependency between function constructor `prototype` and instance `__proto__`
  - Able to create 'class' methods using function `prototype` property
  - Able to set / get object prototype `(optional)`

  - #### ECMAScript Advanced Data Types & Expressions

  - `Set/Map` data types
  - `WeakSet/WeakMap` data types

- #### JavaScript Errors

  - JavaScript Errors (throw, Error class)
  - `try..catch` statement
  - Error handling
  - Error class
  - error logging
  - async error events
  - Custom errors `(optional)`

- #### ECMAScript Advanced

  - Promises
  - Promise states
  - Promise chaining
  - Promise static methods
  - Be able to compare promise and callback patterns `(optional)`
  - Be able to handle errors in promises
  - async/await
  - event loop
  - Garbage collector (concept) `(optional)`

### JavaScript in Browser:

- #### Global object window

  - Location
  - Know browser location structure
  - History API (Global object window)
  - Know browser History APIconcept
  - Be able to navigate within browser history
  - Be able to use history state `(optional)`
  - Navigator `(optional)`
  - Know how to parse user agent `(optional)`
  - Know how to discover client platform, browser
  - Cookies

- #### Page Lifecycle

  - Parsing
  - Reflow
  - Repaint
  - Critical rendering path (CRP) `(optional)`

  - #### Events Basics `(optional)`

  - Custom events `(optional)`

- #### Web components `(optional)`

  - Web components, shadow DOM (concept) `(optional)`

- #### Network requests

  - `Fetch` (with usage)
  - `XMLHTTPRequest` (concept) `(optional)`
  - `WebSocket` (concept) `(optional)`

- #### Timers `(optional)`

  - `requestAnimationFrame` `(optional)`
  - Be able to explain difference between `setTimeout` and `requestAnimationFrame` `(optional)`

- #### Web Storage API & cookies

  - Cookies
  - Difference between localStorage, sessionStorage and cookies

### Typescript:

- #### Ability to write concise TypeScript code using its constructs
  - basic types
  - enums
  - type / interface, differences between them
  - using interfaces with optional properties, read-only properties, etc...
  - function types
  - utitily types `(optional)`
  - typeguards `(optional)`
  - creating custom types
  - generic types (concept)
  - understanding TS (ES6) module system

### Design patterns:

- Creational Design Patterns
- Structural Design Patterns
- Behavioral Design Patterns
- MVC `(optional)`

- #### Intermediate knowledge of patterns and best practices:

  - SOLID principles
  - design patterns used on a student's project, and able to compare these patterns `(optional)`

- #### Software Development Methodologies `(optional)`

  - Agile
  - Scrum / Kanban / Waterfall
  - Estimation

### Testing `(optional)`

- Testing Types
  - Integration Testing
  - E2E
  - Security Testing
  - Perforamance Testing
- Test Pyramid
- Testing approaches `(optional)`
- FIRST
- TDD и BDD
- Frameworks `(optional)`

### Web Communication Protocols: `(optional)`

- #### HTTP vs HTTPS
- #### HTTP 1.x, 2.x, 3.x
- #### HTTP methods, headers, responses, body
- #### HTTP status codes groups (1xx, 2xx, 3xx, 4xx, 5xx)
- #### RESTful API

### Common web-security knowledge `(optional)`

- #### Basic understanding of most common security terms (CORS, XSS) `(optional)`

  - XSS
  - CORS
  - OWASP Top 10
  - Auth (JWT, OAuth, Basic, etc.)

### Coding tasks:

- `Function.prototype.bind` implement polyfill
- `Object.create` implement polyfill
- `Array.flat` implement polyfill
- `Array.reduce` implement polyfill
- `'hello world'.repeating(3)` -> 'hello world hello world hello world'. How to implement?
- `myFunc('!', 4, -10, 34, 0)` -> '4!-10!34!0`. How to implement?
- `five(plus(seven(minus(three()))))` -> 9. How to implement?
- add(5)(9)(-4)(1) -> 11. How to implement?
- `periodOutput(period)` method should output in the console once per every period how mach time has passed since the first function call.
  Example:
  `periodOutput(100) -> 100(after 100 ms), 200(after 100 ms), 300(after 100 ms), ...`
- `extendedPeriodOutput(period)` method should output in the console once per period how mach time has passed since the first function call and then increase the period. Example: `// extendedPeriodOutput(100) -> 100(after 100 ms), 200(after 200 ms), 300(after 300 ms)`
