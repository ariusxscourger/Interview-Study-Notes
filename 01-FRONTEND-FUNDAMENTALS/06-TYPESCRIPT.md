# TYPESCRIPT MASTERY
## Exam-Style Study Notes

---

## TOPIC: TypeScript for Production Applications

### WHY? (Problem Statement)

**JavaScript at Scale Problems:**
- No compile-time type checking
- Refactoring = runtime bugs
- No IDE autocomplete for complex objects
- Documentation drifts from code
- `any` everywhere defeats purpose

**What TypeScript Solves:**
| Problem | TypeScript Solution |
|---------|---------------------|
| Runtime type errors | Compile-time catching |
| Refactoring fear | Rename symbol across codebase |
| Documentation | Types ARE documentation |
| IDE support | IntelliSense, go-to-definition |
| Team scaling | Contracts between modules |

**Real-World Analogy:**
- JavaScript = Building without blueprints
- TypeScript = Building with verified blueprints + inspector

---

### HOW? (Internal Mechanism)

#### 1. **Type System Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                    TYPE HIERARCHY                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  any  ←─────────────────────────────────────────────────┐  │
│   ↑                                                      │  │
│  unknown  ←────────────────────────────────────────────┐ │  │
│   ↑                                                    │ │  │
│  never  (bottom type - no values)                      │ │  │
│   ↑                                                    │ │  │
│  Primitive Types (string, number, boolean, symbol,    │ │  │
│  bigint, null, undefined, void)                       │ │  │
│   ↑                                                    │ │  │
│  Object Types (interfaces, type aliases, classes)     │ │  │
│   ↑                                                    │ │  │
│  Arrays, Tuples, Functions, Constructors              │ │  │
│   ↑                                                    │ │  │
│  Generics, Conditional Types, Mapped Types            │ │  │
│   ↑                                                    │ │  │
│  Template Literal Types, Recursive Types              │ │  │
│                                                             │
│  Structural Typing: "If it walks like a duck..."       │
│  (vs Nominal Typing in C#/Java)                        │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **Compilation Pipeline**

```
.ts/.tsx Files
     │
     ▼
┌─────────────────────────────────────────┐
│         PARSER (TypeScript)             │
│  - Creates AST (Abstract Syntax Tree)   │
│  - Type checking happens HERE           │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│       EMITTER (JavaScript)              │
│  - Removes all types                    │
│  - Transpiles modern JS to target       │
│  - Generates .js + .d.ts + .js.map      │
└─────────────────────────────────────────┘
     │
     ▼
.js Files (No types at runtime!)
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Type Annotations & Inference**

```typescript
// Explicit annotations (when inference fails or for documentation)
let name: string = 'Alice';
let age: number = 30;
let isActive: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;
let id: symbol = Symbol('id');
let big: bigint = 9007199254740991n;

// Inference (TypeScript figures it out)
let inferred = 'hello';  // string
const constant = 'world'; // 'world' (literal type)

// Arrays
const numbers: number[] = [1, 2, 3];
const names: Array<string> = ['Alice', 'Bob'];

// Tuples (fixed length, known types)
const user: [string, number, boolean] = ['Alice', 30, true];
user[0] = 'Bob'; // OK
// user[3] = 'oops'; // Error

// Objects
const user: { name: string; age: number; active?: boolean } = {
  name: 'Alice',
  age: 30
};

// Functions
function add(a: number, b: number): number {
  return a + b;
}

// Arrow function types
const multiply = (a: number, b: number): number => a * b;

// Function with optional/default/rest
function greet(name: string, greeting = 'Hello', ...titles: string[]) {
  return `${greeting}, ${name} ${titles.join(' ')}`;
}
```

#### 2. **Interfaces vs Type Aliases**

```typescript
// Interface - Extensible, declaration merging
interface User {
  id: string;
  name: string;
  email: string;
}

// Can reopen interface
interface User {
  role: 'admin' | 'user' | 'guest';
}

// Extends
interface AdminUser extends User {
  permissions: string[];
}

// Type Alias - More flexible, unions, primitives
type UserRole = 'admin' | 'user' | 'guest';
type UserId = string & { readonly __brand: unique symbol }; // Branded type
type User = {
  id: UserId;
  name: string;
  role: UserRole;
};

// Type alias for unions (interface can't do this)
type Result<T> = 
  | { ok: true; value: T }
  | { ok: false; error: Error };

// When to use which:
// - Interface: Public API, library authors, need declaration merging
// - Type: Unions, intersections, computed types, internal code
```

#### 3. **Generics - The Power Tool**

```typescript
// Basic Generic
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const num = first([1, 2, 3]); // number
const str = first(['a', 'b']); // string

// Generic Constraints
interface HasLength {
  length: number;
}

function logLength<T extends HasLength>(item: T): void {
  console.log(item.length);
}

logLength('hello'); // OK
logLength([1, 2, 3]); // OK
// logLength(42); // Error

// Generic Defaults
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

const userResponse: ApiResponse<User> = { ... };
const genericResponse: ApiResponse = { data: 'anything', ... };

// Generic Utility Types (Built-in)
type Partial<T> = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
type Readonly<T> = { readonly [P in keyof T]: T[P] };
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
type Record<K extends keyof any, T> = { [P in K]: T };
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
type NonNullable<T> = T extends null | undefined ? never : T;
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any;
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

// Practical: Partial for updates
function updateUser(id: string, updates: Partial<User>): User { ... }

// Practical: Pick for API responses
type UserSummary = Pick<User, 'id' | 'name' | 'email'>;

// Practical: Omit for create (no ID)
type CreateUser = Omit<User, 'id' | 'createdAt'>;
```

#### 4. **Advanced Types - Discriminated Unions & Conditional**

```typescript
// Discriminated Union (Best for state machines)
type LoadingState = 
  | { status: 'idle' }
  | { status: 'loading'; progress: number }
  | { status: 'success'; data: User[] }
  | { status: 'error'; error: Error };

function handleState(state: LoadingState) {
  switch (state.status) {
    case 'idle':
      return 'Ready';
    case 'loading':
      return `Loading... ${state.progress}%`; // TypeScript knows progress exists
    case 'success':
      return `Loaded ${state.data.length} users`; // TypeScript knows data exists
    case 'error':
      return `Error: ${state.error.message}`;
  }
}

// Conditional Types
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Distributive Conditional Types (magic!)
type ToArray<T> = T extends any ? T[] : never;

type C = ToArray<string | number>; 
// string[] | number[] (distributes over union!)

// Infer Keyword (Extract types)
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type PromiseValue<T> = T extends Promise<infer V> ? V : never;
type ArrayElement<T> = T extends (infer U)[] ? U : never;

// Template Literal Types (ES2024)
type EventName = `on${Capitalize<string>}`;
// onClick, onSubmit, onUserLogin, etc.

type CssProperty = `margin-${'top' | 'right' | 'bottom' | 'left'}`;
// 'margin-top' | 'margin-right' | 'margin-bottom' | 'margin-left'

// Recursive Types
type JsonValue = 
  | string 
  | number 
  | boolean 
  | null 
  | JsonValue[] 
  | { [key: string]: JsonValue };

type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};
```

#### 5. **Utility Patterns for Real Apps**

```typescript
// 1. Branded Types (Nominal typing simulation)
type UserId = string & { readonly __brand: unique symbol };
type PostId = string & { readonly __brand: unique symbol };

function createUserId(id: string): UserId {
  return id as UserId;
}

// Now you can't mix them
function getUser(id: UserId) { /* ... */ }
function getPost(id: PostId) { /* ... */ }

// getUser(createUserId('123')); // OK
// getUser(createPostId('456')); // Error!

// 2. Strict Event Handlers
type EventMap = {
  click: MouseEvent;
  submit: SubmitEvent;
  keydown: KeyboardEvent;
  change: Event;
  input: InputEvent;
};

function on<K extends keyof EventMap>(
  element: HTMLElement,
  event: K,
  handler: (event: EventMap[K]) => void
) {
  element.addEventListener(event, handler as EventListener);
}

// Usage
on(button, 'click', (e) => {
  e.clientX; // MouseEvent has clientX
});

on(form, 'submit', (e) => {
  e.preventDefault(); // SubmitEvent has preventDefault
});

// 3. Deep Partial / Deep Required
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

type DeepRequired<T> = {
  [P in keyof T]-?: T[P] extends object ? DeepRequired<T[P]> : T[P];
};

// 3. API Route Types (File-based routing style)
type Routes = {
  'GET /users': { query: { page?: string }; response: User[] };
  'POST /users': { body: CreateUser; response: User };
  'GET /users/:id': { params: { id: string }; response: User };
  'PATCH /users/:id': { params: { id: string }; body: Partial<User>; response: User };
};

// 4. Constructor Type
type Constructor<T = {}> = new (...args: any[]) => T;

function withTimestamp<T extends Constructor>(Base: T) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();
  };
}

const UserWithTimestamp = withTimestamp(class {
  name = '';
  email = '';
});
```

#### 6. **Configuration (tsconfig.json)**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    
    /* Strictness (ENABLE ALL) */
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    
    /* Module/Import */
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    
    /* Output */
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    
    /* JSX */
    "jsx": "react-jsx",
    "jsxImportSource": "react",
    
    /* Advanced */
    "skipLibCheck": true,
    "useDefineForClassFields": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| `strict: true` + all strict flags | `strict: false` | Catches 90% of bugs at compile time |
| Discriminated unions for state | `status: string` + `data?: any` | Type-safe, exhaustive checking |
| `unknown` over `any` | `any` everywhere | Forces type narrowing before use |
| Branded types for IDs | `string` for all IDs | Prevents ID mixing bugs |
| `Pick`/`Omit` for derived types | Copy-paste interfaces | Single source of truth |
| `readonly` arrays/objects | Mutable everywhere | Prevents accidental mutations |
| Generics with constraints | `T extends any` | Self-documenting, safer |
| `const` assertions | `as const` missing | Narrowed literal types |

---

### INTERVIEW QUESTIONS (Top 20)

#### 1. **Conceptual**: "Interface vs Type - when to use each?"

**Answer:**
```
INTERFACE:
✓ Declaration merging (auto-merge same name)
✓ Extends/implements (classes)
✓ Better error messages
✓ Public API contracts
✓ Library authors

TYPE:
✓ Unions/Intersections (type A = B | C)
✓ Primitives (type ID = string)
✓ Computed types (mapped, conditional)
✓ Tuples, function types
✓ Internal implementation

RULE: 
- Public API → Interface
- Internal/Complex → Type
- Need union → Type
- Library consumer extends → Interface
```

#### 2. **Code**: "Make this type-safe: `function get(obj, path) { return path.split('.').reduce((o, k) => o?.[k], obj); }`"

```typescript
// Using template literal types + recursion
type Paths<T> = T extends object
  ? { [K in keyof T]-?: 
      K extends string 
        ? T[K] extends object 
          ? `${K}.${Paths<T[K]>}` 
          : K
        : never
    }[keyof T]
  : never;

type PathValue<T, P extends string> = P extends `${infer K}.${infer Rest}`
  ? K extends keyof T
    ? PathValue<T[K], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;

function get<T, P extends Paths<T>>(obj: T, path: P): PathValue<T, P> | undefined {
  return path.split('.').reduce((o: any, k) => o?.[k], obj);
}

// Usage
const user = { profile: { settings: { theme: 'dark' } } };
const theme = get(user, 'profile.settings.theme'); // string | undefined
// get(user, 'profile.invalid'); // Error!
```

#### 3. **Design**: "Type-safe event emitter"

```typescript
type EventMap = {
  login: { user: User; timestamp: Date };
  logout: { userId: string };
  error: { error: Error; context: string };
};

type EventName = keyof EventMap;

class TypedEmitter {
  private listeners = new Map<EventName, Set<(payload: any) => void>>();
  
  on<K extends EventName>(event: K, listener: (payload: EventMap[K]) => void): () => void {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(listener);
    return () => this.off(event, listener);
  }
  
  off<K extends EventName>(event: K, listener: (payload: EventMap[K]) => void): void {
    this.listeners.get(event)?.delete(listener);
  }
  
  emit<K extends EventName>(event: K, payload: EventMap[K]): void {
    this.listeners.get(event)?.forEach(l => {
      try { l(payload); } catch (e) { console.error(e); }
    });
  }
}

// Usage
const emitter = new TypedEmitter();
emitter.on('login', ({ user, timestamp }) => {
  console.log(user.name, timestamp); // Fully typed!
});
emitter.emit('login', { user: { id: '1', name: 'Alice' }, timestamp: new Date() });
```

#### 4. **Advanced**: "Explain `infer` with practical examples"

```typescript
// Extract return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Extract promise resolution
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

// Extract array element
type ElementType<T> = T extends (infer U)[] ? U : never;

// Extract constructor parameters
type ConstructorParams<T> = T extends new (...args: infer P) => any ? P : never;

// Extract class instance type
type InstanceType<T> = T extends new (...args: any[]) => infer I ? I : never;

// Practical: React Component Props
type ComponentProps<T> = T extends React.ComponentType<infer P> ? P : never;

// Practical: Unwrap nested promises
type DeepUnwrap<T> = T extends Promise<infer U> ? DeepUnwrap<U> : T;

type Nested = Promise<Promise<string>>;
type Unwrapped = DeepUnwrap<Nested>; // string
```

#### 5. **Performance**: "Why `const` assertions matter"

```typescript
// Without const assertion
const colors = ['red', 'green', 'blue'];
// type: string[]

// With const assertion
const colors = ['red', 'green', 'blue'] as const;
// type: readonly ['red', 'green', 'blue']

// Benefits:
colors[0] = 'yellow'; // Error! readonly
colors.push('yellow'); // Error! readonly

// Narrowing for object literals
const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3
} as const;

// Type: { readonly apiUrl: "https://api.example.com"; readonly timeout: 5000; readonly retries: 3 }
// Instead of: { apiUrl: string; timeout: number; retries: number; }

// Use with functions
function fetchData(config: typeof config) { ... }
fetchData(config); // Perfect inference
```

#### 6. **Debugging**: "Fix this type error"

```typescript
// Error: Argument of type 'string' is not assignable to parameter of type 'never'
function process(items: string[]) {
  const filtered = items.filter(item => item.length > 3);
  // filtered is string[] (correct)
  
  const mapped = filtered.map(item => item.toUpperCase());
  // mapped is string[] (correct)
  
  const reduced = mapped.reduce((acc, item) => {
    acc[item] = item.length;
    return acc;
  }, {}); // Error! {} is {}, not Record<string, number>
  
  return reduced;
}

// Fix 1: Type the initial value
const reduced = mapped.reduce((acc, item) => {
  acc[item] = item.length;
  return acc;
}, {} as Record<string, number>);

// Fix 2: Use Record utility
const reduced = mapped.reduce((acc: Record<string, number>, item) => {
  acc[item] = item.length;
  return acc;
}, {});

// Fix 3: Use type assertion on reduce
const reduced = mapped.reduce((acc, item) => {
  acc[item] = item.length;
  return acc;
}, <Record<string, number>>{});
```

#### 7. **Architecture**: "Type-safe API client with endpoints"

```typescript
// Define API contract
interface ApiContract {
  'GET /users': {
    query: { page?: number; limit?: number };
    response: { users: User[]; total: number };
  };
  'POST /users': {
    body: CreateUserDto;
    response: User;
  };
  'GET /users/:id': {
    params: { id: string };
    response: User;
  };
  'PATCH /users/:id': {
    params: { id: string };
    body: Partial<User>;
    response: User;
  };
}

// Type-safe client
class ApiClient {
  private baseUrl: string;
  
  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }
  
  private async request<
    TMethod extends keyof ApiContract,
    TEndpoint extends TMethod
  >(
    method: TMethod,
    endpoint: TEndpoint,
    options: {
      query?: ApiContract[TEndpoint]['query'];
      params?: ApiContract[TEndpoint]['params'];
      body?: ApiContract[TEndpoint]['body'];
    } = {}
  ): Promise<ApiContract[TEndpoint]['response']> {
    const url = this.buildUrl(endpoint, options.params);
    const response = await fetch(url, {
      method: method.split(' ')[0],
      headers: { 'Content-Type': 'application/json' },
      body: options.body ? JSON.stringify(options.body) : undefined,
    });
    
    if (!response.ok) {
      throw new ApiError(response.status, await response.text());
    }
    
    return response.json();
  }
  
  // Convenience methods
  getUsers(query?: ApiContract['GET /users']['query']) {
    return this.request('GET /users', '/users', { query });
  }
  
  createUser(body: ApiContract['POST /users']['body']) {
    return this.request('POST /users', '/users', { body });
  }
  
  getUser(id: string) {
    return this.request('GET /users/:id', `/users/${id}`, { params: { id } });
  }
  
  updateUser(id: string, body: ApiContract['PATCH /users/:id']['body']) {
    return this.request('PATCH /users/:id', `/users/${id}`, { params: { id }, body });
  }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **`any` is contagious**: One `any` infects everything it touches
- [ ] **`unknown` requires narrowing**: Must check before use
- [ ] **Excess property checking**: Object literals checked strictly
- [ ] **Function parameter bivariance**: `strictFunctionTypes` fixes this
- [ ] **Index signatures**: `Record<string, unknown>` vs `[key: string]: unknown`
- [ ] **`this` in callbacks**: Use arrow functions or `.bind(this)`
- [ ] **Enum vs const object**: `as const` object better for tree-shaking
- [ ] **Declaration files**: `.d.ts` for JS libs, `declare module` for globals
- [ ] **Type-only imports**: `import type { Foo } from 'bar'` for tree-shaking
- [ ] **Circular references**: Use `type` for recursive, `interface` for mutual

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Utility Types Collection**
```typescript
// From stdlib - know these by heart
type Partial<T> = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
type Readonly<T> = { readonly [P in keyof T]: T[P] };
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
type NonNullable<T> = T extends null | undefined ? never : T;
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any;
type Parameters<T> = T extends (...args: infer P) => any ? P : never;
type ConstructorParameters<T> = T extends new (...args: infer P) => any ? P : never;
type InstanceType<T> = T extends new (...args: any[]) => infer R ? R : any;
type ThisParameterType<T> = T extends (this: infer U, ...args: any[]) => any ? U : unknown;
type OmitThisParameter<T> = T extends (this: any, ...args: infer A) => infer R 
  ? (...args: A) => R 
  : T;
type ThisType<T> = T;

// Custom utilities
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};
type Mutable<T> = { -readonly [P in keyof T]: T[P] };
type RequiredKeys<T> = { [K in keyof T]-?: {} extends Pick<T, K> ? never : K }[keyof T];
type OptionalKeys<T> = { [K in keyof T]-?: {} extends Pick<T, K> ? K : never }[keyof T];
```

#### 2. **Result Pattern (Rust-style)**
```typescript
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

namespace Result {
  export function ok<T>(value: T): Result<T, never> {
    return { ok: true, value };
  }
  
  export function err<E>(error: E): Result<never, E> {
    return { ok: false, error };
  }
  
  export function map<T, U, E>(result: Result<T, E>, fn: (value: T) => U): Result<U, E> {
    return result.ok ? ok(fn(result.value)) : result;
  }
  
  export function flatMap<T, U, E>(result: Result<T, E>, fn: (value: T) => Result<U, E>): Result<U, E> {
    return result.ok ? fn(result.value) : result;
  }
  
  export function unwrap<T, E>(result: Result<T, E>): T {
    if (!result.ok) throw result.error;
    return result.value;
  }
  
  export function unwrapOr<T, E>(result: Result<T, E>, defaultValue: T): T {
    return result.ok ? result.value : defaultValue;
  }
}

// Usage
const result = await fetchUser(id)
  .then(Result.ok)
  .then(r => Result.flatMap(r, user => fetchPosts(user.id)))
  .then(r => Result.map(r, posts => posts.filter(p => p.published)));

if (result.ok) {
  renderPosts(result.value);
} else {
  showError(result.error);
}
```

#### 3. **Type-Safe Object Keys**
```typescript
// keyof returns string | number | symbol
// But for objects, it's only known keys

const user = { name: 'Alice', age: 30, email: 'alice@example.com' };

// keyof typeof user = 'name' | 'age' | 'email'

// Iterate with type safety
(Object.keys(user) as Array<keyof typeof user>).forEach(key => {
  console.log(key, user[key]); // Fully typed!
});

// Typed entries
Object.entries(user).forEach(([key, value]) => {
  // key: 'name' | 'age' | 'email'
  // value: string | number
});

// Typed Object.fromEntries
const doubled = Object.fromEntries(
  Object.entries(user).map(([k, v]) => [k, typeof v === 'number' ? v * 2 : v])
) as { [K in keyof typeof user]: typeof user[K] };
```

---

### PRACTICE PROBLEMS

1. **Implement**: Type-safe form validation with Zod-like API using template literals
2. **Create**: Typed router with file-based route definitions
3. **Build**: Generic CRUD repository with full type inference
4. **Design**: Type-safe event bus for cross-component communication
5. **Debug**: Fix circular type reference in recursive tree structure

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Interface vs Type | Interface: declaration merging, extends. Type: unions, primitives, computed |
| `unknown` vs `any` | `unknown`: safe, requires narrowing. `any`: unsafe, disables checking |
| `infer` keyword | Extracts type in conditional types: `T extends Promise<infer U> ? U : never` |
| Discriminated Union | Common literal property (`status`) narrows union members |
| Branded Type | `type UserId = string & { __brand: unique symbol }` - nominal typing |
| `const` assertion | `as const` → readonly, literal types, narrows inference |
| `Pick` vs `Omit` | Pick: select keys. Omit: remove keys. Both from keyof T |
| Conditional Distribution | `T extends U ? X : Y` distributes over unions automatically |
| `strictFunctionTypes` | Enables contravariance for function parameters (safer) |
| `noUncheckedIndexedAccess` | `obj[key]` returns `T | undefined` instead of `T` |

---

## NEXT TOPIC: `07-GIT-VCS.md`

> **Study Tip**: Enable all strict flags in a new project. Fix every error. Write 5 custom utility types. Build a typed API client.