# HTTP, DNS, AND BROWSER FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: How the Internet Works - HTTP, DNS, Browser

### WHY? (Problem Statement)

**Before Understanding Frontend:**
- Frontend runs in browser → Browser speaks HTTP
- HTTP relies on DNS → DNS resolves names to IPs
- Browser renders HTML/CSS/JS → Understanding parsing matters for performance
- Network is the bottleneck → Optimizing requests = faster apps

**What This Solves:**
| Problem | Understanding Needed |
|---------|---------------------|
| Slow page loads | HTTP/2, caching, critical rendering path |
| CORS errors | Same-origin policy, preflight requests |
| DNS issues | Resolution process, TTL, records |
| Security vulnerabilities | HTTPS, CSP, HSTS, cookies |
| Debugging network issues | DevTools Network tab, headers, timing |

**Real-World Analogy:**
- DNS = Phone book (name → number)
- HTTP = Language for ordering food
- Browser = Restaurant (kitchen = rendering engine, waiter = JS engine)

---

### HOW? (Internal Mechanism)

#### 1. **URL to Pixels - Complete Flow**

```
User types "https://nanolite.tech/products"
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. DNS RESOLUTION                                           │
├─────────────────────────────────────────────────────────────┤
│ Browser checks:                                             │
│   • Browser cache (DNS cache)                               │
│   • OS cache (hosts file, local resolver)                   │
│   • Router cache                                            │
│   • ISP recursive resolver                                  │
│                                                             │
│ Recursive Resolver → Root NS → TLD (.tech) → Authoritative  │
│ Returns: A record (IPv4) or AAAA record (IPv6)              │
│ TTL determines cache duration                               │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. TCP CONNECTION (3-way handshake)                         │
├─────────────────────────────────────────────────────────────┤
│ Client SYN → Server SYN-ACK → Client ACK                    │
│ TLS 1.3: Client Hello → Server Hello + Cert → Finished      │
│ Session resumption (0-RTT) for returning visitors           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. HTTP REQUEST                                             │
├─────────────────────────────────────────────────────────────┤
│ GET /products HTTP/2                                        │
│ Host: nanolite.tech                                         │
│ User-Agent: Mozilla/5.0...                                  │
│ Accept: text/html,application/xhtml+xml...                  │
│ Accept-Encoding: gzip, deflate, br                          │
│ Cookie: session=abc123                                      │
│                                                             │
│ HTTP/2: Single connection, multiplexed streams              │
│ Header compression (HPACK)                                  │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. SERVER PROCESSING                                        │
├─────────────────────────────────────────────────────────────┤
│ • Route matching                                            │
│ • Authentication/Authorization                              │
│ • Business logic                                            │
│ • Database queries                                          │
│ • Template rendering / JSON serialization                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. HTTP RESPONSE                                            │
├─────────────────────────────────────────────────────────────┤
│ HTTP/2 200 OK                                               │
│ Content-Type: text/html; charset=utf-8                      │
│ Content-Encoding: br                                        │
│ Cache-Control: public, max-age=3600                         │
│ ETag: "abc123"                                              │
│ Set-Cookie: session=xyz; HttpOnly; Secure; SameSite=Lax    │
│                                                             │
│ <html>...</html>                                            │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. BROWSER RENDERING (Critical Rendering Path)              │
├─────────────────────────────────────────────────────────────┤
│ HTML → DOM Tree                                            │
│ CSS → CSSOM Tree                                           │
│ DOM + CSSOM → Render Tree                                  │
│ Layout (Reflow) → Paint → Composite                        │
│                                                             │
│ JavaScript: Parsed, Compiled, Executed                     │
│   • Blocks parsing (unless async/defer)                    │
│   • Can modify DOM/CSSOM                                   │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **DNS Resolution Deep Dive**

```
Query: nanolite.tech (A record)

1. Browser Cache          → Hit? Return IP
2. OS Cache (hosts file)  → Hit? Return IP
3. Router Cache           → Hit? Return IP
4. ISP Recursive Resolver → Cache? Return IP
                            │
                            ▼ No cache
5. Root Nameservers (.)   → "Go to .tech TLD"
6. TLD Nameservers (.tech) → "Go to authoritative NS"
7. Authoritative NS       → Returns A record: 192.0.2.1

Key Records:
• A: IPv4 address
• AAAA: IPv6 address
• CNAME: Alias to another domain
• MX: Mail exchanger
• TXT: Text (SPF, DKIM, verification)
• NS: Name server delegation
• SOA: Start of Authority (zone info)

TTL (Time To Live): Seconds to cache. Lower = faster propagation, more queries.
```

#### 3. **HTTP Evolution**

| Version | Year | Key Features |
|---------|------|--------------|
| HTTP/0.9 | 1991 | GET only, no headers |
| HTTP/1.0 | 1996 | Headers, POST, status codes |
| HTTP/1.1 | 1997 | Keep-alive, pipelining, chunked, Host header |
| HTTP/2 | 2015 | Binary, multiplexing, header compression, server push |
| HTTP/3 | 2022 | QUIC (UDP), 0-RTT, better mobile |

**HTTP/2 vs HTTP/1.1:**
```
HTTP/1.1:                    HTTP/2:
┌──────────────┐            ┌──────────────┐
│ Request 1    │            │ Stream 1     │
│ Response 1   │            │ Stream 2     │  ← Single TCP connection
│ Request 2    │   ─────    │ Stream 3     │     Multiplexed
│ Response 2   │            │              │
│ Request 3    │            │ Headers:     │
│ Response 3   │            │ HPACK compressed
└──────────────┘            └──────────────┘
6 connections                1 connection
Head-of-line blocking        No blocking
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **HTTP Methods & Semantics**

| Method | Safe | Idempotent | Cacheable | Purpose |
|--------|------|------------|-----------|---------|
| GET | ✓ | ✓ | ✓ | Retrieve resource |
| HEAD | ✓ | ✓ | ✓ | Headers only |
| POST | ✗ | ✗ | ✗* | Create/submit |
| PUT | ✗ | ✓ | ✗ | Replace resource |
| PATCH | ✗ | ✗ | ✗ | Partial update |
| DELETE | ✗ | ✓ | ✗ | Remove resource |
| OPTIONS | ✓ | ✓ | ✗ | CORS preflight |
| TRACE | ✓ | ✓ | ✗ | Debug loopback |

*POST can be cached with explicit headers

#### 2. **Status Codes (Must Know)**

```csharp
// 1xx Informational
100 Continue
101 Switching Protocols

// 2xx Success
200 OK
201 Created (POST/PUT) - Location header
202 Accepted (async)
204 No Content (DELETE)

// 3xx Redirection
301 Moved Permanently (cache)
302 Found (temporary)
304 Not Modified (conditional GET)
307 Temporary Redirect (preserve method)
308 Permanent Redirect (preserve method)

// 4xx Client Error
400 Bad Request
401 Unauthorized (auth required)
403 Forbidden (auth but no permission)
404 Not Found
409 Conflict (duplicate)
422 Unprocessable Entity (validation)
429 Too Many Requests (rate limit)

// 5xx Server Error
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable (retry-after)
504 Gateway Timeout
```

#### 3. **Critical Headers**

**Request Headers:**
```http
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: en-US,en;q=0.9
Authorization: Bearer <token>
Cache-Control: no-cache, max-age=0
Content-Type: application/json
Cookie: session=abc; preferences=dark
If-None-Match: "etag-value"
If-Modified-Since: Wed, 21 Oct 2023 07:28:00 GMT
Origin: https://app.nanolite.tech
Referer: https://app.nanolite.tech/dashboard
User-Agent: Mozilla/5.0...
X-Requested-With: XMLHttpRequest
```

**Response Headers:**
```http
Cache-Control: public, max-age=31536000, immutable
Content-Encoding: br
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-xyz'
Content-Type: text/html; charset=utf-8
Date: Wed, 21 Oct 2023 07:28:00 GMT
ETag: "abc123-def456"
Expires: Thu, 31 Dec 2037 23:55:55 GMT
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
Location: /new-url
Server: nginx/1.24.0
Set-Cookie: session=xyz; HttpOnly; Secure; SameSite=Lax; Max-Age=2592000
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Accept-Encoding
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
```

#### 4. **Caching Strategies**

```http
// Immutable assets (hashed filenames)
Cache-Control: public, max-age=31536000, immutable

// HTML (always revalidate)
Cache-Control: no-cache, must-revalidate

// API responses
Cache-Control: private, max-age=60, stale-while-revalidate=300

// Conditional requests
ETag: "v1.2.3"
If-None-Match: "v1.2.3" → 304 Not Modified

Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
If-Modified-Since: Wed, 21 Oct 2023 07:28:00 GMT → 304
```

#### 5. **Cookies & Storage**

| Feature | Cookie | localStorage | sessionStorage | IndexedDB |
|---------|--------|--------------|----------------|-----------|
| Capacity | 4KB | 5-10MB | 5-10MB | ~50MB+ |
| Sent to server | Auto | No | No | No |
| Expiry | Configurable | Never | Tab close | Configurable |
| SameSite | Lax/Strict/None | N/A | N/A | N/A |
| HttpOnly | Yes | No | No | No |
| Secure | Yes | HTTPS only | HTTPS only | HTTPS only |

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **HTTPS Everywhere** | Always | HTTP for "non-sensitive" | MITM, SEO penalty |
| **HSTS** | Production | Missing HSTS | Downgrade attacks |
| **CSP** | All apps | No CSP or `unsafe-inline` | XSS |
| **Cache-Control** | Static assets | No caching / `no-store` everything | Slow loads, server load |
| **Compression** | Text resources | No compression / compressing images | Wasted bandwidth |
| **HTTP/2 or 3** | Modern browsers | HTTP/1.1 only | Head-of-line blocking |
| **Preconnect/DNS-prefetch** | Third-party origins | Nothing | Late connections |
| **Service Workers** | Offline/PWA | AppCache (deprecated) | Limited, buggy |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "What happens when you type a URL and press Enter?"

**Answer:** (Condensed - 2 minute version)
```
1. Parse URL → extract protocol, host, path
2. DNS Resolution → IP address (cache hierarchy)
3. TCP Connection → 3-way handshake (+ TLS)
4. HTTP Request → GET /path HTTP/2
5. Server processes → generates response
6. HTTP Response → headers + body
7. Browser parses HTML → DOM tree
8. Fetches subresources (CSS, JS, images)
9. CSSOM + DOM → Render Tree
10. Layout → Paint → Composite
11. JavaScript execution (blocks parsing)
12. Interactive (TTI)
```

#### 2. **Code**: "Implement a fetch wrapper with retry, timeout, and caching"

```typescript
interface FetchOptions extends RequestInit {
  timeout?: number;
  retries?: number;
  cache?: 'force-cache' | 'no-cache' | 'only-if-cached';
}

class HttpClient {
  private cache = new Map<string, { data: any; expires: number }>();
  
  async fetch<T>(url: string, options: FetchOptions = {}): Promise<T> {
    const { timeout = 10000, retries = 3, cache = 'no-cache', ...init } = options;
    const cacheKey = `${init.method || 'GET'}:${url}:${JSON.stringify(init.body)}`;
    
    // Check cache
    if (cache === 'force-cache') {
      const cached = this.cache.get(cacheKey);
      if (cached && cached.expires > Date.now()) return cached.data;
    }
    
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeout);
    
    let lastError: Error;
    
    for (let attempt = 0; attempt <= retries; attempt++) {
      try {
        const response = await fetch(url, {
          ...init,
          signal: controller.signal,
          cache: cache === 'no-cache' ? 'no-cache' : 'default'
        });
        
        clearTimeout(timeoutId);
        
        if (!response.ok) {
          throw new HttpError(response.status, response.statusText);
        }
        
        const data = await response.json();
        
        // Cache successful responses
        if (cache === 'force-cache' && (init.method === 'GET' || !init.method)) {
          const cacheControl = response.headers.get('Cache-Control');
          const maxAge = this.parseMaxAge(cacheControl) || 300;
          this.cache.set(cacheKey, { 
            data, 
            expires: Date.now() + maxAge * 1000 
          });
        }
        
        return data;
      } catch (error) {
        lastError = error as Error;
        if (error instanceof DOMException && error.name === 'AbortError') {
          throw new TimeoutError(`Request timeout after ${timeout}ms`);
        }
        if (attempt < retries) {
          await this.delay(Math.pow(2, attempt) * 1000); // Exponential backoff
        }
      }
    }
    
    throw lastError!;
  }
  
  private parseMaxAge(header: string | null): number | null {
    if (!header) return null;
    const match = header.match(/max-age=(\d+)/);
    return match ? parseInt(match[1], 10) : null;
  }
  
  private delay(ms: number) => new Promise(r => setTimeout(r, ms));
}

class HttpError extends Error {
  constructor(public status: number, public statusText: string) {
    super(`HTTP ${status}: ${statusText}`);
    this.name = 'HttpError';
  }
}

class TimeoutError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'TimeoutError';
  }
}
```

#### 3. **Security**: "Explain CORS and how to fix CORS errors"

**Answer:**
```
CORS (Cross-Origin Resource Sharing):
- Browser security feature (Same-Origin Policy)
- Origin = Protocol + Domain + Port
- Blocks cross-origin requests by default

SIMPLE REQUEST (no preflight):
- GET/HEAD/POST
- Content-Type: text/plain, application/x-www-form-urlencoded, multipart/form-data
- No custom headers

PREFLIGHT REQUEST (OPTIONS):
- Any other method or header
- Browser sends OPTIONS first
- Server responds with allowed methods/headers

SERVER RESPONSE HEADERS:
Access-Control-Allow-Origin: https://app.example.com  (not * with credentials)
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400  (cache preflight)

COMMON FIXES:
1. Server: Add proper CORS headers
2. Proxy: /api/* → backend (same origin)
3. Backend-for-Frontend (BFF): Frontend calls BFF, BFF calls APIs
```

#### 4. **Performance**: "Optimize Critical Rendering Path"

```html
<!-- 1. Minimize critical resources -->
<!-- 2. Minimize critical bytes -->
<!-- 3. Minimize critical path length -->

<!DOCTYPE html>
<html>
<head>
  <!-- Critical CSS inline -->
  <style>/* critical above-fold styles */</style>
  
  <!-- Non-critical CSS async -->
  <link rel="preload" href="/styles.css" as="style" onload="this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/styles.css"></noscript>
  
  <!-- Preconnect to origins -->
  <link rel="preconnect" href="https://api.example.com" crossorigin>
  <link rel="dns-prefetch" href="https://cdn.example.com">
  
  <!-- Critical JS inline (small) -->
  <script>/* inline critical JS */</script>
</head>
<body>
  <!-- Content -->
  
  <!-- Non-critical JS deferred -->
  <script src="/app.js" defer></script>
  
  <!-- Or module (deferred by default) -->
  <script type="module" src="/app.js"></script>
</body>
</html>
```

#### 5. **Code**: "Implement Service Worker for offline caching"

```javascript
// sw.js
const CACHE_NAME = 'nanolite-v1.2.3';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/styles.css',
  '/app.js',
  '/manifest.json'
];

// Install - cache static assets
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_ASSETS))
      .then(() => self.skipWaiting())
  );
});

// Activate - clean old caches
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys()
      .then(keys => Promise.all(
        keys.filter(key => key !== CACHE_NAME)
            .map(key => caches.delete(key))
      ))
      .then(() => self.clients.claim())
  );
});

// Fetch - cache first, then network
self.addEventListener('fetch', event => {
  const { request } = event;
  
  // Skip non-GET
  if (request.method !== 'GET') return;
  
  // Skip chrome-extension, etc.
  if (!request.url.startsWith('http')) return;
  
  event.respondWith(
    caches.match(request)
      .then(cached => {
        // Cache hit
        if (cached) return cached;
        
        // Network request
        return fetch(request)
          .then(response => {
            // Don't cache non-success
            if (!response.ok) return response;
            
            // Clone response (streams can only be consumed once)
            const responseClone = response.clone();
            
            caches.open(CACHE_NAME)
              .then(cache => cache.put(request, responseClone));
            
            return response;
          })
          .catch(() => {
            // Offline fallback
            if (request.mode === 'navigate') {
              return caches.match('/offline.html');
            }
          });
      })
  );
});

// Background sync for failed requests
self.addEventListener('sync', event => {
  if (event.tag === 'sync-forms') {
    event.waitUntil(syncForms());
  }
});

async function syncForms() {
  const db = await openDB('offline-forms', 1);
  const forms = await db.getAll('forms');
  
  for (const form of forms) {
    try {
      await fetch(form.url, { method: 'POST', body: form.data });
      await db.delete('forms', form.id);
    } catch (e) {
      // Will retry on next sync
    }
  }
}
```

#### 6. **Architecture**: "Design a frontend for a real-time dashboard"

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION SHELL                        │
├─────────────────────────────────────────────────────────────┤
│  Header (static)    │  Sidebar (navigation)  │  Main Area  │
├─────────────────────────────────────────────────────────────┤
│                        STATE MANAGEMENT                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Server     │  │  Client     │  │  URL/Router         │  │
│  │  State      │  │  State      │  │  State              │  │
│  │  (TanStack  │  │  (Zustand/  │  │  (Search params,    │  │
│  │   Query)    │  │   Redux)    │  │   filters)          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                        DATA FETCHING                         │
│  ┌─────────────────┐  ┌─────────────────────────────────┐   │
│  │  REST/GraphQL   │  │  WebSocket / Server-Sent Events │   │
│  │  (Queries,      │  │  (Real-time updates,            │   │
│  │   Mutations)    │  │   subscriptions)                │   │
│  └─────────────────┘  └─────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                      RENDERING STRATEGY                      │
│  • Code-split by route (React.lazy + Suspense)              │
│  • Virtualize large lists (react-window)                    │
│  • Memoize expensive components (React.memo, useMemo)       │
│  • Web Workers for heavy computation                        │
└─────────────────────────────────────────────────────────────┘
```

#### 7. **Debugging**: "Network tab shows 304 but UI not updated"

```
CHECKLIST:
☐ Service Worker intercepting? (Check Application tab)
☐ Cache-Control: no-cache vs no-store?
☐ ETag changing on every deploy? (Use content hash)
☐ Browser cache disabled in DevTools?
☐ CDN caching stale version? (Purge CDN)
☐ If-None-Match header sent?
☐ Response has correct ETag/Last-Modified?
☐ Vary header causing cache miss?
☐ Check: Disable cache (DevTools) → Reload → Works?
```

#### 8. **Security**: "Implement Content Security Policy"

```html
<!-- Report-only mode first -->
<meta http-equiv="Content-Security-Policy-Report-Only" 
      content="default-src 'self'; 
               script-src 'self' 'nonce-{RANDOM}'; 
               style-src 'self' 'nonce-{RANDOM}'; 
               img-src 'self' data: https:; 
               font-src 'self' https://fonts.gstatic.com; 
               connect-src 'self' https://api.example.com; 
               frame-ancestors 'none'; 
               base-uri 'self'; 
               form-action 'self';
               report-uri /csp-report">

<!-- Or via header (preferred) -->
Content-Security-Policy: default-src 'self'; 
  script-src 'self' 'nonce-{RANDOM}'; 
  style-src 'self' 'nonce-{RANDOM}'; 
  img-src 'self' data: https:; 
  font-src 'self' https://fonts.gstatic.com; 
  connect-src 'self' https://api.example.com; 
  frame-ancestors 'none'; 
  base-uri 'self'; 
  form-action 'self';
  report-uri /csp-report;
```

#### 9. **Performance**: "Measure and optimize Core Web Vitals"

```javascript
// LCP (Largest Contentful Paint) - Target: < 2.5s
new PerformanceObserver((entries) => {
  const lastEntry = entries.getEntries().pop();
  console.log('LCP:', lastEntry.startTime);
  sendToAnalytics({ lcp: lastEntry.startTime });
}).observe({ type: 'largest-contentful-paint', buffered: true });

// FID (First Input Delay) - Target: < 100ms
new PerformanceObserver((entries) => {
  entries.getEntries().forEach(entry => {
    console.log('FID:', entry.processingStart - entry.startTime);
  });
}).observe({ type: 'first-input', buffered: true });

// CLS (Cumulative Layout Shift) - Target: < 0.1
let clsValue = 0;
new PerformanceObserver((entries) => {
  entries.getEntries().forEach(entry => {
    if (!entry.hadRecentInput) {
      clsValue += entry.value;
    }
  });
}).observe({ type: 'layout-shift', buffered: true });

// INP (Interaction to Next Paint) - Replaces FID
new PerformanceObserver((entries) => {
  entries.getEntries().forEach(entry => {
    console.log('INP:', entry.duration);
  });
}).observe({ type: 'event', buffered: true, durationThreshold: 200 });

// Resource Timing
const resources = performance.getEntriesByType('resource');
resources.forEach(r => {
  console.log(`${r.name}: ${r.duration}ms (${r.transferSize} bytes)`);
});
```

#### 10. **Code**: "Type-safe API client with OpenAPI generation"

```typescript
// Generated from OpenAPI spec (or use orval, openapi-typescript)
// api.ts
export interface User {
  id: string;
  email: string;
  name: string;
  roles: string[];
}

export interface CreateUserRequest {
  email: string;
  name: string;
  password: string;
}

export interface ApiError {
  status: number;
  message: string;
  errors?: Record<string, string[]>;
}

// Client with full type safety
class TypedApiClient {
  private baseUrl: string;
  
  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }
  
  private async request<T>(
    path: string, 
    options: RequestInit = {}
  ): Promise<T> {
    const response = await fetch(`${this.baseUrl}${path}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
      credentials: 'include', // Send cookies
    });
    
    if (!response.ok) {
      const error: ApiError = await response.json();
      throw new ApiError(response.status, error.message, error.errors);
    }
    
    if (response.status === 204) return undefined as T;
    return response.json();
  }
  
  // Users API
  getUsers = (params?: { page?: number; limit?: number }) => 
    this.request<User[]>('/users', { 
      method: 'GET',
      // Auto-serialize query params
    });
  
  getUser = (id: string) => 
    this.request<User>(`/users/${id}`);
  
  createUser = (data: CreateUserRequest) => 
    this.request<User>('/users', { 
      method: 'POST', 
      body: JSON.stringify(data) 
    });
  
  updateUser = (id: string, data: Partial<CreateUserRequest>) => 
    this.request<User>(`/users/${id}`, { 
      method: 'PATCH', 
      body: JSON.stringify(data) 
    });
  
  deleteUser = (id: string) => 
    this.request<void>(`/users/${id}`, { method: 'DELETE' });
}

class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public errors?: Record<string, string[]>
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

// Usage
const api = new TypedApiClient('/api');

try {
  const users = await api.getUsers({ page: 1, limit: 20 });
  const user = await api.createUser({ 
    email: 'test@test.com', 
    name: 'Test', 
    password: 'secret' 
  });
} catch (error) {
  if (error instanceof ApiError) {
    if (error.status === 422) {
      // Handle validation errors
      console.log(error.errors);
    }
  }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Mixed Content**: HTTPS page loading HTTP resources → blocked
- [ ] **CORS Preflight Caching**: Max-Age prevents repeated OPTIONS
- [ ] **Cookie SameSite**: Lax default breaks cross-site POST; use None + Secure for cross-site
- [ ] **Cache Busting**: Use content hash in filename, not query string
- [ ] **HTTP/2 Prioritization**: Browser decides, but can hint with `Priority` header
- [ ] **304 Not Modified**: Must include ETag/Last-Modified in response
- [ ] **Service Worker Scope**: Only controls pages under its scope
- [ ] **IndexedDB Quota**: Request persistent storage for larger quotas
- [ ] **Web Workers**: No DOM access, use `postMessage` for communication
- [ ] **Memory Leaks**: Event listeners, intervals, subscriptions not cleaned up

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Resource Hints**
```html
<!-- Preconnect - full connection (DNS + TCP + TLS) -->
<link rel="preconnect" href="https://api.example.com" crossorigin>

<!-- DNS Prefetch - only DNS -->
<link rel="dns-prefetch" href="https://cdn.example.com">

<!-- Preload - high priority fetch -->
<link rel="preload" href="/critical.css" as="style">
<link rel="preload" href="/hero-image.webp" as="image">

<!-- Prefetch - low priority, next navigation -->
<link rel="prefetch" href="/next-page.html">

<!-- Module Preload - JS modules -->
<link rel="modulepreload" href="/app.js">
```

#### 2. **Fetch with AbortController**
```javascript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch('/api/data', { signal: controller.signal });
  const data = await response.json();
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('Request timed out');
  }
} finally {
  clearTimeout(timeoutId);
}
```

#### 3. **Security Headers Checklist**
```http
# Required for all production sites
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; [your policy]
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

---

### PRACTICE PROBLEMS

1. **Debug**: Page loads slowly - analyze Network tab waterfall, identify bottlenecks
2. **Implement**: Offline-first todo app with Service Worker + IndexedDB
3. **Secure**: Add CSP to existing app without breaking functionality
4. **Optimize**: Reduce bundle size by 50% using code splitting, tree shaking
5. **Design**: Real-time collaborative editor architecture (WebRTC + WebSocket)

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| DNS Resolution Order | Browser cache → OS cache → Router → ISP resolver → Root → TLD → Authoritative |
| HTTP/2 Key Features | Binary, multiplexing, HPACK header compression, server push |
| HTTP/3 Transport | QUIC over UDP (not TCP) |
| CORS Preflight Trigger | Non-simple request (custom headers, non-standard methods, content-type) |
| Cache-Control: no-cache | Must revalidate with server before use (sends If-None-Match) |
| Cache-Control: no-store | Never store any copy (most restrictive) |
| Service Worker Lifecycle | Install → Activate → Fetch/Message/Sync |
| Critical Rendering Path | HTML→DOM, CSS→CSSOM, DOM+CSSOM→Render Tree, Layout→Paint→Composite |
| LCP/FID/CLS Targets | LCP < 2.5s, FID < 100ms, CLS < 0.1 |
| SameSite Cookie Values | Strict (no cross-site), Lax (top-level nav), None (cross-site + Secure) |

---

## NEXT TOPIC: `02-HTML-SEMANTIC-ACCESSIBILITY.md`

> **Study Tip**: Open DevTools Network tab, load a complex site (GitHub, Twitter). Analyze: request waterfall, timing, headers, caching, CORS, security headers. Write down 5 observations.