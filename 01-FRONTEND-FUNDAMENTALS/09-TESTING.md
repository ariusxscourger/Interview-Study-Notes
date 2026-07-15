# TESTING STRATEGIES & TOOLS
## Exam-Style Study Notes

---

## TOPIC: Testing in Modern Frontend & Backend Applications

### WHY? (Problem Statement)

**Without Testing:**
- Bugs reach production → user frustration, revenue loss
- Refactoring fear → technical debt accumulation
- No documentation of expected behavior
- Manual testing doesn't scale
- Deploy anxiety

**What Testing Solves:**
| Problem | Testing Solution |
|---------|------------------|
| Regression bugs | Automated regression suite |
| Refactoring confidence | Tests as safety net |
| Behavior documentation | Tests as living docs |
| Code quality | TDD drives design |
| CI/CD integration | Automated gates |

**Testing Pyramid (Practical):**
```
        E2E Tests (Few)
       /              \
  Integration Tests (Some)
 /                        \
Unit Tests (Many)         Static Analysis
```

**Real-World Analogy:**
- Unit test = Testing individual bricks
- Integration test = Testing brick + mortar bond
- E2E test = Testing the whole house survives a storm

---

### HOW? (Internal Mechanism)

#### 1. **Test Runner Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                     TEST EXECUTION                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SOURCE CODE ──► TRANSPILATION ──► TEST RUNNER             │
│       │              │                   │                 │
│       ▼              ▼                   ▼                 │
│  TypeScript       Babel/SWC           Vitest/Jest        │
│  JSX/TSX          esbuild              Playwright        │
│  CSS Modules      PostCSS              Cypress           │
│                                                             │
│  TEST RUNNER:                                              │
│  ├── Discovery (glob patterns)                             │
│  ├── Isolation (per test)                                  │
│  ├── Execution (parallel)                                  │
│  ├── Reporting (JUnit, HTML, JSON)                         │
│  └── Watch Mode (incremental)                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **Mocking & Spying**

```javascript
// vi.fn() / jest.fn() - Creates mock function
const mockFn = vi.fn();
mockFn('arg1', 'arg2');
// mockFn.mock.calls === [['arg1', 'arg2']]
// mockFn.mock.results === [{type: 'return', value: undefined}]

// Mock implementations
mockFn.mockReturnValue('fixed');
mockFn.mockResolvedValue(Promise.resolve('async'));
mockFn.mockImplementation((a, b) => a + b);

// Spy on existing function
const spy = vi.spyOn(object, 'method');
// object.method() calls spy, spy calls original
spy.mockRestore(); // Restore original
```

---

### WHAT? (Tools & Patterns)

#### 1. **Unit Testing - Vitest (Modern Standard)**

```typescript
// math.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { calculateTotal, formatCurrency, fetchExchangeRate } from './math';

// Basic assertions
describe('calculateTotal', () => {
  it('sums prices with tax', () => {
    expect(calculateTotal([10, 20], 0.1)).toBe(33);
  });
  
  it('handles empty array', () => {
    expect(calculateTotal([], 0.1)).toBe(0);
  });
  
  it('throws on negative price', () => {
    expect(() => calculateTotal([-5], 0.1)).toThrow('Negative price');
  });
});

// Async testing
describe('fetchExchangeRate', () => {
  it('returns rate from API', async () => {
    // Mock global fetch
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ rate: 1.2 })
    });
    
    const rate = await fetchExchangeRate('USD', 'EUR');
    expect(rate).toBe(1.2);
    expect(fetch).toHaveBeenCalledWith('/api/rates?from=USD&to=EUR');
  });
  
  it('handles network error', async () => {
    global.fetch = vi.fn().mockRejectedValue(new Error('Network error'));
    
    await expect(fetchExchangeRate('USD', 'EUR')).rejects.toThrow('Network error');
  });
});

// Mocking modules
vi.mock('./external-api', () => ({
  sendNotification: vi.fn().mockResolvedValue({ sent: true })
}));

// Snapshot testing
it('formats currency correctly', () => {
  expect(formatCurrency(1234.56, 'USD')).toMatchInlineSnapshot('$1,234.56');
});

// Parameterized tests
describe.each([
  [100, 0.1, 110],
  [50, 0.2, 60],
  [0, 0.5, 0],
])('calculateTotal(%d, %d) = %d', (price, tax, expected) => {
  expect(calculateTotal([price], tax)).toBe(expected);
});
```

#### 2. **Component Testing - React Testing Library**

```tsx
// UserProfile.test.tsx
import { render, screen, fireEvent, waitFor, act } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserProfile } from './UserProfile';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// Test wrapper with providers
const renderWithProviders = (ui: React.ReactElement) => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } }
  });
  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
};

describe('UserProfile', () => {
  beforeEach(() => {
    vi.mock('./api', () => ({
      fetchUser: vi.fn().mockResolvedValue({ 
        id: '1', 
        name: 'Alice', 
        email: 'alice@example.com' 
      }),
      updateUser: vi.fn().mockResolvedValue({ success: true })
    }));
  });
  
  it('displays user info', async () => {
    renderWithProviders(<UserProfile userId="1" />);
    
    // Loading state
    expect(screen.getByText('Loading...')).toBeInTheDocument();
    
    // Wait for data
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
      expect(screen.getByText('alice@example.com')).toBeInTheDocument();
    });
  });
  
  it('updates name on form submit', async () => {
    const user = userEvent.setup();
    renderWithProviders(<UserProfile userId="1" />);
    
    await waitFor(() => screen.getByText('Alice'));
    
    const input = screen.getByLabelText('Name');
    await user.clear(input);
    await user.type(input, 'Bob');
    
    await user.click(screen.getByRole('button', { name: 'Save' }));
    
    await waitFor(() => {
      expect(screen.getByText('Bob')).toBeInTheDocument();
    });
  });
  
  it('shows error on failed update', async () => {
    vi.mocked(api.updateUser).mockRejectedValueOnce(new Error('Server error'));
    
    const user = userEvent.setup();
    renderWithProviders(<UserProfile userId="1" />);
    
    await waitFor(() => screen.getByText('Alice'));
    
    await user.click(screen.getByRole('button', { name: 'Edit' }));
    await user.click(screen.getByRole('button', { name: 'Save' }));
    
    await waitFor(() => {
      expect(screen.getByText('Failed to update')).toBeInTheDocument();
    });
  });
});

// Testing custom hooks
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

it('increments counter', () => {
  const { result } = renderHook(() => useCounter(0));
  
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
  
  act(() => result.current.increment());
  expect(result.current.count).toBe(2);
});
```

#### 3. **Integration Testing - API Routes**

```typescript
// users.test.ts (Hono/Express/Fastify)
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { app } from './app';
import { db } from './db';
import { users } from './schema';

describe('Users API', () => {
  beforeAll(async () => {
    await db.migrate();
  });
  
  afterAll(async () => {
    await db.close();
  });
  
  beforeEach(async () => {
    await db.delete(users).execute();
  });
  
  it('creates user', async () => {
    const res = await app.request('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name: 'Alice', email: 'alice@test.com' })
    });
    
    expect(res.status).toBe(201);
    const data = await res.json();
    expect(data).toMatchObject({ name: 'Alice', email: 'alice@test.com' });
    expect(data.id).toBeDefined();
  });
  
  it('returns 400 for invalid email', async () => {
    const res = await app.request('/api/users', {
      method: 'POST',
      body: JSON.stringify({ name: 'Bob', email: 'invalid' })
    });
    
    expect(res.status).toBe(400);
    const data = await res.json();
    expect(data.error).toContain('email');
  });
  
  it('gets user by id', async () => {
    const [user] = await db.insert(users).values({ 
      name: 'Charlie', 
      email: 'charlie@test.com' 
    }).returning();
    
    const res = await app.request(`/api/users/${user.id}`);
    expect(res.status).toBe(200);
    
    const data = await res.json();
    expect(data.name).toBe('Charlie');
  });
});
```

#### 4. **E2E Testing - Playwright**

```typescript
// auth.e2e.test.ts
import { test, expect, Page } from '@playwright/test';

test.describe('Authentication', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });
  
  test('successful login redirects to dashboard', async ({ page }) => {
    await page.fill('[data-testid=email]', 'user@example.com');
    await page.fill('[data-testid=password]', 'password123');
    await page.click('[data-testid=submit]');
    
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toContainText('Welcome');
  });
  
  test('shows error for invalid credentials', async ({ page }) => {
    await page.fill('[data-testid=email]', 'wrong@example.com');
    await page.fill('[data-testid=password]', 'wrong');
    await page.click('[data-testid=submit]');
    
    await expect(page.locator('[data-testid=error]')).toContainText('Invalid credentials');
  });
  
  test('register flow', async ({ page }) => {
    await page.click('text=Sign up');
    await expect(page).toHaveURL('/register');
    
    await page.fill('[data-testid=name]', 'New User');
    await page.fill('[data-testid=email]', 'new@test.com');
    await page.fill('[data-testid=password]', 'secure123');
    await page.fill('[data-testid=confirm]', 'secure123');
    await page.click('[data-testid=submit]');
    
    await expect(page).toHaveURL('/dashboard');
  });
  
  test('protected route redirects to login', async ({ page }) => {
    await page.goto('/settings');
    await expect(page).toHaveURL('/login');
  });
});

// Visual regression
test('dashboard matches snapshot', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot('dashboard.png');
});

// API testing within E2E
test('creates order via API then verifies UI', async ({ page, request }) => {
  // Create via API (fast)
  const response = await request.post('/api/orders', {
    data: { items: [{ productId: '1', quantity: 2 }] }
  });
  expect(response.ok()).toBeTruthy();
  const order = await response.json();
  
  // Verify in UI
  await page.goto(`/orders/${order.id}`);
  await expect(page.locator('[data-testid=order-total]')).toContainText('$50.00');
});
```

#### 5. **Test Utilities & Patterns**

```typescript
// test/utils.tsx
import { ReactElement } from 'react';
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';

// Custom render with all providers
const AllProviders: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: 0 } }
  }));
  
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {children}
      </BrowserRouter>
    </QueryClientProvider>
  );
};

const customRender = (ui: ReactElement, options?: Omit<RenderOptions, 'wrapper'>) => 
  render(ui, { wrapper: AllProviders, ...options });

export * from '@testing-library/react';
export { customRender as render };

// test/factories.ts
import { faker } from '@faker-js/faker';

export const createUser = (overrides = {}) => ({
  id: faker.string.uuid(),
  name: faker.person.fullName(),
  email: faker.internet.email(),
  createdAt: faker.date.past().toISOString(),
  ...overrides
});

export const createProduct = (overrides = {}) => ({
  id: faker.string.uuid(),
  name: faker.commerce.productName(),
  price: parseFloat(faker.commerce.price()),
  ...overrides
});

// test/msw.ts (Mock Service Worker)
import { setupWorker, rest } from 'msw/browser';

export const worker = setupWorker(
  rest.get('/api/users', (req, res, ctx) => 
    res(ctx.json([createUser(), createUser({ name: 'Admin' })]))
  ),
  rest.post('/api/users', async (req, res, ctx) => {
    const body = await req.json();
    return res(ctx.status(201), ctx.json(createUser(body)));
  })
);

// test/setup.ts
import { beforeAll, afterAll, afterEach } from 'vitest';
import { worker } from './msw';

beforeAll(() => worker.start({ onUnhandledRequest: 'error' }));
afterEach(() => worker.resetHandlers());
afterAll(() => worker.stop());
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Test behavior, not implementation** | Testing private methods/state | Refactoring breaks tests |
| **Arrange-Act-Assert (AAA)** | Mixed setup/assertions | Hard to read |
| **Descriptive test names** | `test1`, `shouldWork` | No documentation value |
| **One assertion per test** | Multiple unrelated asserts | Hard to diagnose failures |
| **Fast, isolated unit tests** | Slow integration tests for logic | Slow feedback loop |
| **Test data factories** | Hardcoded objects | Maintenance burden |
| **Mock at boundaries** | Mock everything | Fragile, implementation-coupled |
| **Real timers for async** | `waitFor` with long timeouts | Flaky, slow tests |

---

### INTERVIEW QUESTIONS (Top 20)

#### 1. **Conceptual**: "Unit vs Integration vs E2E - when to use each?"

**Answer:**
```
UNIT TESTS (70%):
✓ Pure functions, utils, hooks, business logic
✓ Fast (<10ms), no I/O, fully isolated
✓ Run on every keystroke (watch mode)

INTEGRATION TESTS (20%):
✓ Component + hooks + context + router
✓ API routes + database
✓ Service interactions
✓ Some I/O (test DB, MSW)

E2E TESTS (10%):
✓ Critical user flows (login, checkout)
✓ Cross-browser verification
✓ Visual regression
✓ Real backend, real network

RULE: Test at the lowest level that gives confidence
```

#### 2. **Code**: "Test this custom hook with complex async logic"

```typescript
// useUser.ts
export function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    let cancelled = false;
    
    async function fetchUser() {
      try {
        setLoading(true);
        const data = await fetch(`/api/users/${userId}`).then(r => r.json());
        if (!cancelled) setUser(data);
      } catch (err) {
        if (!cancelled) setError(err as Error);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }
    
    fetchUser();
    return () => { cancelled = true; };
  }, [userId]);
  
  return { user, error, loading, refetch: () => fetchUser() };
}

// Test
it('fetches user and handles cleanup', async () => {
  const user = createUser({ id: '1', name: 'Alice' });
  server.use(rest.get('/api/users/1', (req, res, ctx) => 
    res(ctx.delay(100), ctx.json(user))
  ));
  
  const { result, unmount } = renderHook(() => useUser('1'));
  
  expect(result.current.loading).toBe(true);
  
  await waitFor(() => expect(result.current.loading).toBe(false));
  expect(result.current.user).toEqual(user);
  
  // Unmount before fetch completes
  unmount();
  
  // Should not update state after unmount
  // (verified by no act warnings)
});
```

#### 3. **Design**: "How to test a component that uses Context + Router + Query Client?"

```typescript
// Create test wrapper
const TestWrapper: FC<{ children: ReactNode }> = ({ children }) => {
  const queryClient = useMemo(() => new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: 0 } }
  }), []);
  
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <AuthProvider>
          <ThemeProvider>
            {children}
          </ThemeProvider>
        </AuthProvider>
      </BrowserRouter>
    </QueryClientProvider>
  );
};

// Use in tests
render(<TestWrapper><MyComponent /></TestWrapper>);

// Or with RTL custom render
const renderWithProviders = (ui, options) => 
  render(ui, { wrapper: TestWrapper, ...options });
```

#### 4. **Performance**: "Tests are slow - how to speed up?"

```typescript
// 1. Parallel execution (Vitest/Jest default)
test.concurrent('independent test 1', () => { ... });
test.concurrent('independent test 2', () => { ... });

// 2. Skip type checking in unit tests
// tsconfig.test.json: { "compilerOptions": { "noEmit": true, "skipLibCheck": true }}

// 3. Use faster test runner
// Vitest > Jest (native ESM, no transform cache needed)

// 4. Reduce test DB overhead
// - Use transactions + rollback (not truncate)
// - SQLite in-memory for unit tests
// - Testcontainers for integration

// 5. Mock external services
// MSW for HTTP, not real network

// 6. Shallow rendering for heavy components
// jest: shallow() | RTL: render(<Component />) with mocked children

// 7. Snapshot only when needed
// toMatchSnapshot() is slow - prefer inline snapshots

// 8. CI optimization
// - Cache node_modules, test results
// - Split test suites across runners
// - Fail fast on first failure (--bail)
```

#### 5. **Debugging**: "Flaky test - how to diagnose?"

```
FLAKY TEST CHECKLIST:
☐ Async timing (waitFor, act, fake timers)
☐ Shared state between tests (cleanup!)
☐ Random data (use seeded faker)
☐ Network dependence (MSW, not real API)
☐ Date/time (mock with fixed date)
☐ Math.random / UUID (mock)
☐ Test order dependence (parallel vs serial)
☐ Resource leaks (timers, listeners, connections)
☐ Animation/transition (disable in test env)
☐ Race conditions in code under test

DEBUGGING:
1. Run in isolation: test.only
2. Run 100x: --repeat=100
3. Add logging to test
4. Check CI vs local differences
5. Use --inspect-brk for debugging
```

#### 6. **Code**: "Test error boundary catches errors"

```tsx
// ErrorBoundary.tsx
class ErrorBoundary extends Component<{ children: ReactNode; fallback: ReactNode }, { error: Error | null }> {
  state = { error: null };
  static getDerivedStateFromError(error) { return { error }; }
  render() { return this.state.error ? this.props.fallback : this.props.children; }
}

// Test
it('renders fallback when child throws', () => {
  const ThrowError = () => { throw new Error('Boom!'); };
  const Fallback = ({ error }: { error: Error }) => <div data-testid="fallback">{error.message}</div>;
  
  render(
    <ErrorBoundary fallback={<Fallback />}>
      <ThrowError />
    </ErrorBoundary>
  );
  
  expect(screen.getByTestId('fallback')).toHaveTextContent('Boom!');
});

// With RTL - use render with wrapper that catches
const renderWithBoundary = (ui) => render(
  <ErrorBoundary fallback={<div data-testid="fallback">Error</div>}>
    {ui}
  </ErrorBoundary>
);
```

#### 7. **Architecture**: "Testing Strategy for Microservices"

```
CONTRACT TESTING (Pact):
Consumer (Frontend)          Provider (API)
     │                           │
     ├─ Defines expectations ───►│
     │                           │
     │◄── Verifies contract ─────┤
     │                           │
     ▼                           ▼

COMPONENT TESTS:
- Each service tested in isolation
- Dependencies mocked/contract tested
- Own database (testcontainers)

INTEGRATION TESTS:
- Key cross-service flows
- Real message broker (Testcontainers)
- Shared test environment

E2E TESTS:
- Critical user journeys only
- Production-like staging
- Synthetic monitoring
```

#### 8. **Security**: "Test for XSS/Injection vulnerabilities"

```typescript
it('sanitizes user input in comments', () => {
  const maliciousInput = '<script>alert("xss")</script><img src=x onerror=alert(1)>';
  
  render(<CommentForm defaultValue={maliciousInput} />);
  
  // Should render escaped, not execute
  expect(screen.getByText(maliciousInput)).toBeInTheDocument();
  expect(screen.queryByRole('alert')).not.toBeInTheDocument();
});

it('prevents SQL injection in search', async () => {
  const sqlInjection = "'; DROP TABLE users; --";
  
  const res = await request(app).get('/api/search').query({ q: sqlInjection });
  
  expect(res.status).toBe(200);
  // Verify no SQL executed (mock DB)
  expect(mockDb.query).not.toHaveBeenCalledWith(expect.stringContaining('DROP'));
});
```

---

### GOTCHAS & EDGE CASES

- [ ] **`act()` warnings** - Wrap state updates in `act()` or use `waitFor`
- [ ] **Timer mocking** - `vi.useFakeTimers()` breaks `setTimeout` in libraries
- [ ] **Module mocking hoisting** - `vi.mock` is hoisted, can't use variables
- [ ] **React 18 strict mode** - Double mounts in dev, not in tests
- [ ] **Cleanup** - `afterEach(() => vi.clearAllMocks())` essential
- [ ] **Snapshot updates** - `u` flag updates, review diffs!
- [ ] **Parallel test interference** - Shared DB? Use transactions or separate DBs
- [ ] **Coverage thresholds** - Don't aim for 100%, aim for meaningful
- [ ] **TypeScript in tests** - `skipLibCheck`, `esModuleInterop`

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Vitest Config**
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react-swc';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./test/setup.ts'],
    include: ['**/*.test.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 70,
        statements: 80,
      },
      exclude: ['**/*.d.ts', '**/*.test.ts', '**/test/**'],
    },
    mockReset: true,
    restoreMocks: true,
    clearMocks: true,
    testTimeout: 10000,
    hookTimeout: 10000,
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
});
```

#### 2. **MSW Setup for API Mocking**
```typescript
// test/handlers.ts
import { http, HttpResponse } from 'msw';
import { createUser } from './factories';

export const handlers = [
  http.get('/api/users', () => HttpResponse.json([createUser(), createUser()])),
  http.get('/api/users/:id', ({ params }) => HttpResponse.json(createUser({ id: params.id }))),
  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(createUser(body), { status: 201 });
  }),
  http.patch('/api/users/:id', async ({ params, request }) => {
    const body = await request.json();
    return HttpResponse.json(createUser({ id: params.id, ...body }));
  }),
  http.delete('/api/users/:id', ({ params }) => 
    new HttpResponse(null, { status: 204 })
  ),
];

// test/setup.ts
import { beforeAll, afterAll, afterEach } from 'vitest';
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

#### 3. **Common Test Helpers**
```typescript
// test/helpers.ts
import { waitFor } from '@testing-library/react';

export const waitForLoading = () => 
  waitFor(() => expect(screen.queryByText('Loading...')).not.toBeInTheDocument());

export const waitForError = (message?: string) =>
  waitFor(() => {
    const error = screen.getByRole('alert');
    if (message) expect(error).toHaveTextContent(message);
  });

export const fillForm = async (fields: Record<string, string>) => {
  const user = userEvent.setup();
  for (const [label, value] of Object.entries(fields)) {
    const input = screen.getByLabelText(label);
    await user.clear(input);
    await user.type(input, value);
  }
};

export const submitForm = async (buttonText = 'Submit') => {
  const user = userEvent.setup();
  await user.click(screen.getByRole('button', { name: buttonText }));
};

export const loginAs = async (role: 'user' | 'admin' = 'user') => {
  const testUser = role === 'admin' ? createAdmin() : createUser();
  server.use(http.get('/api/auth/me', () => HttpResponse.json(testUser)));
  renderWithProviders(<App />);
  await waitFor(() => expect(screen.getByText(testUser.name)).toBeInTheDocument());
  return testUser;
};
```

---

### PRACTICE PROBLEMS

1. **Setup** a React project with Vitest + RTL + MSW + Playwright
2. **Write** tests for a complex form with validation, async submission, error states
3. **Test** a custom hook with WebSocket connection and reconnection logic
4. **Debug** a flaky E2E test - add retries, fix timing, stabilize
5. **Implement** contract testing between frontend and backend with Pact

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Testing Pyramid ratios | Unit 70%, Integration 20%, E2E 10% |
| Unit test characteristics | Fast, isolated, no I/O, deterministic |
| Integration test scope | Multiple units + some I/O (DB, HTTP) |
| E2E test purpose | Critical user flows, real environment |
| AAA pattern | Arrange, Act, Assert |
| RTL principle | Test behavior, not implementation |
| MSW purpose | Mock API at network level (not module) |
| Flaky test causes | Timing, shared state, randomness, network |
| Contract testing | Pact - consumer defines, provider verifies |
| Coverage targets | Lines 80%, Functions 80%, Branches 70% |

---

## NEXT TOPIC: `10-WEB-SECURITY.md`

> **Study Tip**: Build a todo app with: form validation, API calls, auth, error boundaries. Write unit, integration, and E2E tests. Measure coverage.