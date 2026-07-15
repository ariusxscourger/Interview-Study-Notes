# TESTING IN REACT
## Exam-Style Study Notes

---

## TOPIC: Testing Strategies, Tools & Patterns

### WHY? (Problem Statement)

**Without Testing:**
- Bugs reach production
- Refactoring fear
- No documentation of expected behavior
- Manual testing doesn't scale
- No confidence in deployments

**What Testing Solves:**
| Problem | Solution |
|---------|----------|
| Regression bugs | Automated regression suite |
| Refactoring safety | Tests as safety net |
| Behavior documentation | Tests as living docs |
| CI/CD integration | Automated gates |
| Confidence | Green = deployable |

**Testing Pyramid (Practical):**
```
        E2E Tests (Few)
       /              \
  Integration Tests (Some)
 /                        \
Unit Tests (Many)        Static Analysis
```

**Real-World Distribution:**
- Unit: 70% (fast, isolated, many)
- Integration: 20% (component + hooks + context)
- E2E: 10% (critical user flows)

---

### HOW? (Testing Mechanics)

#### 1. **Test Types & When to Use**

| Test Type | Scope | Tools | Speed | Use Case |
|-----------|-------|-------|-------|----------|
| **Unit** | Single function/component | Vitest, Jest | ~ms | Pure logic, utils, hooks |
| **Component** | Single component + children | RTL + Vitest | ~10-100ms | UI behavior, props, events |
| **Integration** | Multiple components + hooks + context | RTL + MSW | ~100-500ms | Flows, data fetching |
| **E2E** | Full app in browser | Playwright, Cypress | ~seconds | Critical paths, auth, payments |

#### 2. **Testing Library Philosophy**

```
GUIDING PRINCIPLES:
─────────────────────────────────────────────────────────────
1. Test BEHAVIOR, not implementation
   ❌ Test internal state, private methods, prop values
   ✅ Test what user sees and interacts with

2. ACCESSIBILITY-FIRST QUERIES
   Priority: getByRole > getByLabelText > getByPlaceholderText > 
             getByText > getByDisplayValue > getByAltText >
             getByTitle > getByTestId (last resort)

3. USER-CENTRIC INTERACTIONS
   fireEvent → userEvent (simulates real browser behavior)
```

---

### WHAT? (Tools & Patterns)

#### 1. **Vitest + React Testing Library Setup**

```tsx
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./test/setup.ts'],
    globals: true,
    include: ['**/*.{test,spec}.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 70,
        statements: 80,
      },
    },
  },
});

// test/setup.ts
import '@testing-library/jest-dom';
import { cleanup } from '@testing-library/react';
import { afterEach, vi } from 'vitest';

afterEach(() => {
  cleanup();
  vi.clearAllMocks();
});

// Mock matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});

// Mock ResizeObserver
global.ResizeObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}));
```

#### 2. **Component Testing Patterns**

```tsx
// Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  // Render helper
  const renderButton = (props = {}) => 
    render(<Button {...props} />);

  it('renders children', () => {
    renderButton({ children: 'Click me' });
    expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
  });

  it('calls onClick when clicked', async () => {
    const handleClick = vi.fn();
    renderButton({ onClick: handleClick });
    
    // userEvent simulates real browser behavior
    await userEvent.click(screen.getByRole('button'));
    
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('does not call onClick when disabled', async () => {
    const handleClick = vi.fn();
    renderButton({ onClick: handleClick, disabled: true });
    
    await userEvent.click(screen.getByRole('button', { disabled: true }));
    
    expect(handleClick).not.toHaveBeenCalled();
  });

  it('applies variant classes', () => {
    renderButton({ variant: 'primary', children: 'Test' });
    expect(screen.getByRole('button')).toHaveClass('btn-primary');
    
    renderButton({ variant: 'secondary', children: 'Test' });
    expect(screen.getByRole('button')).toHaveClass('btn-secondary');
  });

  it('handles keyboard activation', async () => {
    const handleClick = vi.fn();
    renderButton({ onClick: handleClick, children: 'Test' });
    
    const button = screen.getByRole('button');
    button.focus();
    await userEvent.keyboard('{Enter}');
    
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  // Snapshot test (use sparingly)
  it('matches snapshot', () => {
    const { container } = renderButton({ children: 'Snapshot' });
    expect(container).toMatchSnapshot();
  });
});
```

#### 3. **Hook Testing**

```tsx
// useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('initializes with custom value', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('increments counter', () => {
    const { result } = renderHook(() => useCounter(0));
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  it('decrements counter', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => {
      result.current.decrement();
    });
    
    expect(result.current.count).toBe(4);
  });

  it('resets to initial value', () => {
    const { result } = renderHook(() => useCounter(10));
    
    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });
    
    expect(result.current.count).toBe(10);
  });

  // Testing with custom initial value
  it('accepts custom initial value', () => {
    const { result } = renderHook(() => useCounter(42));
    expect(result.current.count).toBe(42);
  });
});

// useLocalStorage.test.ts
import { renderHook, act, waitFor } from '@testing-library/react';
import { useLocalStorage } from './useLocalStorage';

describe('useLocalStorage', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('returns initial value when empty', () => {
    const { result } = renderHook(() => useLocalStorage('test', 'default'));
    expect(result.current[0]).toBe('default');
  });

  it('reads from localStorage', () => {
    localStorage.setItem('test', JSON.stringify('stored'));
    const { result } = renderHook(() => useLocalStorage('test', 'default'));
    expect(result.current[0]).toBe('stored');
  });

  it('updates localStorage on set', async () => {
    const { result } = renderHook(() => useLocalStorage('test', 'default'));
    
    act(() => {
      result.current[1]('new value');
    });
    
    await waitFor(() => {
      expect(localStorage.getItem('test')).toBe(JSON.stringify('new value'));
    });
  });
});
```

#### 4. **Integration Testing with MSW (Mock Service Worker)**

```tsx
// handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', ({ request }) => {
    const url = new URL(request.url);
    const page = url.searchParams.get('page') || '1';
    const limit = url.searchParams.get('limit') || '10';
    
    return HttpResponse.json({
      users: Array.from({ length: +limit }, (_, i) => ({
        id: (+page - 1) * +limit + i + 1,
        name: `User ${(+page - 1) * +limit + i + 1}`,
        email: `user${(+page - 1) * +limit + i + 1}@example.com`,
      })),
      total: 100,
      page: +page,
      limit: +limit,
    });
  }),

  http.post('/api/users', async ({ request }) => {
    const data = await request.json();
    return HttpResponse.json({ id: Date.now(), ...data }, { status: 201 });
  }),

  http.patch('/api/users/:id', async ({ params, request }) => {
    const data = await request.json();
    return HttpResponse.json({ id: params.id, ...data });
  }),

  http.delete('/api/users/:id', () => {
    return new HttpResponse(null, { status: 204 });
  }),
];

// test/setup.ts (add to vitest config)
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// test/UserList.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserList } from './UserList';
import { server } from '../test/setup';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('UserList', () => {
  it('loads and displays users', async () => {
    render(<UserList />);
    
    expect(screen.getByText('Loading...')).toBeInTheDocument();
    
    await waitFor(() => {
      expect(screen.getByText('User 1')).toBeInTheDocument();
    });
    
    expect(screen.getByText('User 10')).toBeInTheDocument();
  });

  it('handles pagination', async () => {
    render(<UserList />);
    
    await waitFor(() => {
      expect(screen.getByText('User 1')).toBeInTheDocument();
    });
    
    await userEvent.click(screen.getByRole('button', { name: 'Next' }));
    
    await waitFor(() => {
      expect(screen.getByText('User 11')).toBeInTheDocument();
    });
  });

  it('handles error state', async () => {
    server.use(
      http.get('/api/users', () => HttpResponse.json({ error: 'Server error' }, { status: 500 }))
    );
    
    render(<UserList />);
    
    await waitFor(() => {
      expect(screen.getByText('Error: Server error')).toBeInTheDocument();
    });
  });

  it('creates new user optimistically', async () => {
    render(<UserList />);
    
    await waitFor(() => {
      expect(screen.getByText('User 1')).toBeInTheDocument();
    });
    
    await userEvent.click(screen.getByRole('button', { name: 'Add User' }));
    await userEvent.type(screen.getByLabelText('Name'), 'New User');
    await userEvent.type(screen.getByLabelText('Email'), 'new@example.com');
    await userEvent.click(screen.getByRole('button', { name: 'Create' }));
    
    await waitFor(() => {
      expect(screen.getByText('New User')).toBeInTheDocument();
    });
  });
});
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Test behavior, not implementation** | Test internal state, private methods | Brittle, breaks on refactor |
| **Query by role/label** | Query by class, id, test-id | Accessibility, real usage |
| **userEvent over fireEvent** | fireEvent for interactions | userEvent = real browser behavior |
| **Test async with waitFor** | setTimeout in tests | Flaky, slow, unreliable |
| **MSW for API mocking** | Mock fetch globally | Realistic, reusable, type-safe |
| **Test user flows** | Test every prop combination | User doesn't care about props |
| **Describe blocks for organization** | Flat test files | Maintainability, readability |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "What's the difference between unit, integration, and E2E tests?"

```
UNIT TESTS:
- Single function/component in isolation
- Fast (< 10ms), many (100s)
- Mock all dependencies
- Test: pure functions, hooks, utils, pure components

INTEGRATION TESTS:
- Multiple units working together
- Medium speed (100-500ms)
- Some real dependencies (context, hooks, router)
- Test: component + hooks + context + API mocks

E2E TESTS:
- Full app in real browser
- Slow (seconds), few (10-20)
- Real backend (or full mock)
- Test: critical user journeys (login, checkout, signup)

PRACTICAL RATIO: 70% Unit, 20% Integration, 10% E2E
```

#### 2. **Code**: "Test a custom hook with async data"

```tsx
// useUser.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useUser } from './useUser';
import { server } from '../test/setup';
import { http, HttpResponse } from 'msw';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('useUser', () => {
  beforeAll(() => server.listen());
  afterEach(() => server.resetHandlers());
  afterAll(() => server.close());

  it('fetches user data', async () => {
    const { result } = renderHook(() => useUser('123'), {
      wrapper: createWrapper(),
    });

    expect(result.current.isLoading).toBe(true);

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual({
      id: '123',
      name: 'John Doe',
      email: 'john@example.com',
    });
  });

  it('handles error', async () => {
    server.use(
      http.get('/api/users/123', () => 
        HttpResponse.json({ error: 'Not found' }, { status: 404 })
      )
    );

    const { result } = renderHook(() => useUser('123'), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error).toBeDefined();
  });
});
```

#### 3. **Code**: "Test a form with validation and submission"

```tsx
// LoginForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';
import { server } from '../test/setup';
import { http, HttpResponse } from 'msw';

describe('LoginForm', () => {
  const setup
  it('shows validation errors', async () => {
    render(<LoginForm />);
    
    await userEvent.click(screen.getByRole('button', { name: 'Sign In' }));
    
    await waitFor(() => {
      expect(screen.getByText('Email is required')).toBeInTheDocument();
      expect(screen.getByText('Password is required')).toBeInTheDocument();
    });
  });

  it('submits form successfully', async () => {
    const onSuccess = vi.fn();
    render(<LoginForm onSuccess={onSuccess} />);
    
    await userEvent.type(screen.getByLabelText('Email'), 'test@example.com');
    await userEvent.type(screen.getByLabelText('Password'), 'password123');
    await userEvent.click(screen.getByRole('button', { name: 'Sign In' }));
    
    await waitFor(() => {
      expect(screen.getByText('Signing in...')).toBeInTheDocument();
    });
    
    await waitFor(() => {
      expect(onSuccess).toHaveBeenCalledWith({
        token: 'mock-jwt-token',
        user: { id: '1', email: 'test@example.com' },
      });
    });
  });

  it('shows server error', async () => {
    server.use(
      http.post('/api/auth/login', () => 
        HttpResponse.json({ message: 'Invalid credentials' }, { status: 401 })
      )
    );
    
    render(<LoginForm />);
    
    await userEvent.type(screen.getByLabelText('Email'), 'wrong@example.com');
    await userEvent.type(screen.getByLabelText('Password'), 'wrong');
    await userEvent.click(screen.getByRole('button', { name: 'Sign In' }));
    
    await waitFor(() => {
      expect(screen.getByText('Invalid credentials')).toBeInTheDocument();
    });
  });
});
```

---

### GOTCHAS & EDGE CASES

- [ ] **Async acts** - Wrap async operations in `act()` or use `waitFor`
- [ ] **Cleanup** - `afterEach(() => cleanup())` + `vi.clearAllMocks()`
- [ ] **MSW handlers** - `server.resetHandlers()` in `afterEach`
- [ ] **Async queries** - Use `waitFor` not `setTimeout`
- [ ] **Port conflicts** - E2E tests need unique ports
- [ ] **Flaky tests** - Usually async timing, fix with proper awaits
- [ ] **Test isolation** - Each test independent, no shared state
- [ ] **Coverage targets** - Don't chase 100%, focus on critical paths

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Test File Template**

```tsx
// Component.test.tsx
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { render, screen, waitFor, cleanup } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Component } from './Component';
import { server } from '../../test/setup';
import { http, HttpResponse } from 'msw';

describe('Component', () => {
  beforeAll(() => server.listen());
  afterEach(() => {
    cleanup();
    vi.clearAllMocks();
    server.resetHandlers();
  });
  afterAll(() => server.close());

  const renderComponent = (props = {}) => render(<Component {...props} />);

  describe('Rendering', () => {
    it('renders correctly', () => {
      renderComponent();
      expect(screen.getByRole('heading')).toHaveTextContent('Title');
    });
  });

  describe('Interactions', () => {
    it('handles click', async () => {
      const handler = vi.fn();
      renderComponent({ onClick: handler });
      
      await userEvent.click(screen.getByRole('button'));
      
      expect(handler).toHaveBeenCalledOnce();
    });
  });

  describe('Async Behavior', () => {
    it('loads data', async () => {
      renderComponent();
      
      await waitFor(() => {
        expect(screen.getByText('Data loaded')).toBeInTheDocument();
      });
    });
  });

  describe('Error Handling', () => {
    it('shows error message', async () => {
      server.use(
        http.get('/api/data', () => HttpResponse.json({ error: 'Failed' }, { status: 500 }))
      );
      
      renderComponent();
      
      await waitFor(() => {
        expect(screen.getByText('Error: Failed')).toBeInTheDocument();
      });
    });
  });
});
```

#### 2. **MSW Setup for Different Scenarios**

```tsx
// Delayed response
http.get('/api/slow', async () => {
  await new Promise(r => setTimeout(r, 1000));
  return HttpResponse.json({ data: 'slow' });
});

// Request validation
http.post('/api/users', async ({ request }) => {
  const body = await request.json();
  if (!body.email.includes('@')) {
    return HttpResponse.json({ error: 'Invalid email' }, { status: 400 });
  }
  return HttpResponse.json({ id: 1, ...body }, { status: 201 });
});

// Network error
http.get('/api/error', () => HttpResponse.error());

// Conditional handlers
server.use(
  http.get('/api/conditional', ({ request }) => {
    const url = new URL(request.url);
    if (url.searchParams.get('fail') === 'true') {
      return HttpResponse.json({ error: 'Forced' }, { status: 500 });
    }
    return HttpResponse.json({ success: true });
  })
);
```

#### 3. **Testing Library Custom Render**

```tsx
// test-utils.tsx
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactNode } from 'react';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  });

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}

export function renderWithProviders(
  ui: ReactNode,
  options?: RenderOptions
) {
  return render(ui, {
    wrapper: createWrapper(),
    ...options,
  });
}

// Re-export everything
export * from '@testing-library/react';
export { renderWithProviders as render };
```

---

### PRACTICE PROBLEMS

1. **Test** a data table component: sorting, pagination, row selection, inline editing
2. **Build** a test suite for a shopping cart: add/remove, quantity, persistence, checkout flow
3. **Create** a test utility for testing protected routes with different auth states
4. **Debug** a flaky test: identify root cause (timing, state leak, async)
5. **Implement** visual regression testing with Storybook + Chromatic

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Testing Pyramid | Unit (70%) → Integration (20%) → E2E (10%) |
| RTL Query Priority | getByRole > getByLabelText > getByPlaceholderText > getByText > getByTestId |
| fireEvent vs userEvent | userEvent simulates real browser behavior (focus, selection, events) |
| Testing async | `waitFor(() => expect(...))` not `setTimeout` |
| MSW | Mock Service Worker - intercepts fetch/XHR at network level |
| renderHook | Tests hooks in isolation with `act` for updates |
| waitFor | Retries callback until passes or timeout |
| act() | Wraps state updates for testing - batches updates |
| MSW Handlers | `http.get/post/patch/delete` with `HttpResponse.json()` |
| Test Isolation | `cleanup()` + `vi.clearAllMocks()` + `server.resetHandlers()` |
| Snapshot Testing | `expect(container).toMatchSnapshot()` - use sparingly |
| E2E vs Integration | E2E = real browser, full stack. Integration = components + mocks |
| Testing Error Boundaries | Wrap in ErrorBoundary, throw in component, check fallback |

---

## NEXT TOPIC: `06-PERFORMANCE.md`

> **Study Tip**: Write tests for 3 components this week: 1 simple (Button), 1 with async (UserCard), 1 form (Login). Use MSW for all API calls.