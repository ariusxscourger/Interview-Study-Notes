# WEB SECURITY FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: Web Application Security

### WHY? (Problem Statement)

**The Reality:**
- 30,000+ websites hacked daily
- Average breach cost: $4.45M (2023)
- OWASP Top 10 unchanged for years
- Developers = first line of defense

**What Security Solves:**
| Threat | Impact | Prevention |
|--------|--------|------------|
| XSS | Session hijack, defacement | CSP, Output encoding |
| CSRF | Unauthorized actions | CSRF tokens, SameSite |
| SQL Injection | Data theft, deletion | Parameterized queries |
| Auth bypass | Account takeover | Proper auth, MFA |
| Data exposure | PII leakage | Encryption, minimal data |

**Real-World Analogy:**
- Web app = House with many doors/windows
- Security = Locks, alarms, cameras, visitor verification
- Vulnerability = Unlocked window on 2nd floor

---

### HOW? (Attack Mechanisms)

#### 1. **XSS (Cross-Site Scripting)**

```
ATTACK FLOW:
1. Attacker injects malicious script via input
2. App stores/reflects without sanitization
3. Victim visits page → Browser executes script
4. Script steals cookies, keylogs, redirects

TYPES:
- Reflected: URL parameter → immediate execution
- Stored: Database → executes on every view
- DOM-based: Client-side JS sinks (innerHTML, eval)
```

#### 2. **CSRF (Cross-Site Request Forgery)**

```
ATTACK FLOW:
1. Victim logged into bank.com (has session cookie)
2. Attacker sends link: <img src="bank.com/transfer?to=attacker&amount=1000">
3. Victim clicks → Browser sends request WITH cookie
4. Bank processes transfer (no CSRF token check)

WHY IT WORKS: Browsers automatically send cookies with requests
```

#### 3. **SQL Injection**

```
VULNERABLE:
query = "SELECT * FROM users WHERE email = '" + email + "'";

ATTACK INPUT: ' OR '1'='1' --
RESULT: SELECT * FROM users WHERE email = '' OR '1'='1' --'

ATTACK TYPES:
- Union-based: Extract data from other tables
- Blind: Boolean/time-based inference
- Second-order: Stored payload triggers later
```

---

### WHAT? (Defense Implementation)

#### 1. **Content Security Policy (CSP)**

```html
<!-- Strict CSP Header -->
Content-Security-Policy: 
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM}' 'strict-dynamic';
  style-src 'self' 'nonce-{RANDOM}';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  object-src 'none';
  upgrade-insecure-requests;
```

```typescript
// Generate nonce per request
app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  next();
});

// In templates
<script nonce="{nonce}">...</script>
<style nonce="{nonce}">...</style>
```

#### 2. **Input Validation & Output Encoding**

```typescript
// Validation (Server-side ALWAYS)
import { z } from 'zod';

const createUserSchema = z.object({
  name: z.string().min(1).max(100).regex(/^[a-zA-Z\s'-]+$/),
  email: z.string().email().max(255),
  age: z.number().int().min(13).max(120),
  bio: z.string().max(500).optional(),
});

// Output Encoding (Context-aware)
// HTML Context
function escapeHtml(str: string): string {
  return str
    .replace(/&/g, '&')
    .replace(/</g, '<')
    .replace(/>/g, '>')
    .replace(/"/g, '"')
    .replace(/'/g, ''');
}

// HTML Attribute Context
function escapeAttr(str: string): string {
  return escapeHtml(str).replace(/`/g, '&#x60;');
}

// JavaScript Context
function escapeJs(str: string): string {
  return str
    .replace(/\\/g, '\\\\')
    .replace(/'/g, "\\'")
    .replace(/"/g, '\\"')
    .replace(/\n/g, '\\n')
    .replace(/\r/g, '\\r')
    .replace(/</g, '\\u003c')
    .replace(/>/g, '\\u003e');
}

// URL Context
function escapeUrl(str: string): string {
  return encodeURIComponent(str);
}

// CSS Context
function escapeCss(str: string): string {
  return str.replace(/[^a-zA-Z0-9]/g, c => `\\${c.charCodeAt(0).toString(16)} `);
}
```

#### 3. **CSRF Protection**

```typescript
// Double Submit Cookie Pattern
app.use(csrf({ cookie: true }));

// Or manual implementation
function csrfProtection(req, res, next) {
  const token = req.cookies.csrf_token || crypto.randomBytes(32).toString('hex');
  
  // Set cookie (HttpOnly, Secure, SameSite)
  res.cookie('csrf_token', token, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000
  });
  
  // For forms: add hidden field
  res.locals.csrfToken = token;
  
  // For AJAX: custom header
  if (req.method !== 'GET' && req.method !== 'HEAD') {
    const headerToken = req.headers['x-csrf-token'];
    if (headerToken !== token) {
      return res.status(403).json({ error: 'Invalid CSRF token' });
    }
  }
  next();
}

// Frontend: Auto-attach token
axios.interceptors.request.use(config => {
  const token = document.cookie.match(/csrf_token=([^;]+)/)?.[1];
  if (token && ['post', 'put', 'patch', 'delete'].includes(config.method)) {
    config.headers['X-CSRF-Token'] = token;
  }
  return config;
});
```

#### 4. **SQL Injection Prevention**

```typescript
// NEVER: String concatenation
// const query = `SELECT * FROM users WHERE email = '${email}'`;

// ALWAYS: Parameterized queries
// PostgreSQL (pg)
const result = await pool.query('SELECT * FROM users WHERE email = $1', [email]);

// MySQL (mysql2)
const [rows] = await pool.execute('SELECT * FROM users WHERE email = ?', [email]);

// SQLite (better-sqlite3)
const user = db.prepare('SELECT * FROM users WHERE email = ?').get(email);

// ORM (Prisma, TypeORM, Drizzle) - SAFE by default
const user = await prisma.user.findUnique({ where: { email } });

// Dynamic queries - use query builders
const query = db.select().from(users);
if (email) query.where(eq(users.email, email));
if (role) query.where(eq(users.role, role));
const results = await query;
```

#### 5. **Authentication Security**

```typescript
// Password Hashing (Argon2id - recommended)
import argon2 from 'argon2';

async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 2 ** 16, // 64 MB
    timeCost: 3,
    parallelism: 1,
  });
}

async function verifyPassword(hash: string, password: string): Promise<boolean> {
  return argon2.verify(hash, password);
}

// JWT Security
const jwtOptions = {
  algorithm: 'RS256', // Asymmetric!
  issuer: 'myapp.com',
  audience: 'myapp.com',
  expiresIn: '15m', // Short-lived access token
};

// Refresh token rotation
async function refreshTokens(refreshToken: string) {
  const stored = await db.refreshToken.findUnique({ where: { token: refreshToken } });
  if (!stored || stored.revoked || stored.expiresAt < new Date()) {
    throw new Error('Invalid refresh token');
  }
  
  // Rotate: revoke old, issue new
  await db.refreshToken.update({ where: { id: stored.id }, data: { revoked: true } });
  
  const newAccessToken = generateAccessToken(stored.userId);
  const newRefreshToken = await createRefreshToken(stored.userId);
  
  return { accessToken: newAccessToken, refreshToken: newRefreshToken };
}

// Rate limiting on auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many attempts, try again later',
  standardHeaders: true,
  legacyHeaders: false,
});
app.post('/login', authLimiter, loginHandler);
app.post('/register', authLimiter, registerHandler);
```

#### 6. **Security Headers**

```typescript
// Helmet.js (Express) or manual
app.use((req, res, next) => {
  // Prevent MIME sniffing
  res.setHeader('X-Content-Type-Options', 'nosniff');
  
  // Prevent clickjacking
  res.setHeader('X-Frame-Options', 'DENY');
  
  // XSS filter (legacy but harmless)
  res.setHeader('X-XSS-Protection', '1; mode=block');
  
  // Referrer policy
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  
  // Permissions policy
  res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
  
  // HSTS (HTTPS only)
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');
  
  // COOP/COEP (isolation)
  res.setHeader('Cross-Origin-Opener-Policy', 'same-origin');
  res.setHeader('Cross-Origin-Embedder-Policy', 'require-corp');
  
  next();
});
```

#### 7. **File Upload Security**

```typescript
const ALLOWED_MIMES = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: MAX_SIZE },
  fileFilter: (req, file, cb) => {
    if (!ALLOWED_MIMES.includes(file.mimetype)) {
      return cb(new Error('Invalid file type'), false);
    }
    // Check magic numbers (first bytes)
    const magicNumbers: Record<string, number[]> = {
      'image/jpeg': [0xFF, 0xD8, 0xFF],
      'image/png': [0x89, 0x50, 0x4E, 0x47],
      'image/webp': [0x52, 0x49, 0x46, 0x46],
      'application/pdf': [0x25, 0x50, 0x44, 0x46],
    };
    cb(null, true);
  },
});

// Sanitize filename
function sanitizeFilename(filename: string): string {
  return filename
    .replace(/[^a-zA-Z0-9.-]/g, '_')
    .substring(0, 255);
}

// Store with random name, serve via signed URL
const key = `${crypto.randomUUID()}-${sanitizeFilename(file.originalname)}`;
await s3.putObject({ Bucket: 'uploads', Key: key, Body: file.buffer }).promise();
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Parameterized queries** | String concatenation | SQLi guaranteed |
| **CSP with nonces** | No CSP / `'unsafe-inline'` | XSS trivial |
| **HttpOnly, Secure, SameSite cookies** | Accessible via JS | Session theft via XSS |
| **Short JWT + refresh rotation** | Long-lived JWT | Token theft = permanent access |
| **Argon2id/bcrypt/scrypt** | MD5/SHA1/SHA256 | Instant cracking |
| **Rate limiting on auth** | No limits | Credential stuffing |
| **Input validation (allowlist)** | Blocklist/sanitization | Bypass guaranteed |
| **Security headers** | Missing headers | Clickjacking, MIME sniffing |
| **Dependency scanning** | Ignoring vulns | Supply chain attacks |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Explain XSS types and prevention"

**Answer:**
```
REFLECTED XSS:
- Payload in URL/query → reflected in response immediately
- Phishing links, search results
- Fix: Validate input, encode output, CSP

STORED XSS:
- Payload stored in DB → executes on every view
- Comments, profiles, messages
- Fix: Same + sanitize on storage/display

DOM-BASED XSS:
- Client-side JS writes attacker-controlled data to DOM
- Sources: location.hash, document.referrer, postMessage
- Sinks: innerHTML, outerHTML, document.write, eval
- Fix: Use textContent, avoid dangerous sinks, CSP
```

#### 2. **Code**: "Fix this XSS vulnerability"

```javascript
// VULNERABLE
app.get('/search', (req, res) => {
  const q = req.query.q;
  res.send(`<h1>Results for: ${q}</h1>`); // XSS!
});

// FIXED
app.get('/search', (req, res) => {
  const q = escapeHtml(req.query.q || '');
  res.send(`<h1>Results for: ${q}</h1>`);
});

// OR with template engine (auto-escaping)
app.get('/search', (req, res) => {
  res.render('search', { query: req.query.q }); // {{query}} auto-escaped
});

// OR with CSP
res.setHeader('Content-Security-Policy', "default-src 'self'; script-src 'self'");
```

#### 3. **Design**: "How to securely implement 'Remember Me'?"

```typescript
// Token stored in DB, not JWT
interface RememberToken {
  id: string;
  userId: string;
  selector: string;      // Public (in cookie)
  validatorHash: string; // bcrypt hash of validator
  expiresAt: Date;
}

// Login with remember
async function loginWithRemember(userId: string, remember: boolean) {
  const accessToken = generateAccessToken(userId);
  let refreshToken = generateRefreshToken(userId);
  
  if (remember) {
    const selector = crypto.randomBytes(16).toString('hex');
    const validator = crypto.randomBytes(32).toString('hex');
    const validatorHash = await bcrypt.hash(validator, 12);
    
    await db.rememberToken.create({
      selector,
      validatorHash,
      userId,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
    });
    
    // Cookie: selector:validator (HttpOnly, Secure, SameSite=Lax)
    res.cookie('remember', `${selector}:${validator}`, {
      httpOnly: true,
      secure: true,
      sameSite: 'lax',
      maxAge: 30 * 24 * 60 * 60 * 1000
    });
  }
  
  return { accessToken, refreshToken };
}

// Auto-login
app.use(async (req, res, next) => {
  if (!req.user && req.cookies.remember) {
    const [selector, validator] = req.cookies.remember.split(':');
    const token = await db.rememberToken.findUnique({ where: { selector } });
    
    if (token && token.expiresAt > new Date() && 
        await bcrypt.compare(validator, token.validatorHash)) {
      // Valid! Create new session
      req.user = await getUser(token.userId);
      // Rotate token
      await rotateRememberToken(token);
    } else {
      // Invalid - clear cookie
      res.clearCookie('remember');
    }
  }
  next();
});
```

#### 4. **Security**: "SameSite cookie values explained"

```
SameSite=Strict:
- Cookie sent ONLY in first-party context
- Not sent on any cross-site request (links, forms, images)
- Best security, breaks some UX (external links lose session)

SameSite=Lax (Default modern):
- Cookie sent on cross-site TOP-LEVEL navigations (links)
- Not sent on subresource requests (images, iframes, AJAX)
- Good balance: CSRF protection + UX

SameSite=None:
- Cookie sent on ALL requests (cross-site included)
- REQUIRES Secure flag (HTTPS only)
- Use for: Embedded widgets, SSO, third-party integrations

RECOMMENDATION:
- Session cookies: SameSite=Lax (or Strict for high security)
- CSRF tokens: SameSite=Strict
- Third-party/embedded: SameSite=None; Secure
```

#### 5. **Code**: "Implement rate limiting with Redis"

```typescript
import { RateLimiterRedis } from 'rate-limiter-flexible';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// General API limiter
const apiLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'ratelimit:api',
  points: 100, // 100 requests
  duration: 60, // per minute
  blockDuration: 60 * 15, // block for 15 min if exceeded
});

// Auth endpoints - stricter
const authLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'ratelimit:auth',
  points: 5,
  duration: 60 * 15, // 5 per 15 min
  blockDuration: 60 * 60, // block 1 hour
});

// Middleware
export async function rateLimit(req, res, next) {
  const limiter = req.path.startsWith('/auth') ? authLimiter : apiLimiter;
  const key = req.ip + ':' + req.path;
  
  try {
    await limiter.consume(key);
    next();
  } catch (rejRes) {
    const retrySecs = Math.round(rejRes.msBeforeNext / 1000) || 1;
    res.set('Retry-After', String(retrySecs));
    res.status(429).json({ 
      error: 'Too Many Requests',
      retryAfter: retrySecs 
    });
  }
}

// Distributed: use same Redis across all instances
```

#### 6. **Architecture**: "Defense in depth for file uploads"

```typescript
// Layer 1: Client-side validation (UX only)
const fileInput = document.getElementById('file');
fileInput.accept = 'image/jpeg,image/png,application/pdf';

// Layer 2: Server validation
const upload = multer({
  limits: { fileSize: 5_000_000 },
  fileFilter: (req, file, cb) => {
    const allowed = ['image/jpeg', 'image/png', 'application/pdf'];
    if (!allowed.includes(file.mimetype)) {
      return cb(new Error('Invalid type'), false);
    }
    cb(null, true);
  }
});

// Layer 3: Magic number verification
function verifyFileType(buffer: Buffer, mimetype: string): boolean {
  const signatures: Record<string, number[][]> = {
    'image/jpeg': [[0xFF, 0xD8, 0xFF]],
    'image/png': [[0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A]],
    'application/pdf': [[0x25, 0x50, 0x44, 0x46]],
  };
  
  const sigs = signatures[mimetype];
  return sigs?.some(sig => sig.every((byte, i) => buffer[i] === byte)) ?? false;
}

// Layer 4: Virus scanning (ClamAV)
async function scanFile(buffer: Buffer): Promise<boolean> {
  const clam = new ClamAV();
  const result = await clam.scanBuffer(buffer);
  return result.isClean;
}

// Layer 5: Storage security
// - Random filenames (no user control)
// - Private bucket (no public access)
// - Serve via signed URLs (expiring)
// - CDN with CSP headers

// Layer 6: Processing isolation
// - Resize/convert in separate container
// - No execution permissions on upload dir
// - Quarantine until scanned
```

#### 7. **Debugging**: "Find the vulnerability in this code"

```javascript
// VULNERABLE CODE SNIPPETS:

// 1. Path Traversal
app.get('/download', (req, res) => {
  const file = req.query.file;
  res.download(`/var/www/uploads/${file}`); 
  // Attack: ../../etc/passwd
});

// FIX: Path resolution + validation
const safePath = path.resolve('/var/www/uploads', file);
if (!safePath.startsWith('/var/www/uploads')) throw Error();
res.download(safePath);

// 2. Open Redirect
app.get('/redirect', (req, res) => {
  res.redirect(req.query.url); 
  // Attack: /redirect?url=https://evil.com
});

// FIX: Allowlist
const allowed = ['/dashboard', '/settings', '/profile'];
if (allowed.includes(req.query.url)) res.redirect(req.query.url);
else res.redirect('/');

// 3. Prototype Pollution
function merge(target, source) {
  for (const key in source) {
    if (source[key] && typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
// Attack: {"__proto__": {"polluted": true}}
// FIX: Object.create(null) or check hasOwnProperty

// 4. Regex DoS (ReDoS)
const emailRegex = /^([a-zA-Z0-9_\-\.]+)@([a-zA-Z0-9_\-\.]+)\.([a-zA-Z]{2,5})$/;
// Attack: "a@a.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"
// FIX: Use validator library (1) Non-backtracking regex, (2) Timeout, (3) Length limit
```

#### 8. **Compliance**: "OWASP Top 10 2021 mapping to defenses"

| Rank | Threat | Defense |
|------|--------|---------|
| A01 | Broken Access Control | Authorization checks on EVERY endpoint, deny by default |
| A02 | Cryptographic Failures | HTTPS everywhere, strong algos, key rotation |
| A03 | Injection | Parameterized queries, input validation, ORM |
| A04 | Insecure Design | Threat modeling, secure design patterns |
| A05 | Security Misconfiguration | Hardening, automation, minimal surface |
| A06 | Vulnerable Components | Dependency scanning, SBOM, patch management |
| A07 | Auth Failures | MFA, rate limiting, secure password reset |
| A08 | Software Integrity | CI/CD signing, provenance, integrity checks |
| A09 | Logging Failures | Structured logs, alerting, audit trails |
| A10 | SSRF | Allowlist URLs, disable internal metadata access |

---

### GOTCHAS & EDGE CASES

- [ ] **CSP breaks inline scripts** → Use nonces/hashes, move to external files
- [ ] **SameSite=Lax breaks some OAuth flows** → Use PKCE, state parameter
- [ ] **Rate limiting by IP fails behind proxy** → Use `X-Forwarded-For` or proxy protocol
- [ ] **JWT in localStorage = XSS steals tokens** → HttpOnly cookies + CSRF protection
- [ ] **Refresh token rotation = logout on multiple tabs** → Grace period or token families
- [ ] **bcrypt 72-char limit** → Pre-hash with SHA-256 if needed
- [ ] **Timing attacks on token comparison** → Use constant-time comparison
- [ ] **CORS misconfiguration** → Never `Access-Control-Allow-Origin: *` with credentials
- [ ] **Subdomain takeover** → Clean DNS records, monitor
- [ ] **Security headers in iframe** → COOP/COEP may break embedding

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Security Middleware Stack**
```typescript
// security.ts
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import csrf from 'csurf';
import { body, validationResult } from 'express-validator';

export const securityMiddleware = [
  // Helmet (all security headers)
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'nonce-{nonce}'", "'strict-dynamic'"],
        styleSrc: ["'self'", "'nonce-{nonce}'"],
        imgSrc: ["'self'", 'data:', 'https:'],
        fontSrc: ["'self'", 'https://fonts.gstatic.com'],
        connectSrc: ["'self'", 'https://api.example.com'],
        frameAncestors: ["'none'"],
        baseUri: ["'self'"],
        formAction: ["'self'"],
        objectSrc: ["'none'"],
        upgradeInsecureRequests: [],
      },
    },
    hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    permissionsPolicy: { features: { geolocation: [], microphone: [], camera: [] } },
  }),
  
  // Rate limiting
  rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 100,
    standardHeaders: true,
    legacyHeaders: false,
    message: { error: 'Too many requests' },
  }),
  
  // CSRF (for form submissions)
  csrf({ cookie: { httpOnly: true, secure: true, sameSite: 'strict' } }),
  
  // Body parsing with limits
  express.json({ limit: '10kb' }),
  express.urlencoded({ extended: true, limit: '10kb' }),
  
  // Validation error handler
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    next();
  },
];
```

#### 2. **Secure Password Reset Flow**
```typescript
// Request reset
async function requestPasswordReset(email: string) {
  const user = await db.user.findUnique({ where: { email } });
  if (!user) return; // Don't reveal existence
  
  const token = crypto.randomBytes(32).toString('hex');
  const hash = await argon2.hash(token);
  
  await db.passwordResetToken.create({
    userId: user.id,
    tokenHash: hash,
    expiresAt: new Date(Date.now() + 60 * 60 * 1000), // 1 hour
  });
  
  await emailService.send(user.email, 'Password Reset', `
    Reset your password: ${process.env.APP_URL}/reset?token=${token}&email=${email}
    Expires in 1 hour.
  `);
}

// Confirm reset
async function confirmPasswordReset(email: string, token: string, newPassword: string) {
  const record = await db.passwordResetToken.findFirst({
    where: { user: { email }, expiresAt: { gt: new Date() } }
  });
  
  if (!record || !(await argon2.verify(record.tokenHash, token))) {
    throw new Error('Invalid or expired token');
  }
  
  // Check password history (prevent reuse)
  const history = await db.passwordHistory.findMany({
    where: { userId: record.userId },
    orderBy: { createdAt: 'desc' },
    take: 5
  });
  
  for (const old of history) {
    if (await argon2.verify(old.hash, newPassword)) {
      throw new Error('Password recently used');
    }
  }
  
  const newHash = await argon2.hash(newPassword);
  
  await db.$transaction([
    db.user.update({ where: { id: record.userId }, data: { passwordHash: newHash } }),
    db.passwordResetToken.delete({ where: { id: record.id } }),
    db.passwordHistory.create({ userId: record.userId, hash: newHash }),
    // Invalidate all sessions
    db.session.deleteMany({ where: { userId: record.userId } }),
  ]);
}
```

#### 3. **Content Security Policy Builder**
```typescript
function buildCSP(nonce: string): string {
  const directives = {
    'default-src': ["'self'"],
    'script-src': ["'self'", `'nonce-${nonce}'`, "'strict-dynamic'"],
    'style-src': ["'self'", `'nonce-${nonce}'`],
    'img-src': ["'self'", 'data:', 'https:'],
    'font-src': ["'self'", 'https://fonts.gstatic.com'],
    'connect-src': ["'self'", 'https://api.example.com', 'wss://ws.example.com'],
    'frame-ancestors': ["'none'"],
    'base-uri': ["'self'"],
    'form-action': ["'self'"],
    'object-src': ["'none'"],
    'upgrade-insecure-requests': [],
    'block-all-mixed-content': [],
  };
  
  return Object.entries(directives)
    .map(([key, values]) => `${key} ${values.join(' ')}`)
    .join('; ');
}

// Per-request nonce
app.use((req, res, next) => {
  res.locals.cspNonce = crypto.randomBytes(16).toString('base64');
  res.setHeader('Content-Security-Policy', buildCSP(res.locals.cspNonce));
  next();
});
```

---

### PRACTICE PROBLEMS

1. **Audit** a sample app for OWASP Top 10 vulnerabilities
2. **Implement** secure authentication: register, login, MFA, password reset, session management
3. **Configure** CSP for a React app with inline styles, Google Fonts, analytics
4. **Write** a security scanner that checks for common misconfigurations
5. **Design** a threat model for a payment processing feature

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| XSS Types | Reflected (URL), Stored (DB), DOM-based (client sinks) |
| CSRF Prevention | SameSite cookies, CSRF tokens, custom headers |
| SQLi Prevention | Parameterized queries / ORM, never string concat |
| CSP Key Directives | default-src, script-src, style-src, connect-src, frame-ancestors |
| SameSite Values | Strict (no cross-site), Lax (top-level nav), None (all + Secure) |
| JWT Security | Short expiry (15m), RS256, refresh rotation, HttpOnly cookie |
| Password Hashing | Argon2id > bcrypt > scrypt > PBKDF2 (never MD5/SHA) |
| Rate Limiting | IP + endpoint, stricter on auth, Redis for distributed |
| File Upload Safety | Validate MIME + magic numbers, random names, virus scan, signed URLs |
| Security Headers | CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy |

---

## NEXT TOPIC: `11-PERFORMANCE.md`

> **Study Tip**: Set up a CSP in report-only mode on a real site. Analyze violations. Gradually tighten until zero violations.