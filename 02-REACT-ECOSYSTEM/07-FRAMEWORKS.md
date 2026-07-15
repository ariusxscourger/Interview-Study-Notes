# REACT FRAMEWORKS OVERVIEW
## Exam-Style Study Notes

---

## TOPIC: Modern React Frameworks (Next.js, Remix, Astro)

### WHY? (Problem Statement)

**React is a Library, Not a Framework:**
- No built-in routing
- No data fetching conventions
- No SSR/SSG/ISR out of the box
- No image optimization
- No file-system routing

**What Frameworks Provide:**
| Feature | Next.js | Remix | Astro |
|---------|---------|-------|-------|
| **Rendering** | SSR, SSG, ISR, RSC | SSR, SPA | SSG, SSR, Islands |
| **Routing** | File-system | File-system | File-system |
| **Data Fetching** | Server Components, loaders | Loaders, actions | Loaders, islands |
| **Optimization** | Image, Font, Script | Prefetch, fine-grained | Partial hydration |
| **Deployment** | Vercel, Node, Docker | Node, Cloudflare, Vercel | Static, Node, Edge |

---

### HOW? (Architecture Comparison)

#### 1. **Next.js (App Router - React Server Components)**

```
NEXT.JS APP ROUTER ARCHITECTURE:
─────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────┐
│                    SERVER (Node.js)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ React Server Components (RSC)                        │   │
│  │ • Fetch data directly (DB, APIs)                     │   │
│  │ • Zero client bundle for server-only code            │   │
│  │ • Stream HTML to client                              │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Streaming + Suspense                                 │   │
│  │ • Send shell first                                   │   │
│  │ • Stream async components                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼ (HTML + RSC Payload)
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT (Browser)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Hydration                                           │   │
│  │ • Client Components hydrate                          │   │
│  │ • Server Components become static                    │   │
│  │ • Interactivity added                                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **Remix (Nested Routes + Loaders/Actions)**

```
REMIX ARCHITECTURE:
─────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────┐
│                    SERVER                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Nested Route Matching                               │   │
│  │ • Multiple loaders run IN PARALLEL                  │   │
│  │ • Each route has independent loader/action          │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Action → Loader Revalidation                        │   │
│  │ • POST → Action runs → Loaders revalidate           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT                                   │
│  • Nested <Outlet /> rendering                              │
│  • <Form method="POST"> → triggers action                   │
│  • Automatic scroll restoration                             │
│  • Optimistic UI with useNavigation                        │
└─────────────────────────────────────────────────────────────┘
```

#### 3. **Astro (Islands Architecture)**

```
ASTRO ISLANDS ARCHITECTURE:
─────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────┐
│                    BUILD TIME                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Static Site Generation (SSG)                         │   │
│  │ • .astro files → static HTML                         │   │
│  │ • Zero JS by default                                 │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    RUNTIME (Client)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Islands of Interactivity                             │   │
│  │ • <Component client:load> = hydrate immediately     │   │
│  │ • <Component client:idle> = hydrate when idle       │   │
│  │ • <Component client:visible> = hydrate when visible │   │
│  │ • <Component client:media> = hydrate on media query │   │
│  │ • <Component client:only> = only render on client   │   │
│  └─────────────────────────────────────────────────────┘   │
│  • Framework agnostic (React, Vue, Svelte, Solid, etc.)  │   │
└─────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Features & Patterns)

#### 1. **Next.js App Router Essentials**

```tsx
// app/layout.tsx (Root Layout - Server Component)
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

// app/page.tsx (Page - Server Component by default)
async function getData() {
  const res = await fetch('https://api.example.com/data');
  return res.json();
}

export default async function Page() {
  const data = await getData(); // Direct DB/API access!
  return <main>{data.title}</main>;
}

// app/dashboard/loading.tsx (Loading UI - Suspense Boundary)
export default function Loading() {
  return <DashboardSkeleton />;
}

// app/dashboard/error.tsx (Error Boundary)
'use client';
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// app/dashboard/page.tsx (Server Component with Suspense)
import { Suspense } from 'react';
import { DashboardCards } from './DashboardCards';
import { AnalyticsChart } from './AnalyticsChart';

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <DashboardCards />
      <Suspense fallback={<ChartSkeleton />}>
        <AnalyticsChart />
      </Suspense>
    </div>
  );
}

// app/actions.ts (Server Actions)
'use server';
export async function createPost(formData: FormData) {
  const title = formData.get('title');
  await db.post.create({ title });
  revalidatePath('/posts'); // Revalidate cache
  redirect('/posts');
}

// app/posts/page.tsx (Client Component for interactivity)
'use client';
import { useState } from 'react';
import { createPost } from './actions';

export default function PostsPage() {
  const [title, setTitle] = useState('');
  
  return (
    <form action={createPost}>
      <input name="title" value={title} onChange={e => setTitle(e.target.value)} />
      <button type="submit">Create</button>
    </form>
  );
}
```

#### 2. **Remix Essentials**

```tsx
// app/routes/users.$id.tsx
import { json, redirect } from '@remix-run/node';
import { useLoaderData, useActionData, Form, useNavigation } from '@remix-run/react';

// Loader - runs BEFORE render
export async function loader({ params }) {
  const user = await db.user.findUnique({ where: { id: params.id } });
  if (!user) throw redirect('/users');
  return json(user);
}

// Action - handles POST/PUT/DELETE
export async function action({ params, request }) {
  const formData = await request.formData();
  const data = Object.fromEntries(formData);
  
  const updated = await db.user.update({
    where: { id: params.id },
    data,
  });
  
  return redirect(`/users/${params.id}`);
}

// Component
export default function UserProfile() {
  const user = useLoaderData<typeof loader>();
  const actionData = useActionData();
  const navigation = useNavigation();
  
  return (
    <Form method="POST">
      <input 
        name="name" 
        defaultValue={user.name} 
        disabled={navigation.state === 'submitting'}
      />
      {actionData?.errors?.name && <span>{actionData.errors.name}</span>}
      <button disabled={navigation.state === 'submitting'}>
        {navigation.state === 'submitting' ? 'Saving...' : 'Save'}
      </button>
    </Form>
  );
}

// Nested routes: app/routes/admin.tsx (parent)
export default function AdminLayout() {
  return (
    <div>
      <nav>
        <Link to="dashboard">Dashboard</Link>
        <Link to="users">Users</Link>
      </nav>
      <Outlet /> {/* Child routes render here */}
    </div>
  );
}

// app/routes/admin.users.tsx (child)
export async function loader() {
  const users = await db.user.findMany();
  return json(users);
}

export default function AdminUsers() {
  const users = useLoaderData();
  return <UserTable users={users} />;
}
```

#### 3. **Astro Essentials**

```astro
---
// src/pages/index.astro
import Layout from '../layouts/Layout.astro';
import Header from '../components/Header.astro';
import InteractiveButton from '../components/InteractiveButton.jsx';
---

<Layout title="Welcome">
  <Header />
  <main>
    <h1>Welcome to Astro</h1>
    <!-- Zero JS by default -->
    <InteractiveButton client:load />
    <!-- Hydrates when visible -->
    <InteractiveChart client:visible />
    <!-- Hydrates when idle -->
    <HeavyComponent client:idle />
  </main>
</Layout>
```

```astro
---
// src/components/InteractiveButton.astro
---
<script>
  export function clickHandler() {
    alert('Clicked!');
  }
</script>

<button onclick="clickHandler()">Click me</button>

<style>
  button { background: blue; color: white; padding: 1rem; }
</style>
```

```jsx
// src/components/InteractiveButton.jsx (React component)
import { useState } from 'react';

export default function InteractiveButton() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Server Components by default** | 'use client' everywhere | Loses RSC benefits, larger bundle |
| **Streaming with Suspense** | Await everything | Blocks render, no streaming |
| **Server Actions for mutations** | API routes for everything | Extra network round-trip |
| **Server Components for data** | useEffect + fetch | Waterfall, no caching |
| **Partial hydration (Astro)** | client:load everywhere | Ships unnecessary JS |
| **Image optimization** | Raw <img> tags | No optimization, CLS |
| **Middleware for auth** | Client-side redirects | Flash of unauthorized content |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Next.js Server Components vs Client Components"

```
SERVER COMPONENTS (Default):
- Run ONLY on server
- Zero client bundle
- Can access DB, filesystem, secrets directly
- No useState, useEffect, event handlers
- Can be async (await fetch)
- Streamed + Suspense
- Zero hydration cost

CLIENT COMPONENTS ('use client'):
- Run on server (SSR) + client (hydration)
- Full React API (hooks, effects, events)
- Bundled to client
- Must be serializable props
- Interactivity required

DECISION TREE:
- Needs interactivity (click, input, state)? → Client
- Needs DB access, secrets, heavy computation? → Server
- Pure UI, no interactivity? → Server
- Event handlers, hooks, browser APIs? → Client
```

#### 2. **Code**: "Next.js Server Action with optimistic update"

```tsx
// app/posts/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = formData.get('title');
  const content = formData.get('content');
  
  const post = await db.post.create({ title, content });
  
  revalidatePath('/posts'); // Invalidate cache
  redirect(`/posts/${post.id}`);
}

// app/posts/page.tsx
'use client';
import { createPost } from './actions';

export default function PostsPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" required />
      <textarea name="content" placeholder="Content" required />
      <button type="submit">Create</button>
    </form>
  );
}

// app/posts/[id]/page.tsx (Server Component with Server Action)
import { deletePost } from './actions';

export default function PostPage({ params }: { params: { id: string } }) {
  const post = await db.post.findUnique({ where: { id: params.id } });
  
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      <form action={deletePost}>
        <input type="hidden" name="id" value={post.id} />
        <button type="submit">Delete</button>
      </form>
    </article>
  );
}

async function deletePost(formData: FormData) {
  'use server';
  const id = formData.get('id');
  await db.post.delete({ where: { id } });
  revalidatePath('/posts');
  redirect('/posts');
}
```

#### 3. **Code**: "Remix Loader + Action Pattern"

```tsx
// app/routes/posts.$id.tsx
import { json, redirect } from '@remix-run/node';
import { useLoaderData, useActionData, Form, useNavigation } from '@remix-run/react';

export async function loader({ params }) {
  const post = await db.post.findUnique({ where: { id: params.id } });
  if (!post) throw redirect('/posts');
  return json(post);
}

export async function action({ params, request }) {
  const formData = await request.formData();
  const intent = formData.get('intent');
  
  if (intent === 'delete') {
    await db.post.delete({ where: { id: params.id } });
    return redirect('/posts');
  }
  
  const data = Object.fromEntries(formData);
  const updated = await db.post.update({ where: { id: params.id }, data });
  return json(updated);
}

export default function PostPage() {
  const post = useLoaderData();
  const actionData = useActionData();
  const navigation = useNavigation();
  
  return (
    <article>
      <h1>{post.title}</h1>
      <Form method="POST">
        <input name="title" defaultValue={post.title} />
        <textarea name="content" defaultValue={post.content} />
        {actionData?.errors?.title && <p>{actionData.errors.title}</p>}
        <button type="submit" disabled={navigation.state === 'submitting'}>
          {navigation.state === 'submitting' ? 'Saving...' : 'Save'}
        </button>
      </Form>
      <Form method="POST">
        <input type="hidden" name="intent" value="delete" />
        <button type="submit" disabled={navigation.state === 'submitting'}>
          Delete
        </button>
      </Form>
    </article>
  );
}
```

#### 4. **Code**: "Astro Partial Hydration"

```astro
---
// src/pages/blog/[slug].astro
import Layout from '../../layouts/Layout.astro';
import { getPost } from '../../lib/posts';
import InteractiveComments from '../components/Comments.svelte';
import RelatedPosts from '../components/RelatedPosts.jsx';
---

{/* Static HTML - zero JS */}
<Layout title={post.title}>
  <article>
    <h1>{post.title}</h1>
    <time>{post.date}</time>
    <div set:html={post.content} />
  </article>
  
  {/* Hydrates when user scrolls to section */}
  <section id="comments">
    <h2>Comments</h2>
    <InteractiveComments client:visible postId={post.id} />
  </section>
  
  {/* Hydrates when browser idle */}
  <RelatedPosts client:idle postId={post.id} />
</Layout>
```

---

### GOTCHAS & EDGE CASES

- [ ] **Next.js: Server Component default** - Forget 'use client' for interactivity
- [ ] **Next.js: Server Actions** - Must be 'use server', not in client components
- [ ] **Next.js: Revalidation** - revalidatePath vs revalidateTag vs on-demand
- [ ] **Remix: Loader revalidation** - Actions auto-revalidate, but only current route
- [ ] **Remix: Nested routes** - Parent loaders run for every child navigation
- [ ] **Astro: Client directives** - client:load vs visible vs idle vs media
- [ ] **Astro: Framework mixing** - React + Vue + Svelte in same page (islands)
- [ ] **Next.js: Middleware** - Runs on Edge, limited APIs, no body parsing
- [ ] **Remix: Resource routes** - For non-HTML responses (JSON, PDF, etc.)
- [ ] **Astro: Content collections** - Type-safe markdown with schema validation

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Next.js Middleware for Auth**

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token')?.value;
  const pathname = request.nextUrl.pathname;
  
  if (pathname.startsWith('/admin') && !token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  
  if (pathname.startsWith('/admin') && token) {
    // Validate token (JWT verification)
    const payload = await verifyToken(token);
    if (!payload || payload.role !== 'admin') {
      return NextResponse.redirect(new URL('/unauthorized', request.url));
    }
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: ['/admin/:path*', '/dashboard/:path*'],
};
```

#### 2. **Remix Resource Route (JSON API)**

```tsx
// app/routes/api.users.tsx
import { json } from '@remix-run/node';

export async function loader({ request }) {
  const users = await db.user.findMany();
  return json(users);
}

export async function action({ request }) {
  const formData = await request.formData();
  const user = await db.user.create({ data: Object.fromEntries(formData) });
  return json(user, { status: 201 });
}

// Usage in component
const response = await fetch('/api/users', {
  method: 'POST',
  body: new FormData(formElement),
});
```

#### 3. **Astro View Transitions**

```astro
---
// src/pages/index.astro
import { ViewTransitions } from 'astro:transitions';
---

<html>
  <head>
    <ViewTransitions />
  </head>
  <body>
    <main>
      <a href="/about" transition:name="hero-link">About</a>
    </main>
  </body>
</html>

---
// src/pages/about.astro
---

<html>
  <head>
    <ViewTransitions />
  </head>
  <body>
    <main>
      <h1 transition:name="hero-link">About Us</h1>
    </main>
  </body>
</html>
```

---

### PRACTICE PROBLEMS

1. **Build** a complete Next.js app: Auth (middleware), Posts (Server Components + Actions), Comments (Client Components)
2. **Build** a Remix app: Nested routes, Loaders/Actions, Optimistic UI, Role-based access
3. **Build** an Astro blog: Content collections, View transitions, Islands for comments/search
4. **Migrate** a CRA app to Next.js: Identify Server vs Client components, add Server Actions
5. **Compare** bundle sizes: Next.js vs Remix vs Astro for same feature set

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Next.js Server Components | Default. Run on server, zero client bundle, async, DB access, no hooks. |
| Next.js Client Components | 'use client'. Hydrated, full React API, interactivity, browser APIs. |
| Next.js Server Actions | 'use server'. Mutations on server, called from client, revalidatePath. |
| React Server Components | Zero bundle, server-only, async, streamed, no hydration cost. |
| Remix Loader | Runs before render, parallel, returns data, throws redirect/json. |
| Remix Action | Handles POST/PUT/DELETE, form submissions, auto-revalidates loaders. |
| Remix Nested Routes | Parent layout with Outlet, child routes match, parallel loaders. |
| Astro Islands | Static HTML + selective hydration. client:load/visible/idle/media/only. |
| Astro Content Collections | Type-safe markdown with schema validation (Zod). |
| Next.js Middleware | Edge runtime, runs before request, auth/redirect/rewrite. |
| Next.js revalidatePath | Purges cache for path, triggers re-fetch on next request. |
| Astro View Transitions | <ViewTransitions /> + transition:name for SPA-like animations. |
| React Server Components | Not in bundle, server-only, streamed, suspense, zero hydration. |
| Next.js App Router | File-system routing, layouts, loading/error templates, parallel routes. |

---

## NEXT TOPIC: `08-INTERVIEW-PREP/01-BEHAVIORAL.md`

> **Study Tip**: Build the same feature in Next.js, Remix, and Astro. Compare: bundle size, DX, performance, deployment.