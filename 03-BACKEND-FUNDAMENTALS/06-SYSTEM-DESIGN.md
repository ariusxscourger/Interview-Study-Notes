# SYSTEM DESIGN FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: Distributed Systems Architecture & Design

### WHY? (Problem Statement)

**Why System Design Matters:**
- Scale: Single server → Millions of users
- Reliability: 99.99% uptime despite failures
- Evolution: Requirements change, architecture must adapt
- Cost: Efficient resource usage at scale
- Team: Multiple teams, independent deployments

**What System Design Solves:**
| Challenge | Solution |
|-----------|----------|
| Scalability | Horizontal scaling, sharding, caching |
| Availability | Replication, failover, multi-region |
| Consistency | Consensus, transactions, eventual consistency |
| Latency | CDN, edge computing, caching |
| Maintainability | Modularity, loose coupling, observability |

**Real-World Analogy:**
- Monolith = One big restaurant kitchen
- Microservices = Food court (independent stalls, shared infrastructure)
- Serverless = Cloud kitchen (no infrastructure management)

---

### HOW? (Core Principles & Patterns)

#### 1. **CAP Theorem & PACELC**

```
CAP THEOREM (Brewer's Theorem):
┌─────────────────────────────────────────────────────────────┐
│  You can only guarantee 2 of 3:                             │
│                                                             │
│  CONSISTENCY    ◄──────►  AVAILABILITY                      │
│       │                  │                                  │
│       │                  │                                  │
│       ▼                  ▼                                  │
│    PARTITION TOLERANCE (Network will fail - mandatory)      │
│                                                             │
│  CP Systems: MongoDB (default), Redis, HBase, PostgreSQL   │
│  AP Systems: Cassandra, DynamoDB, Riak, CouchDB            │
│  CA Systems: Traditional RDBMS (no partition tolerance)    │
└─────────────────────────────────────────────────────────────┘

PACELC EXTENSION:
- If Partition (P) → Choose Availability (A) or Consistency (C)
- Else (E) → Choose Latency (L) or Consistency (C)

System Classification:
- PC/EC: MongoDB, HBase, Redis (Consistency when partitioned, Consistency else)
- PA/EL: Cassandra, DynamoDB (Availability when partitioned, Latency else)
- PA/EC: (Rare)
- PC/EL: (Rare - e.g., Cosmos DB with strong consistency)
```

#### 2. **Scalability Patterns**

```
VERTICAL SCALING (Scale Up):
- Bigger server (more CPU, RAM, SSD)
- Simple, but limited by hardware
- Single point of failure
- Cost grows exponentially

HORIZONTAL SCALING (Scale Out):
- More servers (commodity hardware)
- Linear cost growth
- Requires stateless design
- Needs load balancing, sharding

SCALING DIMENSIONS:
─────────────────────────────────────────────────────────────
1. READ SCALING:          Replicas, Read Replicas, Caching
2. WRITE SCALING:         Sharding, Partitioning, Event Sourcing
3. COMPUTE SCALING:       Microservices, Serverless, Queue Workers
4. STORAGE SCALING:       Sharding, Tiered Storage, Archival
4. GEOGRAPHIC SCALING:    Multi-region, Edge Computing, CDN
```

#### 3. **Database Sharding Strategies**

```
SHARDING KEY SELECTION:
─────────────────────────────────────────────────────────────
1. HASH-BASED:
   shard = hash(key) % N
   ✓ Uniform distribution
   ✗ Resharding requires rehashing all data

2. RANGE-BASED:
   shard = key range (A-M, N-Z)
   ✓ Range queries efficient
   ✗ Hot spots (new users in last shard)

3. DIRECTORY-BASED:
   lookup table maps key → shard
   ✓ Flexible, easy resharding
   ✗ Extra lookup, single point of failure

4. CONSISTENT HASHING:
   virtual nodes on ring
   ✓ Minimal reshuffling on add/remove
   ✓ Used by DynamoDB, Cassandra, Redis Cluster

RESHARDING STRATEGIES:
─────────────────────────────────────────────────────────────
1. OFFLINE: Stop writes, copy data, switch
2. ONLINE (DUAL-WRITE): Write to old + new, backfill, switch
3. SHADOW: Mirror reads to new shard, compare, switch
```

#### 4. **Caching Strategies**

```
CACHE LAYERS:
─────────────────────────────────────────────────────────────
L1: Local (In-Memory)     → Microseconds, per-instance
L2: Distributed (Redis)   → Sub-millisecond, shared
L3: CDN/Edge              → Milliseconds, geographic
L4: Database              → Milliseconds, authoritative

CACHE PATTERNS:
─────────────────────────────────────────────────────────────
1. CACHE-ASIDE (Lazy Loading):
   App → Cache (miss) → App → DB → App → Cache → App
   Simple, but cache miss = 3 hops

2. READ-THROUGH:
   App → Cache (miss) → Cache → DB → Cache → App
   Transparent to app

3. WRITE-THROUGH:
   App → Cache → DB (sync) → App
   Consistent, slower writes

4. WRITE-BEHIND (Write-Back):
   App → Cache → (async) → DB
   Fast writes, risk of data loss

5. REFRESH-AHEAD:
   Async refresh before expiry
   Predictable access patterns

CACHE INVALIDATION:
- TTL (Time-To-Live)
- Explicit invalidation on write
- Version-based (key includes version)
- Event-driven (CDC → invalidate)
```

---

### WHAT? (Architecture Patterns)

#### 1. **Microservices Architecture**

```
SERVICE DECOMPOSITION:
─────────────────────────────────────────────────────────────
BY BUSINESS CAPABILITY (Domain-Driven Design):
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   User      │ │  Order      │ │  Payment    │ │ Inventory   │
│  Service    │ │  Service    │ │  Service    │ │  Service    │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘

COMMUNICATION:
─────────────────────────────────────────────────────────────
SYNCHRONOUS (Request-Response):
- REST/gRPC
- Tight coupling, latency
- Use for queries, immediate consistency

ASYNCHRONOUS (Event-Driven):
- Message Broker (Kafka, RabbitMQ, NATS)
- Loose coupling, resilience
- Use for commands, eventual consistency

DATA MANAGEMENT:
─────────────────────────────────────────────────────────────
- DATABASE PER SERVICE (Polyglot Persistence)
- SAGA Pattern for distributed transactions
- CQRS (Command Query Responsibility Segregation)
- EVENT SOURCING for audit trail

INFRASTRUCTURE:
─────────────────────────────────────────────────────────────
- API GATEWAY (Routing, Auth, Rate Limit)
- SERVICE MESH (Istio, Linkerd) - mTLS, Observability
- SERVICE DISCOVERY (Consul, etcd, DNS)
- CONFIG MANAGEMENT (Consul, Spring Config, etcd)
```

#### 2. **Event-Driven Architecture**

```
EVENT FLOW:
─────────────────────────────────────────────────────────────
Producer → Event Broker → Consumer(s)
                │
                ▼
         ┌─────────────────┐
         │  Event Schema   │
         │  (Avro/Protobuf)│
         └─────────────────┘

EVENT TYPES:
─────────────────────────────────────────────────────────────
1. DOMAIN EVENTS:    Business facts (OrderCreated, PaymentProcessed)
2. INTEGRATION EVENTS: Cross-system (OrderCreated → InventoryReserved)
3. NOTIFICATION EVENTS: User-facing (EmailSent, PushNotification)

MESSAGE BROKER PATTERNS:
─────────────────────────────────────────────────────────────
1. PUB/SUB:          Broadcast to all subscribers
2. WORK QUEUES:      Distribute tasks among workers
3. PUB/SUB + ROUTING: Topic exchanges, routing keys
4. DEAD LETTER:      Failed messages for inspection

GUARANTEES:
─────────────────────────────────────────────────────────────
- AT MOST ONCE:     Fire and forget (may lose)
- AT LEAST ONCE:    Acknowledgment + retry (may duplicate)
- EXACTLY ONCE:     Idempotency + transactions (hard)

IDEMPOTENCY KEYS:
- Client generates unique key per operation
- Server tracks processed keys
- Duplicate requests return same result
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Stateless Services** | Server-side sessions | Can't scale horizontally |
| **API Gateway** | Direct service access | No central auth, rate limit, routing |
| **Circuit Breaker** | No failure handling | Cascade failures |
| **Bulkhead** | Shared thread pools | One failure kills all |
| **CQRS** | Single model for read/write | Read/write conflict, scaling |
| **Saga** | Distributed 2PC | Blocking, not scalable |
| **Event Sourcing** | CRUD only | No audit, no temporal queries |
| **Idempotency** | No duplicate protection | Double charges, duplicate data |

---

### INTERVIEW QUESTIONS (Top 20)

#### 1. **Conceptual**: "Design a URL Shortener (like bit.ly)"

```
REQUIREMENTS:
- Shorten long URL → short code (e.g., bit.ly/abc123)
- Redirect short code → original URL
- 100M URLs, 10B redirects/month
- Custom aliases, analytics, expiration

ARCHITECTURE:
─────────────────────────────────────────────────────────────
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  API Gateway│────►│  Write API  │────►│  Database   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
              ┌───────────┐ ┌───────────┐
              │  Cache    │ │ Analytics │
              │ (Redis)   │ │ (Kafka)   │
              └───────────┘ └───────────┘

COMPONENTS:
1. SHORT CODE GENERATION:
   - Base62 encoding (a-z, A-Z, 0-9) of auto-increment ID
   - Or random 6-8 chars with collision check
   - Custom alias: validate uniqueness

2. STORAGE:
   - PostgreSQL: id, long_url, short_code, user_id, created_at, expires_at
   - Index on short_code (unique), user_id
   - Partition by created_at for archive

3. REDIRECT FLOW (Read-heavy):
   GET /abc123 → Cache (Redis) → hit? → 301 redirect
                        ↓ miss
                   Database → Cache → 301 redirect
   - Cache TTL: 24h (or longer for popular)
   - Bloom filter for non-existent codes

4. ANALYTICS:
   - Async: Click event → Kafka → Flink/Spark → Aggregates
   - Store: click_id, short_code, timestamp, referrer, country, device

5. SCALING:
   - Read replicas for redirects
   - Redis Cluster for cache
   - Sharding by short_code hash
   - CDN for static assets

6. EDGE CASES:
   - Custom alias collision → 409
   - Expired URLs → 410 Gone
   - Malicious URLs → VirusTotal scan, blocklist
   - Rate limiting per IP/user
```

#### 2. **Code**: "Design a Rate Limiter"

```typescript
// Requirements: 1000 req/min per user, distributed

interface RateLimiter {
  allow(key: string, limit: number, windowMs: number): Promise<{ allowed: boolean; remaining: number; resetAt: number }>;
}

// SLIDING WINDOW LOG (Precise)
class SlidingWindowLogLimiter implements RateLimiter {
  constructor(private redis: Redis) {}

  async allow(key: string, limit: number, windowMs: number): Promise<RateLimitResult> {
    const now = Date.now();
    const windowStart = now - windowMs;
    const redisKey = `ratelimit:${key}`;

    const script = `
      -- Remove expired
      redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])
      -- Count
      local count = redis.call('ZCARD', KEYS[1])
      if count >= tonumber(ARGV[3]) then
        return {0, count}
      end
      -- Add current
      redis.call('ZADD', KEYS[1], ARGV[2], ARGV[2])
      redis.call('EXPIRE', KEYS[1], ARGV[4])
      return {1, count + 1}
    `;

    const result = await this.redis.eval(script, 1, `ratelimit:${key}`, windowStart, now, limit, Math.ceil(windowMs / 1000) + 1) as [number, number];
    return { allowed: result[0] === 1, remaining: Math.max(0, limit - result[1]), resetAt: Date.now() + windowMs };
  }
}

// SLIDING WINDOW COUNTER (Approximate, Memory Efficient)
class SlidingWindowCounterLimiter implements RateLimiter {
  async allow(key: string, limit: number, windowMs: number): Promise<RateLimitResult> {
    const now = Date.now();
    const currentWindow = Math.floor(now / windowMs);
    const previousWindow = currentWindow - 1;
    const weight = 1 - (now % windowMs) / windowMs;

    const script = `
      local current = tonumber(redis.call('GET', KEYS[1] .. ':' .. ARGV[1]) or '0')
      local previous = tonumber(redis.call('GET', KEYS[1] .. ':' .. ARGV[2]) or '0')
      local weight = tonumber(ARGV[3])
      local count = math.floor(current + previous * weight)
      if count >= tonumber(ARGV[4]) then return {0, count} end
      redis.call('INCR', KEYS[1] .. ':' .. ARGV[1])
      redis.call('EXPIRE', KEYS[1] .. ':' .. ARGV[1], ARGV[5])
      return {1, count + 1}
    `;

    const result = await this.redis.eval(script, 1, `ratelimit:${key}`, currentWindow, previousWindow, weight.toFixed(4), limit, Math.ceil(windowMs / 1000) * 2) as [number, number];
    return { allowed: result[0] === 1, remaining: Math.max(0, limit - result[1]), resetAt: (currentWindow + 1) * windowMs };
  }
}

// TOKEN BUCKET (Smooth Burst Handling)
class TokenBucketLimiter implements RateLimiter {
  async allow(key: string, capacity: number, refillRate: number): Promise<RateLimitResult> {
    const redisKey = `tokenbucket:${key}`;
    const now = Date.now();

    const script = `
      local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'lastRefill')
      local tokens = tonumber(bucket[1]) or tonumber(ARGV[1])
      local lastRefill = tonumber(bucket[2]) or tonumber(ARGV[2])
      local elapsed = (tonumber(ARGV[3]) - lastRefill) / 1000
      tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[4]))
      if tokens >= 1 then
        tokens = tokens - 1
        redis.call('HMSET', KEYS[1], 'tokens', tokens, 'lastRefill', ARGV[3])
        redis.call('EXPIRE', KEYS[1], ARGV[5])
        return {1, tokens}
      else
        redis.call('HMSET', KEYS[1], 'tokens', tokens, 'lastRefill', lastRefill)
        redis.call('EXPIRE', KEYS[1], ARGV[5])
        return {0, tokens}
      end
    `;

    const result = await this.redis.eval(script, 1, `tokenbucket:${key}`, capacity, now, now, refillRate, Math.ceil(capacity / refillRate) + 10) as [number, number];
    return { allowed: result[0] === 1, remaining: Math.floor(result[1]), resetAt: Date.now() + (1 - result[1]) / refillRate * 1000 };
  }
}
```

#### 3. **Design**: "Design a Notification System (Email, SMS, Push)"

```
REQUIREMENTS:
- Send notifications via multiple channels
- 10M notifications/day
- Retry with exponential backoff
- Priority queue (urgent vs bulk)
- Templates, personalization
- Delivery tracking, analytics

ARCHITECTURE:
─────────────────────────────────────────────────────────────
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  API Gateway│────►│  Notification│────►│  Priority   │
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

COMPONENTS:

1. PRIORITY QUEUES (Redis Streams / RabbitMQ / Kafka):
   - HIGH: OTP, Security alerts (process first)
   - NORMAL: Welcome emails, Updates
   - LOW: Marketing, Newsletters (batch process)

2. TEMPLATE ENGINE:
   - Handlebars / Jinja2
   - Variables: {{user.name}}, {{order.id}}
   - Versioning, A/B testing

3. PROVIDER ABSTRACTION:
   interface NotificationProvider {
     send(notification: Notification): Promise<SendResult>;
     validateConfig(): boolean;
   }
   - Email: SendGrid, SES, Mailgun
   - SMS: Twilio, Vonage, Plivo
   - Push: FCM, APNs, Expo

3. RETRY LOGIC:
   - Exponential backoff: 1min, 5min, 15min, 1hr, 6hr
   - Max retries: 5
   - Dead letter queue after max retries
   - Circuit breaker per provider

4. DELIVERY TRACKING:
   - Status: QUEUED → SENT → DELIVERED / FAILED / BOUNCED
   - Webhooks from providers → Update status
   - Analytics: Open rate, Click rate, Delivery rate

3. IDEMPOTENCY:
   - Deduplication key per notification
   - Prevent duplicate sends on retry

SCALING:
- Horizontal worker scaling per channel
- Separate queues per priority
- Batch processing for low priority
- Rate limiting per provider (API limits)
```

#### 4. **Code**: "Design a Distributed Lock"

```typescript
// Requirements: Mutual exclusion, TTL, auto-release, reentrancy

interface DistributedLock {
  acquire(key: string, ttlMs: number, ownerId: string): Promise<boolean>;
  release(key: string, ownerId: string): Promise<boolean>;
  extend(key: string, ownerId: string, additionalTtlMs: number): Promise<boolean>;
}

class RedisDistributedLock implements DistributedLock {
  constructor(private redis: Redis) {}

  async acquire(key: string, ttlMs: number, ownerId: string): Promise<boolean> {
    const lockKey = `lock:${key}`;
    const lockValue = `${ownerId}:${Date.now()}:${Math.random()}`;
    
    // SET NX EX = atomic acquire with TTL
    const result = await this.redis.set(lockKey, lockValue, 'NX', 'PX', ttlMs);
    return result === 'OK';
  }

  async release(key: string, ownerId: string): Promise<boolean> {
    const lockKey = `lock:${key}`;
    
    // Lua script: only delete if we own it
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(script, 1, `lock:${key}`, `${ownerId}:*`);
    return result === 1;
  }

  async extend(key: string, ownerId: string, additionalTtlMs: number): Promise<boolean> {
    const lockKey = `lock:${key}`;
    
    const script = `
      local current = redis.call("GET", KEYS[1])
      if current and string.find(current, ARGV[1]) == 1 then
        return redis.call("PEXPIRE", KEYS[1], ARGV[2])
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(script, 1, `lock:${key}`, ownerId, additionalTtlMs);
    return result === 1;
  }
}

// Usage with auto-renewal
class AutoRenewLock {
  constructor(private lock: DistributedLock) {}

  async withLock<T>(key: string, ttlMs: number, fn: () => Promise<T>): Promise<T> {
    const ownerId = `${process.env.HOSTNAME}:${process.pid}:${Math.random()}`;
    const acquired = await this.lock.acquire(key, ttlMs, ownerId);
    
    if (!acquired) {
      throw new Error('Could not acquire lock');
    }

    // Auto-renewal
    const renewInterval = setInterval(async () => {
      try {
        await this.lock.extend(key, ownerId, ttlMs);
      } catch {
        // Ignore renewal failures
      }
    }, ttlMs / 2);

    try {
      return await fn();
    } finally {
      clearInterval(renewInterval);
      await this.lock.release(key, ownerId);
    }
  }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Distributed Transactions** - Avoid 2PC; use Saga pattern
- [ ] **Clock Synchronization** - Use NTP; logical clocks for ordering
- [ ] **Network Partitions** - Design for partition tolerance (CP or AP)
- [ ] **Thundering Herd** - Cache stampede; use locks, probabilistic refresh
- [ ] **Split Brain** - Quorum requirements, odd-numbered clusters
- [ ] **Data Consistency** - Eventual consistency ≠ immediate consistency
- [ ] **Idempotency** - Critical for retries, exactly-once semantics
- [ ] **Backpressure** - Handle producer > consumer speed
- [ ] **Cascading Failures** - Circuit breakers, bulkheads, timeouts
- [ ] **Observability** - Correlation IDs, distributed tracing mandatory

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Saga Pattern Implementation**
```typescript
// Order Saga: Create Order → Reserve Payment → Reserve Inventory → Confirm Shipping
type SagaStep<T> = {
  name: string;
  execute: (data: T) => Promise<{ compensationData?: any }>;
  compensate: (data: any) => Promise<void>;
};

class SagaOrchestrator<T> {
  private steps: SagaStep<T>[] = [];
  private executedSteps: Array<{ name: string; compensationData: any }> = [];

  addStep(step: SagaStep<T>): this {
    this.steps.push(step);
    return this;
  }

  async execute(initialData: T): Promise<{ success: boolean; data?: T; error?: Error }> {
    try {
      let data = initialData;
      for (const step of this.steps) {
        const result = await step.execute(data);
        this.executedSteps.push({ name: step.name, compensationData: result.compensationData });
        data = { ...data, ...result };
      }
      return { success: true, data };
    } catch (error) {
      // Compensate in reverse order
      for (const executed of this.executedSteps.reverse()) {
        const step = this.steps.find(s => s.name === executed.name);
        if (step) {
          try {
            await step.compensate(executed.compensationData);
          } catch (compError) {
            // Log compensation failure, alert ops
            console.error(`Compensation failed for ${executed.name}:`, compError);
          }
        }
      }
      return { success: false, error: error as Error };
    }
  }
}

// Usage
const orderSaga = new SagaOrchestrator<OrderData>()
  .addStep({
    name: 'createOrder',
    execute: async (data) => {
      const order = await orderService.create(data);
      return { compensationData: { orderId: order.id } };
    },
    compensate: async (data) => {
      await orderService.cancel(data.orderId);
    },
  })
  .addStep({
    name: 'reservePayment',
    execute: async (data) => {
      const payment = await paymentService.reserve(data.orderId, data.amount);
      return { compensationData: { paymentId: payment.id } };
    },
    compensate: async (data) => {
      await paymentService.refund(data.paymentId);
    },
  })
  .addStep({
    name: 'reserveInventory',
    execute: async (data) => {
      await inventoryService.reserve(data.orderId, data.items);
      return { compensationData: { items: data.items } };
    },
    compensate: async (data) => {
      await inventoryService.release(data.items);
    },
  });

await orderSaga.execute({ userId: '123', items: [...], amount: 99.99 });
```

#### 2. **Circuit Breaker**
```typescript
enum CircuitState { CLOSED, OPEN, HALF_OPEN }

class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failures = 0;
  private lastFailure = 0;
  private successCount = 0;

  constructor(
    private failureThreshold = 5,
    private timeout = 30000, // 30s
    private halfOpenRequests = 3
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.lastFailure > this.timeout) {
        this.state = CircuitState.HALF_OPEN;
        this.successCount = 0;
      } else {
        throw new Error('Circuit breaker OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failures = 0;
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.halfOpenRequests) {
        this.state = CircuitState.CLOSED;
      }
    }
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.failureThreshold) {
      this.state = CircuitState.OPEN;
    }
  }
}
```

---

### PRACTICE PROBLEMS

1. **Design** a distributed unique ID generator (Snowflake, ULID, UUID v7)
2. **Build** a distributed configuration system (etcd/Consul + watchers)
3. **Create** a leader election system (Redis, etcd, ZooKeeper)
4. **Implement** a distributed rate limiter with Redis Cluster
5. **Design** a multi-region active-active database architecture

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| CAP Theorem | Choose 2 of 3: Consistency, Availability, Partition Tolerance |
| PACELC | If P→A or C; Else→Latency or Consistency |
| Consistent Hashing | Virtual nodes on ring; minimal reshuffling on add/remove |
| Sharding Strategies | Hash, Range, Directory, Consistent Hashing |
| Saga Pattern | Choreography (events) or Orchestration (central coordinator) |
| Circuit Breaker | Closed→Open→Half-Open; prevent cascade failures |
| Bulkhead | Isolate resources (thread pools, connections) per service |
| CQRS | Separate read/write models; optimize each |
| Event Sourcing | Store events, not state; rebuild state from events |
| Idempotency Key | Client-generated unique key; server tracks processed keys |
| Backpressure | Handle fast producer, slow consumer (buffer, drop, signal) |
| Sidecar Pattern | Helper container per service (proxy, logging, config) |

---

## NEXT TOPIC: `07-MACHINE-LEARNING.md`

> **Study Tip**: Practice drawing architecture diagrams for: URL shortener, Rate limiter, Notification system, Chat app, Feed system. Time yourself: 30 min design, 15 min deep dive.