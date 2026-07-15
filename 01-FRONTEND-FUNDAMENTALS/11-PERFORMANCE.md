# FRONTEND PERFORMANCE OPTIMIZATION
## Exam-Style Study Notes

---

## TOPIC: Web Performance - Metrics, Optimization, Monitoring

### WHY? (Problem Statement)

**The Reality:**
- 53% of users abandon sites taking >3s to load
- 1s delay = 7% conversion drop (Amazon)
- Core Web Vitals = SEO ranking factor
- Mobile-first indexing = mobile performance critical

**What Performance Solves:**
| Problem | Impact | Solution |
|---------|--------|----------|
| Slow FCP/LCP | Users leave before seeing content | Critical CSS, preload, SSR |
| Layout Shift (CLS) | Frustrating UX, misclicks | Aspect ratios, font loading |
| Slow INP/FID | Unresponsive UI | Code splitting, web workers |
| Large bundles | Slow download, parse, execute | Tree shaking, code splitting |
| Unoptimized images | Wasted bandwidth | WebP/AVIF, responsive images |

**Real-World Analogy:**
- Website = Restaurant
- Performance = Time from order to food on table
- User = Hungry customer who leaves if wait > 10 min

---

### HOW? (Browser Rendering Pipeline)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CRITICAL RENDERING PATH                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  HTML → HTML Parser → DOM Tree                                     │
│                     │                                              │
│                     ▼                                              │
│  CSS → CSS Parser → CSSOM Tree                                     │
│                     │                                              │
│                     ▼                                              │
│  DOM + CSSOM → Render Tree (visible elements only)                │
│                     │                                              │
│                     ▼                                              │
│  Layout (Reflow) → Calculate positions/sizes                      │
│                     │                                              │
│                     ▼                                              │
│  Paint → Pixels on screen (text, colors, borders)                 │
│                     │                                              │
│                     ▼                                              │
│  Composite → Layers combined (GPU accelerated)                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

BLOCKING RESOURCES:
- HTML (always)
- CSS (blocks render + paint)
- JavaScript (blocks parse + can modify DOM/CSSOM)

NON-BLOCKING:
- Images (load async)
- Fonts (font-display: swap)
- Async/defer scripts
```

---

### WHAT? (Core Web Vitals & Metrics)

#### 1. **Core Web Vitals (Google's Ranking Signals)**

| Metric | Good | Needs Improvement | Poor | Measures |
|--------|------|-------------------|------|----------|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | 2.5-4s | > 4s | Loading |
| **INP** (Interaction to Next Paint) | ≤ 200ms | 200-500ms | > 500ms | Interactivity |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | 0.1-0.25 | > 0.25 | Visual Stability |

**LCP Candidates:** `<img>`, `<video>` poster, background-image, text blocks
**INP:** Worst interaction latency (click, tap, key) - replaces FID
**CLS:** Sum of all unexpected layout shifts (score = impact × distance)

#### 2. **Other Key Metrics**

| Metric | Target | Description |
|--------|--------|-------------|
| **FCP** (First Contentful Paint) | ≤ 1.8s | First pixel painted |
| **TTFB** (Time to First Byte) | ≤ 800ms | Server response |
| **TBT** (Total Blocking Time) | ≤ 200ms | Main thread blocked > 50ms |
| **SI** (Speed Index) | ≤ 3.4s | Visual completeness |
| **TTI** (Time to Interactive) | ≤ 3.8s | Fully interactive |

---

### OPTIMIZATION STRATEGIES

#### 1. **Loading Optimization**

```html
<!-- Critical CSS Inlined -->
<style>
  /* Only above-fold styles */
  .hero { display: flex; min-height: 100vh; }
  .btn { padding: 1rem 2rem; background: #0066cc; }
</style>

<!-- Non-critical CSS Async -->
<link rel="preload" href="/styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="/styles.css"></noscript>

<!-- Resource Hints -->
<link rel="preconnect" href="https://fonts.googleapis.com" crossorigin>
<link rel="preconnect" href="https://api.example.com" crossorigin>
<link rel="dns-prefetch" href="https://cdn.example.com">
<link rel="prefetch" href="/next-page.html">
<link rel="prerender" href="/dashboard.html">

<!-- Preload Critical Assets -->
<link rel="preload" href="/hero-image.webp" as="image" type="image/webp">
<link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossorigin>

<!-- Font Optimization -->
@font-face {
  font-family: 'Inter';
  src: url('/inter-var.woff2') format('woff2');
  font-display: swap;        /* Show fallback immediately */
  font-size-adjust: 0.5;     /* Match x-height */
}
```

#### 2. **JavaScript Optimization**

```javascript
// Code Splitting - Route level
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

// Code Splitting - Component level
const HeavyChart = lazy(() => import('./HeavyChart').then(m => ({ default: m.Chart })));

// Dynamic Import for features
async function openModal() {
  const { Modal } = await import('./Modal');
  Modal.open();
}

// Tree Shaking - Use ES Modules
// ✅ import { debounce } from 'lodash-es';
// ❌ import _ from 'lodash';

// Bundle Analysis
// vite.config.ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        'react-vendor': ['react', 'react-dom', 'react-router-dom'],
        'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
        'utils': ['date-fns', 'zod', 'axios'],
      }
    }
  }
}

// Remove Dead Code (Production)
/* @pure */ console.log('debug'); // Removed by terser
if (process.env.NODE_ENV === 'development') { /* dev only */ }

// Web Workers for Heavy Computation
// worker.ts
self.onmessage = (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
};

// main.ts
const worker = new Worker(new URL('./worker.ts', import.meta.url));
worker.postMessage(data);
worker.onmessage = (e) => { /* handle result */ };
```

#### 3. **Image Optimization**

```html
<!-- Responsive Images with WebP/AVIF -->
<picture>
  <source type="image/avif" srcset="hero-400.avif 400w, hero-800.avif 800w, hero-1200.avif 1200w" sizes="(max-width: 768px) 100vw, 50vw">
  <source type="image/webp" srcset="hero-400.webp 400w, hero-800.webp 800w, hero-1200.webp 1200w" sizes="(max-width: 768px) 100vw, 50vw">
  <img src="hero-800.jpg" alt="Hero" width="1200" height="600" loading="lazy" decoding="async">
</picture>

<!-- LCP Image - High Priority -->
<img src="hero.jpg" alt="Hero" width="1200" height="600" 
     fetchpriority="high" loading="eager" 
     style="aspect-ratio: 2/1; width: 100%; height: auto;">

<!-- Lazy Load with Blur Placeholder -->
<img 
  src="tiny-blur.jpg" 
  data-src="full-image.jpg" 
  alt="Description"
  loading="lazy"
  decoding="async"
  class="lazy-image"
  width="800" height="600"
>
```

```javascript
// IntersectionObserver for Lazy Loading (Polyfill-free)
const lazyImages = document.querySelectorAll('img[data-src]');

const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      img.removeAttribute('data-src');
      observer.unobserve(img);
    }
  });
}, { rootMargin: '100px 0px', threshold: 0.01 });

lazyImages.forEach(img => observer.observe(img));
```

#### 4. **Caching Strategy**

```nginx
# Nginx Cache Headers
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
  add_header Vary Accept-Encoding;
}

location / {
  # HTML - no cache, always revalidate
  add_header Cache-Control "no-cache, no-store, must-revalidate";
}

# Service Worker Caching (Workbox)
# workbox-config.js
module.exports = {
  globDirectory: 'dist/',
  globPatterns: ['**/*.{js,css,html,png,jpg,svg,woff2}'],
  swDest: 'dist/sw.js',
  runtimeCaching: [
    {
      urlPattern: /^https:\/\/api\.example\.com/,
      handler: 'NetworkFirst',
      options: {
        cacheName: 'api-cache',
        expiration: { maxEntries: 100, maxAgeSeconds: 5 * 60 },
        networkTimeoutSeconds: 10,
      }
    },
    {
      urlPattern: /^https:\/\/fonts\.googleapis\.com/,
      handler: 'StaleWhileRevalidate',
      options: { cacheName: 'google-fonts' }
    }
  ]
};
```

#### 5. **CLS Prevention**

```css
/* Reserve Space for Images */
img, video {
  aspect-ratio: attr(width) / attr(height);
  width: 100%;
  height: auto;
}

/* Reserve Space for Ads/Embeds */
.ad-slot {
  min-height: 250px; /* Reserve space */
}

/* Font Loading - Prevent FOIT/FOUT */
@font-face {
  font-family: 'CustomFont';
  src: url('/font.woff2') format('woff2');
  font-display: optional; /* Best for CLS */
  /* swap = show fallback immediately, swap when loaded */
  /* optional = use fallback if font not ready in 100ms */
}

/* Aspect Ratio Boxes (Legacy) */
.aspect-ratio-16-9 {
  position: relative;
  width: 100%;
  padding-top: 56.25%; /* 9/16 = 56.25% */
}
.aspect-ratio-16-9 > * {
  position: absolute;
  top: 0; left: 0; width: 100%; height: 100%;
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Preload critical resources** | Preload everything | Bandwidth contention |
| **Code split by route** | Single massive bundle | Load unused code |
| **WebP/AVIF with fallbacks** | PNG/JPG only | 30-50% larger |
| **font-display: swap/optional** | font-display: block | Invisible text |
| **Aspect ratios on media** | No dimensions | Layout shift |
| **Service Worker caching** | No offline support | Poor repeat visits |
| **Compression (Brotli/Gzip)** | Uncompressed | 70%+ size reduction |
| **HTTP/2 or HTTP/3** | HTTP/1.1 | Multiplexing, header compression |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Explain Critical Rendering Path optimization"

**Answer:**
```
CRP = HTML → DOM → CSSOM → Render Tree → Layout → Paint → Composite

OPTIMIZATIONS:
1. Minimize Critical Resources
   - Inline critical CSS
   - Defer non-critical CSS (preload + onload)
   - Defer/async JavaScript

2. Minimize Critical Bytes
   - Minify/compress (Brotli > Gzip)
   - Remove unused CSS (PurgeCSS)
   - Code splitting

3. Minimize Critical Path Length
   - Preconnect to origins
   - Preload key resources
   - Server push (HTTP/2) - use carefully

4. Optimize Render Blocking
   - CSS blocks render → inline critical, async rest
   - JS blocks parse → defer (order preserved) / async (no order)
```

#### 2. **Code**: "Optimize LCP for this hero section"

```html
<!-- BEFORE (Poor LCP) -->
<section class="hero">
  <div class="hero-content">
    <h1>Welcome</h1>
    <p>Description</p>
  </div>
  <img src="hero-large.jpg" alt="Hero">
</section>

<style>
.hero { background: url('hero-bg.jpg') center/cover; }
</style>

<!-- AFTER (Optimized LCP) -->
<!-- 1. Preload LCP image -->
<link rel="preload" as="image" href="hero.webp" type="image/webp" fetchpriority="high">

<!-- 2. Inline critical CSS for hero -->
<style>
.hero { min-height: 100vh; display: flex; align-items: center; }
.hero-img { aspect-ratio: 16/9; width: 100%; object-fit: cover; }
</style>

<!-- 3. Hero with proper sizing, priority hints -->
<section class="hero">
  <div class="hero-content">
    <h1>Welcome</h1>
  </div>
  <img 
    src="hero.webp" 
    alt="Hero description"
    width="1920" height="1080"
    fetchpriority="high"
    loading="eager"
    class="hero-img"
  >
</section>

<!-- 4. Preconnect to image CDN -->
<link rel="preconnect" href="https://images.example.com" crossorigin>
```

#### 3. **Performance**: "Debug high TBT (Total Blocking Time)"

```
TBT DEBUGGING CHECKLIST:

1. Identify Long Tasks (>50ms)
   - Chrome DevTools → Performance → Long Tasks
   - Look for red triangles

2. Common Culprits:
   □ Large bundle parsing/executing
   □ Heavy initial render (React hydration)
   □ Third-party scripts (analytics, ads)
   □ Expensive calculations on main thread
   □ Layout thrashing (read/write DOM alternately)

3. Fixes:
   ✓ Code splitting (reduce initial bundle)
   ✓ Web Workers (move computation)
   ✓ requestIdleCallback (defer non-urgent)
   ✓ Virtualize lists (react-window)
   ✓ Debounce/throttle handlers
   ✓ Avoid forced sync layouts (getBoundingClientRect after write)

4. Measure:
   - TBT = Sum of (task duration - 50ms) for all tasks > 50ms
   - Target: < 200ms
```

#### 4. **Code**: "Implement efficient list rendering"

```jsx
// BAD: Renders all 10,000 items
function BadList({ items }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}

// GOOD: Virtualized (react-window)
import { FixedSizeList as List } from 'react-window';

function VirtualList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style} className="list-item">
      {items[index].name}
    </div>
  );
  
  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
      overscanCount={5}
    >
      {Row}
    </List>
  );
}

// GOOD: Infinite Scroll with IntersectionObserver
function InfiniteList({ fetchMore }) {
  const observer = useRef();
  const lastItemRef = useCallback(node => {
    if (observer.current) observer.current.disconnect();
    observer.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) fetchMore();
    });
    if (node) observer.current.observe(node);
  }, [fetchMore]);
  
  return (
    <div>
      {items.map(item => <Item key={item.id} {...item} />)}
      <div ref={lastItemRef} />
    </div>
  );
}
```

#### 5. **Architecture**: "Design a performance budget"

```javascript
// performance-budget.json
{
  "budgets": [
    {
      "resource": "bundle",
      "type": "javascript",
      "maxSize": "170kb" // gzipped
    },
    {
      "resource": "bundle", 
      "type": "css",
      "maxSize": "50kb"
    },
    {
      "resource": "third-party",
      "maxSize": "100kb"
    },
    {
      "metric": "LCP",
      "maxValue": 2500
    },
    {
      "metric": "INP",
      "maxValue": 200
    },
    {
      "metric": "CLS",
      "maxValue": 0.1
    }
  ],
  
  // CI Integration
  "ci": {
    "failOnOverBudget": true,
    "comparison": "baseline" // or "previous"
  }
}

// Bundle Analysis in CI
// GitHub Actions
- name: Check Bundle Size
  run: |
    npx vite build
    npx bundlesize
    npx lighthouse-ci --config=lighthouserc.json
```

#### 6. **Optimization**: "Third-party script optimization"

```html
<!-- BEFORE: Blocks main thread -->
<script src="https://analytics.example.com/tracker.js"></script>
<script src="https://chat.example.com/widget.js"></script>

<!-- AFTER: Multiple strategies -->

<!-- 1. Defer + Preconnect -->
<link rel="preconnect" href="https://analytics.example.com" crossorigin>
<script defer src="https://analytics.example.com/tracker.js"></script>

<!-- 2. Facade Pattern (Load on Interaction) -->
<div id="chat-facade" class="chat-widget-facade" data-src="https://chat.example.com/widget.js">
  <button onclick="loadChat()">Chat with us</button>
</div>

<script>
function loadChat() {
  const facade = document.getElementById('chat-facade');
  const script = document.createElement('script');
  script.src = facade.dataset.src;
  script.async = true;
  document.body.appendChild(script);
  facade.replaceWith(script);
}

// Auto-load after idle
if ('requestIdleCallback' in window) {
  requestIdleCallback(() => loadChat(), { timeout: 3000 });
} else {
  setTimeout(loadChat, 3000);
}
</script>

<!-- 3. Partytown (Run in Web Worker) -->
<script>
  // partytown config
  partytown = {
    lib: "/~partytown/",
    forward: ["dataLayer.push"],
  };
</script>
<script type="text/partytown" src="https://analytics.example.com/tracker.js"></script>
```

#### 7. **Measurement**: "Real User Monitoring (RUM) Setup"

```javascript
// web-vitals.js - Send to Analytics
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = {
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
    page: window.location.pathname,
    connection: navigator.connection?.effectiveType,
    deviceMemory: navigator.deviceMemory,
  };
  
  // Use sendBeacon for reliability
  navigator.sendBeacon('/api/metrics', JSON.stringify(body));
}

onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);

// Custom Metric: Time to First Interaction
let firstInteraction = null;
['click', 'keydown', 'touchstart'].forEach(event => {
  document.addEventListener(event, () => {
    if (!firstInteraction) {
      firstInteraction = performance.now();
      sendToAnalytics({
        name: 'TTFI',
        value: firstInteraction,
        rating: firstInteraction < 100 ? 'good' : firstInteraction < 300 ? 'needs-improvement' : 'poor'
      });
    }
  }, { once: true, passive: true });
});
```

---

### GOTCHAS & EDGE CASES

- [ ] **Preload everything** → Browser ignores excess, wastes bandwidth
- [ ] **Async CSS with onload** → Fails if JS disabled → Use `<noscript>` fallback
- [ ] **font-display: swap** → Layout shift when font loads → Use `size-adjust` + fallback font matching
- [ ] **Lazy load LCP image** → NEVER lazy load LCP candidate!
- [ ] **Service Worker caches HTML** → Users see stale content → Cache HTML with `no-cache`
- [ ] **HTTP/2 push** → Can hurt if pushing cached resources → Use preload instead
- [ ] **Brotli compression** → Needs HTTPS, server config
- [ ] **Third-party fonts** → Self-host for privacy + performance
- [ ] **Image dimensions** → Always specify width/height for aspect ratio
- [ ] **React 18 concurrent features** → `useTransition`, `useDeferredValue` for non-blocking updates

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Vite Performance Config**
```typescript
// vite.config.ts
export default defineConfig({
  build: {
    target: 'es2020',
    minify: 'esbuild', // or 'terser' for better compression
    cssCodeSplit: true,
    sourcemap: true,
    reportCompressedSize: true,
    chunkSizeWarningLimit: 500,
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('react') || id.includes('react-dom') || id.includes('react-router')) {
              return 'react-vendor';
            }
            if (id.includes('@radix-ui') || id.includes('@headlessui')) {
              return 'ui-vendor';
            }
            if (id.includes('date-fns') || id.includes('zod') || id.includes('axios')) {
              return 'utils';
            }
            return 'vendor';
          }
        },
        chunkFileNames: 'assets/js/[name]-[hash].js',
        entryFileNames: 'assets/js/[name]-[hash].js',
        assetFileNames: 'assets/[ext]/[name]-[hash].[ext]',
      },
    },
  },
  optimizeDeps: {
    include: ['react', 'react-dom', 'react-router-dom'],
  },
  esbuild: {
    drop: ['console', 'debugger'],
    pure: ['console.log'],
  },
});
```

#### 2. **Performance Monitoring Component**
```jsx
// PerformanceMonitor.jsx
import { useEffect } from 'react';
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

export function PerformanceMonitor() {
  useEffect(() => {
    const sendMetric = (metric) => {
      // Send to your analytics endpoint
      fetch('/api/performance', {
        method: 'POST',
        body: JSON.stringify({
          name: metric.name,
          value: Math.round(metric.value),
          rating: metric.rating,
          url: window.location.pathname,
          timestamp: Date.now(),
        }),
        headers: { 'Content-Type': 'application/json' },
        keepalive: true,
      });
    };

    const unsubLCP = onLCP(sendMetric);
    const unsubINP = onINP(sendMetric);
    const unsubCLS = onCLS(sendMetric);
    const unsubFCP = onFCP(sendMetric);
    const unsubTTFB = onTTFB(sendMetric);

    return () => {
      unsubLCP();
      unsubINP();
      unsubCLS();
      unsubFCP();
      unsubTTFB();
    };
  }, []);

  return null; // No UI
}

// Usage: <PerformanceMonitor /> at root of app
```

#### 3. **Optimized Font Loading**
```css
/* 1. Preload critical font */
<link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin>

/* 2. Font Face with optimal settings */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-weight: 100 900;
  font-stretch: 75% 125%;
  font-display: optional; /* Best for CLS */
  font-size-adjust: 0.51; /* Match x-height of fallback */
  ascent-override: 90%;
  descent-override: 25%;
  line-gap-override: 0%;
}

/* 3. Fallback Font Matching */
@font-face {
  font-family: 'Inter Fallback';
  src: local('Arial');
  size-adjust: 60.5%;
  ascent-override: 88%;
  descent-override: 22%;
  line-gap-override: 0%;
}

body {
  font-family: 'Inter', 'Inter Fallback', system-ui, sans-serif;
}

/* 4. Critical CSS Inline - font loading */
<link rel="preload" as="style" href="/fonts.css" onload="this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="/fonts.css"></noscript>
```

---

### PRACTICE PROBLEMS

1. **Audit** a website with Lighthouse - fix all Core Web Vitals issues
2. **Implement** code splitting, lazy loading, and bundle analysis in a React app
3. **Optimize** images: generate WebP/AVIF, responsive sizes, blur placeholders
4. **Configure** Service Worker with Workbox for offline-first caching
5. **Build** a performance dashboard showing RUM data over time

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Core Web Vitals | LCP (loading), INP (interactivity), CLS (stability) |
| LCP Good/Poor | ≤2.5s good, >4s poor |
| INP Good/Poor | ≤200ms good, >500ms poor |
| CLS Good/Poor | ≤0.1 good, >0.25 poor |
| CRP Blocking Resources | HTML, CSS (blocks render), JS (blocks parse) |
| font-display values | block, swap, fallback, optional (best for CLS) |
| Preload vs Prefetch | Preload: current page critical. Prefetch: future navigation |
| TBT vs TTI | TBT: blocking time sum. TTI: when page fully interactive |
| Service Worker Strategy | NetworkFirst (API), StaleWhileRevalidate (assets), CacheFirst (static) |
| LCP Candidate Elements | img, video poster, background-image, text blocks |

---

## NEXT TOPIC: `03-BACKEND-FUNDAMENTALS/01-HTTP-REST-GRPC.md`

> **Study Tip**: Run Lighthouse in CI on every PR. Set budgets. Fix regressions immediately. Use web-vitals library for RUM.