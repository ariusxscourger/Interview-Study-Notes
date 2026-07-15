# REACT FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: React Core Concepts & Component Model

### WHY? (Problem Statement)

**Before React:**
- Direct DOM manipulation (jQuery, vanilla JS)
- Imperative: "How to update UI"
- No component reusability
- State scattered across DOM
- Hard to reason about UI state

**What React Solves:**
| Problem | Solution |
|---------|----------|
| Imperative DOM updates | Declarative: "What UI should look like" |
| No reusability | Component composition |
| State management | Unidirectional data flow |
| Performance | Virtual DOM + Reconciliation |
| Testing | Pure components, easy to test |

**Real-World Analogy:**
- Old way: Chef telling kitchen "chop onion, heat pan, add onion, stir..."
- React way: Chef describes "onion soup" - kitchen figures out steps

---

### HOW? (Internal Mechanism)

#### 1. **Virtual DOM & Reconciliation**

```
REAL DOM (Browser)                    VIRTUAL DOM (React)
─────────────────────────             ─────────────────────
<div class="app">                     { type: 'div',
  <header>                              props: { className: 'app' },
    <h1>Title</h1>                       children: [
  </header>                               { type: 'header',
  <main>                                    children: [...]
    <p>Content</p>                      ]
  </main>                              }
</div>                                   │
     │                                    ▼
     │  DIFFING ALGORITHM (O(n))
     │  1. Same type? Update props
     │  2. Different type? Replace
     │  3. Lists: Use KEYS for identity
     ▼
MINIMAL REAL DOM UPDATES
```

#### 2. **Render Cycle**

```
TRIGGER: setState / props change / parent re-render
         │
         ▼
    ┌─────────────────────┐
    │  RENDER PHASE       │  (Pure, can be interrupted)
    │  • Call component   │
    │  • Create VDOM      │
    │  • Diff with prev   │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  COMMIT PHASE       │  (Synchronous, cannot interrupt)
    │  • Apply DOM changes│
    │  • Run layout effects│
    │  • Run passive effects│
    └─────────────────────┘
```

---

### WHAT? (Core APIs)

#### 1. **Component Types**

```jsx
// 1. Function Component (Modern, Preferred)
function Welcome({ name, age = 25 }) {
  // Hooks for state/side effects
  const [count, setCount] = useState(0);
  
  // Event handlers
  const handleClick = () => setCount(c => c + 1);
  
  // Conditional rendering
  if (!name) return <div>Loading...</div>;
  
  return (
    <div className="welcome">
      <h1>Hello, {name}!</h1>
      <p>Age: {age}</p>
      <button onClick={handleClick}>Count: {count}</button>
    </div>
  );
}

// 2. Arrow Function Component
const Button = ({ children, onClick, variant = 'primary', disabled = false }) => {
  return (
    <button 
      className={`btn btn-${variant}`}
      onClick={onClick}
      disabled={disabled}
    >
      {children}
    </button>
  );
};

// 3. Component with Children (Composition)
function Card({ title, children, footer }) {
  return (
    <div className="card">
      <header><h2>{title}</h2></header>
      <main>{children}</main>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}

// Usage
<Card title="Welcome" footer={<Button>Action</Button>}>
  <p>Content goes here</p>
</Card>
```

#### 2. **Props & PropTypes/TypeScript**

```typescript
// TypeScript Interface (Preferred)
interface UserCardProps {
  name: string;
  email: string;
  age?: number;           // Optional
  role: 'admin' | 'user' | 'guest';  // Union type
  onEdit: (user: User) => void;      // Function prop
  tags: string[];                    // Array
  metadata: Record<string, unknown>; // Object
}

// Component with TypeScript
function UserCard({ 
  name, 
  email, 
  age = 25, 
  role, 
  onEdit, 
  tags = [], 
  metadata 
}: UserCardProps) {
  return (
    <div className={`user-card role-${role}`}>
      <h3>{name}</h3>
      <p>{email}</p>
      {age && <span>Age: {age}</span>}
      <ul>{tags.map(tag => <li key={tag}>{tag}</li>)}</ul>
      <button onClick={() => onEdit({ name, email })}>Edit</button>
    </div>
  );
}

// Default Props (Legacy - use default parameters instead)
UserCard.defaultProps = {
  age: 25,
  tags: [],
};

// PropTypes (If not using TypeScript)
import PropTypes from 'prop-types';

UserCard.propTypes = {
  name: PropTypes.string.isRequired,
  email: PropTypes.string.isRequired,
  age: PropTypes.number,
  role: PropTypes.oneOf(['admin', 'user', 'guest']),
  onEdit: PropTypes.func.isRequired,
  tags: PropTypes.arrayOf(PropTypes.string),
  metadata: PropTypes.object,
};
```

#### 3. **State & useState**

```jsx
import { useState } from 'react';

// Basic State
function Counter() {
  const [count, setCount] = useState(0);
  
  // Functional update (prevents stale closure)
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
}

// Object State (Must spread!)
function UserProfile() {
  const [user, setUser] = useState({
    name: '',
    email: '',
    preferences: { theme: 'light', notifications: true }
  });
  
  // WRONG: setUser({ name: 'New' }) - loses other fields!
  // CORRECT:
  const updateName = (name) => setUser(u => ({ ...u, name }));
  const updatePreferences = (prefs) => setUser(u => ({ 
    ...u, 
    preferences: { ...u.preferences, ...prefs } 
  }));
  
  return <div>{/* ... */}</div>;
}

// Lazy Initialization (Expensive computation)
function ExpensiveComponent({ initialData }) {
  // Only runs ONCE
  const [data, setData] = useState(() => {
    return processExpensiveComputation(initialData);
  });
  
  return <div>{/* ... */}</div>;
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Functional Updates** `setCount(c => c + 1)` | `setCount(count + 1)` | Stale closure bugs |
| **Immutable Updates** `setUser(u => ({...u, name}))` | `setUser({name})` | Loses other state |
| **Lazy Initialization** `useState(() => expensive())` | `useState(expensive())` | Runs every render |
| **Lift State Up** | Duplicate state in siblings | Sync issues, bugs |
| **Derived State** `const fullName = `${first} ${last}`` | `useState` for computed | Sync bugs, extra renders |
| **Component Composition** | Massive monolithic components | Hard to test, reuse |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Virtual DOM vs Real DOM - how does React update the UI?"

**Answer:**
```
1. State/props change triggers render
2. Component function runs → returns new VDOM tree
3. Reconciliation (Diffing):
   - Same element type? Update props/attributes
   - Different type? Destroy old, create new
   - Lists: Match by KEY (not index!)
4. Commit Phase:
   - Apply minimal DOM mutations
   - Run useLayoutEffect (sync)
   - Schedule useEffect (async)

KEY INSIGHT: Keys tell React "this is the SAME element" across renders.
Without keys, React assumes order = identity, causing bugs in lists.
```

#### 2. **Code**: "Fix this bug - list items losing state on reorder"

```jsx
// BUGGY: Using index as key
function TodoList({ items }) {
  return (
    <ul>
      {items.map((item, index) => (
        <TodoItem key={index} item={item} />  // BUG: index as key!
      ))}
    </ul>
  );
}

// FIXED: Use stable unique ID
function TodoList({ items }) {
  return (
    <ul>
      {items.map(item => (
        <TodoItem key={item.id} item={item} />  // GOOD: stable key
      ))}
    </ul>
  );
}

// WHY: When items reorder, index changes but item.id stays same.
// React thinks item at index 0 is "same" even though it's different data.
```

#### 3. **Design**: "Controlled vs Uncontrolled Components"

```jsx
// CONTROLLED (React owns state) - PREFERRED
function ControlledInput() {
  const [value, setValue] = useState('');
  
  return (
    <input 
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );

// UNCONTROLLED (DOM owns state) - Use for file inputs, integration
function UncontrolledInput() {
  const inputRef = useRef(null);
  
  const handleSubmit = () => {
    console.log(inputRef.current.value); // Read directly from DOM
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="Initial" />
      <button type="submit">Submit</button>
    </form>
  );

// WHEN TO USE WHICH:
// Controlled: Form validation, dynamic updates, dependent fields
// Uncontrolled: File inputs, simple forms, third-party integration
```

#### 4. **Code**: "Implement a custom hook for localStorage persistence"

```jsx
import { useState, useEffect } from 'react';

function useLocalStorage(key, initialValue) {
  // Lazy initialization
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  // Sync to localStorage on change
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue];
}

// Usage
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  const [language, setLanguage] = useLocalStorage('lang', 'en');
  
  return (
    <select value={theme} onChange={e => setTheme(e.target.value)}>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
    </select>
  );
}
```

#### 5. **Performance**: "Prevent unnecessary re-renders"

```jsx
import { memo, useCallback, useMemo } from 'react';

// 1. React.memo - Skip render if props unchanged
const ExpensiveComponent = memo(function ExpensiveComponent({ data, onAction }) {
  return <div>{/* heavy rendering */}</div>;
}, (prevProps, nextProps) => {
  // Custom comparison (optional)
  return prevProps.data.id === nextProps.data.id;
});

// 2. useCallback - Stable function reference
function Parent() {
  const [count, setCount] = useState(0);
  
  // Without useCallback: new function every render → child re-renders
  const handleClick = useCallback((id) => {
    console.log('Item clicked:', id);
  }, []); // Empty deps = never changes
  
  return <Child onItemClick={handleClick} items={items} />;
}

// 3. useMemo - Expensive computation
function ProductList({ products, filter }) {
  // Only recalculates when products or filter change
  const filteredProducts = useMemo(() => 
    products.filter(p => p.name.includes(filter)),
    [products, filter]
  );
  
  // Stable object reference for props
  const sortConfig = useMemo(() => ({ 
    field: 'price', 
    direction: 'asc' 
  }), []);
  
  return <ProductGrid products={filteredProducts} sort={sortConfig} />;
}

// 4. useRef - Mutable value without re-render
function Timer() {
  const intervalRef = useRef(null);
  const [seconds, setSeconds] = useState(0);
  
  const start = useCallback(() => {
    intervalRef.current = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);
  }, []);
  
  const stop = useCallback(() => {
    clearInterval(intervalRef.current);
  }, []);
  
  return <div>{/* ... */}</div>;
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Stale Closures** - `useEffect`/`setTimeout` capturing old state → use functional updates or include in deps
- [ ] **Array/Object Identity** - `[1,2] !== [1,2]` → `useMemo`/`useCallback` for stable references
- [ ] **Batch Updates** - React 18+ auto-batches, but async handlers may not
- [ ] **Key Prop** - Must be unique among siblings, stable across renders
- [ ] **setState Async** - `setCount(1); setCount(2)` → only 2 applied (batching)
- [ ] **useEffect Cleanup** - Return function runs before next effect/unmount
- [ ] **Strict Mode** - Double-invokes effects in dev (catches bugs)
- [ ] **Context Re-renders** - All consumers re-render on provider value change → split contexts

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Form Pattern (Controlled)**

```jsx
function ContactForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });
  
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const validate = (data) => {
    const newErrors = {};
    if (!data.name.trim()) newErrors.name = 'Name required';
    if (!data.email.includes('@')) newErrors.email = 'Valid email required';
    if (data.message.length < 10) newErrors.message = 'Message too short';
    return newErrors;
  };
  
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
    // Clear error on change
    if (errors[name]) setErrors(prev => ({ ...prev, [name]: '' }));
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    const newErrors = validate(formData);
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }
    
    setIsSubmitting(true);
    try {
      await submitForm(formData);
      setFormData({ name: '', email: '', message: '' });
    } catch (error) {
      setErrors({ form: error.message });
    } finally {
      setIsSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit} noValidate>
      <div>
        <label htmlFor="name">Name</label>
        <input 
          id="name" name="name" 
          value={formData.name} 
          onChange={handleChange}
          aria-invalid={!!errors.name}
          aria-describedby={errors.name ? 'name-error' : undefined}
        />
        {errors.name && <span id="name-error" role="alert">{errors.name}</span>}
      </div>
      {/* ... email, message fields ... */}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Sending...' : 'Send'}
      </button>
      {errors.form && <div role="alert">{errors.form}</div>}
    </form>
  );
}
```

#### 2. **Component Composition Patterns**

```jsx
// 1. Compound Components (Shared implicit state)
function Tabs({ children, defaultIndex = 0 }) {
  const [activeIndex, setActiveIndex] = useState(defaultIndex);
  
  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div className="tab-list" role="tablist">{children}</div>;
}

function Tab({ index, children }) {
  const { activeIndex, setActiveIndex } = useContext(TabsContext);
  return (
    <button
      role="tab"
      aria-selected={activeIndex === index}
      onClick={() => setActiveIndex(index)}
      className={activeIndex === index ? 'active' : ''}
    >
      {children}
    </button>
  );
}

function TabPanel({ index, children }) {
  const { activeIndex } = useContext(TabsContext);
  if (activeIndex !== index) return null;
  return <div role="tabpanel">{children}</div>;
}

// Usage
<Tabs defaultIndex={0}>
  <TabList>
    <Tab index={0}>Tab 1</Tab>
    <Tab index={1}>Tab 2</Tab>
  </TabList>
  <TabPanel index={0}>Content 1</TabPanel>
  <TabPanel index={1}>Content 2</TabPanel>
</Tabs>

// 2. Render Props (Legacy but still useful)
function DataFetcher({ url, children }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    fetch(url).then(r => r.json()).then(
      d => { setData(d); setLoading(false); },
      e => { setError(e); setLoading(false); }
    );
  }, [url]);
  
  return children({ data, loading, error });
}

// Usage
<DataFetcher url="/api/users">
  {({ data, loading, error }) => loading ? <Spinner /> : error ? <Error /> : <UserList users={data} />}
</DataFetcher>
```

---

### PRACTICE PROBLEMS

1. **Build** a typeahead autocomplete with debouncing, keyboard navigation, and caching
2. **Create** a modal portal with focus trap, ESC to close, and click outside to close
3. **Implement** a drag-and-drop list with react-dnd or @dnd-kit
4. **Debug** a component causing infinite render loop
5. **Build** a wizard form with validation, step persistence, and back/next navigation

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Virtual DOM Purpose | Lightweight JS representation of DOM. Enables diffing for minimal real DOM updates. |
| Reconciliation Algorithm | O(n) heuristic: same type → update props. Different type → replace. Lists use keys for identity. |
| Key Prop Purpose | Stable identity for list items across renders. Prevents bugs on reorder/filter. |
| Controlled vs Uncontrolled | Controlled: React owns state (value + onChange). Uncontrolled: DOM owns state (ref + defaultValue). |
| useState Functional Update | `setCount(c => c + 1)` uses latest state. Avoids stale closure bugs. |
| useEffect Cleanup | Return function runs before next effect execution or unmount. Prevents leaks. |
| useMemo vs useCallback | useMemo: memoizes value. useCallback: memoizes function (equivalent to useMemo(() => fn, deps)). |
| React.memo | HOC that prevents re-render if props shallow equal. Custom compare function optional. |
| Derived State | Compute during render: `const fullName = first + ' ' + last`. Don't useState for derived data. |
| Lifting State Up | Move shared state to common ancestor. Single source of truth. |

---

## NEXT TOPIC: `02-HOOKS-DEEP-DIVE.md`

> **Study Tip**: Build a custom hook library: useLocalStorage, useDebounce, useMediaQuery, useIntersectionObserver, useClickOutside, useCopyToClipboard. Test each in isolation.