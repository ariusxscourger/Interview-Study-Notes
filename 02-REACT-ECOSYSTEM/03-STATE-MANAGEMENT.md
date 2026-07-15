# STATE MANAGEMENT IN REACT
## Exam-Style Study Notes

---

## TOPIC: State Management Patterns & Libraries

### WHY? (Problem Statement)

**State Management Challenges in React:**
- Prop drilling through many layers
- Sibling components needing shared state
- Global state (auth, theme, user preferences)
- Server state vs client state confusion
- Caching, deduplication, background updates
- Optimistic updates & rollbacks

**Evolution of Solutions:**
```
useState/useContext → Redux → MobX → Recoil/Jotai → Zustand → React Query + Zustand
     (Built-in)       (Boilerplate)  (Mutable)   (Atomic)      (Simple)      (Server+Client)
```

---

### HOW? (Mental Models)

#### 1. **State Classification**

| State Type | Examples | Management Approach |
|------------|----------|---------------------|
| **Local UI** | Modal open, input value, dropdown | `useState`, `useReducer` |
| **Feature** | Form data, wizard step, filters | `useReducer`, Context |
| **Global Client** | Auth user, theme, locale, cart | Context + Reducer, Zustand, Jotai |
| **Server State** | Users list, product details, posts | React Query, SWR, RTK Query |
| **URL State** | Filters, pagination, tabs | `useSearchParams`, React Router |
| **Derived State** | Full name from first+last, filtered list | Compute during render (`useMemo`) |

#### 2. **Server State vs Client State**

```
SERVER STATE (Async, External, Cached):
─────────────────────────────────────────────────────────────
- Users list, Product catalog, Notifications
- Owned by server, we cache locally
- Challenges: Stale data, deduplication, background refetch
- Tools: React Query, SWR, RTK Query

CLIENT STATE (Sync, Local, Ephemeral):
─────────────────────────────────────────────────────────────
- Modal open, Theme, Form draft, UI preferences
- Owned by client, no server sync
- Challenges: Persistence, cross-component sharing
- Tools: Zustand, Jotai, Context, localStorage
```

---

### WHAT? (Libraries & Patterns)

#### 1. **React Query (TanStack Query) - Server State Standard**

```tsx
// Setup
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 min
      gcTime: 1000 * 60 * 30,   // 30 min (cache)
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

// Query (GET)
function UserList() {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ['users', { page: 1, limit: 20 }],
    queryFn: () => api.getUsers({ page: 1, limit: 20 }),
    select: (response) => response.data, // Transform
  });
  
  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  
  return (
    <div>
      <button onClick={() => refetch()}>Refresh</button>
      {data.users.map(u => <UserCard key={u.id} user={u} />)}
    </div>
  );
}

// Mutation (POST/PUT/DELETE)
function useUpdateUser() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (variables) => api.updateUser(variables.id, variables.data),
    
    // Optimistic update
    onMutate: async (variables) => {
      await queryClient.cancelQueries({ queryKey: ['users'] });
      const previousUsers = queryClient.getQueryData(['users']);
      
      queryClient.setQueryData(['users'], (old) => 
        old.map(u => u.id === variables.id ? { ...u, ...variables.data } : u)
      );
      
      return { previousUsers };
    },
    
    onError: (err, variables, context) => {
      queryClient.setQueryData(['users'], context.previousUsers);
    },
    
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

// Infinite Query (Pagination)
function InfiniteUserList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['users', 'infinite'],
    queryFn: ({ pageParam = 1 }) => api.getUsers({ page: pageParam, limit: 20 }),
    getNextPageParam: (lastPage) => lastPage.nextPage,
    initialPageParam: 1,
  });
  
  return (
    <div>
      {data.pages.flatMap(page => page.users).map(u => <UserCard key={u.id} user={u} />)}
      <button 
        onClick={() => fetchNextPage()} 
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : 'Load More'}
      </button>
    </div>
  );
}
```

#### 2. **Zustand - Simple Global Client State**

```tsx
// store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface AuthState {
  user: User | null;
  token: string | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
  updateProfile: (data: Partial<User>) => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      token: null,
      
      login: async (credentials) => {
        const { user, token } = await api.login(credentials);
        set({ user, token });
        api.setAuthToken(token);
      },
      
      logout: () => {
        set({ user: null, token: null });
        api.clearAuthToken();
      },
      
      updateProfile: (data) => set((state) => ({
        user: state.user ? { ...state.user, ...data } : null,
      })),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ user: state.user, token: state.token }),
    }
  )
);

// Selective subscriptions (prevent re-renders)
function Header() {
  // Only re-renders when user changes
  const user = useAuthStore((state) => state.user);
  const logout = useAuthStore((state) => state.logout);
  
  return <header>{user?.name} <button onClick={logout}>Logout</button></header>;
}

function UserMenu() {
  // Only re-renders when token changes
  const token = useAuthStore((state) => state.token);
  // ...
}

// Transient updates (no re-render)
function useAuthActions() {
  return useAuthStore(
    useShallow((state) => ({ 
      login: state.login, 
      logout: state.logout,
      updateProfile: state.updateProfile 
    }))
  );
}
```

#### 3. **Jotai - Atomic State (Fine-grained Reactivity)**

```tsx
// atoms.ts
import { atom, useAtom, atomWithStorage, atomWithDefault } from 'jotai';

// Primitive atoms
export const countAtom = atom(0);
export const themeAtom = atomWithStorage('theme', 'light');

// Derived atoms (computed)
export const doubleCountAtom = atom((get) => get(countAtom) * 2);

// Async atoms
export const userAtom = atomWithDefault(async () => {
  const response = await fetch('/api/user');
  return response.json();
});

// Atom family (parameterized)
export const userByIdAtom = atomFamily((id: string) => 
  atomWithDefault(async () => {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
  })
);

// Write-only atom (action)
export const incrementAtom = atom(
  null, // no read
  (get, set, by: number) => {
    set(countAtom, get(countAtom) + by);
  }
);

// Component usage
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const [double] = useAtom(doubleCountAtom);
  
  return (
    <div>
      <p>Count: {count}</p>
      <p>Double: {double}</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
      <button onClick={() => setCount(c => c - 1)}>-1</button>
    </div>
  );
}

// Async atom usage
function UserProfile({ userId }) {
  const [user] = useAtom(userByIdAtom(userId));
  
  if (!user) return <Skeleton />;
  return <div>{user.name}</div>;
}
```

#### 4. **Redux Toolkit (RTK) - Enterprise Standard**

```tsx
// store.ts
import { configureStore, createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// Async thunk
export const fetchUsers = createAsyncThunk(
  'users/fetchAll',
  async (params: { page: number; limit: number }, { rejectWithValue }) => {
    try {
      const response = await api.getUsers(params);
      return response.data;
    } catch (err) {
      return rejectWithValue(err.response.data);
    }
  }
);

// Slice
const usersSlice = createSlice({
  name: 'users',
  initialState: {
    items: [],
    status: 'idle', // 'idle' | 'loading' | 'succeeded' | 'failed'
    error: null,
    pagination: { page: 1, limit: 20, total: 0 },
  },
  reducers: {
    setPage: (state, action) => {
      state.pagination.page = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload.users;
        state.pagination.total = action.payload.total;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload as string;
      });
  },
});

export const { setPage } = usersSlice.actions;
export default usersSlice.reducer;

// Store configuration
export const store = configureStore({
  reducer: {
    users: usersReducer,
    auth: authReducer,
    // ...
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST'],
      },
    }),
});

// Typed hooks
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();

// Component
function UserList() {
  const { items, status, error, pagination } = useAppSelector(
    (state) => state.users
  );
  const dispatch = useAppDispatch();
  
  useEffect(() => {
    dispatch(fetchUsers({ page: pagination.page, limit: pagination.limit }));
  }, [pagination.page, dispatch]);
  
  if (status === 'loading') return <Spinner />;
  if (error) return <Error error={error} onRetry={() => dispatch(fetchUsers({ page: pagination.page }))} />;
  
  return (
    <div>
      {items.map(user => <UserCard key={user.id} user={user} />)}
      <Pagination 
        current={pagination.page}
        total={pagination.total}
        onPageChange={(page) => dispatch(setPage(page))}
      />
    </div>
  );
}
```

#### 5. **Context + Reducer (Built-in, No Dependencies)**

```tsx
// AuthContext.tsx
interface AuthState {
  user: User | null;
  token: string | null;
  login: (creds: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthState | null>(null);

export function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(authReducer, initialState);
  
  const login = useCallback(async (creds) => {
    dispatch({ type: 'LOGIN_START' });
    try {
      const { user, token } = await api.login(creds);
      dispatch({ type: 'LOGIN_SUCCESS', payload: { user, token } });
    } catch (error) {
      dispatch({ type: 'LOGIN_FAILURE', payload: error.message });
    }
  }, []);
  
  const logout = useCallback(() => {
    dispatch({ type: 'LOGOUT' });
  }, []);
  
  const value = useMemo(() => ({ ...state, login, logout }), [state, login, logout]);
  
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// Selective subscriptions (prevent re-renders)
function useAuthSelector<T>(selector: (state: AuthState) => T) {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return useSyncExternalStore(
    context.subscribe,
    () => selector(context.getState()),
    () => selector(context.getInitialState())
  );
}

// Usage
function Header() {
  const user = useAuthSelector(s => s.user); // Only re-renders when user changes
  const logout = useAuthSelector(s => s.logout);
  
  return <header>{user?.name} <button onClick={logout}>Logout</button></header>;
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Server State → React Query** | useState + useEffect for data | No caching, deduping, background refetch |
| **Global UI → Zustand/Jotai** | Context for everything | Re-renders all consumers on any change |
| **Forms → React Hook Form** | Manual useState for each field | Boilerplate, validation complexity |
| **URL State → React Router** | State for filters/pagination | Not shareable, not bookmarkable |
| **Derived State → Compute in Render** | useState for derived | Stale data, sync bugs |
| **Optimistic Updates** | Wait for server | Poor UX, perceived slowness |
| **Selective Subscriptions** | useContext whole object | All consumers re-render on any change |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "When to use React Query vs Redux vs Context?"

```
REACT QUERY (Server State):
- API data, caching, background updates, pagination
- Automatic deduping, stale-while-revalidate
- DevTools, retries, mutations with rollback

REDUX (Complex Client State):
- Large apps, complex state transitions
- Time-travel debugging, middleware ecosystem
- Strict patterns, team scaling

ZUSTAND/JOTAI (Simple Client State):
- Theme, auth, modals, simple global UI
- Less boilerplate, React-friendly APIs
- Fine-grained reactivity (Jotai)

CONTEXT (Built-in, Low Frequency):
- Theme, locale, auth (rarely changes)
- Avoid for high-frequency updates
```

#### 2. **Code**: "Implement optimistic update with rollback"

```tsx
function useLikePost() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ postId, action }) => api.likePost(postId, action),
    
    onMutate: async ({ postId, action }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['posts'] });
      
      // Snapshot previous value
      const previousPosts = queryClient.getQueryData(['posts']);
      
      // Optimistic update
      queryClient.setQueryData(['posts'], (old: Post[]) =>
        old.map(post => 
          post.id === postId 
            ? { ...post, liked: action === 'like', likesCount: post.likesCount + (action === 'like' ? 1 : -1) }
            : post
        )
      );
      
      return { previousPosts };
    },
    
    onError: (err, variables, context) => {
      // Rollback
      queryClient.setQueryData(['posts'], context.previousPosts);
    },
    
    onSettled: () => {
      // Refetch to ensure consistency
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });
}
```

#### 3. **Design**: "Shopping cart - where does state live?"

```
CART ITEMS → Zustand (Client State)
- Add/remove items
- Quantity changes
- Persist to localStorage
- Sync across tabs (BroadcastChannel)

CHECKOUT → React Query Mutation
- POST /orders
- Optimistic: clear cart on success
- Rollback on failure

PRODUCT DATA → React Query
- Cached, background refresh
- Infinite scroll for listings

USER AUTH → Zustand + React Query
- User profile in Zustand (persisted)
- Permissions from React Query
```

#### 4. **Code**: "React Hook Form + Zod validation"

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(2, 'Name too short'),
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password too short').regex(/[A-Z]/, 'Need uppercase'),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords must match',
  path: ['confirmPassword'],
});

type FormData = z.infer<typeof schema>;

function RegistrationForm() {
  const { 
    register, 
    handleSubmit, 
    formState: { errors, isSubmitting, isValid },
    setValue,
    watch,
    reset 
  } = useForm<FormData>({
    resolver: zodResolver(schema),
    mode: 'onBlur',
    defaultValues: { name: '', email: '', password: '', confirmPassword: '' },
  });
  
  // Watch for dependent fields
  const password = watch('password');
  
  const onSubmit = async (data: FormData) => {
    try {
      await api.register(data);
      reset();
      toast.success('Registered!');
    } catch (err) {
      // Set server errors
      err.errors?.forEach(e => setError(e.field, { message: e.message }));
    }
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <Field
        label="Name"
        {...register('name')}
        error={errors.name?.message}
      />
      
      <Field
        label="Email"
        type="email"
        {...register('email')}
        error={errors.email?.message}
      />
      
      <Field
        label="Password"
        type="password"
        {...register('password')}
        error={errors.password?.message}
        helperText={password && password.length < 8 ? 'Min 8 chars' : undefined}
      />
      
      <Field
        label="Confirm Password"
        type="password"
        {...register('confirmPassword')}
        error={errors.confirmPassword?.message}
      />
      
      <button type="submit" disabled={isSubmitting || !isValid}>
        {isSubmitting ? 'Registering...' : 'Register'}
      </button>
    </form>
  );
}
```

#### 5. **Advanced**: "State synchronization across tabs"

```tsx
// zustand with cross-tab sync
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, qty: number) => void;
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],
      
      addItem: (item) => set((state) => {
        const existing = state.items.find(i => i.id === item.id);
        if (existing) {
          return { items: state.items.map(i => i.id === item.id ? { ...i, qty: i.qty + 1 } : i) };
        }
        return { items: [...state.items, { ...item, qty: 1 }] };
      }),
      
      removeItem: (id) => set((state) => ({
        items: state.items.filter(i => i.id !== id)
      })),
      
      updateQuantity: (id, qty) => set((state) => ({
        items: qty <= 0 
          ? state.items.filter(i => i.id !== id)
          : state.items.map(i => i.id === id ? { ...i, qty } : i)
      })),
    }),
    {
      name: 'cart-storage',
      storage: createJSONStorage(() => localStorage),
      onRehydrateStorage: () => (state) => {
        // Sync across tabs
        window.addEventListener('storage', (e) => {
          if (e.key === 'cart-storage' && e.newValue) {
            const newState = JSON.parse(e.newValue);
            get().setState(newState);
          }
        });
      },
    }
  );
}

// BroadcastChannel for real-time sync (better than storage event)
const channel = new BroadcastChannel('cart-sync');

channel.onmessage = (e) => {
  if (e.data.type === 'CART_UPDATE') {
    useCartStore.setState(e.data.payload);
  }
};

// In actions, broadcast after setState
addItem: (item) => {
  set((state) => { /* ... */ });
  channel.postMessage({ type: 'CART_UPDATE', payload: get() });
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **React Query Stale Time** - Default 0 = always stale. Set `staleTime: 5*60*1000` for caching
- [ ] **Zustand Equality** - Default shallow compare. Use `useShallow` for object selectors
- [ ] **React Hook Form** - Uncontrolled → controlled warnings → always provide `defaultValues`
- [ ] **Zustand Persist** - Only persist serializable state. Exclude functions, class instances
- [ ] **React Query Keys** - Use arrays `['users', { page: 1 }]` not strings `'users-page-1'`
- [ ] **Jotai Derived** - `atom((get) => get(a) + get(b))` auto-subscribes to dependencies
- [ ] **Redux Immutability** - RTK uses Immer internally, but don't mutate outside reducers
- [ ] **Form Validation** - Schema on blur, not on change (performance)
- [ ] **Optimistic Updates** - Always have rollback (`onError` with snapshot)
- [ ] **Server State Sync** - `invalidateQueries` vs `setQueryData` - know when to use each

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **React Query Mutation with Optimistic Update**

```tsx
const useUpdatePost = () => {
  const qc = useQueryClient();
  
  return useMutation({
    mutationFn: (post) => api.updatePost(post.id, post),
    onMutate: async (newPost) => {
      await qc.cancelQueries({ queryKey: ['posts'] });
      const previous = qc.getQueryData(['posts']);
      qc.setQueryData(['posts'], (old) => 
        old.map(p => p.id === newPost.id ? newPost : p)
      );
      return { previous };
    },
    onError: (err, vars, ctx) => qc.setQueryData(['posts'], ctx.previous),
    onSettled: () => qc.invalidateQueries({ queryKey: ['posts'] }),
  });
}
```

#### 2. **Zustand with Immer for Nested Updates**

```tsx
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface State {
  user: { profile: { name: string; settings: { theme: string } } } | null;
  updateTheme: (theme: string) => void;
}

export const useStore = create(
  immer((set) => ({
    user: null,
    updateTheme: (theme) => set((state) => {
      if (state.user) state.user.profile.settings.theme = theme;
    }),
  }))
)
```

#### 3. **React Query Infinite Scroll**

```tsx
function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam = 1 }) => api.getPosts({ page: pageParam, limit: 10 }),
    getNextPageParam: (lastPage) => lastPage.hasMore ? lastPage.nextCursor : undefined,
    initialPageParam: 1,
  });
}

function Feed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfinitePosts();
  
  return (
    <div>
      {data?.pages.flatMap(page => page.posts).map(post => <PostCard key={post.id} post={post} />)}
      <button 
        onClick={() => fetchNextPage()} 
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : 'Load More'}
      </button>
    </div>
  );
}
```

---

### PRACTICE PROBLEMS

1. **Build** a complete auth flow: Login → Token storage → Auto-refresh → Protected routes → Logout
2. **Implement** a wizard form with React Hook Form + Zod, step persistence, and back/next navigation
3. **Create** a real-time chat with React Query mutations + WebSocket sync
4. **Debug** a React Query cache issue: duplicate requests, stale data, missing invalidation
5. **Design** state architecture for a dashboard: user prefs, widgets, filters, real-time data

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| React Query staleTime vs gcTime | staleTime: how long data is fresh. gcTime: how long unused data stays in cache. |
| React Query Mutation Flow | onMutate (optimistic) → onError (rollback) → onSettled (invalidate) |
| Zustand vs Redux | Zustand: simpler, hooks-based, mutable (with Immer), no providers. Redux: stricter, DevTools, middleware. |
| Jotai Atom | Primitive unit of state. Derived atoms auto-subscribe. Fine-grained re-renders. |
| React Hook Form vs Controlled | RHF: uncontrolled by default, better perf, less re-renders. Register + handleSubmit. |
| Zod with RHF | `zodResolver(schema)` - validates on blur/submit, infers TypeScript types. |
| Optimistic Update Pattern | Snapshot → Update → Mutate → onError: restore snapshot → onSettled: invalidate |
| React Query Keys | Arrays `['users', {page: 1}]` for structured, type-safe, partial invalidation |
| Zustand Selectors | `useStore(state => state.user)` re-renders on user change. `useShallow` for objects. |
| Context vs Zustand | Context: all consumers re-render on any change. Zustand: selective subscriptions. |

---

## NEXT TOPIC: `04-ROUTING.md`

> **Study Tip**: Set up a project with React Query + Zustand + React Hook Form. Build a feature with: list (infinite scroll), detail, create/edit (optimistic), delete (with confirmation).