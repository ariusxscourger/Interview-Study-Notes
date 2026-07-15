# AUTHENTICATION & AUTHORIZATION
## Exam-Style Study Notes

---

## TOPIC: Identity, Access Control & Security

### WHY? (Problem Statement)

**Without Auth:**
- Anyone can access everything
- No accountability or audit trail
- Data breaches, compliance violations
- No personalization or multi-tenancy

**What Auth Solves:**
| Problem | Solution |
|---------|----------|
| Who are you? | Authentication (AuthN) |
| What can you do? | Authorization (AuthZ) |
| Prove identity later | Tokens, Sessions |
| Delegate access | OAuth, OIDC |
| Audit trail | Structured logging |

**Real-World Analogy:**
- AuthN = ID Card (proves who you are)
- AuthZ = Key Card (what doors you can open)
- Token = Temporary visitor badge (expires, limited scope)

---

### HOW? (Mechanisms & Flows)

#### 1. **Authentication Methods**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FACTORS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  KNOWLEDGE (Something you know)                                 │
│  ├── Password / Passphrase                                      │
│  ├── PIN                                                        │
│  └── Security Questions                                         │
│                                                                  │
│  POSSESSION (Something you have)                                │
│  ├── Phone (SMS, Push, Authenticator App)                       │
│  ├── Hardware Token (YubiKey, RSA SecurID)                      │
│  ├── Smart Card                                                 │
│  └── Email Magic Link                                           │
│                                                                  │
│  INHERENCE (Something you are)                                  │
│  ├── Fingerprint / Face ID / Iris                               │
│  ├── Voice Recognition                                          │
│  └── Behavioral Biometrics                                      │
│                                                                  │
│  LOCATION (Somewhere you are)                                   │
│  ├── GPS / IP Geolocation                                       │
│   └── Corporate Network (VPN, Office WiFi)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

MULTI-FACTOR AUTHENTICATION (MFA):
- 2FA = Any 2 factors
- MFA = 2+ factors
- Adaptive MFA = Risk-based (prompt only when suspicious)
```

#### 2. **Session vs Token**

```
SESSION-BASED (Stateful):
────────────────────────────
1. User logs in → Server creates session
2. Server stores session (Redis, DB, Memory)
3. Server returns Session ID (Cookie)
4. Client sends Cookie with each request
5. Server looks up session → validates

PROS: Revocable immediately, simple, small payload
CONS: Server state, scaling complexity, CSRF risk

TOKEN-BASED (Stateless - JWT):
──────────────────────────────
1. User logs in → Server creates JWT
2. Server returns JWT (Header.Payload.Signature)
3. Client stores JWT (Memory, localStorage, Cookie)
3. Client sends JWT (Authorization: Bearer)
4. Server verifies signature → trusts payload

PROS: No server state, scalable, mobile-friendly
CONS: Can't revoke easily, larger payload, token theft = full access
```

#### 3. **OAuth 2.0 Flows**

```
AUTHORIZATION CODE FLOW (Web Apps - Most Secure):
─────────────────────────────────────────────────
User → Client → Auth Server (consent) → Code → Client → Token Endpoint → Tokens
     ←──────────────────────────────────────────────────────────────────┘

1. Client redirects to /authorize?response_type=code&client_id=...&redirect_uri=...&scope=...&state=...
2. User authenticates + consents
3. Auth Server redirects to redirect_uri?code=...&state=...
4. Client exchanges code for tokens (POST /token)
5. Client receives access_token, refresh_token, id_token

PKCE (Proof Key for Code Exchange) - Required for SPA/Mobile:
- Client generates code_verifier (random)
- Sends code_challenge = SHA256(code_verifier) in /authorize
- Sends code_verifier in /token exchange
- Prevents authorization code interception

CLIENT CREDENTIALS FLOW (Service-to-Service):
─────────────────────────────────────────────
Client → Token Endpoint (client_id, client_secret) → Access Token
- No user involvement
- Machine-to-machine
- Short-lived tokens

DEVICE AUTHORIZATION FLOW (TV, CLI, IoT):
─────────────────────────────────────────
1. Device requests device_code + user_code
2. User visits URL, enters user_code
3. Device polls token endpoint until authorized
```

#### 4. **OpenID Connect (OIDC)**

```
OIDC = OAuth 2.0 + Identity Layer
─────────────────────────────────
- Adds id_token (JWT with user info)
- Standardized scopes: openid, profile, email, address, phone
- UserInfo endpoint for additional claims
- Discovery document (.well-known/openid-configuration)
- Dynamic client registration
- Session management (logout, backchannel)
```

---

### WHAT? (Implementation)

#### 1. **JWT Implementation**

```typescript
// JWT Structure
// Header.Payload.Signature
// eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJuYW1lIjoiQWxpY2UiLCJpYXQiOjE1MTYyMzkwMjJ9.signature

interface JWTPayload {
  iss: string;        // Issuer
  sub: string;        // Subject (User ID)
  aud: string | string[]; // Audience
  exp: number;        // Expiration (Unix timestamp)
  nbf: number;        // Not Before
  iat: number;        // Issued At
  jti: string;        // JWT ID (unique)
  // Custom claims
  roles?: string[];
  permissions?: string[];
  sessionId?: string;
}

// Token Service
class TokenService {
  constructor(
    private privateKey: string,  // RS256
    public publicKey: string,
    private issuer: string,
    private audience: string
  ) {}

  // Generate Access Token (Short-lived: 15-30 min)
  generateAccessToken(user: User, sessionId: string): string {
    const payload: JWTPayload = {
      iss: this.issuer,
      sub: user.id,
      aud: this.audience,
      exp: Math.floor(Date.now() / 1000) + 15 * 60, // 15 min
      nbf: Math.floor(Date.now() / 1000),
      iat: Math.floor(Date.now() / 1000),
      jti: crypto.randomUUID(),
      roles: user.roles,
      permissions: user.permissions,
      sessionId,
    };
    return this.sign(payload);
  }

  // Generate Refresh Token (Long-lived: 30 days)
  generateRefreshToken(userId: string, sessionId: string): string {
    const payload: JWTPayload = {
      iss: this.issuer,
      sub: userId,
      aud: this.audience,
      exp: Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60,
      iat: Math.floor(Date.now() / 1000),
      jti: crypto.randomUUID(),
      type: 'refresh',
      sessionId,
    };
    return this.sign(payload);
  }

  // Verify Token
  verifyToken(token: string): JWTPayload | null {
    try {
      return this.verify(token, this.publicKey, {
        issuer: this.issuer,
        audience: this.audience,
        algorithms: ['RS256'],
      });
    } catch {
      return null;
    }
  }

  // RS256 Signing (Asymmetric - Public key can verify, private key signs)
  private sign(payload: JWTPayload): string {
    const header = { alg: 'RS256', typ: 'JWT' };
    const encodedHeader = this.base64UrlEncode(JSON.stringify(header));
    const encodedPayload = this.base64UrlEncode(JSON.stringify(payload));
    const signature = crypto.sign('RSA-SHA256', Buffer.from(`${encodedHeader}.${encodedPayload}`), this.privateKey);
    return `${encodedHeader}.${encodedPayload}.${this.base64UrlEncode(signature)}`;
  }

  private verify(token: string, publicKey: string, options: any): JWTPayload {
    // Implementation using jose or jsonwebtoken library
    // Validates: signature, exp, nbf, iss, aud, algorithms
  }
}
```

#### 2. **Session Management with Redis**

```typescript
class SessionManager {
  constructor(private redis: Redis) {}

  // Create Session
  async createSession(user: User, metadata: SessionMetadata): Promise<Session> {
    const sessionId = crypto.randomUUID();
    const session: Session = {
      id: sessionId,
      userId: user.id,
      roles: user.roles,
      permissions: user.permissions,
      createdAt: Date.now(),
      lastAccessedAt: Date.now(),
      ip: metadata.ip,
      userAgent: metadata.userAgent,
      mfaVerified: metadata.mfaVerified || false,
    };

    await this.redis.setex(
      `session:${sessionId}`,
      30 * 24 * 60 * 60, // 30 days
      JSON.stringify(session)
    );

    // Track user's active sessions (for "log out all devices")
    await this.redis.sadd(`user:sessions:${user.id}`, sessionId);
    await this.redis.expire(`user:sessions:${user.id}`, 30 * 24 * 60 * 60);

    return session;
  }

  // Get & Refresh Session
  async getSession(sessionId: string): Promise<Session | null> {
    const data = await this.redis.get(`session:${sessionId}`);
    if (!data) return null;

    const session = JSON.parse(data);
    
    // Extend TTL on access (sliding expiration)
    await this.redis.expire(`session:${sessionId}`, 30 * 24 * 60 * 60);
    session.lastAccessedAt = Date.now();
    await this.redis.set(`session:${sessionId}`, JSON.stringify(session));

    return session;
  }

  // Revoke Session
  async revokeSession(sessionId: string): Promise<void> {
    const data = await this.redis.get(`session:${sessionId}`);
    if (data) {
      const session = JSON.parse(data);
      await this.redis.srem(`user:sessions:${session.userId}`, sessionId);
    }
    await this.redis.del(`session:${sessionId}`);
  }

  // Revoke All User Sessions
  async revokeAllUserSessions(userId: string): Promise<number> {
    const sessionIds = await this.redis.smembers(`user:sessions:${userId}`);
    if (sessionIds.length === 0) return 0;

    const pipeline = this.redis.pipeline();
    for (const id of sessionIds) {
      pipeline.del(`session:${id}`);
    }
    pipeline.del(`user:sessions:${userId}`);
    await pipeline.exec();

    return sessionIds.length;
  }

  // Validate Session for Middleware
  async validateSession(sessionId: string): Promise<AuthContext | null> {
    const session = await this.getSession(sessionId);
    if (!session) return null;

    return {
      userId: session.userId,
      roles: session.roles,
      permissions: session.permissions,
      sessionId: session.id,
      mfaVerified: session.mfaVerified,
    };
  }
}
```

#### 3. **Authorization (RBAC + ABAC)**

```typescript
// Role-Based Access Control (RBAC)
enum Role {
  ADMIN = 'admin',
  MANAGER = 'manager',
  USER = 'user',
  VIEWER = 'viewer',
}

enum Permission {
  USER_CREATE = 'user:create',
  USER_READ = 'user:read',
  USER_UPDATE = 'user:update',
  USER_DELETE = 'user:delete',
  ORDER_CREATE = 'order:create',
  ORDER_READ = 'order:read',
  ORDER_UPDATE = 'order:update',
  ADMIN_PANEL = 'admin:panel',
}

// Role → Permissions Mapping
const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  [Role.ADMIN]: Object.values(Permission),
  [Role.MANAGER]: [
    Permission.USER_READ, Permission.USER_UPDATE,
    Permission.ORDER_CREATE, Permission.ORDER_READ, Permission.ORDER_UPDATE,
  ],
  [Role.USER]: [
    Permission.USER_READ, Permission.ORDER_CREATE, Permission.ORDER_READ,
  ],
  [Role.VIEWER]: [
    Permission.USER_READ, Permission.ORDER_READ,
  ],
};

// Attribute-Based Access Control (ABAC)
interface ABACContext {
  user: { id: string; roles: Role[]; department?: string };
  resource: { id: string; ownerId: string; department?: string; status?: string };
  action: string;
  environment: { time: Date; ip: string };
}

class AuthorizationService {
  // RBAC Check
  hasPermission(userRoles: Role[], required: Permission | Permission[]): boolean {
    const permissions = new Set(
      userRoles.flatMap(r => ROLE_PERMISSIONS[r])
    );
    const requiredPerms = Array.isArray(required) ? required : [required];
    return requiredPerms.every(p => permissions.has(p));
  }

  // ABAC Policy Evaluation
  async evaluate(context: ABACContext): Promise<boolean> {
    // Policy: Users can read/update own resources
    if (context.action.startsWith('read') || context.action.startsWith('update')) {
      if (context.resource.ownerId === context.user.id) return true;
    }

    // Policy: Managers can manage their department
    if (context.user.roles.includes(Role.MANAGER)) {
      if (context.resource.department === context.user.department) return true;
    }

    // Policy: Admins can do everything
    if (context.user.roles.includes(Role.ADMIN)) return true;

    // Policy: Time-based restrictions
    if (context.environment.time.getHours() < 6 || context.environment.time.getHours() > 22) {
      if (!context.user.roles.includes(Role.ADMIN)) return false;
    }

    return false;
  }

  // Resource-Level Authorization Middleware
  authorize(resourceType: string, action: string) {
    return async (req: Request, res: Response, next: NextFunction) => {
      const userId = req.user.id;
      const resourceId = req.params.id;
      
      const resource = await this.getResource(resourceType, resourceId);
      if (!resource) return res.status(404).json({ error: 'Not found' });

      const context: ABACContext = {
        user: { id: userId, roles: req.user.roles, department: req.user.department },
        resource: { id: resourceId, ownerId: resource.ownerId, department: resource.department, status: resource.status },
        action,
        environment: { time: new Date(), ip: req.ip },
      };

      const allowed = await this.evaluate(context);
      if (!allowed) return res.status(403).json({ error: 'Forbidden' });

      req.resource = resource;
      next();
    };
  }
}
```

#### 4. **Complete Auth Flow (Login + MFA + Tokens)**

```typescript
class AuthController {
  constructor(
    private userService: UserService,
    private sessionManager: SessionManager,
    private tokenService: TokenService,
    private mfaService: MFAService,
    private emailService: EmailService
  ) {}

  // Step 1: Username/Password
  async login(req: Request, res: Response) {
    const { email, password, mfaCode, rememberMe } = req.body;

    const user = await this.userService.findByEmail(email);
    if (!user || !await this.verifyPassword(password, user.passwordHash)) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    if (!user.isActive) {
      return res.status(403).json({ error: 'Account disabled' });
    }

    // Check MFA
    if (user.mfaEnabled) {
      if (!mfaCode) {
        return res.status(200).json({ requiresMFA: true, userId: user.id });
      }
      const valid = await this.mfaService.verify(user.mfaSecret, mfaCode);
      if (!valid) return res.status(401).json({ error: 'Invalid MFA code' });
    }

    // Create Session
    const session = await this.sessionManager.createSession(user, {
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      mfaVerified: user.mfaEnabled,
    });

    // Generate Tokens
    const accessToken = this.tokenService.generateAccessToken(user, session.id);
    const refreshToken = this.tokenService.generateRefreshToken(user.id, session.id);

    // Set Cookies (HttpOnly, Secure, SameSite)
    this.setAuthCookies(res, accessToken, refreshToken, rememberMe);

    res.json({
      user: { id: user.id, email: user.email, name: user.name, roles: user.roles },
      accessToken, // Also return for mobile/SPA
    });
  }

  // Step 2: Refresh Access Token
  async refresh(req: Request, res: Response) {
    const refreshToken = req.cookies?.refreshToken || req.body.refreshToken;
    if (!refreshToken) return res.status(401).json({ error: 'No refresh token' });

    const payload = this.tokenService.verifyToken(refreshToken);
    if (!payload || payload.type !== 'refresh') {
      return res.status(401).json({ error: 'Invalid refresh token' });
    }

    // Check session still valid
    const session = await this.sessionManager.getSession(payload.sessionId);
    if (!session) return res.status(401).json({ error: 'Session expired' });

    // Rotate refresh token (security best practice)
    const newAccessToken = this.tokenService.generateAccessToken(
      { id: payload.sub, roles: session.roles, permissions: session.permissions },
      session.id
    );
    const newRefreshToken = this.tokenService.generateRefreshToken(payload.sub, session.id);

    // Revoke old refresh token (optional: maintain family)
    // await this.revokeRefreshToken(refreshToken);

    this.setAuthCookies(res, newAccessToken, newRefreshToken, true);
    res.json({ accessToken: newAccessToken });
  }

  // Logout
  async logout(req: Request, res: Response) {
    const sessionId = req.cookies?.sessionId || req.headers['x-session-id'];
    if (sessionId) {
      await this.sessionManager.revokeSession(sessionId);
    }
    this.clearAuthCookies(res);
    res.json({ success: true });
  }

  // Logout All Devices
  async logoutAll(req: Request, res: Response) {
    await this.sessionManager.revokeAllUserSessions(req.user.id);
    this.clearAuthCookies(res);
    res.json({ success: true });
  }

  private setAuthCookies(res: Response, accessToken: string, refreshToken: string, rememberMe: boolean) {
    const cookieOptions = {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax' as const,
      path: '/',
    };

    res.cookie('accessToken', accessToken, { ...cookieOptions, maxAge: 15 * 60 * 1000 });
    res.cookie('refreshToken', refreshToken, { 
      ...cookieOptions, 
      maxAge: rememberMe ? 30 * 24 * 60 * 60 * 1000 : 24 * 60 * 60 * 1000 
    });
  }

  private clearAuthCookies(res: Response) {
    res.clearCookie('accessToken', { path: '/' });
    res.clearCookie('refreshToken', { path: '/' });
  }
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Short-lived Access + Refresh** | Long-lived JWT only | Token theft = permanent access |
| **RS256 (Asymmetric)** | HS256 (Symmetric) | Any service can sign tokens |
| **HttpOnly Secure Cookies** | localStorage for tokens | XSS steals tokens |
| **Refresh Token Rotation** | Static refresh token | Stolen refresh = permanent access |
| **MFA for Sensitive Actions** | Password only | Phishing, credential stuffing |
| **Short Session + Sliding Expiry** | Infinite sessions | Forgotten sessions = risk |
| **Centralized Auth Service** | Auth logic in every service | Inconsistent, hard to update |
| **Audit Log All Auth Events** | No logging | Can't investigate breaches |

---

### INTERVIEW QUESTIONS (Top 20)

#### 1. **Conceptual**: "JWT vs Session - when to use which?"

```
SESSION (Stateful):
✓ Immediate revocation
✓ Small payload (just session ID)
✓ Simple implementation
✓ CSRF protection built-in (SameSite cookie)
✗ Server state (Redis/DB)
✗ Scaling complexity
✗ Cross-domain tricky

JWT (Stateless):
✓ No server state
✓ Horizontal scaling trivial
✓ Works great for microservices
✓ Mobile/SPA friendly
✗ Can't revoke without blocklist
✗ Larger payload
✗ Token theft = full access until expiry
✗ Algorithm confusion attacks

USE SESSIONS FOR:
- Traditional web apps
- High security requirements
- Need immediate logout everywhere

USE JWT FOR:
- Microservices
- Mobile apps
- Third-party API access
- Serverless functions

HYBRID (BEST OF BOTH):
- JWT for access tokens (short 15min)
- Session/DB for refresh tokens (revocable)
- Session stores metadata (IP, device, MFA status)
```

#### 2. **Code**: "Implement JWT with refresh rotation and family tracking"

```typescript
class RefreshTokenManager {
  constructor(private redis: Redis) {}

  // Store refresh token with family tracking
  async storeRefreshToken(
    userId: string, 
    sessionId: string, 
    refreshToken: string, 
    familyId: string
  ): Promise<void> {
    const tokenHash = await this.hashToken(refreshToken);
    const data = {
      userId,
      sessionId,
      familyId,
      createdAt: Date.now(),
      expiresAt: Date.now() + 30 * 24 * 60 * 60 * 1000,
    };

    // Store token → metadata
    await this.redis.setex(
      `refresh:${tokenHash}`,
      30 * 24 * 60 * 60,
      JSON.stringify(data)
    );

    // Track family for rotation detection
    await this.redis.sadd(`refresh:family:${familyId}`, tokenHash);
    await this.redis.expire(`refresh:family:${familyId}`, 30 * 24 * 60 * 60);
  }

  // Validate and rotate
  async rotateRefreshToken(
    oldRefreshToken: string
  ): Promise<{ newRefreshToken: string; accessToken: string; user: User } | null> {
    const tokenHash = await this.hashToken(oldRefreshToken);
    const stored = await this.redis.get(`refresh:${tokenHash}`);
    
    if (!stored) return null; // Not found or expired

    const data = JSON.parse(stored);
    
    // Check if token was already used (reuse detection)
    if (data.used) {
      // TOKEN REUSE DETECTED - Revoke entire family!
      await this.revokeFamily(data.familyId);
      throw new SecurityError('Refresh token reuse detected - possible theft');
    }

    // Mark old token as used
    data.used = true;
    data.replacedAt = Date.now();
    await this.redis.set(`refresh:${tokenHash}`, JSON.stringify(data));

    // Generate new tokens
    const familyId = data.familyId;
    const newRefreshToken = crypto.randomBytes(32).toString('hex');
    const newTokenHash = await this.hashToken(newRefreshToken);

    // Store new token in same family
    await this.storeRefreshToken(data.userId, data.sessionId, newRefreshToken, familyId);

    // Generate new access token
    const user = await this.userService.findById(data.userId);
    const accessToken = this.tokenService.generateAccessToken(user, data.sessionId);

    return { newRefreshToken, accessToken, user };
  }

  // Revoke entire token family (on logout or reuse detection)
  async revokeFamily(familyId: string): Promise<void> {
    const tokenHashes = await this.redis.smembers(`refresh:family:${familyId}`);
    const pipeline = this.redis.pipeline();
    for (const hash of tokenHashes) {
      pipeline.del(`refresh:${hash}`);
    }
    pipeline.del(`refresh:family:${familyId}`);
    await pipeline.exec();
  }

  private async hashToken(token: string): Promise<string> {
    return crypto.createHash('sha256').update(token).digest('hex');
  }
}
```

#### 3. **Design**: "OAuth 2.0 Authorization Server Architecture"

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTHORIZATION SERVER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ /authorize   │  │  /token      │  │ /introspect  │          │
│  │ (Consent,    │  │ (Code,       │  │ (Validate    │          │
│  │  Login,      │  │  Refresh,    │  │  Access      │          │
│  │  MFA)        │  │  Client Cred)│  │  Tokens)     │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         ▼                 ▼                 ▼                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    CORE SERVICES                          │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐  │   │
│  │  │ Client      │ │ User        │ │ Token               │  │   │
│  │  │ Registry    │ │ Management  │ │ Service (JWT/Opaque)│  │   │
│  │  │ (client_id, │ │ (Credentials│ │ (Sign, Verify,      │  │   │
│  │  │  secrets,   │ │  MFA,       │ │  Rotate, Revoke)    │  │   │
│  │  │  redirects, │ │  Sessions)  │ │                     │  │   │
│  │  │  scopes)    │ │             │ │                     │  │   │
│  │  └─────────────┘ └─────────────┘ └─────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    STORAGE                                 │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐  │   │
│  │  │ PostgreSQL  │ │ Redis       │ │ HSM / KMS           │  │   │
│  │  │ (Clients,   │ │ (Sessions,  │ │ (Signing Keys,      │  │   │
│  │  │  Users,     │ │  Tokens,    │ │  Encryption)        │  │   │
│  │  │  Consents)  │ │  Codes)     │ │                     │  │   │
│  │  └─────────────┘ └─────────────┘ └─────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

KEY ENDPOINTS:
GET  /authorize?response_type=code&client_id=...&redirect_uri=...&scope=...&state=...&code_challenge=...
POST /token (grant_type=authorization_code|refresh_token|client_credentials)
POST /introspect (token=...) → { active: true, scope, client_id, exp, ... }
POST /revoke (token=..., token_type_hint=access_token|refresh_token)
GET  /userinfo (Authorization: Bearer ...) → { sub, name, email, ... }
GET  /.well-known/openid-configuration
GET  /.well-known/jwks.json
```

#### 4. **Security**: "Prevent Common Auth Vulnerabilities"

```typescript
// 1. BRUTE FORCE PROTECTION
class BruteForceProtection {
  constructor(private redis: Redis) {}

  async checkAttempt(identifier: string): Promise<{ allowed: boolean; retryAfter?: number }> {
    const key = `bruteforce:${identifier}`;
    const attempts = await this.redis.incr(key);
    
    if (attempts === 1) {
      await this.redis.expire(key, 15 * 60); // 15 min window
    }

    const maxAttempts = 5;
    if (attempts > maxAttempts) {
      const ttl = await this.redis.ttl(key);
      return { allowed: false, retryAfter: ttl };
    }
    return { allowed: true };
  }

  async resetAttempts(identifier: string): Promise<void> {
    await this.redis.del(`bruteforce:${identifier}`);
  }
}

// 2. PASSWORD SECURITY
class PasswordSecurity {
  // Argon2id (recommended)
  static async hash(password: string): Promise<string> {
    return argon2.hash(password, {
      type: argon2.argon2id,
      memoryCost: 2 ** 16, // 64 MB
      timeCost: 3,
      parallelism: 1,
    });
  }

  static async verify(hash: string, password: string): Promise<boolean> {
    return argon2.verify(hash, password);
  }

  // Password Strength Policy
  static validate(password: string): { valid: boolean; errors: string[] } {
    const errors: string[] = [];
    if (password.length < 12) errors.push('At least 12 characters');
    if (!/[A-Z]/.test(password)) errors.push('At least one uppercase');
    if (!/[a-z]/.test(password)) errors.push('At least one lowercase');
    if (!/[0-9]/.test(password)) errors.push('At least one number');
    if (!/[^A-Za-z0-9]/.test(password)) errors.push('At least one special character');
    
    // Check against common passwords (haveibeenpwned API)
    // Check against user info (name, email)
    
    return { valid: errors.length === 0, errors };
  }

  // Breached Password Check (k-anonymity)
  static async isBreached(password: string): Promise<boolean> {
    const hash = crypto.createHash('sha1').update(password).digest('hex').toUpperCase();
    const prefix = hash.substring(0, 5);
    const suffix = hash.substring(5);
    
    const response = await fetch(`https://api.pwnedpasswords.com/range/${prefix}`);
    const data = await response.text();
    return data.split('\n').some(line => line.startsWith(suffix));
  }
}

// 3. TOKEN BINDING (Prevent Token Theft)
class TokenBinding {
  // Bind token to device fingerprint
  static generateDeviceFingerprint(req: Request): string {
    const components = [
      req.headers['user-agent'] || '',
      req.headers['accept-language'] || '',
      req.ip || '',
      // Add more stable identifiers
    ];
    return crypto.createHash('sha256').update(components.join('|')).digest('hex');
  }

  // Store binding with token
  static async bindToken(tokenId: string, fingerprint: string): Promise<void> {
    await redis.setex(`token:binding:${tokenId}`, 30 * 24 * 60 * 60, fingerprint);
  }

  // Verify on each request
  static async verifyBinding(tokenId: string, fingerprint: string): Promise<boolean> {
    const stored = await redis.get(`token:binding:${tokenId}`);
    return stored === fingerprint;
  }
}

// 4. SECURE HEADERS MIDDLEWARE
function securityHeaders() {
  return (req: Request, res: Response, next: NextFunction) => {
    // Prevent clickjacking
    res.setHeader('X-Frame-Options', 'DENY');
    // Prevent MIME sniffing
    res.setHeader('X-Content-Type-Options', 'nosniff');
    // XSS Protection (legacy but harmless)
    res.setHeader('X-XSS-Protection', '1; mode=block');
    // Referrer Policy
    res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
    // Permissions Policy
    res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
    // HSTS (HTTPS only)
    if (process.env.NODE_ENV === 'production') {
      res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');
    }
    next();
  }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Algorithm Confusion** - Always specify allowed algorithms in verification
- [ ] **Token Replay** - Use nonce, short expiry, or token binding
- [ ] **Refresh Token Theft** - Rotate on every use, detect reuse, revoke families
- [ ] **Clock Skew** - Allow small leeway (30-60s) for exp/nbf validation
- [ ] **Sub Claim** - Use stable identifier (user ID), not email
- [ ] **Audience Validation** - Always verify `aud` claim matches your API
- [ ] **Scope Validation** - Check scopes on every request, not just at issuance
- [ ] **PKCE for SPA/Mobile** - Mandatory for public clients
- [ ] **State Parameter** - Prevent CSRF in OAuth flow
- [ ] **Token Storage** - HttpOnly cookies > localStorage > sessionStorage > memory

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete JWT Verification Middleware**
```typescript
function authenticateToken(tokenService: TokenService, sessionManager: SessionManager) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const accessToken = req.cookies?.accessToken || req.headers.authorization?.replace('Bearer ', '');
    
    if (!accessToken) {
      return res.status(401).json({ error: 'No token provided' });
    }

    const payload = tokenService.verifyToken(accessToken);
    if (!payload) {
      return res.status(401).json({ error: 'Invalid or expired token' });
    }

    // Optional: Verify session still valid (for revocation)
    const session = await sessionManager.getSession(payload.sessionId);
    if (!session) {
      return res.status(401).json({ error: 'Session revoked' });
    }

    // Attach user context
    req.user = {
      id: payload.sub,
      roles: payload.roles,
      permissions: payload.permissions,
      sessionId: payload.sessionId,
      mfaVerified: session.mfaVerified,
    };

    next();
  };
}
```

#### 2. **Permission Decorator**
```typescript
function RequirePermission(...permissions: Permission[]) {
  return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;

    descriptor.value = async function (req: Request, res: Response, next: NextFunction) {
      const userPermissions = req.user?.permissions || [];
      const hasPermission = permissions.every(p => userPermissions.includes(p));

      if (!hasPermission) {
        return res.status(403).json({ 
          error: 'Insufficient permissions',
          required: permissions,
          current: userPermissions,
        });
      }

      return originalMethod.apply(this, [req, res, next]);
    };

    return descriptor;
  };
}

// Usage
class OrderController {
  @RequirePermission(Permission.ORDER_CREATE)
  async createOrder(req: Request, res: Response) { /* ... */ }

  @RequirePermission(Permission.ORDER_READ, Permission.ORDER_UPDATE)
  async updateOrder(req: Request, res: Response) { /* ... */ }
}
```

#### 3. **OAuth Client Registration**
```typescript
interface OAuthClient {
  clientId: string;
  clientSecret: string; // Hashed
  name: string;
  redirectUris: string[];
  allowedScopes: string[];
  grantTypes: ('authorization_code' | 'refresh_token' | 'client_credentials')[];
  responseTypes: ('code' | 'token')[];
  tokenEndpointAuthMethod: 'client_secret_basic' | 'client_secret_post' | 'none';
  requirePkce: boolean;
  accessTokenLifetime: number; // seconds
  refreshTokenLifetime: number; // seconds
}

class ClientRegistry {
  async register(client: Omit<OAuthClient, 'clientId' | 'clientSecret'>): Promise<OAuthClient> {
    const clientId = `client_${crypto.randomBytes(16).toString('hex')}`;
    const clientSecret = crypto.randomBytes(32).toString('hex');
    const secretHash = await argon2.hash(clientSecret);

    const newClient: OAuthClient = {
      clientId,
      clientSecret: secretHash,
      ...client,
    };

    await this.db.clients.insert(newClient);
    return { ...newClient, clientSecret }; // Return plain secret once
  }

  async validateClient(clientId: string, secret: string): Promise<OAuthClient | null> {
    const client = await this.db.clients.findByClientId(clientId);
    if (!client) return null;

    const valid = await argon2.verify(client.clientSecret, secret);
    return valid ? client : null;
  }

  async validateRedirectUri(clientId: string, redirectUri: string): Promise<boolean> {
    const client = await this.db.clients.findByClientId(clientId);
    return client?.redirectUris.includes(redirectUri) ?? false;
  }
}
```

---

### PRACTICE PROBLEMS

1. **Implement** a complete OAuth 2.0 Authorization Server with PKCE support
2. **Build** a session management system with concurrent session limits
3. **Create** an ABAC policy engine with JSON-based policy definitions
4. **Design** a passwordless authentication system (magic links, WebAuthn)
5. **Implement** Single Sign-On (SSO) with SAML 2.0 and OIDC federation

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| AuthN vs AuthZ | AuthN = Who (Identity). AuthZ = What (Permissions) |
| JWT Structure | Header.Payload.Signature (Base64Url encoded) |
| Access Token Lifetime | 15-30 minutes (short) |
| Refresh Token Lifetime | 7-30 days (long), with rotation |
| RS256 vs HS256 | RS256 = Asymmetric (public/private). HS256 = Symmetric (shared secret) |
| PKCE Purpose | Prevent auth code interception in public clients (SPA, mobile) |
| Token Binding | Bind token to device fingerprint to prevent theft |
| Refresh Token Rotation | Issue new refresh token on each use, detect reuse, revoke family |
| OIDC Scopes | openid, profile, email, address, phone |
| Token Introspection | Validate opaque tokens via /introspect endpoint |
| JWKS | JSON Web Key Set - public keys for JWT verification |
| Brute Force Protection | Rate limit by IP/user, exponential backoff, CAPTCHA |

---

## NEXT TOPIC: `05-OBSERVABILITY.md`

> **Study Tip**: Build a mini auth service: Register → Login (MFA) → JWT + Refresh Rotation → Protected Route → Logout All. Test token theft scenario.