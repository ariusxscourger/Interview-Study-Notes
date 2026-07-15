# JAVASCRIPT ADVANCED PATTERNS
## Exam-Style Study Notes

---

## TOPIC: Advanced JavaScript Patterns & Concepts

### WHY? (Problem Statement)

**Beyond Basics - What Separates Senior from Junior:**
- Architecture decisions that scale
- Performance optimization techniques
- Type safety without TypeScript
- Testing strategies
- Framework internals understanding

**What Advanced JS Solves:**
| Problem | Advanced Pattern |
|---------|------------------|
| Spaghetti callback code | Promises, async/await, generators |
| Prop drilling | Context, State Management |
| Memory leaks | WeakRef, FinalizationRegistry |
| Type errors at runtime | JSDoc, Type Guards, Branded Types |
| Bundle size | Code splitting, Tree shaking |
| Race conditions | AbortController, Request deduplication |

---

### HOW? (Internal Mechanism)

#### 1. **Proxy & Reflect - Meta Programming**

```javascript
// Proxy wraps object to intercept operations
const target = { name: 'Alice', age: 30 };

const handler = {
  get(target, prop, receiver) {
    console.log(`Getting ${prop}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    console.log(`Setting ${prop} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  },
  has(target, prop) {
    return prop.startsWith('_') ? false : Reflect.has(target, prop);
  },
  deleteProperty(target, prop) {
    if (prop === 'id') throw new Error('Cannot delete id');
    return Reflect.deleteProperty(target, prop);
  },
  ownKeys(target) {
    return Reflect.ownKeys(target).filter(k => !k.toString().startsWith('_'));
  }
};

const proxy = new Proxy(target, handler);
proxy.name;      // Logs, returns 'Alice'
proxy.age = 31;  // Logs, sets
'name' in proxy; // true
'_secret' in proxy; // false (hidden)
```

**Real-World Uses:**
- Validation/Reactive systems (Vue 3)
- API request/response transformation
- Immutable data structures
- Lazy loading / Virtual objects

#### 2. **WeakRef & FinalizationRegistry - Memory Management**

```javascript
// WeakRef - doesn't prevent GC
class Cache {
  #cache = new Map();
  
  set(key, value) {
    this.#cache.set(key, new WeakRef(value));
  }
  
  get(key) {
    const ref = this.#cache.get(key);
    return ref?.deref(); // Returns undefined if GC'd
  }
}

// FinalizationRegistry - cleanup after GC
const registry = new FinalizationRegistry((heldValue) => {
  console.log('Object garbage collected:', heldValue);
  // Cleanup: close connections, remove listeners, etc.
});

class Resource {
  constructor(id) {
    this.id = id;
    registry.register(this, id, this); // heldValue = this
  }
  
  close() {
    registry.unregister(this);
  }
}

// Usage
const resource = new Resource('db-connection-1');
resource = null; // Eventually GC runs, calls callback
```

#### 3. **Generators & Iterators - Lazy Evaluation**

```javascript
// Generator function - returns iterator
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
fib.next();  // { value: 0, done: false }
fib.next();  // { value: 1, done: false }
fib.next();  // { value: 1, done: false }

// Practical: Infinite scroll pagination
async function* fetchAllPages(url) {
  let nextUrl = url;
  while (nextUrl) {
    const response = await fetch(nextUrl);
    const data = await response.json();
    yield* data.items;
    nextUrl = data.nextPage;
  }
}

// Consume lazily
for await (const item of fetchAllPages('/api/items')) {
  renderItem(item);
  if (shouldStop) break; // Stops fetching!
}

// Iterator Helpers (ES2024 / Stage 3)
const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.values()
  .filter(n => n % 2 === 0)
  .map(n => n * 2)
  .take(3); // [4, 8] (lazy!)
```

#### 4. **Async Iterators & Streams**

```javascript
// Async Generator
async function* readLines(fileHandle) {
  for await (const chunk of fileHandle) {
    const lines = chunk.split('\n');
    for (const line of lines) yield line;
  }
}

// Transform Stream
const transformStream = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk.toUpperCase());
  }
});

// Readable Stream from Async Iterator
const readable = new ReadableStream({
  async start(controller) {
    for await (const item of fetchAllPages('/api/items')) {
      controller.enqueue(item);
    }
    controller.close();
  }
});

// Pipe through transforms
fetch('/api/data')
  .then(res => res.body)
  .then(body => body
    .pipeThrough(new TextDecoderStream())
    .pipeThrough(transformStream)
    .pipeTo(new WritableStream({
      write(chunk) { console.log(chunk); }
    }))
  );
```

---

### WHAT? (Key Patterns & APIs)

#### 1. **Module Patterns for Organization**

```javascript
// Revealing Module Pattern (ES5 style - still useful)
const ApiClient = (() => {
  // Private
  const baseUrl = '/api/v1';
  const cache = new Map();
  
  async function request(endpoint, options = {}) {
    const url = `${baseUrl}${endpoint}`;
    const response = await fetch(url, {
      headers: { 'Content-Type': 'application/json', ...options.headers },
      ...options
    });
    if (!response.ok) throw new ApiError(response.status);
    return response.json();
  }
  
  // Public API
  return {
    get: (endpoint) => request(endpoint),
    post: (endpoint, data) => request(endpoint, { method: 'POST', body: JSON.stringify(data) }),
    clearCache: () => cache.clear()
  };
})();

// Modern ES Modules (Preferred)
// api/client.js
export const baseUrl = '/api/v1';
export async function request(endpoint, options = {}) { /* ... */ }
export const get = (endpoint) => request(endpoint);
export const post = (endpoint, data) => request(endpoint, { method: 'POST', body: JSON.stringify(data) });

// Barrel export
// api/index.js
export * from './client.js';
export * from './errors.js';
```

#### 2. **State Management Patterns**

```javascript
// 1. Simple Observable Store
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();
  
  return {
    getState: () => state,
    setState: (partial) => {
      state = typeof partial === 'function' ? partial(state) : { ...state, ...partial };
      listeners.forEach(fn => fn(state));
    },
    subscribe: (fn) => {
      listeners.add(fn);
      return () => listeners.delete(fn);
    }
  };
}

// Usage
const store = createStore({ user: null, theme: 'light' });
store.subscribe(state => console.log('State changed:', state));
store.setState({ theme: 'dark' });

// 2. Redux-like with Middleware
function createStore(reducer, preloadedState, enhancer) {
  if (enhancer) return enhancer(createStore)(reducer, preloadedState);
  
  let state = preloadedState;
  const listeners = [];
  
  return {
    getState: () => state,
    dispatch: (action) => {
      state = reducer(state, action);
      listeners.forEach(fn => fn());
      return action;
    },
    subscribe: (fn) => { listeners.push(fn); return () => { /* remove */ }; }
  };
}

// Middleware
const logger = store => next => action => {
  console.log('dispatching', action);
  const result = next(action);
  console.log('next state', store.getState());
  return result;
};

const thunk = store => next => action => 
  typeof action === 'function' ? action(store.dispatch, store.getState) : next(action);

// 3. Signals (Fine-grained reactivity - SolidJS style)
function createSignal(initialValue) {
  let value = initialValue;
  const subscribers = new Set();
  
  const read = () => {
    // Track dependency if in tracking context
    if (currentSubscriber) subscribers.add(currentSubscriber);
    return value;
  };
  
  const write = (newValue) => {
    value = typeof newValue === 'function' ? newValue(value) : newValue;
    subscribers.forEach(fn => fn());
  };
  
  return [read, write];
}
```

#### 3. **Type-Safe JavaScript (JSDoc + Type Guards)**

```javascript
/**
 * @typedef {Object} User
 * @property {string} id
 * @property {string} name
 * @property {string} email
 * @property {'admin'|'user'|'guest'} role
 * @property {Date} createdAt
 */

/**
 * @param {User} user
 * @returns {string}
 */
function formatUser(user) {
  return `${user.name} (${user.role})`;
}

/**
 * @template T
 * @typedef {Object} Result
 * @property {boolean} ok
 * @property {T} [value]
 * @property {Error} [error]
 */

/**
 * @template T
 * @param {Promise<T>} promise
 * @returns {Promise<Result<T>>}
 */
async function to(promise) {
  try {
    const value = await promise;
    return { ok: true, value };
  } catch (error) {
    return { ok: false, error };
  }
}

// Branded Types (Nominal Typing)
/** @typedef {string & { readonly _brand: unique symbol }} UserId */
/** @typedef {string & { readonly _brand: unique symbol }} PostId */

/**
 * @param {string} id
 * @returns {UserId}
 */
function createUserId(id) { return /** @type {UserId} */(id); }

// Type Guards
/**
 * @param {unknown} value
 * @returns {value is User}
 */
function isUser(value) {
  return (
    typeof value === 'object' && value !== null &&
    'id' in value && 'name' in value && 'email' in value
  );
}

// Discriminated Union
/** @typedef {{ type: 'success', data: User } | { type: 'error', message: string } } ApiResponse */

/**
 * @param {ApiResponse} response
 */
function handleResponse(response) {
  if (response.type === 'success') {
    console.log(response.data.name); // TypeScript knows data exists
  } else {
    console.error(response.message);
  }
}
```

#### 4. **Testing Patterns**

```javascript
// test/utils.js
export function renderWithProviders(ui, { providers = [] } = {}) {
  const wrapper = ({ children }) => 
    providers.reduceRight((acc, Provider) => <Provider>{acc}</Provider>, children);
  
  return render(ui, { wrapper });
}

// test/matchers.js
expect.extend({
  toBeWithinRange(received, floor, ceiling) {
    const pass = received >= floor && received <= ceiling;
    return {
      pass,
      message: () => pass 
        ? `expected ${received} not to be within range ${floor} - ${ceiling}`
        : `expected ${received} to be within range ${floor} - ${ceiling}`
    };
  }
});

// test/factories.js
export function createUser(overrides = {}) {
  return {
    id: crypto.randomUUID(),
    name: 'Test User',
    email: 'test@example.com',
    role: 'user',
    createdAt: new Date(),
    ...overrides
  };
}

// Integration test pattern
describe('User Registration', () => {
  it('registers new user and sends welcome email', async () => {
    // Arrange
    const emailService = { sendWelcome: jest.fn() };
    const userRepo = { save: jest.fn().mockResolvedValue(undefined) };
    
    // Act
    const result = await registerUser({ 
      email: 'new@test.com', 
      password: 'secure123' 
    }, { emailService, userRepo });
    
    // Assert
    expect(result.ok).toBe(true);
    expect(userRepo.save).toHaveBeenCalledWith(
      expect.objectContaining({ email: 'new@test.com' })
    );
    expect(emailService.sendWelcome).toHaveBeenCalledWith('new@test.com');
  });
});

// Property-based testing (fast-check)
import { test, prop } from 'fast-check';

test('reverse(reverse(arr)) === arr', () => {
  prop([fc.array(fc.integer())], (arr) => {
    expect(reverse(reverse(arr))).toEqual(arr);
  });
});
```

#### 5. **Performance Patterns**

```javascript
// 1. Request Deduplication
function createDeduplicatedFetcher() {
  const pending = new Map();
  
  return async function fetchOnce(key, fetcher) {
    if (pending.has(key)) return pending.get(key);
    
    const promise = fetcher().finally(() => pending.delete(key));
    pending.set(key, promise);
    return promise;
  };
}

const fetchUser = createDeduplicatedFetcher();

// Multiple calls = single request
const [user1, user2, user3] = await Promise.all([
  fetchUser('user:1', () => fetch('/api/users/1')),
  fetchUser('user:1', () => fetch('/api/users/1')),
  fetchUser('user:1', () => fetch('/api/users/1')),
]);

// 2. Virtualized List (React example concept)
function VirtualList({ items, itemHeight, windowHeight, renderItem }) {
  const [scrollTop, setScrollTop] = useState(0);
  
  const startIndex = Math.floor(scrollTop / itemHeight);
  const visibleCount = Math.ceil(windowHeight / itemHeight);
  const endIndex = Math.min(startIndex + visibleCount + 1, items.length);
  
  const visibleItems = items.slice(startIndex, endIndex);
  
  return (
    <div style={{ height: windowHeight, overflow: 'auto' }} onScroll={e => setScrollTop(e.target.scrollTop)}>
      <div style={{ height: items.length * itemHeight, position: 'relative' }}>
        {visibleItems.map((item, index) => (
          <div key={startIndex + index} style={{ 
            position: 'absolute', 
            top: (startIndex + index) * itemHeight,
            height: itemHeight 
          }}>
            {renderItem(item)}
          </div>
        ))}
      </div>
    </div>
  );
}

// 3. Memoization with WeakMap (for object keys)
function memoize(fn) {
  const cache = new WeakMap();
  return function(arg) {
    if (cache.has(arg)) return cache.get(arg);
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}

// 4. Web Workers for Heavy Computation
// worker.js
self.onmessage = async (e) => {
  const { type, payload, id } = e.data;
  try {
    let result;
    switch (type) {
      case 'PROCESS_IMAGE': result = await processImage(payload); break;
      case 'CALCULATE': result = heavyCalculation(payload); break;
    }
    self.postMessage({ id, type: 'SUCCESS', result });
  } catch (error) {
    self.postMessage({ id, type: 'ERROR', error: error.message });
  }
};

// main.js
function createWorker() {
  const worker = new Worker('worker.js');
  const pending = new Map();
  
  worker.onmessage = (e) => {
    const { id, type, result, error } = e.data;
    const { resolve, reject } = pending.get(id);
    pending.delete(id);
    type === 'SUCCESS' ? resolve(result) : reject(new Error(error));
  };
  
  return {
    run: (type, payload) => new Promise((resolve, reject) => {
      const id = crypto.randomUUID();
      pending.set(id, { resolve, reject });
      worker.postMessage({ id, type, payload });
    }),
    terminate: () => worker.terminate()
  };
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Composition over Inheritance** | Deep class hierarchies | Rigid, hard to modify |
| **Dependency Injection** | Singleton imports | Untestable, tight coupling |
| **Explicit Error Types** | Throwing strings/Error | Hard to handle specifically |
| **Pure Functions** | Mutable global state | Unpredictable, untestable |
| **Early Returns** | Deep nesting | Cognitive load |
| **Immer for Immutability** | Manual spread operators | Error-prone, verbose |
| **Zod/Valibot for Validation** | Manual validation | Inconsistent, incomplete |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Design**: "Implement a rate limiter (token bucket)"

```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillRate; // tokens per second
    this.lastRefill = Date.now();
  }
  
  consume(tokens = 1) {
    this.refill();
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }
    return false;
  }
  
  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillRate);
    this.lastRefill = now;
  }
  
  // For async waiting
  async waitForToken(tokens = 1) {
    while (!this.consume(tokens)) {
      await new Promise(r => setTimeout(r, 100));
    }
  }
}

// Usage: 10 requests per second
const limiter = new TokenBucket(10, 10);

app.use(async (req, res, next) => {
  if (!limiter.consume()) {
    return res.status(429).json({ error: 'Rate limited' });
  }
  next();
});
```

#### 2. **Code**: "Implement `Promise.race` with timeout"

```javascript
function withTimeout(promise, ms, timeoutError = new Error('Timeout')) {
  return Promise.race([
    promise,
    new Promise((_, reject) => setTimeout(() => reject(timeoutError), ms))
  ]);
}

// Usage
try {
  const data = await withTimeout(fetchData(), 5000);
} catch (err) {
  if (err.message === 'Timeout') { /* handle */ }
}

// Advanced: Timeout with cleanup
function withTimeoutCleanup(promise, ms) {
  let timeoutId;
  const timeoutPromise = new Promise((_, reject) => {
    timeoutId = setTimeout(() => reject(new Error('Timeout')), ms);
  });
  
  try {
    return await Promise.race([promise, timeoutPromise]);
  } finally {
    clearTimeout(timeoutId);
  }
}
```

#### 3. **Architecture**: "Design a plugin system"

```javascript
// Core
class PluginManager {
  constructor() {
    this.plugins = new Map();
    this.hooks = new Map();
  }
  
  register(name, plugin) {
    if (this.plugins.has(name)) throw new Error(`Plugin ${name} already registered`);
    this.plugins.set(name, plugin);
    plugin.install?.(this);
  }
  
  unregister(name) {
    const plugin = this.plugins.get(name);
    plugin.uninstall?.(this);
    this.plugins.delete(name);
  }
  
  // Hook system
  addHook(name, fn) {
    if (!this.hooks.has(name)) this.hooks.set(name, []);
    this.hooks.get(name).push(fn);
  }
  
  async runHook(name, ...args) {
    const fns = this.hooks.get(name) || [];
    for (const fn of fns) {
      await fn(...args);
    }
  }
  
  // Filter hook (each plugin can modify)
  async filterHook(name, value, ...args) {
    let result = value;
    for (const fn of this.hooks.get(name) || []) {
      result = await fn(result, ...args);
    }
    return result;
  }
}

// Plugin Example
const loggerPlugin = {
  name: 'logger',
  install(manager) {
    manager.addHook('request:start', (req) => console.log('Request:', req.url));
    manager.addHook('request:end', (req, res) => console.log('Response:', res.status));
  },
  uninstall(manager) { /* cleanup */ }
};

// Usage
const manager = new PluginManager();
manager.register('logger', loggerPlugin);
await manager.runHook('request:start', { url: '/api/users' });
```

#### 4. **Debugging**: "Find the bug in this async code"

```javascript
// BUGGY: Sequential when should be parallel
async function getDashboardData(userId) {
  const user = await fetchUser(userId);      // 100ms
  const posts = await fetchPosts(userId);    // 200ms (waits for user!)
  const comments = await fetchComments(userId); // 150ms (waits for posts!)
  return { user, posts, comments };          // Total: 450ms
}

// FIXED: Parallel
async function getDashboardData(userId) {
  const [user, posts, comments] = await Promise.all([
    fetchUser(userId),
    fetchPosts(userId),
    fetchComments(userId)
  ]);
  return { user, posts, comments };          // Total: 200ms
}

// BUGGY: forEach with async
async function processAll(items) {
  items.forEach(async (item) => {  // forEach doesn't wait!
    await processItem(item);
  });
  console.log('Done');  // Runs immediately!
}

// FIXED: for...of or Promise.all
async function processAll(items) {
  for (const item of items) {
    await processItem(item);  // Sequential
  }
  // OR parallel:
  await Promise.all(items.map(processItem));
}
```

#### 5. **Security**: "Prevent Prototype Pollution"

```javascript
// VULNERABLE
function merge(target, source) {
  for (const key of Object.keys(source)) {
    if (source[key] && typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

// Attack: merge({}, JSON.parse('{"__proto__": {"polluted": true}}'))
// Now ALL objects have .polluted = true

// SECURE
function safeMerge(target, source, visited = new WeakSet()) {
  if (visited.has(source)) return target; // Circular
  visited.add(source);
  
  for (const key of Object.keys(source)) {
    // Block dangerous keys
    if (['__proto__', 'constructor', 'prototype'].includes(key)) continue;
    
    const srcVal = source[key];
    const tgtVal = target[key];
    
    if (srcVal && typeof srcVal === 'object' && !Array.isArray(srcVal)) {
      if (tgtVal && typeof tgtVal === 'object') {
        target[key] = safeMerge(tgtVal, srcVal, visited);
      } else {
        target[key] = safeMerge({}, srcVal, visited);
      }
    } else {
      target[key] = srcVal;
    }
  }
  return target;
}

// Or use: Object.create(null) for prototype-less objects
const safeObj = Object.create(null);
safeMerge(safeObj, userInput);
```

---

### GOTCHAS & EDGE CASES

- [ ] **`for...in` iterates prototype chain**: Use `Object.hasOwn()` or `for...of Object.keys()`
- [ ] **`Array.from()` vs Spread**: `Array.from(arrayLike)` works on iterables, spread needs iterator
- [ ] **`Promise.all` fails fast**: Use `Promise.allSettled` for independent operations
- [ ] **`JSON.stringify` loses**: Functions, undefined, Symbol, circular refs, BigInt
- [ ] **`Date` serialization**: `toISOString()` not `toString()` for APIs
- [ ] **`sort()` mutates**: Always `[...arr].sort()` or `arr.toSorted()` (ES2023)
- [ ] **`parseInt` radix**: Always `parseInt(str, 10)` - octal surprises
- [ ] **Floating point**: `0.1 + 0.2 !== 0.3` → use `Math.round((0.1+0.2)*100)/100` or `decimal.js`
- [ ] **Event loop microtasks**: `queueMicrotask` runs before next macrotask
- [ ] **`this` in callbacks**: Arrow functions capture `this`, regular functions don't

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Error Boundary Pattern**
```javascript
class Result {
  static ok(value) { return new Result({ ok: true, value }); }
  static err(error) { return new Result({ ok: false, error }); }
  
  constructor({ ok, value, error }) { this.ok = ok; this.value = value; this.error = error; }
  
  map(fn) { return this.ok ? Result.ok(fn(this.value)) : this; }
  mapErr(fn) { return this.ok ? this : Result.err(fn(this.error)); }
  flatMap(fn) { return this.ok ? fn(this.value) : this; }
  unwrap() { if (!this.ok) throw this.error; return this.value; }
  unwrapOr(defaultValue) { return this.ok ? this.value : defaultValue; }
}

// Usage
const result = await Result.ok(fetchUser(id))
  .flatMap(user => Result.ok(fetchPosts(user.id)))
  .map(posts => posts.filter(p => p.published));

if (result.ok) { /* use result.value */ }
else { /* handle result.error */ }
```

#### 2. **Event Emitter with Types**
```javascript
/**
 * @template T
 * @typedef {{[K in keyof T]: Array<(payload: T[K]) => void>}} EventMap
 */

class TypedEventEmitter {
  constructor() {
    /** @type {Map<string, Set<Function>>} */
    this.events = new Map();
  }
  
  /** @param {string} event @param {Function} listener */
  on(event, listener) {
    if (!this.events.has(event)) this.events.set(event, new Set());
    this.events.get(event).add(listener);
    return () => this.off(event, listener);
  }
  
  off(event, listener) { this.events.get(event)?.delete(listener); }
  
  /** @param {string} event @param {any} payload */
  emit(event, payload) {
    this.events.get(event)?.forEach(fn => {
      try { fn(payload); } catch (err) { console.error(err); }
    });
  }
  
  once(event, listener) {
    const onceFn = (payload) => { listener(payload); this.off(event, onceFn); };
    return this.on(event, onceFn);
  }
}
```

#### 3. **Retry with Circuit Breaker**
```javascript
class CircuitBreaker {
  constructor(fn, { failureThreshold = 5, timeout = 30000 } = {}) {
    this.fn = fn;
    this.failureThreshold = failureThreshold;
    this.timeout = timeout;
    this.failures = 0;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.lastFailure = 0;
  }
  
  async execute(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailure > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker OPEN');
      }
    }
    
    try {
      const result = await this.fn(...args);
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }
  
  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
    }
  }
}

// Usage
const breaker = new CircuitBreaker(fetchUser, { failureThreshold: 3 });
const user = await breaker.execute(userId);
```

---

### PRACTICE PROBLEMS

1. **Implement**: Reactive state system with computed values and effects
2. **Build**: Plugin architecture with lifecycle hooks (init, destroy, config)
3. **Create**: Type-safe event emitter using JSDoc generics
4. **Debug**: Memory leak in long-running WebSocket connection
5. **Optimize**: Bundle size analysis and code splitting strategy

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Proxy Traps | get, set, has, deleteProperty, ownKeys, apply, construct |
| WeakRef Use Case | Cache that doesn't prevent GC |
| FinalizationRegistry | Cleanup after GC (close connections, remove listeners) |
| Generator vs Async Generator | yield vs await yield (for sync/async iteration) |
| Token Bucket Algorithm | Capacity + refill rate, consume tokens per request |
| Circuit Breaker States | CLOSED (normal) → OPEN (failing) → HALF_OPEN (testing) |
| Request Deduplication | Map of pending promises, return existing for same key |
| Virtualized List | Only render visible items + buffer, absolute positioning |
| JSDoc @template | Generic types in JavaScript |
| Branded Types | Nominal typing: `type UserId = string & { _brand: unique }` |

---

## NEXT TOPIC: `06-TYPESCRIPT.md`

> **Study Tip**: Build a mini framework: Router + State + Components using Proxies. Add TypeScript types via JSDoc.