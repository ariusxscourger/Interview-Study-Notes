# REACT PERFORMANCE OPTIMIZATION
## Exam-Style Study Notes

---

## TOPIC: React Performance Patterns & Optimization

### WHY? (Problem Statement)

**Without Performance Optimization:**
- Slow initial load (large bundles)
- Janky interactions (blocked main thread)
- Unnecessary re-renders (wasted work)
- Poor Core Web Vitals (LCP, INP, CLS)
- User abandonment, SEO penalty

**What Performance Optimization Solves:**
| Problem | Solution |
|---------|----------|
| Large bundle | Code splitting, tree shaking |
| Slow render | Memoization, virtualization |
| Blocked main thread | Web Workers, offscreen rendering |
| Layout shift | Aspect ratios, skeleton loaders |
| Hydration mismatch | Proper SSR, suspense boundaries |

**Real-World Analogy:**
- Unoptimized = Moving house with everything in one trip (slow, exhausting)
- Optimized = Packed boxes, labeled, multiple trips, unpack only what's needed

---

### HOW? (React Rendering Mechanics)

#### 1. **Render Phases**

```
RENDER PHASE (Can be async, interruptible)
──────────────────────────────────────────
1. Trigger: setState, props change, parent re-render
2. Component function executes
3. New React Element tree created (Virtual DOM)
4. Reconciliation (Diffing Algorithm - O(n))
   • Same type? Update props
   • Different type? Destroy old, create new
   • Lists: Use KEYS for identity
5. Fiber tree built with effects

COMMIT PHASE (Synchronous, cannot interrupt)
──────────────────────────────────────────
1. DOM mutations applied
2. useLayoutEffect cleanup/create (sync)
3. Browser PAINT (user sees update)
4. useEffect cleanup/create (async, after paint)
```

#### 2. **Reconciliation Algorithm (Diffing)**

```javascript
// O(n) algorithm - key assumptions:
// 1. Two elements of different types produce different trees
// 2. Elements with stable key stay same across renders

// DIFFING RULES:
// 1. Different element type → tear down old, build new
// <div> → <span> : destroy div, create span

// 2. Same type, different props → update props
// <div className="a" /> → <div className="b" /> : update className

// 3. Same type, children → recurse
// <ul><li>A</li></ul> → <ul><li>A</li><li>B</li></ul> : insert B

// 4. Lists: KEYS determine identity
// <li key="a">A</li> → <li key="b">B</li> : move a→b, insert b
// WITHOUT KEYS: index-based (breaks on reorder!)
```

---

### WHAT? (Optimization Techniques)

#### 1. **Code Splitting & Lazy Loading**

```tsx
// Route-level splitting
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const AdminDashboard = lazy(() => import('./AdminDashboard'));
const UserProfile = lazy(() => import('./UserProfile'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/admin/*" element={<AdminDashboard />} />
        <Route path="/users/:id" element={<UserProfile />} />
      </Routes>
    </Suspense>
  );
}

// Component-level splitting
const HeavyChart = lazy(() => import('./HeavyChart').then(module => ({
  default: module.Chart // named export
})));

function Dashboard() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <HeavyChart data={data} />
    </Suspense>
  );
}

// Dynamic import with preload
function SearchResults() {
  const [query, setQuery] = useState('');
  
  const handleSearch = useCallback(() => {
    // Preload when user focuses search
    import('./SearchResults').then(module => {
      module.preload?.();
    });
    setQuery(query);
  }, [query]);
  
  return <input onFocus={handleSearch} />;
}
```

#### 2. **Memoization Techniques**

```tsx
// useMemo: Memoize EXPENSIVE COMPUTATION
function ProductList({ products, filter, sortBy }) {
  const filtered = useMemo(() => 
    products
      .filter(p => p.name.includes(filter))
      .sort((a, b) => a[sortBy] - b[sortBy]),
    [products, filter, sortBy]
  );
  
  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// useCallback: Memoize FUNCTION REFERENCE
function Parent() {
  const [count, setCount] = useState(0);
  
  // Without useCallback: new function every render
  const handleClick = useCallback((id: string) => {
    console.log('Item clicked:', id);
  }, []); // Empty deps = never recreated
  
  return <Child onItemClick={handleClick} items={items} />;
}

// React.memo: Memoize COMPONENT (shallow prop comparison)
const ExpensiveChild = memo(function ExpensiveChild({ data, onAction }) {
  return <div>{/* heavy rendering */}</div>;
}, (prevProps, nextProps) => {
  // Custom comparison (optional)
  return prevProps.data.id === nextProps.data.id;
});

// useMemo for stable object/array references
function Parent() {
  const [filter, setFilter] = useState('');
  
  // Stable reference for style object
  const containerStyle = useMemo(() => ({
    padding: 20,
    maxWidth: 800,
    margin: '0 auto',
  }), []);
  
  // Stable config object
  const tableConfig = useMemo(() => ({
    sortable: true,
    pageSize: 20,
    columns: ['name', 'email', 'role'],
  }), []);
  
  return <Child style={containerStyle} config={tableConfig} />;
}
```

#### 2. **Virtualization (Large Lists)**

```tsx
import { FixedSizeList as List } from 'react-window';
import { FixedSizeGrid as Grid } from 'react-window';

function VirtualizedList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style} className="list-row">
      {items[index].name}
    </div>
  );
  
  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
      overscanCount={5} // Render extra items for smooth scroll
    >
      {Row}
    </List>
  );
}

// Grid for 2D virtualization
function VirtualizedGrid({ items, columns = 4 }) {
  const cellRenderer = ({ columnIndex, rowIndex, style }) => (
    <div style={style} className="grid-cell">
      {items[rowIndex * columns + columnIndex]?.name}
    </div>
  );
  
  return (
    <Grid
      columnCount={columns}
      rowCount={Math.ceil(items.length / columns)}
      columnWidth={200}
      rowHeight={150}
      width={800}
      height={600}
    >
      {cellRenderer}
    </Grid>
  );
}

// Intersection Observer for lazy loading
function LazyImage({ src, alt }) {
  const [isVisible, setIsVisible] = useState(false);
  const imgRef = useRef<HTMLImageElement>(null);
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => entry.isIntersecting && setIsVisible(true),
      { rootMargin: '100px' }
    );
    if (imgRef.current) observer.observe(imgRef.current);
    return () => observer.disconnect();
  }, []);
  
  return (
    <div ref={imgRef} className={isVisible ? 'visible' : ''}>
      {isVisible && <img src={src} alt={alt} loading="lazy" />}
      {!isVisible && <div className="placeholder" />}
    </div>
  );
}
```

#### 3. **State Colocation & Avoiding Re-renders**

```tsx
// BAD: State at top level causes all children to re-render
function App() {
  const [count, setCount] = useState(0);
  const [user, setUser] = useState(null);
  
  return (
    <div>
      <Header user={user} />        // Re-renders on count change!
      <Counter count={count} />     // Re-renders on user change!
      <Sidebar />                   // Re-renders on both!
    </div>
  );
}

// GOOD: Colocate state where it's used
function App() {
  return (
    <div>
      <Header />      // Owns user state
      <Counter />     // Owns count state
      <Sidebar />     // Owns its own state
    </div>
  );
}

// Context splitting for performance
const UserContext = createContext(null);
const ThemeContext = createContext(null);

function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <AppContent />
      </ThemeProvider>
    </UserProvider>
  );
}

// Consumer only re-renders when ITS context changes
function Header() {
  const theme = useContext(ThemeContext); // Only re-renders on theme change
  return <header className={theme}>...</header>;
}
```

#### 4. **Advanced Patterns**

```tsx
// useTransition for non-blocking updates (React 18)
function SearchResults() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value); // Urgent: update input immediately
    startTransition(() => {
      setSearchQuery(value); // Non-urgent: defer filtering
    });
  };
  
  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <Results query={searchQuery} />}
    </div>
  );
}

// useDeferredValue for deferred rendering
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  
  // Expensive filtering uses deferred value
  const results = useMemo(() => 
    expensiveFilter(items, deferredQuery), 
    [deferredQuery]
  );
  
  return <List items={results} />;
}

// useSyncExternalStore for external subscriptions
import { useSyncExternalStore } from 'react';

function useOnlineStatus() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine,
    () => true // SSR fallback
  );
}

// useId for stable IDs (SSR safe)
function FormField({ label }) {
  const id = useId();
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </div>
  );
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Lazy + Suspense** | Import all routes | Large initial bundle |
| **React.memo** | Memo everything | Overhead > benefit for simple components |
| **useMemo/useCallback** | Memoize everything | Overhead, stale closures |
| **Virtualization** | Render 1000+ items | DOM nodes = memory + layout cost |
| **Code splitting** | Single bundle | Large initial load |
| **State colocation** | Global state for everything | Unnecessary re-renders |
| **Web Workers** | Heavy calc on main thread | Blocked UI, jank |
| **CSS containment** | Layout thrashing | Forced synchronous layouts |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "How does React's reconciliation work?"

```
1. Trigger: setState/props change → schedule update
2. Render phase: Component functions execute → new fiber tree
3. Reconciliation (diffing):
   - Same type + same key → UPDATE (props change)
   - Different type → REPLACE (destroy old, create new)
   - Lists: KEYS determine identity (NOT index!)
4. Commit phase:
   - DOM mutations applied
   - useLayoutEffect (sync)
   - Browser paint
   - useEffect (async)
   
KEY INSIGHT: Keys tell React "this is the SAME element" across renders.
Without keys, React assumes index = identity (breaks on reorder/filter).
```

#### 2. **Code**: "Fix the performance issue"

```tsx
// BUGGY: Re-renders all items on any change
function UserList({ users, onSelect }) {
  return (
    <ul>
      {users.map(user => (
        <UserItem 
          key={user.id} 
          user={user} 
          onSelect={onSelect} 
        />
      ))}
    </ul>
  );
}

function UserItem({ user, onSelect }) {
  // BUG: New function every render
  const handleClick = () => onSelect(user.id);
  
  return <li onClick={handleClick}>{user.name}</li>;
}

// FIXED
const UserItem = memo(function UserItem({ user, onSelect }) {
  // Stable callback reference
  const handleClick = useCallback(() => onSelect(user.id), [onSelect, user.id]);
  
  return <li onClick={handleClick}>{user.name}</li>;
});

// Parent provides stable callback
function UserList({ users, onSelect }) {
  const handleSelect = useCallback((id) => {
    // handle selection
  }, []);
  
  return (
    <ul>
      {users.map(user => (
        <UserItem key={user.id} user={user} onSelect={handleSelect} />
      ))}
    </ul>
  );
}
```

#### 3. **Code**: "Implement virtualized list with search"

```tsx
import { FixedSizeList as List } from 'react-window';
import { useMemo, useState } from 'react';

function VirtualizedSearchList({ allItems }) {
  const [query, setQuery] = useState('');
  
  // Filter outside render (memoized)
  const filteredItems = useMemo(() => 
    allItems.filter(item => 
      item.name.toLowerCase().includes(query.toLowerCase())
    ), 
    [allItems, query]
  );
  
  const Row = ({ index, style }) => (
    <div style={style} className="item-row">
      {filteredItems[index].name}
    </div>
  );
  
  return (
    <div>
      <input 
        value={query} 
        onChange={e => setQuery(e.target.value)} 
        placeholder="Search..."
      />
      <List
        height={500}
        itemCount={filteredItems.length}
        itemSize={50}
        width="100%"
        overscanCount={5}
      >
        {Row}
      </List>
    </div>
  );
}
```

#### 4. **Advanced**: "When does React re-render?"

```
RE-RENDER TRIGGERS:
1. setState() called in component
2. forceUpdate() called
3. Parent re-renders (unless memoized)
4. Context value changes (for consumers)
4. Hooks return new references (useState, useReducer dispatch)

RE-RENDER PREVENTION:
1. React.memo(Component) - shallow prop comparison
2. useMemo(value, deps) - memoize computed value
3. useCallback(fn, deps) - memoize function reference
4. useRef - mutable container, no re-render
5. State colocation - move state down
6. Context splitting - separate contexts for different update frequencies

BENCHMARK RULE: Measure first! React DevTools Profiler → "Record why each component rendered"
```

#### 5. **Code**: "Bundle analysis and optimization"

```bash
# Analyze bundle
npm run build -- --analyze
# or
npx vite-bundle-analyzer

# Webpack Bundle Analyzer
npm install --save-dev webpack-bundle-analyzer
# Add to webpack config:
plugins: [new BundleAnalyzerPlugin()]

# Vite
npm run build -- --mode analyze
```

```tsx
// vite.config.ts optimization
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          'utils': ['date-fns', 'zod', 'axios'],
        },
      },
    },
    minify: 'esbuild', // or 'terser' for better compression
    cssCodeSplit: true,
    cssMinify: true,
  },
  esbuild: {
    drop: ['console', 'debugger'], // Remove in prod
    pure: ['console.log'], // Remove console.log calls
  },
});
```

---

### GOTCHAS & EDGE CASES

- [ ] **Premature Optimization** - Measure first (React DevTools Profiler)
- [ ] **Memo Overhead** - `memo`/`useMemo` has cost; only for expensive components
- [ ] **Stale Closures** - `useCallback` deps missing → stale values
- [ ] **Index as Key** - Breaks on reorder/filter → always use stable IDs
- [ ] **Inline Objects** - `<Child style={{}} />` creates new object each render
- [ ] **Context Re-renders** - Single context = all consumers re-render on ANY change
- [ ] **SSR Hydration** - Client/server mismatch → hydration errors
- [ ] **Web Workers** - Transferable objects for large data transfer
- [ ] **CSS Containment** - `contain: layout paint` isolates layout
- [ ] **Third-party Scripts** - `defer`, `async`, `partytown` for off-main-thread

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Profiler Wrapper for Development**

```tsx
// ProfilerWrapper.tsx
import { Profiler, ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (
  id, phase, actualDuration, baseDuration, startTime, commitTime
) => {
  if (actualDuration > 16) { // Slower than 60fps frame
    console.warn(`[Perf] ${id} ${phase}: ${actualDuration.toFixed(2)}ms`);
  }
};

export function ProfilerWrapper({ children, id = 'App' }) {
  return (
    <Profiler id={id} onRender={onRender}>
      {children}
    </Profiler>
  );
}

// Usage
<ProfilerWrapper id="UserList">
  <UserList users={users} />
</ProfilerWrapper>
```

#### 2. **Web Worker for Heavy Computation**

```tsx
// worker.ts
self.onmessage = (e) => {
  const { data, type } = e.data;
  let result;
  
  switch (type) {
    case 'PROCESS_LARGE_DATASET':
      result = heavyProcessing(data);
      break;
    case 'SORT':
      result = data.sort((a, b) => a[sortBy] - b[sortBy]);
      break;
  }
  
  self.postMessage({ result, type });
};

// Component
function useWebWorker() {
  const workerRef = useRef<Worker>();
  
  useEffect(() => {
    workerRef.current = new Worker(new URL('./worker.ts', import.meta.url));
    return () => workerRef.current?.terminate();
  }, []);
  
  const runTask = useCallback(<T,>(type: string, data: any): Promise<T> => {
    return new Promise((resolve) => {
      const handler = (e: MessageEvent) => {
        if (e.data.type === type) {
          workerRef.current?.removeEventListener('message', handler);
          resolve(e.data.result);
        }
      };
      workerRef.current?.addEventListener('message', handler);
      workerRef.current?.postMessage({ type, data });
    }, []);
    
    return { runTask };
  }, []);
```

#### 3. **CSS Containment for Layout Isolation**

```css
/* Isolate layout/paint to component subtree */
.virtual-list {
  contain: layout paint style size;
  /* layout: internal layout doesn't affect outside */
  /* paint: descendants don't display outside bounds */
  /* size: size computed without children */
  /* style: no style inheritance from outside */
}

/* Content-visibility for offscreen content */
.offscreen-item {
  content-visibility: auto;
  contain-intrinsic-size: 200px; /* Reserve space */
}

/* Will-change for animations */
.animate-on-scroll {
  will-change: transform, opacity;
}
```

---

### PRACTICE PROBLEMS

1. **Profile** a React app: Identify top 3 slow components, optimize each
2. **Implement** a search with debounce + virtualization + suspense
3. **Optimize** a dashboard with 50 widgets: code split, lazy load, memoize
4. **Debug** a memory leak: detached DOM nodes, uncleaned subscriptions
5. **Audit** bundle size: identify duplicates, unused deps, large packages

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| React Render Phases | 1. Render (async, interruptible): diff, fiber tree. 2. Commit (sync): DOM mutations, layout effects, paint, effects |
| Reconciliation Rules | 1. Different type → replace. 2. Same type → update props. 3. Lists: keys for identity |
| Key Purpose | Stable identity across renders. Enables move detection. Never use index! |
| useMemo vs useCallback | useMemo: memoizes VALUE. useCallback: memoizes FUNCTION (≈ useMemo(() => fn, deps)) |
| React.memo | HOC: shallow prop comparison. Custom compare: `memo(Comp, (prev, next) => bool)` |
| useTransition | Marks update as non-urgent. `startTransition(() => setState())` - keeps UI responsive |
| useDeferredValue | Defers value update. `const deferred = useDeferredValue(value)` - for expensive rendering |
| Virtualization | Render only visible items. `react-window` / `react-virtualized`. Overscan for smooth scroll |
| Code Splitting | `lazy(() => import())` + `<Suspense>`. Route-level + component-level |
| Bundle Analysis | `vite-bundle-analyzer`, `webpack-bundle-analyzer`. Look for: duplicates, large deps, unused code |
| React.memo Pitfall | Props must be stable (useCallback/useMemo) or custom compare needed |
| Profiler API | `<Profiler id="X" onRender={(id, phase, actual, base) => {...}}>` - measure render times |

---

## NEXT TOPIC: `07-FRAMEWORKS.md` (Next.js, Astro, Remix)

> **Study Tip**: Open React DevTools Profiler, record a slow interaction, identify "why did this render", fix top 3 offenders. Measure bundle with `npm run build -- --analyze`.