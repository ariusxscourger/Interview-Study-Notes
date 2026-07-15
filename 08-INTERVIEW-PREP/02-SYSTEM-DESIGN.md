# SYSTEM DESIGN INTERVIEW MASTERY
## Exam-Style Study Notes

---

## TOPIC: System Design Interview Framework

### WHY? (Problem Statement)

**System Design interviews test:**
- Ability to design scalable, reliable systems
- Trade-off analysis and decision-making
- Communication and collaboration skills
- Real-world engineering experience

**What System Design Solves:**
| Interview Need | Preparation |
|----------------|-------------|
| Structure | Framework (REQUIREMENTS → API → DATA → ARCHITECTURE → SCALE) |
| Depth | Deep-dive areas (caching, sharding, consistency) |
| Trade-offs | Explicit pros/cons for each decision |
| Clarity | Diagrams, API contracts, data models |

---

### HOW? (The System Design Framework)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM DESIGN FRAMEWORK (45-60 min)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. REQUIREMENTS (5-10 min)          2. API DESIGN (5-10 min)             │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │ • Functional requirements       │  │ • REST/GraphQL/gRPC             │  │
│  │ • Non-functional (NFR)          │  │ • Request/Response contracts    │  │
│  │ • Scale estimates (QPS, DAU)    │  │ • Error codes                   │  │
│  │ • Constraints                   │  │ • Versioning strategy           │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                             │
│  3. DATA MODEL (10 min)              4. HIGH-LEVEL ARCHITECTURE (10 min)  │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │ • Entities & relationships      │  │ • Service decomposition         │  │
│  │ • SQL vs NoSQL                  │  │ • Communication (sync/async)    │  │
│  │ • Partitioning strategy         │  │ • Data flow                     │  │
│  │ • Indexing strategy             │  │ • External dependencies         │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                             │
│  5. DEEP DIVES (15-20 min)           6. TRADE-OFFS & BOTTLENECKS (5 min)  │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │ • Caching strategy              │  │ • Consistency vs Availability   │  │
│  │ • Database scaling              │  │ • Latency vs Consistency        │  │
│  │ • Consistency models            │  │ • Cost vs Performance           │  │
│  │ • Rate limiting                 │  │ • Build vs Buy                  │  │
│  │ • Observability                 │  │ • Single points of failure      │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### HOW? (Back-of-Envelope Estimation)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BACK-OF-ENVELOPE MATH                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STANDARD NUMBERS:                                                          │
│  ─────────────────────────────────────────────────────────────────────────  │
│  1 million = 10^6                                                           │
│  1 billion = 10^9                                                           │
│  1 KB = 10^3 bytes | 1 MB = 10^6 | 1 GB = 10^9 | 1 TB = 10^12              │
│                                                                             │
│  THROUGHPUT:                                                                │
│  ─────────────────────────────────────────────────────────────────────────  │
│  1M DAU, 10 actions/day = 10M req/day = ~115 QPS average                   │
│  Peak = 10x average = ~1,150 QPS                                            │
│                                                                             │
│  STORAGE:                                                                   │
│  ─────────────────────────────────────────────────────────────────────────  │
│  1M users × 1KB/profile = 1 GB                                              │
│  100M tweets × 500 bytes = 50 GB                                            │
│  1B logs × 200 bytes = 200 GB                                               │
│                                                                             │
│  LATENCY BUDGETS:                                                           │
│  ─────────────────────────────────────────────────────────────────────────  │
│  L1 cache: 0.5 ns | L2 cache: 7 ns | RAM: 100 ns                           │
│  SSD: 50-150 μs | Network (DC): 0.5-2 ms | Network (cross-region): 50-100ms│
│                                                                             │
│  RULE OF THUMB:                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│  "Designer's dozen": 1M users → 100 QPS average, 1K QPS peak               │
│  1 server handles ~1K QPS (simple CRUD), ~100 QPS (complex)                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Core Patterns)

#### 1. **API Design Patterns**

```http
# RESTful API Design
GET    /api/v1/users                    # List users
GET    /api/v1/users/{id}               # Get user
POST   /api/v1/users                    # Create user
PATCH  /api/v1/users/{id}               # Partial update
DELETE /api/v1/users/{id}               # Delete user

# Query Parameters
GET /api/v1/users?page=2&limit=20&sort=-createdAt&filter[role]=admin

# Error Responses (RFC 7807)
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Email already exists",
  "instance": "/api/v1/users",
  "errors": [
    { "field": "email", "message": "already in use", "code": "unique" }
  ]
}

# Pagination (Cursor-based for large datasets)
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTAwfQ==",
    "hasMore": true
  }
}
```

#### 2. **Database Scaling Patterns**

```
SHARDING STRATEGIES:
─────────────────────────────────────────────────────────────────────────────
1. HASH-BASED: shard = hash(key) % N
   ✓ Uniform distribution ✗ Resharding = rehash all

2. RANGE-BASED: shard = key range (A-M, N-Z)
   ✓ Range queries efficient ✗ Hot spots (new users = last shard)

3. DIRECTORY-BASED: lookup table maps key → shard
   ✓ Flexible ✗ Extra lookup, SPOF

4. CONSISTENT HASHING: virtual nodes on ring
   ✓ Minimal reshuffling on add/remove ✓ Used by DynamoDB, Cassandra

READ REPLICAS:
─────────────────────────────────────────────────────────────────────────────
Primary (Writes) → Async Replication → Replicas (Reads)
- Read scaling: Add replicas
- Lag: Usually <100ms, can be seconds under load
- Solution: Route critical reads to primary
```

#### 3. **Caching Strategies**

```
CACHE PATTERNS:
─────────────────────────────────────────────────────────────────────────────
1. CACHE-ASIDE (Lazy Loading):
   App → Cache (miss) → App → DB → App → Cache → App
   
2. READ-THROUGH:
   App → Cache (miss) → Cache → DB → Cache → App
   
3. WRITE-THROUGH:
   App → Cache → DB (sync) → App
   
4. WRITE-BEHIND (Write-Back):
   App → Cache → (async) → DB

CACHE KEYS:
─────────────────────────────────────────────────────────────────────────────
user:{id}:profile          → TTL 1hr, invalidate on profile update
user:{id}:posts:{page}     → TTL 5min, invalidate on new post
post:{id}:comments         → TTL 10min, invalidate on comment
feed:{userId}              → TTL 15min, rebuild async

CACHE STAMPEDE PREVENTION:
─────────────────────────────────────────────────────────────────────────────
1. Probabilistic early expiration (random TTL jitter)
2. Mutex lock (single flight)
3. Stale-while-revalidate (serve stale, refresh async)
```

#### 4. **Consistency Models**

```
CONSISTENCY SPECTRUM:
─────────────────────────────────────────────────────────────────────────────
STRONG ────────────────────────────────────────────────────────────────── EVENTUAL
  │                                                                    │
  ▼                                                                    ▼
Linearizable          Sequential          Causal          Eventual
  │                                                                    │
  ├── Every read          ├── Operations     ├── Causally       ├── Reads
  │   sees latest         │   appear in      │   related        │   may
  │   write               │   some order     │   ops ordered    │   see
  │                       │                  │                  │   stale
  │                       │                  │                  │   data
  ▼                       ▼                  ▼                  ▼
CP Systems               -                  -                  AP Systems
(PostgreSQL,                                    (Cassandra,
 etcd, ZooKeeper)                                 DynamoDB)
```

---

### INTERVIEW QUESTIONS (Top 10 System Design Problems)

#### 1. **Design a URL Shortener (bit.ly)**

```
REQUIREMENTS:
- Shorten long URL → short code (e.g., bit.ly/abc123)
- Redirect short code → original URL
- 100M URLs, 10B redirects/month
- Custom aliases, analytics, expiration

ARCHITECTURE:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  API GW     │────►│  Write API  │────►│  Database   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
              ┌───────────┐ ┌───────────┐
              │  Cache    │ │ Analytics │
              │ (Redis)   │ │ (Kafka)   │
              └───────────┘ └───────────┘

KEY DECISIONS:
1. SHORT CODE: Base62 of auto-increment ID (6 chars = 62^6 = 56B)
2. STORAGE: PostgreSQL (id, long_url, short_code, user_id, created_at, expires_at)
3. REDIRECT: Cache TTL 24h, bloom filter for non-existent
4. ANALYTICS: Async via Kafka → Flink/Spark → Aggregates
5. CUSTOM ALIAS: Validate uniqueness, reserve namespace
```

#### 2. **Design a Rate Limiter**

```typescript
// Sliding Window Log (Precise)
class SlidingWindowLogLimiter {
  async allow(key: string, limit: number, windowMs: number): Promise<RateLimitResult> {
    const now = Date.now();
    const windowStart = now - windowMs;
    const redisKey = `ratelimit:${key}`;

    const script = `
      redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])
      local count = redis.call('ZCARD', KEYS[1])
      if count >= tonumber(ARGV[3]) then return {0, count} end
      redis.call('ZADD', KEYS[1], ARGV[2], ARGV[2])
      redis.call('EXPIRE', KEYS[1], ARGV[4])
      return {1, count + 1}
    `;

    const result = await this.redis.eval(script, 1, `ratelimit:${key}`, windowStart, now, limit, Math.ceil(windowMs / 1000) + 1) as [number, number];
    return { allowed: result[0] === 1, remaining: Math.max(0, limit - result[1]) };
  }
}

// Token Bucket (Smooth burst handling)
class TokenBucketLimiter { /* ... */ }
```

#### 3. **Design a Notification System**

```
REQUIREMENTS:
- Email, SMS, Push notifications
- 10M notifications/day
- Retry with exponential backoff
- Priority queue (urgent vs bulk)
- Templates, personalization
- Delivery tracking, analytics

ARCHITECTURE:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  API GW     │────►│Notification │────►│  Priority   │
└─────────────┘     │  Service    │     │  Queues     │
                    └──────┬──────┘     └──────┬──────┘
                           │                   │
                    ┌──────┴──────┐            │
                    ▼             ▼            ▼
              ┌───────────┐ ┌───────────┐ ┌───────────┐
              │  Email    │ │   SMS     │ │  Push     │
              │  Workers  │ │  Workers  │ │  Workers  │
              └───────────┘ └───────────┘ └───────────┘
                           │                   │
                    ┌──────┴───────────────────┴──────┐
                    ▼                                 ▼
              ┌───────────┐                   ┌───────────┐
              │  Retry    │                   │  Analytics│
              │  Service  │                   │  Service  │
              └───────────┘                   └───────────┘

KEY PATTERNS:
1. Provider abstraction (SendGrid, Twilio, FCM, APNs)
2. Exponential backoff: 1m, 5m, 15m, 1hr, 6hr
3. Idempotency keys per notification
4. Dead letter queue after max retries
```

#### 4. **Design a Chat System (WhatsApp/Slack)**

```
REQUIREMENTS:
- 1:1 and group chat
- Real-time messaging
- Media sharing (images, files)
- Read receipts, typing indicators
- 1B messages/day

ARCHITECTURE:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  API GW     │────►│  Chat       │────►│  Message    │
└─────────────┘     │  Service    │     │  Store      │
                    └──────┬──────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
              ┌───────────┐ ┌───────────┐
              │  Redis    │ │  Kafka    │
              │ (Presence)│ │ (Message) │
              └───────────┘ └───────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        ┌─────────────┐           ┌─────────────┐
        │  Push       │           │  WebSocket  │
        │  Service    │           │  Server     │
        └─────────────┘           └─────────────┘

KEY DECISIONS:
1. MESSAGE STORE: Cassandra (time-series, partition by conversation_id)
2. REAL-TIME: WebSocket per user, Redis Pub/Sub for presence
3. MEDIA: S3 + CDN, thumbnail generation async
4. OFFLINE: Message queue per user, sync on reconnect
5. SEARCH: Elasticsearch (message content, participants)
```

---

### GOTCHAS & EDGE CASES

- [ ] **Always clarify requirements first** - Don't assume scale
- [ ] **Explicit trade-offs** - "I chose X over Y because..."
- [ ] **Bottleneck identification** - "The bottleneck here is..."
- [ ] **Failure scenarios** - "If X fails, we handle it by..."
- [ ] **Cost awareness** - "This would cost ~$X/month"
- [ ] **Monitoring** - "We'd add metrics for X, Y, Z"
- [ ] **Security** - Auth, rate limiting, encryption, PII
- [ ] **Migration path** - "V1 → V2 migration strategy..."

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete System Design Template**

```
SYSTEM DESIGN: [Name]

1. REQUIREMENTS
   Functional: [List]
   Non-Functional: [Latency < Xms, Availability 99.9%, ...]
   Scale: [DAU, QPS, Storage, Bandwidth]

2. API DESIGN
   GET /api/resource
   POST /api/resource
   ...

3. DATA MODEL
   Entity: { fields }
   Relationships: [ER diagram description]
   
4. ARCHITECTURE
   [Component diagram description]
   
5. KEY COMPONENTS
   - Caching: [Strategy, TTL, Invalidation]
   - Database: [SQL/NoSQL, Sharding, Replication]
   - Queue: [Kafka/RabbitMQ, Topics, Consumers]
   - Cache: [Redis, TTL, Patterns]
   
6. SCALING
   Horizontal: [Stateless services, read replicas]
   Database: [Sharding strategy, read replicas]
   
7. TRADE-OFFS
   Consistency vs Availability: [Choice + reasoning]
   Latency vs Consistency: [Choice + reasoning]
   Build vs Buy: [Decision]

8. FAILURE HANDLING
   Circuit Breaker, Retry, Dead Letter Queue, Graceful Degradation
```

---

### PRACTICE PROBLEMS

1. **Design** a distributed unique ID generator (Snowflake, ULID, UUID v7)
2. **Build** a distributed configuration system (etcd/Consul + watchers)
3. **Create** a leader election system (Redis, etcd, ZooKeeper)
4. **Implement** a distributed rate limiter with Redis Cluster
4. **Design** a multi-region active-active database architecture
5. **Design** a payment processing system (idempotency, exactly-once)

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| System Design Framework | Requirements → API → Data Model → Architecture → Deep Dives → Trade-offs |
| Back-of-envelope: 1M DAU | ~115 QPS avg, ~1K QPS peak |
| Consistent Hashing | Virtual nodes on ring, minimal reshuffling on add/remove |
| Cache Patterns | Cache-Aside, Read-Through, Write-Through, Write-Behind |
| Cache Stampede Fix | Locks, probabilistic early expiry, stale-while-revalidate |
| CAP Theorem | Pick 2 of 3: Consistency, Availability, Partition Tolerance |
| PACELC | If Partition → A or C. Else → Latency or Consistency |
| Sharding Strategies | Hash, Range, Directory, Consistent Hashing |
| Consistency Levels | Strong → Sequential → Causal → Eventual |
| Rate Limiter Algorithms | Sliding Window Log, Sliding Window Counter, Token Bucket |
| Read Replicas | Async replication, read scaling, lag <100ms typically |
| Write-Behind Cache | Async write to DB, fast writes, risk of data loss |
| Circuit Breaker | Closed→Open→Half-Open, prevents cascade failures |

---

## NEXT TOPIC: `03-CODING-PATTERNS.md`

> **Study Tip**: Practice 2 system designs per week. Time yourself: 5min requirements, 5min API, 10min data, 10min arch, 15min deep-dives, 5min tradeoffs. Draw diagrams.