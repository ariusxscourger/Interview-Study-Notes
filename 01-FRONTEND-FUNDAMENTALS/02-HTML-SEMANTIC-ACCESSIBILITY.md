# HTML SEMANTICS & ACCESSIBILITY
## Exam-Style Study Notes

---

## TOPIC: Semantic HTML & Web Accessibility (a11y)

### WHY? (Problem Statement)

**Before Semantic HTML:**
- `<div class="header">`, `<div class="nav">`, `<div class="article">`
- Screen readers see generic containers
- Search engines can't understand content structure
- Developers can't maintain consistent patterns
- No keyboard navigation by default

**What Semantic HTML + a11y Solves:**
| Problem | Solution |
|---------|----------|
| Screen reader navigation | Landmarks, headings, roles |
| SEO understanding | Semantic elements, microdata |
| Keyboard users | Focus management, native controls |
| Maintainability | Standard vocabulary |
| Legal compliance | WCAG 2.1 AA (law in many countries) |

**Real-World Analogy:**
- Non-semantic = Building with unmarked boxes
- Semantic = Building with labeled rooms (kitchen, bedroom, bathroom)
- Accessibility = Ramps, elevators, braille signs - usable by everyone

---

### HOW? (Internal Mechanism)

#### 1. **Accessibility Tree (Parallel to DOM)**

```
DOM TREE                          ACCESSIBILITY TREE
┌─────────────────────┐           ┌─────────────────────┐
│ <div class="btn">   │           │ (missing - no role) │
│ <button>Click</button>         │ button "Click"      │
│ <img src="logo.png">           │ image "Company logo"│
│ <h1>Title</h1>                  │ heading level 1     │
└─────────────────────┘           └─────────────────────┘

Browser builds both trees. AT (Assistive Tech) uses Accessibility Tree.
```

#### 2. **ARIA (Accessible Rich Internet Applications)**

```
ARIA fills gaps when native HTML can't express the semantics.

Roles:          States:              Properties:
──────────────────────────────────────────────────────────────
button          aria-disabled        aria-label
dialog          aria-expanded        aria-labelledby
tablist         aria-selected        aria-describedby
menu            aria-hidden          aria-controls
alert           aria-checked         aria-live
status          aria-pressed         aria-atomic
region          aria-busy            aria-relevant
```

**First Rule of ARIA:** Don't use ARIA if native HTML works.

---

### WHAT? (Key Concepts & APIs)

#### 1. **Semantic HTML Elements**

**Document Structure:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Page Title - Site Name</title>
</head>
<body>
  <a href="#main" class="skip-link">Skip to main content</a>
  
  <header>
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/products">Products</a></li>
      </ul>
    </nav>
  </header>
  
  <main id="main">
    <article>
      <header>
        <h1>Article Title</h1>
        <time datetime="2024-01-15">January 15, 2024</time>
      </header>
      <section>
        <h2>Section Heading</h2>
        <p>Content...</p>
      </section>
      <aside>
        <h3>Related</h3>
      </aside>
    </article>
  </main>
  
  <footer>
    <address>Contact: <a href="mailto:info@nanolite.tech">info@nanolite.tech</a></address>
    <small>&copy; 2024 Nanolite Tech</small>
  </footer>
</body>
</html>
```

**Interactive Elements:**
```html
<!-- Native button - preferred -->
<button type="button">Submit</button>

<!-- Link for navigation -->
<a href="/login">Login</a>

<!-- Form with proper labels -->
<form>
  <div class="form-group">
    <label for="email">Email Address</label>
    <input type="email" id="email" name="email" required 
           autocomplete="email" aria-describedby="email-hint">
    <span id="email-hint">We'll never share your email</span>
  </div>
  
  <fieldset>
    <legend>Account Type</legend>
    <label><input type="radio" name="type" value="personal"> Personal</label>
    <label><input type="radio" name="type" value="business"> Business</label>
  </fieldset>
  
  <button type="submit">Create Account</button>
</form>
```

**Media & Tables:**
```html
<!-- Images -->
<img src="chart.png" alt="Bar chart showing 40% growth in Q4">
<img src="decoration.png" alt=""> <!-- Empty alt = decorative -->
<picture>
  <source srcset="chart.avif" type="image/avif">
  <source srcset="chart.webp" type="image/webp">
  <img src="chart.png" alt="Revenue growth chart">
</picture>

<!-- Tables -->
<table>
  <caption>Quarterly Revenue 2024</caption>
  <thead>
    <tr>
      <th scope="col">Quarter</th>
      <th scope="col">Revenue</th>
      <th scope="col">Growth</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">Q1</th>
      <td>$1.2M</td>
      <td>+5%</td>
    </tr>
  </tbody>
</table>
```

#### 2. **ARIA Patterns (When Native HTML Isn't Enough)**

**Custom Select (Combobox):**
```html
<div class="combobox" role="combobox" aria-expanded="false" aria-owns="listbox" aria-haspopup="listbox">
  <label for="combobox-input">Choose option</label>
  <input type="text" 
         id="combobox-input" 
         role="textbox" 
         aria-autocomplete="list" 
         aria-controls="listbox"
         autocomplete="off">
  <svg aria-hidden="true"><use href="#chevron-down"></use></svg>
  
  <ul id="listbox" role="listbox" hidden>
    <li role="option" aria-selected="true">Option 1</li>
    <li role="option">Option 2</li>
    <li role="option">Option 3</li>
  </ul>
</div>
```

**Modal Dialog:**
```html
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title" aria-describedby="dialog-desc">
  <h2 id="dialog-title">Confirm Delete</h2>
  <p id="dialog-desc">This action cannot be undone.</p>
  <button>Cancel</button>
  <button>Delete</button>
</div>

/* CSS for focus trap */
[role="dialog"]:focus-within {
  /* Focus stays inside */
}
```

**Live Regions (Announcements):**
```html
<!-- Polite - waits for pause -->
<div aria-live="polite" aria-atomic="true" class="sr-only">
  Item added to cart
</div>

<!-- Assertive - interrupts -->
<div aria-live="assertive" aria-atomic="true" class="sr-only" role="alert">
  Error: Payment failed
</div>

<!-- Status - implicit polite -->
<div role="status" aria-live="polite" class="sr-only">
  Loading...
</div>
```

#### 3. **Keyboard Navigation**

```html
<!-- Focus visible -->
:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}

<!-- Skip links -->
<a href="#main" class="skip-link">Skip to main content</a>

<style>
.skip-link {
  position: absolute;
  top: -100%;
  left: 50%;
  transform: translateX(-50%);
  padding: 1rem;
  background: #000;
  color: #fff;
  z-index: 10000;
}
.skip-link:focus {
  top: 1rem;
}
</style>

<!-- Tab order -->
<div tabindex="0">Focusable div (avoid if possible)</div>
<button tabindex="-1">Removed from tab order</button>

<!-- Roving tabindex (for widgets) -->
<ul role="tablist">
  <li role="tab" aria-selected="true" tabindex="0">Tab 1</li>
  <li role="tab" aria-selected="false" tabindex="-1">Tab 2</li>
  <li role="tab" aria-selected="false" tabindex="-1">Tab 3</li>
</ul>
```

#### 4. **Color & Contrast**

```css
/* WCAG AA: 4.5:1 normal text, 3:1 large text (18pt/14pt bold) */
/* WCAG AAA: 7:1 normal, 4.5:1 large */

:root {
  --color-primary: #0066cc;     /* 4.5:1 on white */
  --color-primary-dark: #004499; /* 7:1 on white */
  --color-text: #1a1a1a;        /* 12:1 on white */
  --color-text-muted: #595959;  /* 4.5:1 on white */
  --color-background: #ffffff;
  --color-error: #cc0000;       /* 4.5:1 on white */
}

/* Don't rely on color alone */
.error {
  color: var(--color-error);
}
.error::before {
  content: "⚠ "; /* Icon for non-color indicator */
}

/* High contrast mode */
@media (prefers-contrast: high) {
  :root {
    --color-primary: #0000ee;
    --color-text: #000;
  }
}

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

#### 5. **Forms & Validation**

```html
<form novalidate>
  <div class="field">
    <label for="password">Password</label>
    <div class="password-wrapper">
      <input type="password" 
             id="password" 
             name="password"
             required
             minlength="8"
             aria-describedby="password-requirements password-error"
             aria-invalid="false">
      <button type="button" aria-label="Toggle password visibility">
        <svg aria-hidden="true"><use href="#eye"></use></svg>
      </button>
    </div>
    <div id="password-requirements" class="hint">
      At least 8 characters, one uppercase, one number
    </div>
    <div id="password-error" class="error" role="alert" hidden>
      Password must be at least 8 characters
    </div>
  </div>
  
  <button type="submit">Register</button>
</form>

<script>
// Client-side validation with ARIA
const form = document.querySelector('form');
form.addEventListener('submit', async (e) => {
  e.preventDefault();
  
  const password = form.password;
  const errorEl = document.getElementById('password-error');
  
  if (password.value.length < 8) {
    password.setAttribute('aria-invalid', 'true');
    errorEl.hidden = false;
    password.focus();
    return;
  }
  
  password.setAttribute('aria-invalid', 'false');
  errorEl.hidden = true;
  
  // Submit...
});
</script>
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| `<button>` for actions | `<div onclick>` | No keyboard, no screen reader support |
| `<label for="id">` | `<div>Name</div><input>` | Clickable area, screen reader association |
| `<h1>`-`<h6>` in order | `<h3>` then `<h1>` | Broken heading hierarchy |
| `alt=""` for decorative | No alt attribute | Screen reader reads filename |
| `aria-live` for updates | Visual-only changes | Screen reader misses updates |
| Focus visible outline | `outline: none` | Keyboard users lost |
| Semantic landmarks | All `<div>` | No navigation structure |
| Native `<select>` | Custom dropdown (unless needed) | Complex ARIA, keyboard support |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Why does semantic HTML matter for accessibility?"

**Answer:**
```
1. SCREEN READERS: Build accessibility tree from semantics
   - <nav> → landmark "navigation"
   - <h1> → heading level 1
   - <button> → role="button", keyboard accessible

2. KEYBOARD NAVIGATION: Native elements work without JS
   - <a href> → Enter to follow
   - <button> → Space/Enter to activate
   - <input> → Arrow keys, typing

3. SEO: Search engines understand content structure
   - <article>, <section>, <header>, <footer>
   - Heading hierarchy = content outline

4. MAINTAINABILITY: Standard vocabulary
   - New devs understand intent
   - Consistent patterns

5. LEGAL: WCAG 2.1 AA compliance (ADA, EAA, AODA)
```

#### 2. **Code**: "Make this accessible: `<div class="tab" onclick="openTab(1)">Tab 1</div>`"

```html
<!-- BEFORE (Inaccessible) -->
<div class="tab" onclick="openTab(1)">Tab 1</div>

<!-- AFTER (Accessible) -->
<div role="tablist" aria-label="Product details">
  <button role="tab" 
          id="tab-1" 
          aria-selected="true" 
          aria-controls="panel-1" 
          tabindex="0">
    Tab 1
  </button>
  <button role="tab" 
          id="tab-2" 
          aria-selected="false" 
          aria-controls="panel-2" 
          tabindex="-1">
    Tab 2
  </button>
</div>

<div role="tabpanel" 
     id="panel-1" 
     aria-labelledby="tab-1" 
     hidden="">
  Content for tab 1
</div>
<div role="tabpanel" 
     id="panel-2" 
     aria-labelledby="tab-2" 
     hidden="">
  Content for tab 2
</div>

<script>
// Keyboard: Arrow keys switch tabs
// Enter/Space activates
// Home/End for first/last
</script>
```

#### 3. **Design**: "How to handle focus management in a modal?"

```javascript
function openModal(modal) {
  // 1. Save previous focus
  const previousFocus = document.activeElement;
  
  // 2. Show modal
  modal.hidden = false;
  modal.setAttribute('aria-modal', 'true');
  
  // 3. Trap focus
  const focusableElements = modal.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const firstElement = focusableElements[0];
  const lastElement = focusableElements[focusableElements.length - 1];
  
  function handleTab(e) {
    if (e.key !== 'Tab') return;
    
    if (e.shiftKey) {
      if (document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      }
    } else {
      if (document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    }
  }
  
  modal.addEventListener('keydown', handleTab);
  firstElement.focus();
  
  // 4. Return focus on close
  function closeModal() {
    modal.hidden = true;
    modal.removeEventListener('keydown', handleTab);
    previousFocus?.focus();
  }
  
  return { closeModal };
}
```

#### 4. **Testing**: "How to test accessibility?"

```bash
# Automated (catches ~30-50%)
# axe-core
npx @axe-core/cli https://example.com

# Lighthouse
npx lighthouse https://example.com --only-categories=accessibility

# eslint-plugin-jsx-a11y (React)
# Stylelint a11y (CSS)

# Manual (catches rest)
# 1. Keyboard only: Tab, Shift+Tab, Enter, Space, Arrows, Escape
# 2. Screen reader: NVDA (Windows), VoiceOver (Mac), Orca (Linux)
# 3. Zoom: 200% browser zoom
# 4. High contrast mode
# 5. Reduced motion
# 6. Color blindness simulator

# Checklist:
# - All interactive elements reachable?
# - Focus visible?
# - Reading order logical?
# - Form labels associated?
# - Error messages announced?
# - Images have alt?
# - Heading hierarchy correct?
# - Landmarks present?
# - Language declared?
```

#### 5. **ARIA**: "When to use `role="button"` vs native `<button>`?"

**Answer:**
```
NEVER use role="button" on non-button elements if you can avoid it.

Native <button> gives you FOR FREE:
✓ Keyboard focus (Tab)
✓ Activation (Enter, Space)
✓ Form submission (type="submit")
✓ Disabled state (disabled attr)
✓ Form validation
✓ Implicit role="button"
✓ No ARIA needed

Only use role="button" when:
- Wrapping complex interactive component
- Legacy code migration (temporary)
- Truly cannot use native element

If you MUST use role="button":
<div role="button" tabindex="0" aria-pressed="false" 
     onkeydown="handleKeyDown(event)" 
     onclick="handleClick()">
</div>

// You MUST implement:
function handleKeyDown(e) {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault(); // Prevent scroll on Space
    handleClick();
  }
}
```

#### 6. **Images**: "Alt text decision tree"

```
Is the image purely decorative?
  YES → alt="" (empty string, not missing!)
  NO  → Does it convey information?
    YES → Is it a link/button?
      YES → alt = destination/action ("Search", "Home")
      NO  → alt = concise description of information
    NO  → Is it text as image?
      YES → alt = same text
      NO  → alt = concise description

Examples:
✓ <img src="logo.png" alt="Nanolite Tech"> (linked logo)
✓ <img src="chart.png" alt="Bar chart: Q1 1.2M, Q2 1.5M, Q3 1.4M, Q4 1.8M">
✓ <img src="spacer.gif" alt=""> (decorative)
✓ <img src="icon.png" alt=""> (icon with text label next to it)
✗ <img src="photo.jpg"> (missing alt)
✗ <img src="chart.png" alt="chart"> (too vague)
```

#### 7. **Forms**: "Accessible form validation pattern"

```html
<form novalidate>
  <div class="field">
    <label for="email">Email Address <span aria-hidden="true">*</span></label>
    <input type="email" 
           id="email" 
           name="email" 
           required
           autocomplete="email"
           aria-describedby="email-hint email-error"
           aria-invalid="false">
    <span id="email-hint" class="hint">We'll send a verification link</span>
    <span id="email-error" class="error" role="alert" hidden></span>
  </div>
  
  <button type="submit">Submit</button>
</form>

<script>
// On submit
form.addEventListener('submit', (e) => {
  const email = form.email;
  const error = document.getElementById('email-error');
  
  if (!email.validity.valid) {
    e.preventDefault();
    email.setAttribute('aria-invalid', 'true');
    error.hidden = false;
    error.textContent = email.validity.valueMissing 
      ? 'Email is required' 
      : 'Invalid email format';
    email.focus();
  } else {
    email.setAttribute('aria-invalid', 'false');
    error.hidden = true;
  }
});

// On input (clear error)
email.addEventListener('input', () => {
  if (email.getAttribute('aria-invalid') === 'true') {
    email.setAttribute('aria-invalid', 'false');
    error.hidden = true;
  }
});
</script>
```

#### 8. **WCAG**: "Four principles of WCAG (POUR)"

```
PERCEIVABLE:
- Text alternatives for non-text content
- Captions for multimedia
- Adaptable content (structure separable from presentation)
- Distinguishable (contrast, color, audio control)

OPERABLE:
- Keyboard accessible
- Enough time (no timeouts, or extendable)
- No seizures (flash threshold)
- Navigable (headings, landmarks, focus order)
- Input modalities (touch, voice, gesture)

UNDERSTANDABLE:
- Readable (language, unusual words, abbreviations)
- Predictable (consistent nav, consistent identification)
- Input assistance (labels, instructions, error prevention)

ROBUST:
- Compatible (valid HTML, ARIA)
- Parsable (complete tags, unique IDs)
```

---

### GOTCHAS & EDGE CASES

- [ ] **`alt=""` vs missing alt**: Empty = decorative, Missing = screen reader reads filename
- [ ] **`aria-hidden="true"`**: Removes from accessibility tree, but NOT from DOM
- [ ] **`role="presentation"`**: Removes semantics, keeps children
- [ ] **`<label>` wrapping vs `for`**: Both work, wrapping = no ID needed
- [ ] **`placeholder` ≠ label**: Placeholder disappears, low contrast, not accessible
- [ ] **`title` attribute**: Not reliable for accessibility (touch devices, keyboard)
- [ ] **Focus order**: DOM order = tab order (don't use tabindex > 0)
- [ ] **Modal focus trap**: Must trap focus, restore on close
- [ ] **Skip links**: Must be first focusable element
- [ ] **Language**: `<html lang="en">` + `<html lang="ar" dir="rtl">` for RTL

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Skip Link**
```html
<a href="#main" class="skip-link">Skip to main content</a>
<main id="main">...</main>

<style>
.skip-link {
  position: absolute;
  top: -100%;
  left: 50%;
  transform: translateX(-50%);
  padding: 1rem 2rem;
  background: #000;
  color: #fff;
  z-index: 10000;
  text-decoration: none;
}
.skip-link:focus { top: 1rem; }
</style>
```

#### 2. **Visually Hidden (Screen Reader Only)**
```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

#### 3. **Focus Visible (Modern)**
```css
:focus:not(:focus-visible) { outline: none; }
:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}

/* Fallback for older browsers */
.js-focus-visible :focus:not(.focus-visible) { outline: none; }
```

#### 4. **Landmark Structure**
```html
<body>
  <header role="banner">...</header>
  <nav role="navigation" aria-label="Main">...</nav>
  <main role="main" id="main">...</main>
  <aside role="complementary">...</aside>
  <footer role="contentinfo">...</footer>
</body>
```

#### 5. **Accessible Autocomplete**
```html
<label for="search">Search</label>
<input type="search" 
       id="search" 
       role="combobox" 
       aria-autocomplete="list" 
       aria-controls="search-results"
       aria-expanded="false">
<ul id="search-results" role="listbox" hidden>
  <li role="option" aria-selected="true">Result 1</li>
  <li role="option">Result 2</li>
</ul>
```

---

### PRACTICE PROBLEMS

1. **Audit**: Run axe-core on a page, fix top 5 issues
2. **Build**: Accessible modal dialog with focus trap, ESC to close
3. **Fix**: Convert `<div>`-based tab component to semantic + ARIA
4. **Test**: Navigate a page using only keyboard, document issues
5. **Write**: Alt text for 10 different image types

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Semantic HTML benefits | Screen readers, SEO, keyboard nav, maintainability, legal |
| First rule of ARIA | Don't use ARIA if native HTML works |
| alt="" vs missing alt | Empty = decorative (ignored). Missing = screen reader reads filename |
| Focus visible CSS | `:focus-visible { outline: 3px solid blue; }` |
| Skip link purpose | Keyboard users bypass navigation to main content |
| Modal focus trap | Save previous focus → trap in modal → restore on close |
| Heading hierarchy | h1 → h2 → h3 (never skip levels) |
| Landmark roles | banner, navigation, main, complementary, contentinfo |
| Live regions | polite (wait), assertive (interrupt), status (implicit polite) |
| Color contrast AA | 4.5:1 normal text, 3:1 large text (18pt/14pt bold) |

---

## NEXT TOPIC: `03-CSS-LAYOUT-RESPONSIVE.md`

> **Study Tip**: Disable CSS in browser, navigate page - does structure make sense? Then use only keyboard for 10 minutes.