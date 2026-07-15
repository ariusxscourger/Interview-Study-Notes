# REACT ROUTING (React Router v6+)
## Exam-Style Study Notes

---

## TOPIC: Client-Side Routing with React Router

### WHY? (Problem Statement)

**Before Client-Side Routing:**
- Full page reloads on navigation
- No state preservation between pages
- Poor UX (flash of white, lost scroll position)
- SEO challenges with SPAs

**What React Router Solves:**
| Problem | Solution |
|---------|----------|
| URL ↔ UI sync | Declarative route definitions |
| Nested layouts | Outlet + nested routes |
| Data loading | Loaders + Actions (v6.4+) |
| Protected routes | Wrapper components |
| Type safety | Route params inference |

---

### HOW? (Router Architecture)

#### 1. **Router Types**

```tsx
// BrowserRouter - Uses History API (clean URLs)
import { BrowserRouter } from 'react-router-dom';

<BrowserRouter>
  <App />
</BrowserRouter>

// HashRouter - Uses URL hash (legacy browser support)
import { HashRouter } from 'react-router-dom';

<HashRouter>
  <App />
</HashRouter>

// MemoryRouter - Testing, non-browser environments
import { MemoryRouter } from 'react-router-dom';

<MemoryRouter initialEntries={['/users/1']}>
  <App />
</MemoryRouter>

// StaticRouter - SSR (Node.js)
import { StaticRouter } from 'react-router-dom/server';

<StaticRouter location={req.url}>
  <App />
</StaticRouter>
```

#### 2. **Route Configuration (v6.4+ Data APIs)**

```tsx
// router.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
      {
        path: 'users',
        element: <UsersLayout />,
        children: [
          { index: true, element: <UserList /> },
          { path: ':userId', element: <UserProfile /> },
          { path: ':userId/edit', element: <EditUser /> },
        ],
      },
      {
        path: 'admin',
        element: <AdminLayout />,
        loader: adminLoader,      // Data loading BEFORE render
        action: adminAction,      // Form submissions
        children: [
          { index: true, element: <AdminDashboard /> },
          { path: 'users', element: <AdminUsers /> },
        ],
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} fallback={<GlobalLoading />} />;
}
```

#### 3. **Loaders & Actions (Data Loading)**

```tsx
// loaders.ts
import { json, redirect } from 'react-router-dom';

// Loader: Fetch data BEFORE component renders
export async function userLoader({ params, request }) {
  const userId = params.userId;
  const response = await fetch(`/api/users/${userId}`);
  
  if (!response.ok) {
    throw json({ message: 'User not found' }, { status: 404 });
  }
  
  return response.json();
}

// Action: Handle form submissions (POST/PUT/DELETE)
export async function userAction({ params, request }) {
  const formData = await request.formData();
  const data = Object.fromEntries(formData);
  
  const response = await fetch(`/api/users/${params.userId}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  
  if (!response.ok) {
    return { errors: await response.json() }; // Return to actionData
  }
  
  return redirect(`/users/${params.userId}`);
}

// Component using loader data
function UserProfile() {
  const user = useLoaderData(); // Typed with loader return type
  const actionData = useActionData(); // Form errors
  const navigation = useNavigation(); // Pending state
  
  return (
    <form method="POST">
      <input 
        name="name" 
        defaultValue={user.name} 
        disabled={navigation.state === 'submitting'}
      />
      {actionData?.errors?.name && <span>{actionData.errors.name}</span>}
      <button disabled={navigation.state === 'submitting'}>
        {navigation.state === 'submitting' ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

---

### WHAT? (Core Patterns)

#### 1. **Route Parameters & Search Params**

```tsx
// Route with params
<Route path="users/:userId/posts/:postId" element={PostDetail} />

// Component
function PostDetail() {
  const { userId, postId } = useParams(); // { userId: '123', postId: '456' }
  const [searchParams, setSearchParams] = useSearchParams();
  
  // Read search params
  const tab = searchParams.get('tab') || 'overview';
  const page = parseInt(searchParams.get('page') || '1');
  
  // Update search params (preserves others)
  const handleTabChange = (newTab) => {
    setSearchParams({ ...Object.fromEntries(searchParams), tab: newTab });
  };
  
  // Navigate with params
  const navigate = useNavigate();
  const goToEdit = () => navigate(`/users/${userId}/posts/${postId}/edit`);
  
  return <div>{/* ... */}</div>;
}

// Type-safe params (TypeScript)
interface PostParams { userId: string; postId: string; }
function PostDetail() {
  const params = useParams<PostParams>(); // Typed!
  // params.userId, params.postId are typed
}
```

#### 2. **Nested Routes & Outlets**

```tsx
// Layout with nested routes
function UsersLayout() {
  return (
    <div className="users-layout">
      <aside className="sidebar">
        <NavLink to="/users">List</NavLink>
        <NavLink to="/users/new">New User</NavLink>
      </aside>
      <main className="content">
        <Outlet /> {/* Child routes render here */}
      </main>
    </div>
  );
}

// Routes
{
  path: 'users',
  element: <UsersLayout />,
  children: [
    { index: true, element: <UserList /> },           // /users
    { path: 'new', element: <CreateUser /> },         // /users/new
    { path: ':userId', element: <UserProfile /> },    // /users/123
    { path: ':userId/edit', element: <EditUser /> },  // /users/123/edit
  ]
}
```

#### 3. **Protected Routes & Role-Based Access**

```tsx
// ProtectedRoute.tsx
interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRole?: string[];
  fallback?: React.ReactNode;
}

export function ProtectedRoute({ 
  children, 
  requiredRole = [], 
  fallback = <Navigate to="/login" replace /> 
}: ProtectedRouteProps) {
  const { user, loading } = useAuth();
  
  if (loading) return <Spinner />;
  if (!user) return fallback;
  
  if (requiredRole.length && !requiredRole.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />;
  }
  
  return <>{children}</>;
}

// Usage in routes
{
  path: 'admin',
  element: (
    <ProtectedRoute requiredRole={['admin']}>
      <AdminLayout />
    </ProtectedRoute>
  ),
  children: [
    { path: 'users', element: <AdminUsers /> },
  ]
}
```

#### 4. **Navigation Patterns**

```tsx
// Imperative navigation
function LoginForm() {
  const navigate = useNavigate();
  const location = useLocation();
  const from = useSearchParams()[0].get('redirect') || '/';
  
  const handleSubmit = async (data) => {
    try {
      await api.login(data);
      navigate(from, { replace: true });
    } catch (error) {
      setError(error.message);
    }
  };
  
  return <form onSubmit={handleSubmit}>...</form>;
}

// Navigation with state
function UserCard({ user }) {
  const navigate = useNavigate();
  
  const handleClick = () => {
    navigate(`/users/${user.id}`, { 
      state: { from: 'user-list' },
      replace: false 
    });
  };
  
  return <div onClick={handleClick}>...</div>;
}

// Receiving state
function UserProfile() {
  const location = useLocation();
  const fromList = location.state?.from === 'user-list';
  
  return <div>{fromList && <BackButton />}</div>;
}

// Programmatic navigation with replace
function FormWizard() {
  const navigate = useNavigate();
  const [step, setStep] = useState(1);
  
  const next = () => navigate(`/wizard/${step + 1}`, { replace: true });
  const back = () => navigate(`/wizard/${step - 1}`, { replace: true });
  
  return <div>...</div>;
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Loader/Action pattern** | useEffect for data | Race conditions, no caching, no pending UI |
| **Outlet for layouts** | Duplicate layout in each route | DRY violation, layout shift on nav |
| **Search params for filters** | State for URL sync | Not shareable, not bookmarkable |
| **Lazy loading routes** | Bundle all routes | Large initial bundle |
| **Navigate replace** | Push for form submits | Back button breaks form flow |
| **Relative paths** | Absolute paths everywhere | Refactoring breaks routes |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "How does React Router v6.4+ data loading differ from useEffect?"

```
OLD (useEffect):
- Component mounts → useEffect fires → fetch → setState → re-render
- Problems: Waterfall requests, no caching, race conditions, no pending UI

NEW (Loader/Action):
- Router calls loader BEFORE rendering component
- Benefits:
  ✓ Parallel data fetching (multiple loaders run concurrently)
  ✓ Automatic caching & deduplication
  ✓ Built-in pending/error states (useNavigation, useRouteError)
  ✓ Form submissions via Action (progressive enhancement)
  ✓ Optimistic updates via action + loader coordination
  ✓ SEO-friendly (SSR loads data before HTML)
```

#### 2. **Code**: "Implement a route guard with loading state"

```tsx
// AuthGuard.tsx
export async function requireAuth({ request }) {
  const token = request.headers.get('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    throw redirect('/login?redirect=' + new URL(request.url).pathname);
  }
  
  try {
    const user = await validateToken(token);
    return user;
  } catch {
    throw redirect('/login');
  }
}

// Role guard
export async function requireRole({ params, request }) {
  const user = await requireAuth({ request });
  
  const requiredRoles = params.requiredRoles?.split(',') || [];
  if (requiredRoles.length && !requiredRoles.includes(user.role)) {
    throw redirect('/unauthorized');
  }
  
  return user;
}

// Route config
{
  path: 'admin',
  element: <AdminLayout />,
  loader: requireAuth,
  children: [
    { 
      path: 'users', 
      element: <AdminUsers />, 
      loader: ({ params, request }) => requireRole({ 
        params: { requiredRoles: 'admin' }, 
        request 
      }) 
    },
  ]
}
```

#### 3. **Code**: "Parallel data loading with defer"

```tsx
// Heavy data loaded after initial render
export async function dashboardLoader({ request }) {
  // Critical data - blocks render
  const user = await fetchUser();
  const notifications = await fetchNotifications();
  
  // Non-critical - streams in later
  const analytics = defer(fetchAnalytics());
  const recommendations = defer(fetchRecommendations());
  
  return { user, notifications, analytics, recommendations };
}

function Dashboard() {
  const { user, notifications, analytics, recommendations } = useLoaderData();
  
  return (
    <div>
      <header>Welcome, {user.name}</header>
      <Notifications data={notifications} />
      
      {/* Suspense boundaries for deferred data */}
      <Suspense fallback={<AnalyticsSkeleton />}>
        <Await resolve={analytics}>
          {(data) => <AnalyticsChart data={data} />}
        </Await>
      </Suspense>
      
      <Suspense fallback={<RecsSkeleton />}>
        <Await resolve={recommendations}>
          {(data) => <Recommendations data={data} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

#### 4. **Code**: "Lazy loading routes with Suspense"

```tsx
// routes.tsx
import { lazy, Suspense } from 'react';

const AdminDashboard = lazy(() => import('./AdminDashboard'));
const AdminUsers = lazy(() => import('./AdminUsers'));
const UserProfile = lazy(() => import('./UserProfile'));

// Route config
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      { index: true, element: <Home /> },
      { 
        path: 'admin/*', 
        element: (
          <Suspense fallback={<AdminSkeleton />}>
            <AdminRoutes />
          </Suspense>
        ) 
      },
      { 
        path: 'users/:userId', 
        element: (
          <Suspense fallback={<ProfileSkeleton />}>
            <UserProfile />
          </Suspense>
        ),
        loader: userLoader,
      },
    ],
  ],
);

// AdminRoutes.tsx (separate file for code splitting)
const AdminRoutes = () => (
  <Suspense fallback={<AdminSkeleton />}>
    <Routes>
      <Route path="dashboard" element={<AdminDashboard />} />
      <Route path="users" element={<AdminUsers />} />
    </Routes>
  </Suspense>
);
```

#### 5. **Advanced**: "URL State Management for Complex Filters"

```tsx
// hooks/useFilters.ts
function useFilters<T extends Record<string, any>>(defaults: T) {
  const [searchParams, setSearchParams] = useSearchParams();
  
  // Parse params with defaults
  const filters = useMemo(() => {
    const parsed: T = { ...defaults };
    for (const key of Object.keys(defaults)) {
      const value = searchParams.get(key);
      if (value !== null) {
        const defaultVal = defaults[key];
        if (typeof defaultVal === 'number') {
          parsed[key] = parseFloat(value) as any;
        } else if (typeof defaultVal === 'boolean') {
          parsed[key] = value === 'true';
        } else if (Array.isArray(defaultVal)) {
          parsed[key] = value.split(',') as any;
        } else {
          parsed[key] = value as any;
        }
      }
    }
    return parsed;
  }, [searchParams]);
  
  const setFilters = useCallback((updates: Partial<T>) => {
    setSearchParams((prev) => {
      const next = new URLSearchParams(prev);
      for (const [key, value] of Object.entries(updates)) {
        if (value === defaults[key] || value === undefined || value === null) {
          next.delete(key);
        } else if (Array.isArray(value)) {
          next.set(key, value.join(','));
        } else {
          next.set(key, String(value));
        }
      }
      return next;
    });
  }, [defaults]);
  
  const resetFilters = useCallback(() => {
    setSearchParams({});
  }, [setSearchParams]);
  
  return { filters, setFilters, resetFilters };
}

// Usage
function ProductList() {
  const { filters, setFilters, resetFilters } = useFilters({
    category: 'all',
    minPrice: 0,
    maxPrice: 1000,
    sort: 'newest',
    page: 1,
    tags: [],
  });
  
  const { data } = useQuery({
    queryKey: ['products', filters],
    queryFn: () => fetchProducts(filters),
  });
  
  return (
    <div>
      <FilterSidebar filters={filters} onChange={setFilters} onReset={resetFilters} />
      <ProductGrid products={data} />
    </div>
  );
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Loader Errors** - Must throw `json()` or `redirect()`, not regular Error
- [ ] **Action Revalidation** - Actions auto-revalidate loaders, but only for current route
- [ ] **Navigate Replace** - Use `replace: true` for form submits, wizards
- [ ] **Search Params** - Always strings, need parsing for numbers/booleans
- [ ] **Lazy + Suspense** - Must wrap lazy component in Suspense boundary
- [ ] **Relative Paths** - `navigate('../edit')` vs absolute `/users/123/edit`
- [ ] **Form Actions** - Default to current route, can target specific route
- [ ] **Navigation Blocking** - `useBlocker` for unsaved changes warning
- [ ] **Hydration Mismatch** - SSR + client data must match
- [ ] **Route Rank** - More specific routes match first (exact > params > splat)

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Router Setup with Types**

```tsx
// router.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { RootLayout } from './layouts/RootLayout';
import { Home } from './pages/Home';
import { Login } from './pages/Login';
import { UserLayout } from './layouts/UserLayout';
import { UserProfile } from './pages/UserProfile';
import { ErrorPage } from './pages/ErrorPage';

export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      { index: true, element: <Home /> },
      { path: 'login', element: <Login /> },
      {
        path: 'app',
        element: <UserLayout />,
        loader: requireAuth,
        children: [
          { path: 'profile', element: <UserProfile /> },
          { path: 'settings', element: <Settings /> },
        ],
      },
    ],
  },
);

// main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { router } from './router';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <RouterProvider router={router} fallback={<GlobalLoading />} />
  </StrictMode>
);
```

#### 2. **Form with Action + Optimistic UI**

```tsx
function EditProfile() {
  const user = useLoaderData<User>();
  const actionData = useActionData<{ errors?: Partial<User> }>();
  const navigation = useNavigation();
  
  return (
    <Form method="POST" id="profile-form">
      <div>
        <label htmlFor="name">Name</label>
        <input 
          name="name" 
          id="name" 
          defaultValue={user.name}
          disabled={navigation.state === 'submitting'}
        />
        {actionData?.errors?.name && <p className="error">{actionData.errors.name}</p>}
      </div>
      
      <div>
        <label htmlFor="email">Email</label>
        <input 
          name="email" 
          id="email" 
          type="email" 
          defaultValue={user.email}
          disabled={navigation.state === 'submitting'}
        />
      </div>
      
      <Button type="submit" disabled={navigation.state === 'submitting'}>
        {navigation.state === 'submitting' ? 'Saving...' : 'Save Changes'}
      </Button>
    </Form>
  );
}

// Action
export async function profileAction({ request, params }) {
  const formData = await request.formData();
  const data = Object.fromEntries(formData);
  
  const response = await fetch(`/api/users/${params.userId}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  
  if (!response.ok) {
    return { errors: await response.json() };
  }
  
  return redirect(`/app/profile`);
}
```

#### 3. **Navigation Blocking for Unsaved Changes**

```tsx
function EditForm() {
  const [formData, setFormData] = useState({ name: '', email: '' });
  const [isDirty, setIsDirty] = useState(false);
  
  // Block navigation when form is dirty
  useBlocker(
    ({ currentLocation, nextLocation }) => 
      isDirty && currentLocation.pathname !== nextLocation.pathname
  );
  
  const handleChange = (e) => {
    setFormData(prev => ({ ...prev, [e.target.name]: e.target.value }));
    setIsDirty(true);
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    await api.updateProfile(formData);
    setIsDirty(false);
    navigate('/profile');
  };
  
  return (
    <Form onSubmit={handleSubmit}>
      <input name="name" defaultValue={formData.name} onChange={handleChange} />
      <input name="email" defaultValue={formData.email} onChange={handleChange} />
      <button type="submit">Save</button>
    </Form>
  );
}
```

---

### PRACTICE PROBLEMS

1. **Build** a multi-step wizard with URL sync, back/next, and data persistence
2. **Implement** nested routes with shared layout, sidebar, and breadcrumbs
3. **Create** a route-based permission system with roles and resource ownership
4. **Debug** a hydration mismatch between SSR loader and client data
5. **Implement** deep linking with shareable filter/sort/pagination state

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| React Router v6.4+ Data APIs | Loader (GET), Action (POST/PUT/DELETE), automatic revalidation |
| Loader vs Action | Loader: data fetching, runs before render. Action: mutations, form submissions |
| useLoaderData | Returns loader data, typed with loader return type |
| useActionData | Returns action return value (usually form errors) |
| useNavigation | Returns navigation state: 'idle' | 'loading' | 'submitting' |
| defer() + Await | Streams non-critical data after initial render |
| Lazy + Suspense | `lazy(() => import())` + `<Suspense fallback>` for code splitting |
| Protected Route Pattern | Wrapper component with auth check, redirects if unauthorized |
| Search Params for Filters | `useSearchParams()` - shareable, bookmarkable, browser history |
| useBlocker | Prevents navigation when condition true (unsaved changes) |
| navigate replace | `navigate('/path', { replace: true })` - replaces history entry |
| Route Rank | Exact > params > splat. More specific matches first |

---

## NEXT TOPIC: `07-FRAMEWORKS.md` (Next.js, Remix, Astro)

> **Study Tip**: Build a complete feature with React Router: list → detail → edit → save, with loaders, actions, optimistic updates, and protected routes.