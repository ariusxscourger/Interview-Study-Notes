# HTTP, REST & GRPC
## Exam-Style Study Notes

---

## TOPIC: API Communication Protocols

### WHY? (Problem Statement)

**Before Standardized APIs:**
- Custom protocols per application
- Tight coupling between client/server
- No interoperability
- Difficult to evolve

**What Standard Protocols Solve:**
| Problem | Solution |
|---------|----------|
| Communication standard | HTTP/REST, gRPC, GraphQL |
| Language agnostic | JSON/Protobuf serialization |
| Evolution | Versioning, backward compatibility |
| Tooling | Universal clients, debugging |
| Caching/Proxy | HTTP semantics |

**Real-World Analogy:**
- REST = Universal shipping containers (standardized, inspectable)
- gRPC = High-speed pneumatic tubes (fast, binary, contract-first)
- GraphQL = Custom order form (ask for exactly what you need)

---

### HOW? (Protocol Mechanics)

#### 1. **HTTP Request/Response Cycle**

```
CLIENT                                    SERVER
  │                                         │
  ├────── TCP Connection (3-way handshake) ─►│
  │                                         │
  ├────── TLS Handshake (if HTTPS) ────────►│
  │                                         │
  ├────── HTTP Request ────────────────────►│
  │  GET /api/users/123 HTTP/1.1            │
  │  Host: api.example.com                  │
  │  Authorization: Bearer <token>          │
  │  Accept: application/json               │
  │                                         │
  │◄────── HTTP Response ───────────────────┤
  │  HTTP/1.1 200 OK                        │
  │  Content-Type: application/json         │
  │  Cache-Control: public, max-age=3600    │
  │  { "id": 123, "name": "Alice" }         │
  │                                         │
  ├────── Connection Close / Keep-Alive ────►│
  │                                         │
```

#### 2. **HTTP Methods & Semantics**

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

*POST can be cached with explicit freshness headers

#### 3. **Status Codes (Must Know)**

```http
1xx Informational
100 Continue
101 Switching Protocols

2xx Success
200 OK
201 Created (Location header required)
202 Accepted (async processing)
204 No Content (successful delete)

3xx Redirection
301 Moved Permanently (cache)
302 Found (temporary)
304 Not Modified (conditional GET)
307 Temporary Redirect (preserve method)
308 Permanent Redirect (preserve method)

4xx Client Error
400 Bad Request (validation)
401 Unauthorized (auth required)
403 Forbidden (auth but no permission)
404 Not Found
409 Conflict (duplicate, version mismatch)
422 Unprocessable Entity (semantic errors)
429 Too Many Requests (rate limit)

5xx Server Error
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable (Retry-After)
504 Gateway Timeout
```

---

### WHAT? (REST & gRPC Implementation)

#### 1. **REST API Design Principles**

```yaml
# Resource-Oriented URLs
GET    /users              # List users
POST   /users              # Create user
GET    /users/{id}         # Get user
PATCH  /users/{id}         # Partial update
PUT    /users/{id}         # Full replace
DELETE /users/{id}         # Delete user

# Nested Resources
GET    /users/{id}/posts
POST   /users/{id}/posts
GET    /users/{id}/posts/{postId}

# Query Parameters for Filtering/Pagination
GET /users?page=2&limit=20&sort=-createdAt&filter[role]=admin
GET /posts?include=author,comments&fields=id,title,slug

# Versioning
# Option 1: URL
GET /api/v1/users
GET /api/v2/users

# Option 2: Header
Accept: application/vnd.myapi.v2+json

# Option 3: Query Param (discouraged)
GET /api/users?version=2
```

#### 2. **REST Response Standards**

```json
// Success Response
{
  "data": { "id": "123", "name": "Alice" },
  "meta": { "timestamp": "2024-01-15T10:30:00Z" }
}

// Collection Response
{
  "data": [{ "id": "1" }, { "id": "2" }],
  "meta": { 
    "pagination": { "page": 1, "limit": 20, "total": 100, "pages": 5 }
  }
}

// Error Response (RFC 7807 Problem Details)
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Email already exists",
  "instance": "/api/users",
  "errors": [
    { "field": "email", "message": "already in use", "code": "unique" }
  ]
}

// Created Response
HTTP/1.1 201 Created
Location: /api/users/123
Content-Type: application/json

{ "data": { "id": "123", "name": "Alice" } }
```

#### 3. **gRPC (Protocol Buffers + HTTP/2)**

```protobuf
// user.proto
syntax = "proto3";

package user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc WatchUsers(WatchUsersRequest) returns (stream User); // Server streaming
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  Role role = 4;
  google.protobuf.Timestamp created_at = 5;
}

enum Role {
  ROLE_UNSPECIFIED = 0;
  ROLE_ADMIN = 1;
  ROLE_USER = 2;
  ROLE_GUEST = 3;
}

message GetUserRequest {
  string id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

message UpdateUserRequest {
  string id = 1;
  string name = 2;
  string email = 3;
}

message DeleteUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string filter = 3;
  string order_by = 4;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
  int32 page = 3;
  int32 page_size = 4;
}

message WatchUsersRequest {
  string filter = 1;
}
```

#### 4. **gRPC Server Implementation (Go/Node/.NET)**

```typescript
// TypeScript gRPC Server
import { UserService } from './proto/user_grpc_pb';
import { User, GetUserRequest, CreateUserRequest } from './proto/user_pb';

class UserServiceImpl implements UserService {
  async getUser(call: { request: GetUserRequest }, callback: (error: Error | null, response: User) => void) {
    try {
      const user = await userRepository.findById(call.request.getId());
      if (!user) {
        callback({ code: 5, message: 'Not found' }, null); // grpc.NOT_FOUND
        return;
      }
      const response = new User();
      response.setId(user.id);
      response.setName(user.name);
      response.setEmail(user.email);
      response.setRole(user.role);
      callback(null, response);
    } catch (err) {
      callback({ code: 13, message: 'Internal error' }, null); // grpc.INTERNAL
    }
  }
  
  async createUser(call: { request: CreateUserRequest }, callback: (error: Error | null, response: User) => void) {
    // Validation, hashing, etc.
  }
  
  // Server Streaming
  async *watchUsers(call: { request: WatchUsersRequest }) {
    for await (const user of userRepository.watch(call.request.getFilter())) {
      yield user;
    }
  }
}

// Server setup
const server = new Server();
server.addService(UserService, new UserServiceImpl());
server.bindAsync('0.0.0.0:50050', ServerCredentials.createInsecure(), () => {
  server.start();
});
```

#### 5. **gRPC Client Usage**

```typescript
// TypeScript Client
import { UserServiceClient } from './proto/user_grpc_pb';
import { GetUserRequest, CreateUserRequest } from './proto/user_pb';

const client = new UserServiceClient('https://api.example.com', credentials);

// Unary call
const request = new GetUserRequest();
request.setId('123');

try {
  const user = await new Promise<User>((resolve, reject) => {
    client.getUser(request, (err, response) => {
      if (err) reject(err);
      else resolve(response);
    });
  });
  console.log(user.getName());
} catch (err) {
  if (err.code === 5) { /* Not found */ }
}

// Streaming call
const watchRequest = new WatchUsersRequest();
watchRequest.setFilter('role=admin');

const stream = client.watchUsers(watchRequest);
for await (const user of stream) {
  console.log('User changed:', user.getName());
}

// Metadata (auth, tracing)
const metadata = new grpc.Metadata();
metadata.add('authorization', `Bearer ${token}`);
metadata.add('x-correlation-id', correlationId);

client.getUser(request, metadata, (err, response) => { /* ... */ });
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Resource-oriented URLs** | RPC-style URLs (`/getUser`, `/createUser`) | Not cacheable, not discoverable |
| **Proper HTTP Status** | 200 for everything | Clients can't distinguish errors |
| **Plural nouns** | Singular (`/user/123`) | Inconsistent |
| **Nested for relationships** | Deep nesting (`/a/b/c/d/e`) | Complexity |
| **Query params for filtering** | Path params for filters | Overloading path |
| **gRPC for internal services** | REST for everything internal | Overhead, no streaming |
| **REST for public APIs** | gRPC for public | Browser support limited |
| **Idempotency keys** | No idempotency | Duplicate operations |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "REST vs gRPC - when to use which?"

**Answer:**
```
REST:
✓ Public APIs (browser, mobile, third-party)
✓ Human-readable (JSON, easy debugging)
✓ Universal tooling (curl, Postman, browsers)
✓ Caching (HTTP semantics)
✓ Firewall friendly (port 80/443)
✓ Stateless, scalable

gRPC:
✓ Internal microservices communication
✓ High performance (binary, HTTP/2 multiplexing)
✓ Contract-first (protobuf schema)
✓ Streaming (server/client/bidirectional)
✓ Strong typing (code generation)
✓ Polyglot (generate clients for any language)

HYBRID:
- Public API: REST + OpenAPI
- Internal: gRPC
- Gateway: gRPC-Gateway translates REST → gRPC
```

#### 2. **Code**: "Design a REST API for a blog with posts, comments, tags"

```yaml
# Posts
GET    /posts?page=1&limit=10&tag=tech&author=123
POST   /posts
GET    /posts/{id}
PATCH  /posts/{id}
DELETE /posts/{id}

# Comments (nested)
GET    /posts/{postId}/comments
POST   /posts/{postId}/comments
PATCH  /posts/{postId}/comments/{id}
DELETE /posts/{postId}/comments/{id}

# Tags
GET    /tags
POST   /tags
GET    /tags/{id}
GET    /tags/{id}/posts

# Search
GET    /search?q=keyword&type=post,comment

# Responses
# GET /posts
{
  "data": [{ "id": "1", "title": "...", "slug": "...", "author": {...}, "tags": [...] }],
  "meta": { "pagination": { "page": 1, "limit": 10, "total": 100 } }
}

# POST /posts
Request: { "title": "...", "content": "...", "tagIds": ["1", "2"] }
Response: 201 Created, Location: /posts/123
{ "data": { "id": "123", "title": "...", "slug": "..." } }
```

#### 3. **Design**: "API Versioning Strategy"

```http
# Strategy 1: URL Versioning (Explicit, Easy)
GET /api/v1/users
GET /api/v2/users

# Strategy 2: Header Versioning (Clean URLs)
GET /api/users
Accept: application/vnd.myapi.v1+json
Accept: application/vnd.myapi.v2+json

# Strategy 3: Media Type Parameter
Accept: application/json; version=1

# Best Practice: Header + Semantic Versioning
# v1, v2 for breaking changes
# Minor versions for additive changes (backward compatible)

# Deprecation Headers
Deprecation: true
Sunset: Sat, 01 Jan 2025 00:00:00 GMT
Link: <https://api.example.com/api/v2/users>; rel="successor-version"
```

#### 4. **Code**: "Implement Idempotency for POST endpoints"

```typescript
// Idempotency Key Header: Idempotency-Key: <uuid>
const idempotencyStore = new Map<string, { status: number; body: any; expires: number }>();

function idempotencyMiddleware(req, res, next) {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    return next();
  }
  
  const key = req.headers['idempotency-key'];
  if (!key) return next(); // Optional
  
  const cached = idempotencyStore.get(key);
  if (cached && cached.expires > Date.now()) {
    return res.status(cached.status).json(cached.body);
  }
  
  // Capture response
  const originalJson = res.json;
  res.json = function(body) {
    idempotencyStore.set(key, {
      status: res.statusCode,
      body,
      expires: Date.now() + 24 * 60 * 60 * 1000 // 24 hours
    });
    return originalJson.call(this, body);
  };
  
  next();
}

// Client Usage
fetch('/api/orders', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Idempotency-Key': crypto.randomUUID()
  },
  body: JSON.stringify({ items: [...] })
});
```

#### 5. **Performance**: "Optimize gRPC Performance"

```protobuf
// Use efficient field types
message User {
  string id = 1;           // Good for UUID
  // bytes id = 1;         // Better for binary IDs (16 bytes vs 36 chars)
  
  int32 age = 2;           // varint encoding
  // fixed32 age = 2;      // If values often large
  
  bool active = 3;         // varint (1 byte)
  
  repeated string tags = 4; // Packed by default in proto3
  // repeated int32 tags = 4 [packed = true]; // Explicit packing
}

// Avoid deep nesting - flatten
// BAD:
message User {
  Profile profile = 1;
}
message Profile {
  Settings settings = 1;
}

// GOOD:
message User {
  string name = 1;
  string avatar_url = 2;
  string theme = 3;
  bool notifications_enabled = 4;
}

// Connection Pooling
const channel = new grpc.Client(
  'api.example.com:443',
  grpc.credentials.createSsl(),
  {
    'grpc.max_receive_message_length': 10 * 1024 * 1024,
    'grpc.max_send_message_length': 10 * 1024 * 1024,
    'grpc.keepalive_time_ms': 30000,
    'grpc.keepalive_timeout_ms': 10000,
    'grpc.http2.max_pings_without_data': 0,
  }
);
```

#### 6. **Security**: "API Security Checklist"

```typescript
// 1. Authentication on all endpoints
app.use(authMiddleware); // Validates JWT, sets req.user

// 2. Authorization (RBAC/ABAC)
function requireRole(...roles: string[]) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

app.delete('/users/:id', requireRole('admin'), deleteUser);

// 3. Rate Limiting
const limiter = rateLimit({ windowMs: 60000, max: 100 });
app.use('/api/', limiter);

// Stricter on auth
const authLimiter = rateLimit({ windowMs: 900000, max: 5 });
app.post('/auth/login', authLimiter, login);

// 4. Input Validation
const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(12).regex(/[A-Z]/).regex(/[0-9]/).regex(/[^A-Za-z0-9]/),
  name: z.string().min(1).max(100),
});

app.post('/users', validate(createUserSchema), createUser);

// 5. Output Sanitization (prevent data leakage)
function sanitizeUser(user: User): PublicUser {
  return {
    id: user.id,
    name: user.name,
    email: user.email,
    // Exclude: passwordHash, ssn, internalNotes, etc.
  };
}

// 6. Security Headers
app.use(helmet());

// 7. CORS - Restrict origins
app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'Idempotency-Key'],
}));

// 8. Request Size Limits
app.use(express.json({ limit: '100kb' }));
app.use(express.urlencoded({ extended: true, limit: '100kb' }));
```

#### 7. **Architecture**: "GraphQL vs REST vs gRPC"

| Feature | REST | GraphQL | gRPC |
|---------|------|---------|------|
| **Data Fetching** | Multiple endpoints | Single endpoint, exact data | RPC calls |
| **Over-fetching** | Common | None | N/A |
| **Under-fetching** | Common | None | N/A |
| **Caching** | HTTP caching | Complex (normalized cache) | Application-level |
| **Real-time** | Webhooks/SSE | Subscriptions | Streaming |
| **Type Safety** | OpenAPI/Manual | Schema-first | Schema-first |
| **Learning Curve** | Low | Medium | Medium |
| **Browser Support** | Native | Fetch + Apollo/Urql | gRPC-Web needed |
| **Best For** | Public APIs, simple CRUD | Complex queries, multiple clients | Internal services, high perf |

---

### GOTCHAS & EDGE CASES

- [ ] **REST PUT vs PATCH**: PUT = full replace (must send all fields), PATCH = partial
- [ ] **gRPC HTTP/2**: Requires TLS in most environments, not plain HTTP
- [ ] **Protobuf evolution**: Never change field numbers, only add new fields
- [ ] **gRPC streaming**: Unary = request/response, Server/Client/Bidi streaming
- [ ] **Idempotency**: Only safe methods (GET, HEAD, PUT, DELETE) are naturally idempotent
- [ ] **REST pagination**: Use cursor-based for large datasets, offset for small
- [ ] **GraphQL N+1**: Use DataLoader for batching resolver calls
- [ ] **API versioning**: Don't break existing clients, deprecate gracefully
- [ ] **gRPC errors**: Use standard gRPC codes (NOT_FOUND, ALREADY_EXISTS, etc.)
- [ ] **REST filtering**: Use query params, not custom headers

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete REST Controller (TypeScript/Express)**
```typescript
import { Router, Request, Response, NextFunction } from 'express';
import { z } from 'zod';
import { UserService } from './user.service';
import { authMiddleware, requireRole } from '../auth';
import { validate } from '../validation';

const router = Router();
const userService = new UserService();

// Schemas
const createUserSchema = z.object({
  body: z.object({
    email: z.string().email(),
    password: z.string().min(12),
    name: z.string().min(1).max(100),
  }),
});

const updateUserSchema = z.object({
  params: z.object({ id: z.string().uuid() }),
  body: z.object({
    name: z.string().min(1).max(100).optional(),
    email: z.string().email().optional(),
  }).refine(obj => Object.keys(obj).length > 0, 'At least one field required'),
});

const listUsersSchema = z.object({
  query: z.object({
    page: z.coerce.number().int().positive().default(1),
    limit: z.coerce.number().int().positive().max(100).default(20),
    sort: z.enum(['createdAt', '-createdAt', 'name', '-name']).default('-createdAt'),
    role: z.enum(['admin', 'user', 'guest']).optional(),
  }),
});

// Routes
router.get('/', authMiddleware, validate(listUsersSchema), async (req, res, next) => {
  try {
    const result = await userService.list(req.query);
    res.json({ data: result.users, meta: { pagination: result.pagination } });
  } catch (err) { next(err); }
});

router.post('/', authMiddleware, requireRole('admin'), validate(createUserSchema), async (req, res, next) => {
  try {
    const user = await userService.create(req.body);
    res.status(201).location(`/api/users/${user.id}`).json({ data: user });
  } catch (err) { next(err); }
});

router.get('/:id', authMiddleware, async (req, res, next) => {
  try {
    const user = await userService.findById(req.params.id);
    if (!user) return res.status(404).json({ error: 'Not found' });
    res.json({ data: user });
  } catch (err) { next(err); }
});

router.patch('/:id', authMiddleware, validate(updateUserSchema), async (req, res, next) => {
  try {
    const user = await userService.update(req.params.id, req.body);
    res.json({ data: user });
  } catch (err) { next(err); }
});

router.delete('/:id', authMiddleware, requireRole('admin'), async (req, res, next) => {
  try {
    await userService.delete(req.params.id);
    res.status(204).send();
  } catch (err) { next(err); }
});

export default router;
```

#### 2. **gRPC Error Handling**
```typescript
// Server Interceptor for consistent errors
const errorInterceptor: grpc.ServerUnaryCallInterceptor = (call, next) => {
  try {
    return next(call);
  } catch (err) {
    if (err instanceof AppError) {
      const status = GRPC_CODE_MAP[err.code] || grpc.status.INTERNAL;
      throw { code: status, message: err.message, metadata: err.metadata };
    }
    throw { code: grpc.status.INTERNAL, message: 'Internal server error' };
  }
};

const GRPC_CODE_MAP = {
  'NOT_FOUND': grpc.status.NOT_FOUND,
  'ALREADY_EXISTS': grpc.status.ALREADY_EXISTS,
  'INVALID_ARGUMENT': grpc.status.INVALID_ARGUMENT,
  'UNAUTHENTICATED': grpc.status.UNAUTHENTICATED,
  'PERMISSION_DENIED': grpc.status.PERMISSION_DENIED,
  'FAILED_PRECONDITION': grpc.status.FAILED_PRECONDITION,
};

// Client-side retry with exponential backoff
async function withRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let i = 0; ; i++) {
    try {
      return await fn();
    } catch (err) {
      if (i >= maxRetries || !RETRYABLE_CODES.includes(err.code)) throw err;
      await sleep(Math.min(1000 * 2 ** i, 10000) + Math.random() * 1000);
    }
  }
}

const RETRYABLE_CODES = [
  grpc.status.UNAVAILABLE,
  grpc.status.DEADLINE_EXCEEDED,
  grpc.status.RESOURCE_EXHAUSTED,
];
```

#### 3. **OpenAPI/Swagger Generation**
```typescript
// swagger.ts
import swaggerJsdoc from 'swagger-jsdoc';
import swaggerUi from 'swagger-ui-express';

const options = {
  definition: {
    openapi: '3.0.3',
    info: {
      title: 'Nanolite API',
      version: '1.0.0',
      description: 'API Documentation',
      contact: { name: 'Nanolite Tech', email: 'api@nanolite.tech' },
    },
    servers: [{ url: 'https://api.example.com/api/v1' }],
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      },
      schemas: {
        User: {
          type: 'object',
          properties: {
            id: { type: 'string', format: 'uuid' },
            name: { type: 'string' },
            email: { type: 'string', format: 'email' },
            role: { type: 'string', enum: ['admin', 'user', 'guest'] },
            createdAt: { type: 'string', format: 'date-time' },
          },
        },
        Error: {
          type: 'object',
          properties: {
            type: { type: 'string' },
            title: { type: 'string' },
            status: { type: 'integer' },
            detail: { type: 'string' },
          },
        },
      },
    },
    security: [{ bearerAuth: [] }],
  },
  apis: ['./src/routes/*.ts'],
};

const specs = swaggerJsdoc(options);
app.use('/docs', swaggerUi.serve, swaggerUi.setup(specs));
```

---

### PRACTICE PROBLEMS

1. **Design** a REST API for an e-commerce system (products, orders, payments, inventory)
2. **Implement** gRPC service for real-time notifications with streaming
3. **Create** GraphQL schema for a social media feed with posts, comments, likes
4. **Build** API Gateway that routes REST → gRPC with protocol translation
5. **Implement** API versioning with backward compatibility and deprecation headers

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| REST vs gRPC | REST: public, JSON, caching. gRPC: internal, binary, streaming, contract-first |
| HTTP Methods | GET (read), POST (create), PUT (replace), PATCH (partial), DELETE (remove) |
| Idempotent Methods | GET, HEAD, PUT, DELETE, OPTIONS, TRACE |
| 201 Created | Must include Location header |
| 422 vs 400 | 400: malformed request. 422: valid syntax, semantic error |
| gRPC Status Codes | OK, CANCELLED, UNKNOWN, INVALID_ARGUMENT, DEADLINE_EXCEEDED, NOT_FOUND, ALREADY_EXISTS, PERMISSION_DENIED, UNAUTHENTICATED, RESOURCE_EXHAUSTED, FAILED_PRECONDITION, ABORTED, OUT_OF_RANGE, UNIMPLEMENTED, INTERNAL, UNAVAILABLE, DATA_LOSS |
| Protobuf Evolution | Never change field numbers, only add new optional fields |
| REST Pagination | Cursor-based for large, offset for small. Include total count in meta |
| API Versioning | Header (Accept: application/vnd.api.v2+json) or URL (/v1/, /v2/) |

---

## NEXT TOPIC: `02-DATABASES-SQL.md`

> **Study Tip**: Create a REST API with OpenAPI spec, then generate gRPC protobuf from it. Compare the two approaches.