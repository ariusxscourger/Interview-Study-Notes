# JAVASCRIPT FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: JavaScript Core Language Features

### WHY? (Problem Statement)

**JavaScript is the language of the web - you cannot escape it:**
- Frontend: React, Vue, Angular, Svelte all compile to JS
- Backend: Node.js, Deno, Bun
- Mobile: React Native, Capacitor, NativeScript
- Desktop: Electron, Tauri
- Edge: Cloudflare Workers, Vercel Edge Functions

**What Deep JS Knowledge Solves:**
| Problem | Understanding Needed |
|---------|---------------------|
| "Why is `this` undefined?" | Execution context, binding rules |
| "Why does my async code run out of order?" | Event loop, microtasks, macrotasks |
| "Memory leak in production" | Closures, references, GC |
| "TypeError: Cannot read property of undefined" | Prototypes, optional chaining, nullish coalescing |
| "Race condition in API calls" | Promises, AbortController, concurrency |

**Real-World Analogy:**
- JavaScript = English (ubiquitous, flexible, quirky)
- TypeScript = English with grammar checker
- Runtime = Conversation partner (V8, SpiderMonkey, JavaScriptCore)

---

### HOW? (Internal Mechanism)

#### 1. **Execution Context & Call Stack**

```
┌─────────────────────────────────────────────────────────────┐
│                    CALL STACK (LIFO)                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Global Execution Context (GEC)                             │
│  ├── Creation Phase:                                        │
│  │   ├── Create Global Object (window/global)              │
│  │   ├── Create `this` = Global Object                     │
│  │   ├── Hoist variables (var → undefined, let/const → TDZ)│
│  │   ├── Hoist functions (full declaration)                │
│  │   └── Scope Chain = [Global]                            │
│  │                                                         │
│  └── Execution Phase: Line-by-line execution               │
│                                                             │
│  Function Execution Context (FEC) - Created per call       │
│  ├── Creation Phase:                                        │
│  │   ├── Create Arguments Object                           │
│  │   ├── Create `this` (depends on call site)              │
│  │   ├── Hoist local variables/functions                   │
│  │   └── Scope Chain = [Local, ...Outer, Global]           │
│  │                                                         │
│  └── Execution Phase                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **Event Loop (The Heart of JS Concurrency)**

```
┌─────────────────────────────────────────────────────────────────┐
│                        MAIN THREAD                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │
│  │ Call Stack  │    │  Microtask  │    │    Macrotask        │ │
│  │  (Sync)     │    │   Queue     │    │    Queue            │ │
│  │             │    │  (Promises, │    │  (setTimeout,       │ │
│  │             │    │   queueMicro │    │   setInterval,     │ │
│  │             │    │   task,     │    │   I/O, UI render)  │ │
│  │             │    │   MutationObs)│   │                     │ │
│  └──────┬──────┘    └──────┬──────┘    └─────────┬───────────┘ │
│         │                  │                      │            │
│         ▼                  ▼                      ▼            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    EVENT LOOP                           │  │
│  │  while (stack not empty) execute stack                 │  │
│  │  while (microtask queue not empty) execute microtask   │  │
│  │  execute ONE macrotask                                  │  │
│  │  render if needed                                       │  │
│  │  repeat                                                 │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

ORDER OF EXECUTION:
1. Sync code (Call Stack)
2. All microtasks (Promise .then, queueMicrotask)
3. ONE macrotask (setTimeout callback)
4. Render
5. Repeat from step 2
```

#### 3. **Prototypal Inheritance**

```
Object Creation & Prototype Chain:

const animal = { 
  alive: true,
  breathe() { console.log('breathing'); }
};

const dog = Object.create(animal);
dog.bark = () => console.log('woof');
dog.alive;        // true (inherited)
dog.breathe();    // 'breathing' (inherited)

dog.__proto__ === animal;  // true
dog.__proto__.__proto__ === Object.prototype;  // true
dog.__proto__.__proto__.__proto__ === null;    // true

// Class syntax (syntactic sugar)
class Animal {
  constructor(name) { this.name = name; }
  breathe() { console.log('breathing'); }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);  // Calls Animal constructor
    this.breed = breed;
  }
  bark() { console.log('woof'); }
}
```

#### 4. **Closures & Lexical Scoping**

```
function outer(x) {
  const secret = 'hidden';
  
  function inner(y) {
    // Closure: inner remembers outer's scope
    return x + y + secret.length;
  }
  
  return inner;
}

const fn = outer(10);
fn(5);  // 10 + 5 + 6 = 21

// Practical use: Module pattern, memoization, event handlers
function createCounter() {
  let count = 0;  // Private, enclosed
  return {
    increment: () => ++count,
    decrement: () => --count,
    getValue: () => count
  };
}
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Variables & Types**

```javascript
// Primitives (immutable, passed by value)
const str = 'hello';
const num = 42;
const bool = true;
const und = undefined;
const nul = null;
const sym = Symbol('id');
const big = 9007199254740991n;

// Objects (mutable, passed by reference)
const obj = { a: 1 };
const arr = [1, 2, 3];
const fn = () => {};
const date = new Date();
const regex = /pattern/;
const map = new Map();
const set = new Set();
const weakMap = new WeakMap();
const weakSet = new WeakSet();

// Type Detection
typeof 'str'        // 'string'
typeof 42           // 'number'
typeof true         // 'boolean'
typeof undefined    // 'undefined'
typeof null         // 'object' (bug)
typeof Symbol()     // 'symbol'
typeof 1n           // 'bigint'
typeof {}           // 'object'
typeof []           // 'object'
typeof () => {}     // 'function'

// Better type checking
Array.isArray([])           // true
Object.prototype.toString.call([])  // '[object Array]'
Object.prototype.toString.call({})  // '[object Object]'
Object.prototype.toString.call(null) // '[object Null]'
```

#### 2. **Equality & Comparison**

```javascript
// Strict Equality (ALWAYS USE)
1 === 1        // true
1 === '1'      // false
null === undefined  // false
NaN === NaN    // false

// Loose Equality (NEVER USE)
1 == '1'       // true
null == undefined  // true
'' == 0        // true
'0' == false   // true

// Object Equality
{} === {}      // false (different references)
const a = {}; a === a  // true

// Object.is (handles NaN, -0)
Object.is(NaN, NaN)        // true
Object.is(0, -0)           // false
Object.is(0, 0)            // true

// Comparison
'a' > 'b'   // false (lexicographic)
'10' > '2'  // false (string comparison!)
10 > 2      // true
```

#### 3. **Functions - All Patterns**

```javascript
// 1. Function Declaration (hoisted)
function add(a, b) { return a + b; }

// 2. Function Expression (not hoisted)
const subtract = function(a, b) { return a - b; };

// 3. Arrow Function (no this, no arguments, no prototype)
const multiply = (a, b) => a * b;
const divide = (a, b) => {
  if (b === 0) throw new Error('Divide by zero');
  return a / b;
};

// 4. Default Parameters
function greet(name = 'Guest', greeting = 'Hello') {
  return `${greeting}, ${name}!`;
}

// 5. Rest Parameters
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4);  // 10

// 6. Destructuring Parameters
function createUser({ name, email, age = 18 } = {}) {
  return { name, email, age, id: crypto.randomUUID() };
}

// 7. IIFE (Immediately Invoked Function Expression)
const result = (function() {
  const private = 'secret';
  return { getPrivate: () => private };
})();

// 8. Higher-Order Functions
const withLogging = (fn) => (...args) => {
  console.log('Calling', fn.name, args);
  const result = fn(...args);
  console.log('Result', result);
  return result;
};

// 9. Function Properties
function factory() {}
factory.displayName = 'MyComponent';
factory.defaultProps = { value: 0 };
```

#### 4. **Async JavaScript - Complete Patterns**

```javascript
// PROMISES
const fetchUser = (id) => 
  fetch(`/api/users/${id}`)
    .then(res => {
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    })
    .then(data => data.user)
    .catch(err => {
      console.error('Fetch failed:', err);
      throw err;  // Re-throw for caller
    });

// Promise Utilities
Promise.all([fetchUser(1), fetchUser(2)])      // All succeed or first fails
Promise.allSettled([p1, p2, p3])               // Wait for all, get status
Promise.race([fetchUser(1), timeout(5000)])    // First to settle
Promise.any([p1, p2, p3])                      // First to fulfill

// ASYNC/AWAIT (syntactic sugar)
async function getUserData(userId) {
  try {
    const [user, posts, comments] = await Promise.all([
      fetchUser(userId),
      fetchPosts(userId),
      fetchComments(userId)
    ]);
    return { user, posts, comments };
  } catch (error) {
    // Handle any error from any promise
    logger.error('getUserData failed', { userId, error });
    throw new UserDataError(userId, error);
  }
}

// Parallel vs Sequential
// SEQUENTIAL (slow)
const user = await fetchUser(id);
const posts = await fetchPosts(user.id);

// PARALLEL (fast)
const [user, posts] = await Promise.all([
  fetchUser(id),
  fetchPosts(id)
]);

// Sequential with dependency
const user = await fetchUser(id);
const posts = await fetchPosts(user.id);  // Needs user.id

// ERROR HANDLING PATTERNS
// 1. Try/catch per await
try { const data = await fetchData(); }
catch (err) { handleError(err); }

// 2. Wrapper function
const safeAsync = async (promise) => {
  try { return [await promise, null]; }
  catch (err) { return [null, err]; }
};
const [data, error] = await safeAsync(riskyCall());

// 3. AbortController for cancellation
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch(url, { signal: controller.signal });
} catch (err) {
  if (err.name === 'AbortError') { /* handle timeout */ }
} finally {
  clearTimeout(timeout);
}

// GENERATORS (for complex async flows)
function* fetchAllPages(url) {
  let nextUrl = url;
  while (nextUrl) {
    const response = yield fetch(nextUrl).then(r => r.json());
    yield response.data;
    nextUrl = response.nextPage;
  }
}

// Usage
const iterator = fetchAllPages('/api/items');
let result = iterator.next();
while (!result.done) {
  const page = await result.value;
  process(page);
  result = iterator.next(page);
}
```

#### 5. **Array & Object Methods (Must Know)**

```javascript
// ARRAY TRANSFORMATION
const users = [
  { id: 1, name: 'Alice', age: 25, active: true },
  { id: 2, name: 'Bob', age: 30, active: false },
  { id: 3, name: 'Charlie', age: 35, active: true }
];

// Map - transform each element
const names = users.map(u => u.name);  // ['Alice', 'Bob', 'Charlie']

// Filter - select elements
const active = users.filter(u => u.active);  // [{id:1...}, {id:3...}]

// Reduce - accumulate to single value
const totalAge = users.reduce((sum, u) => sum + u.age, 0);  // 90
const byId = users.reduce((acc, u) => ({ ...acc, [u.id]: u }), {});

// Find - first match
const alice = users.find(u => u.name === 'Alice');  // {id:1...}

// FindIndex - index of first match
const idx = users.findIndex(u => u.age > 30);  // 2

// Some/Every - boolean checks
const hasActive = users.some(u => u.active);  // true
const allActive = users.every(u => u.active);  // false

// Flat/Map - flatten arrays
const nested = [[1, 2], [3, 4]];
const flat = nested.flat();  // [1, 2, 3, 4]
const flatMap = users.flatMap(u => [u.name, u.age]);  // ['Alice', 25, 'Bob', 30, ...]

// Sort - MUTATES ORIGINAL
users.sort((a, b) => a.age - b.age);  // Ascending age

// Object Methods
const obj = { a: 1, b: 2, c: 3 };
Object.keys(obj);    // ['a', 'b', 'c']
Object.values(obj);  // [1, 2, 3]
Object.entries(obj); // [['a', 1], ['b', 2], ['c', 3]]
Object.fromEntries([['a', 1], ['b', 2]]);  // { a: 1, b: 2 }

// Merge objects
const merged = { ...obj, d: 4 };  // { a: 1, b: 2, c: 3, d: 4 }
Object.assign({}, obj, { d: 4 });  // Same

// Freeze/Seal
Object.freeze(obj);   // Immutable (shallow)
Object.seal(obj);     // Can't add/delete, can modify
```

#### 6. **Modern Syntax (ES2020+)**

```javascript
// Optional Chaining
user?.profile?.settings?.theme  // undefined if any null/undefined

// Nullish Coalescing
const theme = user.settings?.theme ?? 'light';  // Only falls back for null/undefined

// Logical Assignment
a ??= b;  // a = a ?? b
a ||= b;  // a = a || b
a &&= b;  // a = a && b

// Private Class Fields
class Wallet {
  #balance = 0;  // Truly private
  
  getBalance() { return this.#balance; }
  
  deposit(amount) {
    if (amount <= 0) throw new Error('Invalid amount');
    this.#balance += amount;
  }
  
  #validateAmount(amount) { return amount > 0; }
}

// Static Private
class Config {
  static #instance = null;
  static getInstance() {
    return this.#instance ??= new Config();
  }
}

// Top-level Await (modules)
const data = await fetch('/api/data').then(r => r.json());

// Array/Object destructuring with defaults
const { name = 'Anonymous', age = 0, ...rest } = user;
const [first, second, ...others] = array;

// Import/Export
export const PI = 3.14159;
export function circleArea(r) { return PI * r * r; }
export default class Circle { }

// Dynamic Import (code splitting)
const module = await import('./heavy-module.js');
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| `const` by default, `let` if needed | `var` everywhere | Block scope, hoisting issues |
| `===` / `!==` | `==` / `!=` | Type coercion bugs |
| Optional chaining `?.` | `user && user.profile && user.profile.name` | Verbose, error-prone |
| Nullish coalescing `??` | `||` for defaults | `||` catches `0`, `''`, `false` |
| `Promise.all` for parallel | Sequential `await` in loop | Unnecessarily slow |
| `Array.from` / spread for copying | `arr.slice()` for objects | Shallow copy confusion |
| Early returns | Deep nesting | Cognitive complexity |
| Named exports | Default exports only | Refactoring, tree-shaking |
| Functional array methods | `for` loops with index | Mutations, off-by-one errors |

---

### INTERVIEW QUESTIONS (Top 20)

#### 1. **Conceptual**: "Explain the Event Loop"

**Answer:**
```
JavaScript is single-threaded but non-blocking.

Call Stack: Executes sync code (LIFO)
Web APIs: setTimeout, fetch, DOM events (browser provides)
Callback Queue: Macrotasks (setTimeout, I/O)
Microtask Queue: Promises, queueMicrotask, MutationObserver

Event Loop Algorithm:
1. Execute Call Stack until empty
2. Execute ALL microtasks (drain queue)
3. Execute ONE macrotask
4. Render (if needed)
5. Repeat from step 1

Example Output:
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
console.log('5');

// Output: 1, 5, 3, 4, 2
```

#### 2. **Code**: "What does this output?"

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}

// Output: 3, 3, 3 (var is function-scoped, loop ends with i=3)

// Fix 1: let (block-scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2

// Fix 2: IIFE
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 100))(i);
}
// Output: 0, 1, 2

// Fix 3: bind
for (var i = 0; i < 3; i++) {
  setTimeout(console.log.bind(console, i), 100);
}
```

#### 3. **Code**: "Implement `debounce` and `throttle`"

```javascript
// Debounce: Wait until calls stop
function debounce(fn, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Usage: Search input
const debouncedSearch = debounce(query => {
  fetchResults(query);
}, 300);

// Throttle: Limit rate
function throttle(fn, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// Usage: Scroll handler
const throttledScroll = throttle(() => {
  updatePosition();
}, 100);

// Advanced: Leading + Trailing edge
function throttleAdvanced(fn, delay, { leading = true, trailing = true } = {}) {
  let timeoutId;
  let lastArgs;
  let lastThis;
  let lastTime = 0;
  
  const invoke = () => {
    fn.apply(lastThis, lastArgs);
    lastTime = Date.now();
  };
  
  return function(...args) {
    const now = Date.now();
    const remaining = delay - (now - lastTime);
    
    lastArgs = args;
    lastThis = this;
    
    if (leading && remaining <= 0) {
      invoke();
    } else if (trailing && !timeoutId) {
      timeoutId = setTimeout(() => {
        timeoutId = null;
        if (trailing) invoke();
      }, remaining);
    }
  };
}
```

#### 4. **Code**: "Implement `Promise.all` from scratch"

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completed = 0;
    
    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(
        value => {
          results[index] = value;
          completed++;
          if (completed === promises.length) {
            resolve(results);
          }
        },
        reject  // Reject immediately on first failure
      );
    });
    
    // Handle empty array
    if (promises.length === 0) resolve(results);
  });
}

// Promise.allSettled
function promiseAllSettled(promises) {
  return Promise.all(
    promises.map(p => Promise.resolve(p)
      .then(value => ({ status: 'fulfilled', value }))
      .catch(reason => ({ status: 'rejected', reason }))
    )
  );
}
```

#### 5. **Conceptual**: "Closure - explain with practical example"

```
A closure is a function that remembers its lexical scope 
even when executed outside that scope.

PRACTICAL USE CASES:

1. DATA PRIVACY (Module Pattern)
const Counter = (() => {
  let count = 0;  // Private!
  return {
    increment: () => ++count,
    getCount: () => count
  };
})();

2. MEMOIZATION
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

3. EVENT HANDLERS WITH STATE
function createClickHandler(element) {
  let clicks = 0;
  return function handler() {
    clicks++;
    console.log(`Clicked ${clicks} times`);
  };
}

4. CURRYING
const add = a => b => a + b;
const add5 = add(5);
add5(3);  // 8
```

#### 6. **Code**: "Deep clone an object"

```javascript
// Simple (loses functions, dates, regex, circular refs)
const deepClone = obj => JSON.parse(JSON.stringify(obj));

// Robust implementation
function deepClone(obj, visited = new WeakMap()) {
  // Primitives
  if (obj === null || typeof obj !== 'object') return obj;
  
  // Circular reference
  if (visited.has(obj)) return visited.get(obj);
  
  // Special objects
  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof RegExp) return new RegExp(obj);
  if (obj instanceof Map) return new Map(obj);
  if (obj instanceof Set) return new Set(obj);
  if (obj instanceof ArrayBuffer) return obj.slice(0);
  
  // Array or Object
  const clone = Array.isArray(obj) ? [] : {};
  visited.set(obj, clone);
  
  for (const key of Object.keys(obj)) {
    clone[key] = deepClone(obj[key], visited);
  }
  
  // Preserve prototype
  Object.setPrototypeOf(clone, Object.getPrototypeOf(obj));
  
  // Symbol keys
  for (const sym of Object.getOwnPropertySymbols(obj)) {
    clone[sym] = deepClone(obj[sym], visited);
  }
  
  return clone;
}

// Modern: structuredClone (browser/Node 17+)
const cloned = structuredClone(original);
```

#### 7. **Architecture**: "Event Emitter Pattern"

```javascript
class EventEmitter {
  constructor() {
    this.events = new Map();
  }
  
  on(event, listener) {
    if (!this.events.has(event)) this.events.set(event, new Set());
    this.events.get(event).add(listener);
    return () => this.off(event, listener); // Return unsubscribe
  }
  
  off(event, listener) {
    this.events.get(event)?.delete(listener);
  }
  
  emit(event, ...args) {
    this.events.get(event)?.forEach(listener => {
      try { listener(...args); }
      catch (err) { console.error(`Error in ${event}:`, err); }
    });
  }
  
  once(event, listener) {
    const onceWrapper = (...args) => {
      listener(...args);
      this.off(event, onceWrapper);
    };
    this.on(event, onceWrapper);
  }
}

// Usage
const emitter = new EventEmitter();
const unsubscribe = emitter.on('data', (data) => console.log(data));
emitter.emit('data', { id: 1 });
unsubscribe();
```

#### 8. **Performance**: "Memory Leak Patterns & Fixes"

```javascript
// 1. Forgotten Timers
// LEAK
setInterval(() => doSomething(), 1000);

// FIX
const timer = setInterval(() => doSomething(), 1000);
clearInterval(timer);  // On cleanup

// 2. Event Listeners Not Removed
// LEAK
element.addEventListener('click', handler);

// FIX
element.addEventListener('click', handler);
// Later:
element.removeEventListener('click', handler);
// Or use AbortController
const controller = new AbortController();
element.addEventListener('click', handler, { signal: controller.signal });
controller.abort();

// 3. Closures Holding References
// LEAK
function createHandler() {
  const hugeData = fetchHugeData();  // Kept in closure
  return () => console.log('click');
}

// FIX
function createHandler() {
  return () => {
    const hugeData = fetchHugeData();  // Fetch when needed
    console.log('click');
  };
}

// 4. Global Cache Without Eviction
// LEAK
const cache = new Map();
function getData(key) {
  if (!cache.has(key)) cache.set(key, expensiveCompute(key));
  return cache.get(key);
}

// FIX - LRU Cache with size limit
class LRUCache {
  constructor(maxSize = 100) {
    this.cache = new Map();
    this.maxSize = maxSize;
  }
  
  get(key) {
    if (!this.cache.has(key)) return null;
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);  // Move to end (most recent)
    return value;
  }
  
  set(key, value) {
    if (this.cache.has(key)) this.cache.delete(key);
    else if (this.cache.size >= this.maxSize) {
      this.cache.delete(this.cache.keys().next().value); // Remove oldest
    }
    this.cache.set(key, value);
  }
}

// 5. WeakMap/WeakSet for Metadata (auto cleanup)
const metadata = new WeakMap();
function attachMeta(obj, data) {
  metadata.set(obj, data);  // Doesn't prevent GC
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **`this` binding**: Arrow functions don't have `this` - they inherit from enclosing scope
- [ ] **`arguments` object**: Not available in arrow functions - use rest params
- [ ] **Automatic Semicolon Insertion (ASI)**: Can break `return\nvalue` → returns `undefined`
- [ ] **Floating Point**: `0.1 + 0.2 === 0.30000000000000004` → use `Number.EPSILON` or integer math
- [ ] **Array `sort()` mutates**: Always `[...arr].sort()` if you need original
- [ ] **`for...in` vs `for...of`**: `in` = keys (strings), `of` = values
- [ ] **`map` vs `forEach`**: `map` returns new array, `forEach` returns undefined
- [ ] **Promise constructor anti-pattern**: `new Promise(resolve => resolve(fetch(...)))` → just return the promise
- [ ] **Top-level `await`**: Only in modules (`type: "module"` or `.mjs`)
- [ ] **`Object.freeze` is shallow**: Nested objects still mutable

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Utility Functions**
```javascript
// Pipe - compose functions left to right
const pipe = (...fns) => x => fns.reduce((v, f) => f(v), x);

// Compose - right to left
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);

// Deep get/set
const get = (obj, path, defaultValue) => 
  path.split('.').reduce((o, k) => (o && o[k] !== undefined ? o[k] : defaultValue), obj);

const set = (obj, path, value) => {
  const keys = path.split('.');
  const last = keys.pop();
  const target = keys.reduce((o, k) => o[k] = o[k] || {}, obj);
  target[last] = value;
  return obj;
};

// Retry with backoff
async function retry(fn, retries = 3, delay = 1000) {
  try { return await fn(); }
  catch (err) {
    if (retries === 0) throw err;
    await new Promise(r => setTimeout(r, delay));
    return retry(fn, retries - 1, delay * 2);
  }
}

// Type Guards
const isString = (v): v is string => typeof v === 'string';
const isNumber = (v): v is number => typeof v === 'number' && !isNaN(v);
const isPlainObject = (v): v is Record<string, unknown> => 
  Object.prototype.toString.call(v) === '[object Object]';
```

#### 2. **Error Handling**
```javascript
// Custom Error Classes
class AppError extends Error {
  constructor(message, public code: string, public statusCode = 500) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message, public fields: string[]) {
    super(message, 'VALIDATION_ERROR', 400);
  }
}

class NotFoundError extends AppError {
  constructor(resource, id) {
    super(`${resource} not found: ${id}`, 'NOT_FOUND', 404);
  }
}

// Global Error Handler (Express-style)
function errorHandler(err, req, res, next) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: err.message, code: err.code });
  }
  console.error('Unexpected error:', err);
  res.status(500).json({ error: 'Internal server error', code: 'INTERNAL_ERROR' });
}
```

#### 3. **Testing Helpers**
```javascript
// Mock timers
jest.useFakeTimers();
await act(async () => {
  fireEvent.click(button);
  jest.advanceTimersByTime(300);
});
jest.useRealTimers();

// Mock fetch
global.fetch = jest.fn(() => Promise.resolve({
  ok: true,
  json: () => Promise.resolve({ data: 'test' })
}));

// Async test wrapper
const waitFor = (fn, { timeout = 1000, interval = 50 } = {}) => {
  const start = Date.now();
  return new Promise((resolve, reject) => {
    const check = () => {
      try { resolve(fn()); }
      catch (e) {
        if (Date.now() - start > timeout) reject(e);
        else setTimeout(check, interval);
      }
    };
    check();
  });
};
```

---

### PRACTICE PROBLEMS

1. **Implement**: LRU Cache with O(1) get/set
2. **Build**: Event emitter with `once`, `off`, wildcard events
3. **Create**: Promise-based rate limiter (max N concurrent)
4. **Debug**: Find memory leak in event listener code
5. **Write**: Type-safe `Object.keys` / `Object.entries` helpers
6. **Implement**: Pub/Sub with message history replay

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Event Loop Order | Sync → All Microtasks → 1 Macrotask → Render → Repeat |
| Microtask Examples | Promise.then, queueMicrotask, MutationObserver |
| Macrotask Examples | setTimeout, setInterval, I/O, UI Events |
| Promise.all vs allSettled | all: fails fast. allSettled: waits for all, returns status |
| Debounce vs Throttle | Debounce: wait for pause. Throttle: limit rate |
| Closure Definition | Function + lexical environment where it was created |
| var vs let vs const | var: function scope, hoisted. let/const: block scope, TDZ |
| == vs === | ==: type coercion. ===: strict (no coercion) |
| Optional Chaining | obj?.prop?.nested - short circuits on null/undefined |
| Nullish Coalescing | a ?? b - only falls back for null/undefined |
| Array.sort() | MUTATES original. Returns reference to same array |
| Structured Clone | structuredClone() - deep clone, handles circular, dates, maps |
| WeakMap Use Case | Private data, metadata - doesn't prevent GC |
| this in Arrow Fn | Inherits from enclosing lexical scope |

---

## NEXT TOPIC: `05-JAVASCRIPT-ADVANCED.md`

> **Study Tip**: Open browser console, run the Event Loop examples. Predict output before running. Build a Promise library from scratch.