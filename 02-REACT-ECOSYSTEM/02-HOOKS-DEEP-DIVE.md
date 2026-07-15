# REACT HOOKS DEEP DIVE
## Exam-Style Study Notes

---

## TOPIC: Advanced React Hooks Patterns & Internals

### WHY? (Problem Statement)

**Before Hooks (Class Components):**
- `this` binding issues
- Lifecycle methods split logic across methods
- HOC/Render Props for reuse → wrapper hell
- No state in function components

**What Hooks Solve:**
| Problem | Hook Solution |
|---------|---------------|
| State in functions | `useState`, `useReducer` |
| Side effects | `useEffect`, `useLayoutEffect` |
| Context access | `useContext` |
| Refs | `useRef`, `useImperativeHandle` |
| Memoization | `useMemo`, `useCallback` |
| Custom reuse | Custom hooks |
| Performance | `useMemo`, `useCallback`, `useDeferredValue`, `useTransition` |

---

### HOW? (Internal Mechanism)

#### 1. **Hook Rules & Dispatcher**

```
RULES OF HOOKS:
─────────────────────────────────────────────────────────────
1. Only call at TOP LEVEL (not in loops, conditions, nested)
2. Only call from React functions (components, custom hooks)
3. Same order every render (React relies on call order)

INTERNAL: React maintains a linked list of hooks per component
┌─────────────────────────────────────────────────────────────┐
│  Component Fiber                                            │
│  ├── memoizedState → Hook 1 (useState)                     │
│  │    ├── baseState: 0                                      │
│  │    ├── queue: { pending: null, dispatch: fn }           │
│  │    └── next → Hook 2 (useEffect)                        │
│  │         ├── create: fn                                   │
│  │         ├── destroy: fn                                  │
│  │         ├── deps: [dep1, dep2]                           │
│  │         └── next → Hook 3 (useContext)                  │
│  │              └── context: ContextObject                  │
│  │              └── next → null                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **useState vs useReducer Internals**

```javascript
// useState → Creates update queue, processes in batch
function useState(initialState) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = hook.baseState = initialState;
  
  const queue = {
    pending: null,
    dispatch: null,  // Will be set to dispatchAction.bind(null, fiber, queue)
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  
  return [hook.memoizedState, queue.dispatch];
}

// useReducer → More complex, supports complex logic
function useReducer(reducer, initialArg, init) {
  const hook = mountWorkInProgressHook();
  const initialState = init ? init(initialArg) : initialArg;
  hook.memoizedState = hook.baseState = initialState;
  
  const queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  
  return [hook.memoizedState, queue.dispatch];
}
```

#### 3. **useEffect vs useLayoutEffect Timing**

```
RENDER PHASE (Can be async, interruptible)
────────────────────────────────────────────
1. React calls component function
2. Creates new fiber tree
3. Reconciliation (diffing)
4. Determines DOM mutations needed

COMMIT PHASE (Synchronous, cannot interrupt)
────────────────────────────────────────────
1. DOM mutations applied
2. useLayoutEffect CLEANUP (sync) ← BEFORE paint
3. useLayoutEffect CREATE (sync)
4. Browser PAINT (user sees update)
5. useEffect CLEANUP (async) ← AFTER paint
6. useEffect CREATE (async)

KEY DIFFERENCE:
- useLayoutEffect: Runs SYNCHRONOUSLY after DOM mutations, before paint
  Use for: DOM measurements, focus management, preventing visual flicker
- useEffect: Runs ASYNCHRONOUSLY after paint
  Use for: Data fetching, subscriptions, analytics, non-urgent work
```

---

### WHAT? (Advanced Patterns)

#### 1. **useReducer for Complex State**

```jsx
// Complex form state with validation
const initialState = {
  values: { name: '', email: '', password: '' },
  errors: {},
  touched: {},
  isSubmitting: false,
  submitSuccess: false,
};

function formReducer(state, action) {
  switch (action.type) {
    case 'FIELD_CHANGE':
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        // Clear error when user types
        errors: { ...state.errors, [action.field]: '' },
        touched: { ...state.touched, [action.field]: true },
      };
      
    case 'FIELD_BLUR':
      return {
        ...state,
        touched: { ...state.touched, [action.field]: true },
        errors: { 
          ...state.errors, 
          [action.field]: validateField(action.field, state.values[action.field]) 
        },
      };
      
    case 'SET_ERRORS':
      return { ...state, errors: action.errors };
      
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true, submitSuccess: false };
      
    case 'SUBMIT_SUCCESS':
      return { ...state, isSubmitting: false, submitSuccess: true };
      
    case 'SUBMIT_FAILURE':
      return { 
        ...state, 
        isSubmitting: false, 
        errors: { ...state.errors, ...action.errors } 
      };
      
    case 'RESET':
      return initialState;
      
    default:
      return state;
  }
}

function RegistrationForm() {
  const [state, dispatch] = useReducer(formReducer, initialState);
  
  const handleChange = (field) => (e) => {
    dispatch({ type: 'FIELD_CHANGE', field, value: e.target.value });
  };
  
  const handleBlur = (field) => (e) => {
    dispatch({ type: 'FIELD_BLUR', field });
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    dispatch({ type: 'SUBMIT_START' });
    
    try {
      await api.register(state.values);
      dispatch({ type: 'SUBMIT_SUCCESS' });
    } catch (error) {
      dispatch({ type: 'SUBMIT_FAILURE', errors: error.fields });
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Field
        name="name"
        value={state.values.name}
        onChange={handleChange('name')}
        onBlur={handleBlur('name')}
        error={state.touched.name && state.errors.name}
      />
      {/* ... */}
      <button type="submit" disabled={state.isSubmitting}>
        {state.isSubmitting ? 'Registering...' : 'Register'}
      </button>
    </form>
  );
}
```

#### 2. **useEffect Advanced Patterns**

```jsx
// 1. Cleanup Function Pattern (Prevent memory leaks)
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    
    // CLEANUP: Runs before next effect or unmount
    return () => {
      connection.disconnect();
    };
  }, [roomId]); // Re-run when roomId changes
  
  // ...
}

// 2. Dependency Array Gotchas
function SearchResults({ query }) {
  // WRONG: Function recreated every render
  useEffect(() => {
    const fetchData = async () => { /* ... */ };
    fetchData();
  }, [query, fetchData]); // fetchData changes every render!
  
  // CORRECT: Move inside or useCallback
  useEffect(() => {
    const fetchData = async () => { /* ... */ };
    fetchData();
  }, [query]);
  
  // ALSO CORRECT: Stable reference
  const fetchData = useCallback(async () => { /* ... */ }, [query]);
  useEffect(() => { fetchData(); }, [fetchData]);
}

// 3. Conditional Effects (Use early return)
function UserProfile({ userId }) {
  useEffect(() => {
    if (!userId) return; // Skip if no userId
    
    let cancelled = false;
    fetchUser(userId).then(data => {
      if (!cancelled) setUser(data);
    });
    return () => { cancelled = true; };
  }, [userId]);
}

// 4. Effect Event Pattern (React 19+ / useEffectEvent RFC)
// Separate "event" logic from effect lifecycle
function Component() {
  const onClick = useEffectEvent(() => {
    // Always has latest props/state
    analytics.track('click', { userId: props.userId });
  });
  
  useEffect(() => {
    subscription.onData(onClick);
    return () => subscription.offData(onClick);
  }, []); // Empty deps! onClick is stable
}
```

#### 3. **useMemo vs useCallback vs useRef**

```jsx
// useMemo: Memoize VALUE (computation result)
function ProductList({ products, filter, sortBy }) {
  // Only recomputes when dependencies change
  const filtered = useMemo(() => 
    products
      .filter(p => p.name.includes(filter))
      .sort((a, b) => a[sortBy] - b[sortBy]),
    [products, filter, sortBy]
  );
  
  // Also useful for stable object references
  const sortConfig = useMemo(() => ({ field: 'price', dir: 'asc' }), []);
  
  return <ProductGrid products={filtered} sortConfig={sortConfig} />;
}

// useCallback: Memoize FUNCTION (reference stability)
function Parent() {
  const [count, setCount] = useState(0);
  
  // Without useCallback: new function every render
  const handleClick = useCallback((id) => {
    console.log('Item clicked:', id);
  }, []); // Empty deps = never changes
  
  return <Child onItemClick={handleClick} items={items} />;
}

// useRef: Mutable container WITHOUT re-render
function AutoFocusInput() {
  const inputRef = useRef(null);
  
  useEffect(() => {
    inputRef.current?.focus(); // Imperative access
  }, []);
  
  return <input ref={inputRef} />;
}

// IMPORTANT: useRef vs useState
// useState: Changing triggers re-render
// useRef: Changing does NOT trigger re-render

// Use useRef for:
// - DOM refs
// - Mutable values that don't need to trigger render
// - Storing previous values
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; }, [value]);
  return ref.current;
}
```

#### 4. **Custom Hooks - Reusable Logic**

```jsx
// 1. useLocalStorage
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });
  
  const setValue = useCallback((value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);
  
  return [storedValue, setValue];
}

// 2. useDebounce
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);
  
  return debouncedValue;
}

// 3. useIntersectionObserver (Lazy loading, infinite scroll)
function useIntersectionObserver(options = {}) {
  const [isIntersecting, setIsIntersecting] = useState(false);
  const elementRef = useRef(null);
  
  useEffect(() => {
    const element = elementRef.current;
    if (!element) return;
    
    const observer = new IntersectionObserver(([entry]) => {
      setIsIntersecting(entry.isIntersecting);
    }, options);
    
    observer.observe(element);
    return () => observer.disconnect();
  }, [options.root, options.rootMargin, options.threshold]);
  
  return [elementRef, isIntersecting];
}

// Usage
function Image({ src, alt }) {
  const [ref, isVisible] = useIntersectionObserver({ rootMargin: '100px' });
  
  return (
    <div ref={ref}>
      {isVisible && <img src={src} alt={alt} />}
      {!isVisible && <div className="placeholder" />}
    </div>
  );
}

// 4. useMediaQuery (Responsive hooks)
function useMediaQuery(query) {
  const [matches, setMatches] = useState(false);
  
  useEffect(() => {
    const media = window.matchMedia(query);
    if (media.matches !== matches) setMatches(media.matches);
    
    const listener = (e) => setMatches(e.matches);
    media.addEventListener('change', listener);
    return () => media.removeEventListener('change', listener);
  }, [matches, query]);
  
  return matches;
}

// 5. useClickOutside (Modals, dropdowns)
function useClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (event) => {
      if (!ref.current || ref.current.contains(event.target)) return;
      handler(event);
    };
    
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}

// 6. useAsync (Data fetching with state)
function useAsync(asyncFn, deps = []) {
  const [state, setState] = useState({ data: null, error: null, loading: true });
  
  useEffect(() => {
    let cancelled = false;
    
    async function execute() {
      setState(prev => ({ ...prev, loading: true, error: null }));
      try {
        const data = await asyncFn();
        if (!cancelled) setState({ data, error: null, loading: false });
      } catch (error) {
        if (!cancelled) setState({ data: null, error, loading: false });
      }
    }
    
    execute();
    return () => { cancelled = true; };
  }, deps);
  
  return state;
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useAsync(
    () => fetchUser(userId), 
    [userId]
  );
  
  if (loading) return <Skeleton />;
  if (error) return <Error message={error.message} />;
  if (!user) return null;
  
  return <UserCard user={user} />;
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Custom Hooks** | Copy-paste logic | DRY violation, bug propagation |
| **useReducer for complex state** | Multiple useState calls | Related state out of sync |
| **useCallback for event handlers** | Inline functions in JSX | Child re-renders, breaks memo |
| **useMemo for expensive calc** | Inline computation | Recalculates every render |
| **useRef for mutable refs** | useState for non-render data | Unnecessary re-renders |
| **useEffect cleanup** | Missing cleanup | Memory leaks, double subscriptions |
| **Stable deps** | Functions/objects in deps | Effect fires too often |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "useEffect vs useLayoutEffect - when to use which?"

```
useLayoutEffect:
- Runs SYNCHRONOUSLY after DOM mutations, BEFORE paint
- Blocks visual update until complete
- Use for: DOM measurements (getBoundingClientRect), focus management, preventing layout shift

useEffect:
- Runs ASYNCHRONOUSLY after paint
- Non-blocking
- Use for: Data fetching, subscriptions, analytics, timers

RULE: Start with useEffect. Switch to useLayoutEffect only if visual flicker or measurement issues.
```

#### 2. **Code**: "Fix the infinite loop"

```jsx
// BUGGY: Object in dependency array
function UserList({ user }) {
  const [users, setUsers] = useState([]);
  
  // BUG: filterOptions is NEW object every render
  const filterOptions = { active: true, role: user.role };
  
  useEffect(() => {
    fetchUsers(filterOptions).then(setUsers);
  }, [filterOptions]); // Triggers every render!
  
  return <UserList users={users} />;
}

// FIX 1: useMemo for stable object
const filterOptions = useMemo(
  () => ({ active: true, role: user.role }),
  [user.role]
);

// FIX 2: useEffectEvent (React 19) or separate deps
useEffect(() => {
  fetchUsers({ active: true, role: user.role }).then(setUsers);
}, [user.role]); // Primitive deps only

// FIX 3: Primitive values only in deps
useEffect(() => {
  fetchUsers({ active: true, role: user.role }).then(setUsers);
}, [user.role]); // user.role is primitive
```

#### 3. **Code**: "Implement a debounced search hook"

```jsx
function useDebouncedSearch(searchFn, delay = 300) {
  const [query, setQuery] = useState('');
  const [debouncedQuery, setDebouncedQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  
  // Debounce query
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedQuery(query), delay);
    return () => clearTimeout(timer);
  }, [query, delay]);
  
  // Execute search when debounced query changes
  useEffect(() => {
    if (!debouncedQuery) {
      setResults([]);
      return;
    }
    
    let cancelled = false;
    setLoading(true);
    
    searchFn(debouncedQuery)
      .then(data => {
        if (!cancelled) {
          setResults(data);
          setLoading(false);
        }
      })
      .catch(() => {
        if (!cancelled) {
          setResults([]);
          setLoading(false);
        }
      });
    
    return () => { cancelled = true; };
  }, [debouncedQuery, searchFn]);
  
  return { query, setQuery, results, loading };
}

// Usage
function SearchComponent() {
  const { query, setQuery, results, loading } = useDebouncedSearch(
    (q) => api.searchUsers(q),
    300
  );
  
  return (
    <div>
      <input 
        value={query} 
        onChange={e => setQuery(e.target.value)} 
        placeholder="Search users..."
      />
      {loading && <Spinner />}
      <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>
    </div>
  );
}
```

#### 4. **Advanced**: "useReducer vs useState for forms"

```jsx
// WHEN TO USE useReducer FOR FORMS:
// - Multiple related fields
// - Complex validation logic
// - Dependent fields (field B depends on field A)
// - Need to reset entire form
// - Complex submission logic

// useState is fine for:
// - 1-2 simple fields
// - Independent fields
// - Simple validation

// Hybrid approach: useReducer + useState
function ComplexForm() {
  const [formState, dispatch] = useReducer(formReducer, initialState);
  const [touched, setTouched] = useState({}); // Simple state for touched
  
  // ...
}
```

#### 5. **Performance**: "Why is my component re-rendering?"

```jsx
// DEBUG: Why Did You Render (library)
// import whyDidYouRender from '@welldone-software/why-did-you-render';

// Common causes & fixes:

// 1. NEW OBJECT/PROPS every render
<Child style={{ margin: 10 }} />  // BAD: new object
<Child style={styles.container} /> // GOOD: stable reference

// 2. FUNCTION PROPS
<Child onClick={() => handleClick(id)} /> // BAD: new function
<Child onClick={handleClick} /> // GOOD: useCallback

// 3. CONTEXT VALUE
<MyContext.Provider value={{ user, theme }}> // BAD: new object
<MyContext.Provider value={memoizedValue} /> // GOOD: useMemo

// 4. STATE IN PARENT
function Parent() {
  const [count, setCount] = useState(0);
  // Child re-renders every time count changes
  return <Child />; 
}

// FIX: Move state down or memoize child
const MemoizedChild = memo(Child);
function Parent() {
  const [count, setCount] = useState(0);
  return <MemoizedChild />;
}
```

#### 6. **Advanced**: "Custom hook for optimistic updates"

```jsx
function useOptimisticMutation(mutationFn, options = {}) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);
  
  const mutate = useCallback(async (variables) => {
    // 1. Optimistic update
    if (options.onMutate) {
      await options.onMutate(variables);
    }
    
    setLoading(true);
    setError(null);
    
    try {
      const result = await mutationFn(variables);
      setData(result);
      
      // 2. On success
      if (options.onSuccess) {
        await options.onSuccess(result, variables);
      }
    } catch (err) {
      setError(err);
      
      // 3. Rollback on error
      if (options.onError) {
        await options.onError(err, variables);
      }
    } finally {
      setLoading(false);
      // 4. Always settle
      if (options.onSettled) {
        await options.onSettled(data, error, variables);
      }
    }
  }, [mutationFn, options]);
  
  return { mutate, data, error, loading };
}

// Usage with React Query style
function useUpdateUser() {
  return useOptimisticMutation(
    (variables) => api.updateUser(variables.id, variables.data),
    {
      onMutate: async (variables) => {
        // Cancel outgoing refetches
        await queryClient.cancelQueries(['user', variables.id]);
        
        // Snapshot previous value
        const previousUser = queryClient.getQueryData(['user', variables.id]);
        
        // Optimistically update
        queryClient.setQueryData(['user', variables.id], (old) => ({
          ...old,
          ...variables.data,
        }));
        
        return { previousUser };
      },
      onError: (err, variables, context) => {
        queryClient.setQueryData(['user', variables.id], context.previousUser);
      },
      onSettled: (data, error, variables) => {
        queryClient.invalidateQueries(['user', variables.id]);
      },
    }
  );
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Stale Closures** - `useEffect` capturing old state → use functional updates or include in deps
- [ ] **Function in Deps** - New function every render → `useCallback` or move inside effect
- [ ] **Object/Array in Deps** - New reference every render → `useMemo` or primitive deps
- [ ] **Missing Cleanup** - Subscriptions, timers, listeners not cleaned → memory leaks
- [ ] **Conditional Hooks** - Hooks in if/loops → violates rules → violates rules
- [ ] **useEffect vs useLayoutEffect** - Visual flicker → useLayoutEffect for DOM measurements
- [ ] **Strict Mode Double Invoke** - Dev only, effects run twice → cleanup handles this
- [ ] **useState Lazy Init** - Expensive initial computation → `useState(() => expensive())`
- [ ] **useReducer vs useState** - Complex related state → useReducer
- [ ] **Custom Hook Naming** - Must start with `use` → React can't apply rules otherwise

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Custom Hook Template**

```jsx
import { useState, useEffect, useCallback, useRef, useMemo } from 'react';

function useCustomHook(param1, param2, options = {}) {
  // Destructure options with defaults
  const { 
    enabled = true, 
    onSuccess, 
    onError,
    debounceMs = 0 
  } = options;
  
  // State
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);
  
  // Refs for values that shouldn't trigger re-renders
  const mountedRef = useRef(true);
  const abortControllerRef = useRef(null);
  
  // Stable callbacks
  const execute = useCallback(async (...args) => {
    if (!enabled) return;
    
    // Cancel previous request
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
    abortControllerRef.current = new AbortController();
    
    setLoading(true);
    setError(null);
    
    try {
      const result = await param1(param2, ...args, {
        signal: abortControllerRef.current.signal
      });
      
      if (mountedRef.current) {
        setData(result);
        onSuccess?.(result, ...args);
      }
    } catch (err) {
      if (err.name === 'AbortError') return;
      if (mountedRef.current) {
        setError(err);
        onError?.(err, ...args);
      }
    } finally {
      if (mountedRef.current) {
        setLoading(false);
      }
    }
  }, [param1, param2, enabled, onSuccess, onError]);
  
  // Debounced execution
  const debouncedExecute = useMemo(
    () => debounce(execute, debounceMs),
    [execute, debounceMs]
  );
  
  // Cleanup on unmount
  useEffect(() => {
    mountedRef.current = true;
    return () => {
      mountedRef.current = false;
      abortControllerRef.current?.abort();
    };
  }, []);
  
  // Return stable API
  return useMemo(() => ({
    data,
    error,
    loading,
    execute: debouncedExecute,
    reset: useCallback(() => {
      setData(null);
      setError(null);
      setLoading(false);
    }, []),
  }), [data, error, loading, debouncedExecute]);
}
```

#### 2. **Performance Optimization Checklist**

```jsx
// ✅ GOOD: Stable references
const styles = useMemo(() => ({ container: { padding: 10 } }), []);
const handleClick = useCallback((id) => doSomething(id), [dependency]);
const value = useMemo(() => computeExpensive(a, b), [a, b]);

// ✅ GOOD: Primitive deps
useEffect(() => { fetchData(userId); }, [userId]); // number/string

// ✅ GOOD: Context splitting
const UserContext = createContext();
const ThemeContext = createContext();
// Prevents all consumers re-rendering when theme changes

// ✅ GOOD: Component composition over prop drilling
<Parent>
  <ChildA />
  <ChildB />
</Parent>

// ❌ BAD: New object every render
<Child style={{ margin: 10 }} />

// ❌ BAD: Function in render
<Child onClick={() => handleClick(id)} />

// ❌ BAD: Object in deps
useEffect(() => {}, [options]); // options = { a: 1 } new every render

// ❌ BAD: Missing cleanup
useEffect(() => {
  const sub = api.subscribe();
  // Missing return () => sub.unsubscribe()
}, []);
```

---

### PRACTICE PROBLEMS

1. **Build** `useForm` hook with validation, dirty tracking, and submit handling
2. **Create** `useWebSocket` hook with reconnection, message queue, and binary support
3. **Implement** `useInfiniteScroll` with IntersectionObserver and loading states
4. **Debug** a component with `why-did-you-render` - find 3 unnecessary renders
5. **Build** `useAsyncQueue` for sequential async operations with concurrency control

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| useEffect vs useLayoutEffect | useLayoutEffect: sync after DOM, before paint. useEffect: async after paint. |
| useMemo vs useCallback | useMemo = memoizes VALUE. useCallback = memoizes FUNCTION (useMemo(() => fn, deps)). |
| useRef vs useState | useRef: mutable, no re-render. useState: immutable, triggers re-render. |
| useReducer vs useState | useReducer: complex state logic, multiple sub-values, next state depends on prev. |
| useEffect Cleanup | Return function runs before next effect or unmount. Prevents leaks. |
| Dependency Array Rules | Same order, same count, same values every render. Primitives preferred. |
| Stale Closure Fix | Functional updates: `setCount(c => c + 1)` or include in deps. |
| Custom Hook Naming | MUST start with `use` - enables React linting & rules. |
| useMemo for Object Props | `style={useMemo(() => ({ margin: 10 }), [])}` prevents child re-renders. |
| Context Splitting | Separate contexts for values that change at different frequencies. |
| React.memo | Shallow prop comparison. Custom comparator: `memo(Comp, (prev, next) => ...)`. |
| useLayoutEffect Use Case | DOM measurements, focus management, preventing visual flicker. |

---

## NEXT TOPIC: `03-STATE-MANAGEMENT.md`

> **Study Tip**: Build 5 custom hooks this week. Each must: handle cleanup, support TypeScript, have tests, and handle edge cases (rapid calls, unmount during async, etc.)