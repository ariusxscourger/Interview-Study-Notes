# CSS LAYOUT & RESPONSIVE DESIGN
## Exam-Style Study Notes

---

## TOPIC: Modern CSS Layout Systems & Responsive Design

### WHY? (Problem Statement)

**Before Modern CSS:**
- Floats for layout → Clearing hacks, brittle
- Fixed widths → Horizontal scroll on mobile
- Media queries only → No container queries
- No native variables → Preprocessors required
- Centering → `margin: auto` only works with width

**What Modern CSS Solves:**
| Problem | Solution |
|---------|----------|
| Complex layouts | Flexbox, Grid, Container Queries |
| Responsive without breakpoints | Fluid typography, `clamp()`, `min()/max()` |
| Theming | CSS Custom Properties (variables) |
| Component encapsulation | Shadow DOM, CSS Modules, Cascade Layers |
| Performance | `content-visibility`, `contain` |

**Real-World Analogy:**
- Floats = Building with bricks and mortar (messy, rigid)
- Flexbox = Magnetic strips (1D alignment)
- Grid = Graph paper (2D layout)
- Container Queries = Self-measuring components

---

### HOW? (Internal Mechanism)

#### 1. **Flexbox Algorithm (1D)**

```
Main Axis (flex-direction)
┌─────────────────────────────────────┐
│  Start → → → → → → → → → → → End   │
│  │  item  │  item  │  item  │      │
│  │ flex:1  │ flex:2 │ flex:1 │      │
│  └────────────────────────────────┘
│  Cross Axis (align-items)
│  stretch (default) / center / start / end
```

**Flex Item Sizing Algorithm:**
```
1. Base Size = width/height or content (max-content)
2. Flex Grow: Distribute positive free space proportionally
3. Flex Shrink: Distribute negative free space proportionally
4. Flex Basis: Initial size before grow/shrink
```

#### 2. **Grid Algorithm (2D)**

```
Explicit Grid (defined)          Implicit Grid (auto)
┌───┬───┬───┐                    ┌───┬───┬───┐
│   │   │   │  grid-template-    │   │   │   │  grid-auto-flow:
│   │   │   │  columns: 1fr 2fr  │   │   │   │  row (default)
│   │   │   │  grid-template-    │   │   │   │  grid-auto-rows:
│   │   │   │  rows: 100px 1fr   │   │   │   │  auto
└───┴───┴───┘                    └───┴───┴───┘
```

**Grid Placement:**
```css
.item {
  /* Line-based */
  grid-column: 1 / 3;        /* Lines 1 to 3 */
  grid-row: 2 / span 2;      /* Line 2, span 2 rows */
  
  /* Named lines */
  grid-column: sidebar-start / content-end;
  
  /* Named areas */
  grid-area: header;
}

/* Parent */
.grid {
  grid-template-areas: 
    "header header"
    "sidebar content"
    "footer footer";
}
```

#### 3. **Container Queries (The Game Changer)**

```
Traditional: Viewport-based (media queries)
  @media (min-width: 768px) { .card { ... } }

Modern: Container-based (container queries)
  .card-container { container-type: inline-size; }
  @container (min-width: 400px) { .card { ... } }

Benefit: Component adapts to ITS container, not viewport
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **CSS Custom Properties (Variables)**

```css
:root {
  /* Colors */
  --color-primary: #0066cc;
  --color-primary-hover: #0052a3;
  --color-background: #ffffff;
  --color-text: #1a1a1a;
  --color-text-muted: #595959;
  --color-border: #d9d9d9;
  --color-error: #cc0000;
  --color-success: #008000;
  
  /* Spacing Scale */
  --space-xs: 0.25rem;   /* 4px */
  --space-sm: 0.5rem;    /* 8px */
  --space-md: 1rem;      /* 16px */
  --space-lg: 1.5rem;    /* 24px */
  --space-xl: 2rem;      /* 32px */
  --space-2xl: 3rem;     /* 48px */
  
  /* Typography */
  --font-family-base: system-ui, -apple-system, sans-serif;
  --font-family-mono: ui-monospace, monospace;
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;
  --font-size-3xl: 2rem;
  --font-size-4xl: 3rem;
  --line-height-tight: 1.1;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;
  
  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
  
  /* Border Radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-full: 9999px;
  
  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-normal: 250ms ease;
  --transition-slow: 350ms ease;
  
  /* Z-Index Scale */
  --z-dropdown: 100;
  --z-sticky: 200;
  --z-fixed: 300;
  --z-modal-backdrop: 400;
  --z-modal: 500;
  --z-popover: 600;
  --z-tooltip: 700;
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --color-background: #0f0f0f;
    --color-text: #fafafa;
    --color-text-muted: #a3a3a3;
    --color-border: #333;
  }
}

/* Usage */
.card {
  background: var(--color-background);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  padding: var(--space-lg);
  box-shadow: var(--shadow-md);
  transition: box-shadow var(--transition-normal);
}

.card:hover {
  box-shadow: var(--shadow-lg);
}
```

#### 2. **Fluid Typography & Spacing**

```css
/* Clamp = fluid between min and max */
:root {
  --fs-fluid-sm: clamp(0.875rem, 0.875rem + 0vw, 0.875rem);
  --fs-fluid-base: clamp(1rem, 0.95rem + 0.25vw, 1.125rem);
  --fs-fluid-lg: clamp(1.125rem, 1rem + 0.625vw, 1.5rem);
  --fs-fluid-xl: clamp(1.25rem, 1.125rem + 1.25vw, 2rem);
  --fs-fluid-2xl: clamp(1.5rem, 1.25rem + 2.5vw, 3rem);
  --fs-fluid-3xl: clamp(2rem, 1.5rem + 5vw, 4rem);
  --fs-fluid-4xl: clamp(3rem, 2rem + 10vw, 6rem);
}

/* Fluid spacing */
:root {
  --space-fluid-xs: clamp(0.25rem, 0.25rem + 0vw, 0.5rem);
  --space-fluid-sm: clamp(0.5rem, 0.5rem + 0vw, 1rem);
  --space-fluid-md: clamp(1rem, 0.875rem + 0.625vw, 1.5rem);
  --space-fluid-lg: clamp(1.5rem, 1.25rem + 1.25vw, 3rem);
  --space-fluid-xl: clamp(2rem, 1.5rem + 2.5vw, 4rem);
}

/* Fluid container */
.container {
  width: min(100% - 2rem, 1200px);
  margin-inline: auto;
}
```

#### 3. **Modern Layout Patterns**

**Holy Grail Layout (Grid):**
```css
.holy-grail {
  display: grid;
  grid-template-columns: 1fr 3fr 1fr;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header header"
    "nav main aside"
    "footer footer footer";
  min-height: 100vh;
  gap: var(--space-md);
}

.header { grid-area: header; }
.nav { grid-area: nav; }
.main { grid-area: main; }
.aside { grid-area: aside; }
.footer { grid-area: footer; }

/* Responsive: Stack on mobile */
@media (max-width: 768px) {
  .holy-grail {
    grid-template-columns: 1fr;
    grid-template-areas:
      "header"
      "main"
      "nav"
      "aside"
      "footer";
  }
}
```

**Card Grid (Auto-fit):**
```css
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--space-lg);
}

/* Or with container queries */
.card-container {
  container-type: inline-size;
}

@container (min-width: 600px) {
  .card-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@container (min-width: 900px) {
  .card-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

**Sidebar Layout (Flexbox):**
```css
.sidebar-layout {
  display: flex;
  min-height: 100vh;
}

.sidebar {
  width: 280px;
  flex-shrink: 0;
  border-right: 1px solid var(--color-border);
}

.main {
  flex: 1;
  min-width: 0; /* Critical for text truncation */
  padding: var(--space-lg);
  overflow: auto;
}

@media (max-width: 768px) {
  .sidebar-layout { flex-direction: column; }
  .sidebar { width: 100%; border-right: none; border-bottom: 1px solid var(--color-border); }
}
```

#### 4. **Responsive Images**

```html
<!-- Art Direction (different crops) -->
<picture>
  <source media="(max-width: 480px)" srcset="hero-mobile.webp" type="image/webp">
  <source media="(max-width: 768px)" srcset="hero-tablet.webp" type="image/webp">
  <img src="hero-desktop.webp" alt="Hero" width="1200" height="600" loading="eager">
</picture>

<!-- Resolution Switching (same aspect ratio) -->
<img 
  src="image-400.webp"
  srcset="
    image-400.webp 400w,
    image-800.webp 800w,
    image-1200.webp 1200w,
    image-1600.webp 1600w
  "
  sizes="
    (max-width: 480px) 100vw,
    (max-width: 768px) 50vw,
    (max-width: 1200px) 33vw,
    25vw
  "
  alt="Description"
  width="400"
  height="300"
  loading="lazy"
  decoding="async"
>

<!-- CSS for aspect ratio -->
img {
  aspect-ratio: 4 / 3;
  object-fit: cover;
  width: 100%;
  height: auto;
}
```

#### 5. **Container Queries (Component-Level Responsive)**

```css
/* Parent establishes containment */
.card-wrapper {
  container-type: inline-size; /* or size for both dimensions */
  container-name: card;
}

/* Child queries its container */
@container card (min-width: 400px) {
  .card {
    display: flex;
    flex-direction: row;
    align-items: center;
  }
  
  .card__image {
    width: 120px;
    flex-shrink: 0;
  }
  
  .card__content {
    flex: 1;
  }
}

@container card (min-width: 600px) {
  .card__title { font-size: var(--fs-fluid-xl); }
}

/* Style queries (based on computed styles) */
@container style(--theme: dark) {
  .card { background: var(--color-surface-dark); }
}
```

#### 6. **Cascade Layers (Organization)**

```css
@layer reset, base, components, utilities, overrides;

/* Order matters - later layers win */
@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
  * { margin: 0; padding: 0; }
}

@layer base {
  body { font-family: var(--font-family-base); line-height: var(--line-height-normal); }
}

@layer components {
  .button { /* button styles */ }
  .card { /* card styles */ }
}

@layer utilities {
  .sr-only { /* screen reader only */ }
  .visually-hidden { /* same */ }
}

/* Import with layer */
@import url('tailwind.css') layer(utilities);
@import url('custom.css') layer(components);
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| `clamp()` for fluid type | Fixed px at every breakpoint | Maintenance nightmare |
| Container queries | Media queries for components | Components break in different contexts |
| `aspect-ratio` | Padding-top hack | Cleaner, works with grid/flex |
| `gap` in flex/grid | Margin on children | No double margins, cleaner |
| Logical properties | Physical (margin-left) | RTL support automatic |
| `minmax()` in grid | Fixed columns | True responsiveness |
| CSS Variables | Sass variables | Runtime updates, JS access |
| `:where()` for low specificity | `!important` | Maintainable specificity |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Flexbox vs Grid - when to use each?"

**Answer:**
```
FLEXBOX (1D) - One-dimensional layouts
✓ Navigation bars
✓ Form controls (label + input)
✓ Centering (vertical + horizontal)
✓ Card content (header, body, footer)
✓ Any single-row/column distribution

GRID (2D) - Two-dimensional layouts
✓ Page layout (header, sidebar, main, footer)
✓ Card grids (auto-fit columns)
✓ Complex forms (labels + inputs aligned)
✓ Image galleries
✓ Overlapping elements

RULE OF THUMB:
- "I need to align items in a row/column" → Flexbox
- "I need rows AND columns" → Grid
- Often: Grid for page, Flexbox for components
```

#### 2. **Code**: "Center a div both horizontally and vertically (3 ways)"

```css
/* 1. Flexbox (modern, easy) */
.parent { display: flex; justify-content: center; align-items: center; }

/* 2. Grid (modern, one-liner) */
.parent { display: grid; place-content: center; }

/* 3. Absolute + Transform (legacy support) */
.parent { position: relative; }
.child { 
  position: absolute; 
  top: 50%; left: 50%; 
  transform: translate(-50%, -50%);
}

/* 4. Margin Auto (Flex/Grid child) */
.parent { display: flex; } /* or grid */
.child { margin: auto; }
```

#### 3. **Design**: "Responsive card that stacks on mobile, side-by-side on desktop"

```css
.card {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--space-md);
  container-type: inline-size;
}

@container (min-width: 480px) {
  .card {
    grid-template-columns: 150px 1fr;
    align-items: center;
  }
}

@container (min-width: 720px) {
  .card {
    grid-template-columns: 200px 1fr;
  }
}

/* Image maintains aspect ratio */
.card__image {
  aspect-ratio: 4/3;
  object-fit: cover;
  width: 100%;
  border-radius: var(--radius-md);
}
```

#### 4. **Code**: "Implement a sticky footer (footer at bottom if content short)"

```css
/* Method 1: Flexbox on body */
body {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

main { flex: 1; }

/* Method 2: Grid */
body {
  min-height: 100vh;
  display: grid;
  grid-template-rows: auto 1fr auto;
}

/* Method 3: Legacy (calc) */
.content { min-height: calc(100vh - 200px); } /* header + footer height */
```

#### 5. **Performance**: "What is `content-visibility` and when to use it?"

```css
/* Skip rendering off-screen content */
.heavy-section {
  content-visibility: auto;
  contain-intrinsic-size: 1000px; /* Estimated size for scrollbar */
}

/* Values:
   visible (default) - normal rendering
   auto - browser skips layout/paint until near viewport
   hidden - always skipped (like display: none but keeps state)
*/

/* Use cases:
   - Long lists (virtualize instead if possible)
   - Below-fold sections
   - Heavy components (charts, maps)
*/
```

#### 6. **CSS Logic**: "How does `clamp(min, preferred, max)` work?"

```css
/* clamp(MIN, PREFERRED, MAX) */
/* Returns: max(MIN, min(PREFERRED, MAX)) */

.font-size {
  /* Minimum 1rem, preferred 2.5vw, maximum 2rem */
  font-size: clamp(1rem, 2.5vw, 2rem);
  
  /* Breakdown:
     - Viewport < 40vw: uses 1rem (min)
     - Viewport 40-80vw: uses 2.5vw (fluid)
     - Viewport > 80vw: uses 2rem (max)
  */
}

/* Practical fluid spacing */
.section {
  padding-block: clamp(2rem, 5vw, 6rem);
}

/* Fluid container width */
.container {
  width: min(90%, 1200px); /* Same as: width: 90%; max-width: 1200px; */
}
```

#### 7. **Modern CSS**: "Logical Properties - why and how?"

```css
/* Physical (breaks in RTL) */
.box { margin-left: 1rem; padding-right: 2rem; border-top: 1px solid; }

/* Logical (works in RTL/LTR automatically) */
.box { 
  margin-inline-start: 1rem; 
  padding-inline-end: 2rem; 
  border-block-start: 1px solid; 
}

/* Mapping:
   margin-left → margin-inline-start
   margin-right → margin-inline-end
   margin-top → margin-block-start
   margin-bottom → margin-block-end
   
   padding-left → padding-inline-start
   border-top → border-block-start
   border-radius-top-left → border-start-start-radius
   
   left → inset-inline-start
   top → inset-block-start
   width → inline-size
   height → block-size
*/

/* Writing M
   max-width → max-inline-size
*/
```

#### 8. **Architecture**: "Organize CSS for a design system"

```css
/* 1. Design Tokens (CSS Variables) */
:root {
  --color-brand-50: #eff6ff;
  --color-brand-500: #3b82f6;
  --color-brand-900: #1e3a8a;
  --space-1: 0.25rem;
  --space-4: 1rem;
  --radius-md: 0.375rem;
}

/* 2. Reset/Normalize */
@layer reset { /* ... */ }

/* 3. Base Styles */
@layer base { /* typography, links, forms */ }

/* 4. Layout Primitives */
@layer layout {
  .stack { display: flex; flex-direction: column; gap: var(--space-4); }
  .inline { display: flex; gap: var(--space-4); align-items: center; }
  .grid { display: grid; gap: var(--space-4); }
  .container { width: min(100% - 2rem, 1200px); margin-inline: auto; }
}

/* 5. Components */
@layer components {
  .btn { /* ... */ }
  .card { /* ... */ }
  .input { /* ... */ }
}

/* 6. Utilities (highest specificity) */
@layer utilities {
  .visually-hidden { /* ... */ }
  .text-center { text-align: center; }
  .d-none { display: none; }
}

/* 7. Themes (override tokens) */
@media (prefers-color-scheme: dark) {
  :root { --color-bg: #000; --color-text: #fff; }
}

[data-theme="dark"] { /* ... */ }
```

#### 9. **Debugging**: "Element has unexpected spacing - debug steps"

```
1. DevTools → Elements → Computed → Box Model
   - Shows margin, border, padding, content

2. Check:
   ☐ Margin collapsing (adjacent vertical margins)
   ☐ Default browser styles (user-agent stylesheet)
   ☐ Flex/Grid gap vs margin
   ☐ line-height creating space
   ☐ Font metrics (ascender/descender)
   ☐ Inline element whitespace
   ☐ box-sizing: border-box?

3. Fixes:
   - Reset: * { box-sizing: border-box; margin: 0; }
   - Gap instead of margin in flex/grid
   - font-size: 0 on parent for inline-block whitespace
   - line-height: 1 on icon containers
```

#### 10. **Specificity**: "Cascade Layers vs `!important` vs Specificity"

```css
/* Specificity: (inline, ID, class, element) */
#id .class { color: red; }        /* 0,1,1,0 = 110 */
.class { color: blue; }           /* 0,0,1,0 = 10 */

/* !important beats specificity */
.class { color: green !important; }  /* Wins */

/* Cascade Layers: Order of layers beats specificity */
@layer base, components, utilities;

@layer base { .btn { color: red; } }
@layer components { .btn { color: blue; } }  /* Wins! */

/* Layer specificity:
   1. Layer order (last wins)
   2. Specificity within layer
   3. !important within layer (reversed!)
   
   !important in base layer > !important in components layer
*/
```

---

### GOTCHAS & EDGE CASES

- [ ] **Flexbox gap**: Not supported in Safari < 14.1 (use margin hack or `@supports`)
- [ ] **Grid subgrid**: `grid-template-rows: subgrid` - not in all browsers
- [ ] **Container queries**: Need `container-type` on parent, not `:root`
- [ ] **Aspect ratio**: `aspect-ratio` on replaced elements (img, video) uses intrinsic ratio
- [ ] **Clamp units**: Preferred value must use viewport units (vw, vh, vmin, vmax)
- [ ] **Custom properties**: Not interpolated in `calc()` without `@property`
- [ ] **Cascade layers**: Unlayered styles = highest layer (after all `@layer`)
- [ ] **Focus-visible**: Polyfill needed for older browsers
- [ ] **Subpixel rendering**: `translateZ(0)` or `will-change: transform` for smooth animation
- [ ] **Scrollbar gutters**: `scrollbar-gutter: stable` prevents layout shift

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Reset + Base**
```css
*, *::before, *::after { box-sizing: border-box; }
* { margin: 0; padding: 0; }

html { 
  font-size: 100%; /* 16px default */
  -webkit-text-size-adjust: 100%;
  scroll-behavior: smooth;
}

body {
  min-height: 100vh;
  font-family: var(--font-family-base);
  line-height: var(--line-height-normal);
  color: var(--color-text);
  background: var(--color-background);
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
}

img, picture, video, canvas, svg {
  display: block;
  max-width: 100%;
  height: auto;
}

input, button, textarea, select { font: inherit; }

a { color: inherit; text-decoration: none; }
```

#### 2. **Utility Classes**
```css
/* Spacing */
.m-0 { margin: 0; } .m-auto { margin: auto; }
.p-0 { padding: 0; }

/* Flexbox */
.flex { display: flex; }
.flex-col { flex-direction: column; }
.items-center { align-items: center; }
.justify-center { justify-content: center; }
.justify-between { justify-content: space-between; }
.gap-4 { gap: 1rem; }
.flex-1 { flex: 1 1 0%; }

/* Grid */
.grid { display: grid; }
.grid-cols-2 { grid-template-columns: repeat(2, 1fr); }
.grid-cols-3 { grid-template-columns: repeat(3, 1fr); }

/* Display */
.block { display: block; }
.inline-flex { display: inline-flex; }
.hidden { display: none; }

/* Sizing */
.w-full { width: 100%; }
.h-full { height: 100%; }
.min-h-screen { min-height: 100vh; }

/* Typography */
.text-center { text-align: center; }
.font-bold { font-weight: 700; }
.text-sm { font-size: 0.875rem; }
.text-lg { font-size: 1.125rem; }

/* Colors */
.bg-primary { background-color: var(--color-primary); }
.text-white { color: white; }

/* Borders */
.rounded { border-radius: var(--radius-md); }
.rounded-full { border-radius: var(--radius-full); }
.border { border: 1px solid var(--color-border); }

/* Shadows */
.shadow { box-shadow: var(--shadow-md); }
.shadow-lg { box-shadow: var(--shadow-lg); }

/* Transitions */
.transition { transition: all var(--transition-normal); }
```

#### 3. **Media Query Breakpoints (Mobile-First)**
```css
/* Mobile First - Base styles for < 640px */

/* sm: 640px */
@media (min-width: 640px) { }

/* md: 768px */
@media (min-width: 768px) { }

/* lg: 1024px */
@media (min-width: 1024px) { }

/* xl: 1280px */
@media (min-width: 1280px) { }

/* 2xl: 1536px */
@media (min-width: 1536px) { }

/* Container Queries (Component-level) */
.card-container { container-type: inline-size; }

@container (min-width: 320px) { }
@container (min-width: 480px) { }
@container (min-width: 640px) { }
```

---

### PRACTICE PROBLEMS

1. **Build**: Responsive navbar (hamburger menu on mobile, horizontal on desktop)
2. **Implement**: Masonry layout with Grid (auto-flow: dense)
3. **Create**: Fluid type scale using `clamp()` for all headings
4. **Debug**: Fix layout shift caused by web font loading
5. **Optimize**: Add `content-visibility: auto` to long article page

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Flexbox vs Grid | Flexbox: 1D (row OR column). Grid: 2D (rows AND columns) |
| clamp() syntax | clamp(min, preferred, max) - fluid between min/max |
| Container query setup | Parent: container-type: inline-size. Child: @container (min-width: 400px) |
| Logical property for margin-left | margin-inline-start |
| gap in flexbox | Supported in all modern browsers (Safari 14.1+) |
| Aspect ratio on img | Uses intrinsic ratio; on div creates box |
| content-visibility: auto | Skips render until near viewport; needs contain-intrinsic-size |
| Cascade layer order | Last declared layer wins (regardless of specificity) |
| Mobile-first media query | @media (min-width: 768px) - styles for 768px AND UP |
| Fluid container width | width: min(90%, 1200px) |

---

## NEXT TOPIC: `04-JAVASCRIPT-FUNDAMENTALS.md`

> **Study Tip**: Build a component library with: Button, Card, Input, Modal. Use CSS Variables, Container Queries, Cascade Layers. Test in RTL mode.