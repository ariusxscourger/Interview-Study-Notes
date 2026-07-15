# CACHING STRATEGIES & IMPLEMENTATION
## Exam-Style Study Notes

---

## TOPIC: Caching in Distributed Systems

### WHY? (Problem Statement)

**Without Caching:**
- Every request hits database → High latency, high load
- Repeated computations → Wasted CPU
- External API calls → Rate limits, latency, costs
- No resilience → Downstream failure = total failure

**What Caching Solves:**
| Problem | Solution |
|---------|----------|
| Latency | Sub-millisecond reads (Redis: ~0.5ms) |
| Database load | 90%+ read reduction typical |
| External API limits | Local cache with TTL |
| Computation reuse | Memoization, pre-computation |
| Resilience | Stale data better than errors |

**Real-World Analogy:**
- Database = Library (slow, authoritative)
- Cache = Desk reference books (fast, limited, occasionally stale)

---

### HOW? (Caching Patterns & Mechanisms)

#### 1. **Cache Patterns**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CACHE PATTERNS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CACHE-ASIDE (Lazy Loading)          READ-THROUGH               │
│  ─────────────────────────           ─────────────              │
│  App checks cache                      App calls cache          │
│  Miss → App loads from DB              Miss → Cache loads DB    │
│  App stores in cache                   Cache returns data       │
│  Simple, App controls                  Transparent to App       │
│                                                                  │
│  WRITE-THROUGH                       WRITE-BEHIND               │
│  ───────────────                      ─────────────             │
│  App writes to cache                   App writes to cache      │
│  Cache writes to DB                    Cache async writes DB    │
│  Consistent, slower                    Fast, eventual           │
│                                                                  │
│  REFRESH-AHEAD                       WRITE-AROUND               │
│  ──────────────                       ───────────               │
│  Async refresh before expiry          Write to DB only          │
│  Predictable access patterns          Cache invalidated         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. **Cache Invalidation Strategies**

```
INVALIDATION METHODS:
────────────────────
1. TTL (Time-To-Live)          - Auto-expire, simple, stale reads possible
2. Explicit Invalidation       - On write, delete/update cache keys
3. Version/Tag Based           - Increment version, cache key includes version
4. Event-Driven (CDC)          - Database changes → invalidate cache
5. Probabilistic               - Random early expiry (prevent thundering herd)

CACHE KEY DESIGN:
────────────────
user:{id}:profile          → TTL 1hr, invalidate on profile update
user:{id}:posts:{page}     → TTL 5min, invalidate on new post
post:{id}:comments         → TTL 10min, invalidate on comment
feed:{userId}              → TTL 15min, rebuild async
```

---

### WHAT? (Implementation Details)

#### 1. **Redis Cache Implementation**

```typescript
// Cache Service with Multiple Patterns
class CacheService {
  constructor(private redis: Redis) {}

  // ============ CACHE-ASIDE ============
  async getOrSet<T>(key: string, factory: () => Promise<T>, ttl = 300): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached);

    const value = await factory();
    await this.redis.setex(key, ttl, JSON.stringify(value));
    return value;
  }

  // ============ READ-THROUGH ============
  async readThrough<T>(key: string, loader: (key: string) => Promise<T>, ttl = 300): Promise<T> {
    // Lua script for atomic read-through
    const script = `
      local val = redis.call('GET', KEYS[1])
      if val then return val end
      -- Can't call external loader from Lua, handle in app
      return nil
    `;
    const cached = await this.redis.eval(script, 1, key);
    if (cached) return JSON.parse(cached);

    const value = await loader(key);
    await this.redis.setex(key, ttl, JSON.stringify(value));
    return value;
  }

  // ============ WRITE-THROUGH ============
  async writeThrough<T>(key: string, value: T, writer: (value: T) => Promise<void>, ttl = 300): Promise<void> {
    // Write to DB first, then cache (or use transaction)
    await writer(value);
    await this.redis.setex(key, ttl, JSON.stringify(value));
  }

  // ============ WRITE-BEHIND (Async) ============
  private writeQueue: Map<string, { value: any; writer: Function; retries: number }> = new Map();

  async writeBehind<T>(key: string, value: T, writer: (value: T) => Promise<void>, ttl = 300): Promise<void> {
    // Immediate cache write
    await this.redis.setex(key, ttl, JSON.stringify(value));
    
    // Queue DB write
    this.writeQueue.set(key, { value, writer, retries: 0 });
    this.processWriteQueue();
  }

  private async processWriteQueue(): Promise<void> {
    for (const [key, item] of this.writeQueue) {
      try {
        await item.writer(item.value);
        this.writeQueue.delete(key);
      } catch (err) {
        item.retries++;
        if (item.retries >= 3) {
          // Move to dead letter queue
          await this.redis.lpush('dlq:writes', JSON.stringify({ key, ...item }));
          this.writeQueue.delete(key);
        }
      }
    }
  }

  // ============ INVALIDATION ============
  async invalidate(pattern: string): Promise<number> {
    let cursor = '0';
    let deleted = 0;
    do {
      const [nextCursor, keys] = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = nextCursor;
      if (keys.length > 0) {
        deleted += await this.redis.del(...keys);
      }
    } while (cursor !== '0');
    return deleted;
  }

  async invalidateByTags(tags: string[]): Promise<void> {
    // Tag-based invalidation: key → set of tags
    for (const tag of tags) {
      const keys = await this.redis.smembers(`tag:${tag}`);
      if (keys.length > 0) {
        await this.redis.del(...keys);
        await this.redis.del(`tag:${tag}`);
      }
    }
  }

  // Tag a key
  async tagKey(key: string, tags: string[]): Promise<void> {
    for (const tag of tags) {
      await this.redis.sadd(`tag:${tag}`, key);
    }
  }
}
```

#### 2. **Multi-Level Caching (L1 + L2)**

```typescript
class MultiLevelCache {
  constructor(
    private l1Cache: Map<string, { value: any; expires: number }>, // In-memory
    private l2Cache: CacheService, // Redis
    private l1MaxSize = 1000,
    private l1DefaultTtl = 60 // seconds
  ) {}

  async get<T>(key: string): Promise<T | null> {
    // L1 Check
    const l1Entry = this.l1Cache.get(key);
    if (l1Entry && l1Entry.expires > Date.now()) {
      return l1Entry.value;
    }

    // L2 Check
    const l2Value = await this.l2Cache.get<T>(key);
    if (l2Value) {
      // Populate L1
      this.setL1(key, l2Value);
      return l2Value;
    }

    return null;
  }

  async set<T>(key: string, value: T, ttl = 300): Promise<void> {
    // Write to both
    await Promise.all([
      this.setL1(key, value, Math.min(ttl, this.l1DefaultTtl)),
      this.l2Cache.set(key, value, ttl),
    ]);
  }

  private setL1(key: string, value: any, ttl: number): void {
    // LRU eviction if needed
    if (this.l1Cache.size >= this.l1MaxSize) {
      const firstKey = this.l1Cache.keys().next().value;
      this.l1Cache.delete(firstKey);
    }
    this.l1Cache.set(key, { value, expires: Date.now() + ttl * 1000 });
  }
}
```

#### 3. **Cache Stampede Prevention**

```typescript
class StampedeProtection {
  constructor(private redis: Redis) {}

  // Method 1: Probabilistic Early Expiration
  async getWithProbabilisticExpiry<T>(
    key: string, 
    factory: () => Promise<T>, 
    ttl = 300,
    beta = 1.0 // Higher = more aggressive early refresh
  ): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) {
      const { value, expiresAt } = JSON.parse(cached);
      const now = Date.now();
      
      // Probabilistic early refresh
      if (expiresAt - now < ttl * 1000 * 0.1) { // Within 10% of TTL
        const probability = Math.min(1, beta * (ttl * 1000) / (expiresAt - now));
        if (Math.random() < probability) {
          // Trigger async refresh, return stale
          this.refreshAsync(key, factory, ttl);
        }
      }
      return value;
    }
    
    return this.setWithLock(key, factory, ttl);
  }

  // Method 2: Mutex/Lock for Single Flight
  private async setWithLock<T>(key: string, factory: () => Promise<T>, ttl: number): Promise<T> {
    const lockKey = `lock:${key}`;
    const lockValue = `${Date.now()}:${Math.random()}`;
    
    const acquired = await this.redis.set(lockKey, lockValue, 'NX', 'EX', 10);
    if (!acquired) {
      // Wait for lock holder to finish
      await this.waitForLock(key);
      return this.get(key); // Recursive call
    }

    try {
      const value = await factory();
      await this.redis.setex(key, ttl, JSON.stringify({ 
        value, 
        expiresAt: Date.now() + ttl * 1000 
      }));
      return value;
    } finally {
      // Release lock only if we own it
      const script = `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
          return redis.call("DEL", KEYS[1])
        else
          return 0
        end
      `;
      await this.redis.eval(script, 1, lockKey, lockValue);
    }
  }

  private async waitForLock(key: string, maxWait = 5000): Promise<void> {
    const start = Date.now();
    while (Date.now() - start < maxWait) {
      const exists = await this.redis.exists(`lock:${key}`);
      if (!exists) return;
      await new Promise(r => setTimeout(r, 10));
    }
  }

  private async refreshAsync<T>(key: string, factory: () => Promise<T>, ttl: number): Promise<void> {
    try {
      const value = await factory();
      await this.redis.setex(key, ttl, JSON.stringify({ 
        value, 
        expiresAt: Date.now() + ttl * 1000 
      }));
    } catch (err) {
      console.error('Async refresh failed:', err);
    }
  }
}
```

#### 4. **HTTP Caching (CDN/Edge)**

```typescript
// Express Middleware for HTTP Caching
function httpCache(options: { 
  maxAge: number; 
  staleWhileRevalidate?: number; 
  vary?: string[];
} = { maxAge: 60 }) {
  return (req: Request, res: Response, next: NextFunction) => {
    // Only cache GET/HEAD
    if (!['GET', 'HEAD'].includes(req.method)) {
      return next();
    }

    const key = `http:${req.originalUrl}:${req.headers['accept-encoding'] || ''}`;
    
    // Generate ETag from content hash
    const originalSend = res.send;
    res.send = function(body: any) {
      if (res.statusCode === 200 && req.method === 'GET') {
        const etag = crypto.createHash('md5').update(JSON.stringify(body)).digest('hex');
        res.setHeader('ETag', `"${etag}"`);
      }
      return originalSend.call(this, body);
    };

    // Check If-None-Match
    const ifNoneMatch = req.headers['if-none-match'];
    if (ifNoneMatch) {
      // Would need to check cached ETag
      // For simplicity, skip in middleware
    }

    // Set Cache-Control
    const directives = [`max-age=${options.maxAge}`];
    if (options.staleWhileRevalidate) {
      directives.push(`stale-while-revalidate=${options.staleWhileRevalidate}`);
    }
    if (options.vary) {
      res.setHeader('Vary', options.vary.join(', '));
    }
    res.setHeader('Cache-Control', directives.join(', '));

    next();
  };
}

// Usage
app.get('/api/products', httpCache({ 
  maxAge: 60, 
  staleWhileRevalidate: 300,
  vary: ['Accept', 'Authorization']
}), getProducts);

// Cache Invalidation Endpoint
app.post('/api/cache/invalidate', authenticate, authorize('admin'), async (req, res) => {
  const { pattern } = req.body;
  const deleted = await cache.invalidate(pattern);
  res.json({ deleted });
});
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Cache-Aside for reads** | Cache everything blindly | Cache pollution, memory waste |
| **TTL + Invalidation** | Only TTL | Stale data served |
| **Cache key versioning** | Manual key deletion | Missed invalidations |
| **Async write-behind** | Sync write-through for high write | Write latency |
| **Local L1 + Redis L2** | Only Redis | Network latency every read |
| **Probabilistic refresh** | Fixed TTL only | Thundering herd |
| **Cache warming** | Cold start | Poor first-user experience |
| **Circuit breaker** | No fallback | Cascading failures |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Explain Cache Aside vs Read Through vs Write Through"

```
CACHE-ASIDE (Lazy Loading):
App → Cache (miss) → App → DB → App → Cache → App
- App manages cache logic
- Simple, flexible
- Cache miss = 3 hops

READ-THROUGH:
App → Cache (miss) → Cache → DB → Cache → App
- Cache manages loading
- Transparent to app
- Cache library handles logic

WRITE-THROUGH:
App → Cache → DB (sync) → App
- Data always consistent
- Write latency = cache + DB
- Good for critical data

WRITE-BEHIND (Write-Back):
App → Cache → (async) → DB
- Fast writes
- Risk of data loss
- Complex (queue, retry, DLQ)
```

#### 2. **Code**: "Implement a distributed rate limiter with Redis"

```typescript
class RateLimiter {
  constructor(private redis: Redis) {}

  // Sliding Window Log (Precise)
  async slidingWindowLog(key: string, limit: number, windowMs: number): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
    const now = Date.now();
    const windowStart = now - windowMs;
    const redisKey = `ratelimit:${key}`;

    const script = `
      -- Remove expired entries
      redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])
      -- Count current
      local count = redis.call('ZCARD', KEYS[1])
      if count >= tonumber(ARGV[3]) then
        return {0, count}
      end
      -- Add current request
      redis.call('ZADD', KEYS[1], ARGV[2], ARGV[2])
      redis.call('EXPIRE', KEYS[1], ARGV[4])
      return {1, count + 1}
    `;

    const result = await this.redis.eval(
      script, 
      1, 
      redisKey, 
      windowStart, 
      now, 
      limit, 
      Math.ceil(windowMs / 1000) + 1
    ) as [number, number];

    return {
      allowed: result[0] === 1,
      remaining: Math.max(0, limit - result[1]),
      resetAt: now + windowMs
    };
  }

  // Sliding Window Counter (Approximate, More Efficient)
  async slidingWindowCounter(key: string, limit: number, windowMs: number): Promise<{ allowed: boolean; remaining: number }> {
    const now = Date.now();
    const currentWindow = Math.floor(now / windowMs);
    const previousWindow = currentWindow - 1;
    
    const script = `
      local current = redis.call('GET', KEYS[1] .. ':' .. ARGV[1])
      local previous = redis.call('GET', KEYS[1] .. ':' .. ARGV[2])
      current = tonumber(current) or 0
      previous = tonumber(previous) or 0
      local weight = 1 - (tonumber(ARGV[3]) / tonumber(ARGV[4]))
      local count = math.floor(current + previous * weight)
      if count >= tonumber(ARGV[5]) then
        return {0, count}
      end
      redis.call('INCR', KEYS[1] .. ':' .. ARGV[1])
      redis.call('EXPIRE', KEYS[1] .. ':' .. ARGV[1], ARGV[6])
      return {1, count + 1}
    `;

    const result = await this.redis.eval(
      script,
      1,
      `ratelimit:${key}`,
      currentWindow,
      previousWindow,
      now % windowMs,
      windowMs,
      limit,
      Math.ceil(windowMs / 1000) * 2
    ) as [number, number];

    return { allowed: result[0] === 1, remaining: Math.max(0, limit - result[1]) };
  }

  // Token Bucket (For Smooth Rate Limiting)
  async tokenBucket(key: string, capacity: number, refillRate: number): Promise<{ allowed: boolean; tokens: number }> {
    const redisKey = `tokenbucket:${key}`;
    const now = Date.now();

    const script = `
      local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'lastRefill')
      local tokens = tonumber(bucket[1]) or tonumber(ARGV[1])
      local lastRefill = tonumber(bucket[2]) or tonumber(ARGV[2])
      
      local now = tonumber(ARGV[3])
      local elapsed = (now - lastRefill) / 1000
      local refill = elapsed * tonumber(ARGV[4])
      tokens = math.min(tonumber(ARGV[1]), tokens + refill)
      
      if tokens >= 1 then
        tokens = tokens - 1
        redis.call('HMSET', KEYS[1], 'tokens', tokens, 'lastRefill', now)
        redis.call('EXPIRE', KEYS[1], ARGV[5])
        return {1, tokens}
      else
        redis.call('HMSET', KEYS[1], 'tokens', tokens, 'lastRefill', lastRefill)
        redis.call('EXPIRE', KEYS[1], ARGV[5])
        return {0, tokens}
      end
    `;

    const result = await this.redis.eval(
      script,
      1,
      `tokenbucket:${key}`,
      capacity,
      now,
      now,
      refillRate,
      Math.ceil(capacity / refillRate) + 10
    ) as [number, number];

    return { allowed: result[0] === 1, tokens: result[1] };
  }
}
```

#### 3. **Design**: "Design a caching layer for a high-traffic e-commerce product catalog"

```
REQUIREMENTS:
- 10M products, 100K RPS reads, 1K RPS writes
- Product data: name, price, inventory, images, variants
- Sub-10ms p99 latency
- Strong consistency for price/inventory

ARCHITECTURE:

┌─────────────────────────────────────────────────────────────┐
│                     API GATEWAY                               │
└────────────────────────────┬────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
       ┌─────────────┐               ┌─────────────┐
       │  CDN/Edge   │               │  App Cache  │
       │  (Cloudflare)               │  (Redis)    │
       │  TTL: 5min                  │  L1: 100ms  │
       │  Purge on update            │  L2: 5min   │
       └─────────────┘               └──────┬──────┘
                                            │
                    ┌───────────────────────┼───────────────────┐
                    ▼                       ▼                   ▼
             ┌─────────────┐          ┌─────────────┐    ┌─────────────┐
             │  Primary DB │          │  Search     │    │  Analytics  │
             │  (PostgreSQL)           │  (Elastic)  │    │  (ClickHouse)│
             │  Price/Stock            │  Full-text  │    │  Aggregates │
             └─────────────┘          └─────────────┘    └─────────────┘

CACHE STRATEGY:
1. PRODUCT DETAIL: Cache-Aside, TTL 5min, Invalidate on update
   Key: product:{id}:detail → {id, name, price, stock, images, variants}

2. PRODUCT LIST: Cache-Aside, TTL 1min, Tag-based invalidation
   Key: products:list:{filters}:{page} → [{id, name, price, thumbnail}]

3. INVENTORY: Read-Through, TTL 30s, Write-Through for updates
   Key: inventory:{id} → {available, reserved, incoming}

4. PRICE: Write-Through, TTL 1min, Strong consistency
   Key: price:{id} → {current, original, currency, saleEnd}

INVALIDATION:
- Product update → Invalidate product:{id}:* + products:list:*
- Price change → Invalidate price:{id} + product:{id}:detail
- Stock change → Invalidate inventory:{id} + product:{id}:detail

CACHE WARMING:
- Daily job: Pre-load top 10K products
- On deploy: Warm new instances before traffic
- Black Friday: Extend TTLs, pre-load all

MONITORING:
- Hit rate > 95%
- p99 latency < 5ms
- Memory usage < 80%
- Eviction rate < 1%
```

#### 4. **Debugging**: "Cache hit rate is 60% - how to improve?"

```
DEBUGGING CHECKLIST:
────────────────────
1. ANALYZE KEYS:
   - SCAN for patterns
   - Check TTL distribution
   - Identify hot keys (MONITOR, SLOWLOG)

2. COMMON CAUSES:
   □ Too short TTL
   □ Overly granular keys (per-user, per-request)
   □ Missing cache warming
   □ Cache bypass paths (admin, preview)
   □ Invalidation too aggressive
   □ Wrong cache pattern (cache-aside for static data)

3. QUICK WINS:
   □ Increase TTL for stable data
   □ Consolidate keys (merge per-user into shared)
   □ Add cache warming job
   □ Implement L1 cache for hottest keys
   □ Use probabilistic early expiration

4. MEASURE:
   - Hit rate by key pattern
   - Latency distribution
   - Memory fragmentation
   - Eviction rate
```

---

### GOTCHAS & EDGE CASES

- [ ] **Thundering Herd** - Use locks, probabilistic refresh, or stale-while-revalidate
- [ ] **Cache Pollution** - Don't cache personalized/low-reuse data
- [ ] **Stale Reads** - Acceptable for most reads, not for payments/inventory
- [ ] **Memory Fragmentation** - Redis defrag, avoid huge objects
- [ ] **Hot Key Skew** - Use local L1 cache, request collapsing
- [ ] **TTL Synchronization** - Related keys should expire together
- [ ] **Cache Invalidation** - "Two hard things: cache invalidation, naming, off-by-one"
- [ ] **Redis Single Thread** - Avoid blocking ops (KEYS, FLUSHALL, complex Lua)
- [ ] **Connection Pooling** - Reuse connections, pipeline commands
- [ ] **Serialization** - JSON vs MessagePack vs Protobuf (size/speed tradeoff)

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Redis Cache with All Patterns**
```typescript
class RedisCache {
  constructor(private redis: Redis) {}

  // Cache key with version for easy invalidation
  versionedKey(base: string, version: string | number): string {
    return `${base}:v${version}`;
  }

  // Get with automatic version resolution
  async get<T>(baseKey: string, currentVersion: string): Promise<T | null> {
    return this.redis.get(this.versionedKey(baseKey, currentVersion)) as Promise<T | null>;
  }

  // Set with version
  async set<T>(baseKey: string, version: string, value: T, ttl: number): Promise<void> {
    await this.redis.setex(this.versionedKey(baseKey, version), ttl, JSON.stringify(value));
  }

  // Increment version (invalidates all old keys)
  async bumpVersion(baseKey: string): Promise<string> {
    const newVersion = await this.redis.incr(`version:${baseKey}`);
    return newVersion.toString();
  }

  // Atomic get-or-set with lock
  async getOrSetWithLock<T>(key: string, factory: () => Promise<T>, ttl: number): Promise<T> {
    const lockKey = `lock:${key}`;
    const lockValue = `${Date.now()}:${Math.random()}`;
    
    const acquired = await this.redis.set(lockKey, lockValue, 'NX', 'EX', 10);
    if (!acquired) {
      await this.waitForLock(key);
      const existing = await this.redis.get(key);
      if (existing) return JSON.parse(existing);
      return this.getOrSetWithLock(key, factory, ttl); // Retry
    }

    try {
      const value = await factory();
      await this.redis.setex(key, ttl, JSON.stringify(value));
      return value;
    } finally {
      await this.releaseLock(lockKey, lockValue);
    }
  }

  private async waitForLock(key: string): Promise<void> {
    const lockKey = `lock:${key}`;
    for (let i = 0; i < 50; i++) {
      if (!(await this.redis.exists(lockKey))) return;
      await new Promise(r => setTimeout(r, 20));
    }
  }

  private async releaseLock(lockKey: string, lockValue: string): Promise<void> {
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;
    await this.redis.eval(script, 1, lockKey, lockValue);
  }
}
```

#### 2. **Cache Decorator for Methods**
```typescript
function Cached(options: { 
  key: (args: any[]) => string; 
  ttl: number; 
  condition?: (result: any) => boolean;
} = { key: () => '', ttl: 300 }) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    const cache = new RedisCache(redisClient);

    descriptor.value = async function (...args: any[]) {
      const cacheKey = options.key(args);
      if (!cacheKey) return originalMethod.apply(this, args);

      const cached = await cache.get(cacheKey);
      if (cached) return cached;

      const result = await originalMethod.apply(this, args);
      
      if (!options.condition || options.condition(result)) {
        await cache.set(cacheKey, result, options.ttl);
      }
      
      return result;
    };

    return descriptor;
  };
}

// Usage
class ProductService {
  @Cached({ key: (args) => `product:${args[0]}`, ttl: 300 })
  async getProduct(id: string): Promise<Product> { /* ... */ }

  @Cached({ key: (args) => `products:list:${JSON.stringify(args[0])}`, ttl: 60 })
  async listProducts(filters: ProductFilters): Promise<Product[]> { /* ... */ }
}
```

---

### PRACTICE PROBLEMS

1. **Implement** a distributed cache with consistent hashing for sharding
2. **Build** a cache warming system that pre-loads based on access patterns
3. **Design** a cache invalidation system using database change data capture (CDC)
4. **Optimize** Redis for a write-heavy workload (pipeline, batch, Lua)
5. **Create** a multi-region cache with active-active replication

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Cache Patterns | Cache-Aside, Read-Through, Write-Through, Write-Behind, Refresh-Ahead |
| Cache Invalidation | TTL, Explicit, Versioning, Event-Driven, Probabilistic |
| Thundering Herd | Many requests for expired key → DB overload. Fix: locks, probabilistic refresh, stale-while-revalidate |
| Write-Through vs Write-Behind | Through: sync, consistent, slower. Behind: async, fast, risk of loss |
| L1 + L2 Cache | L1: In-memory (fast, small). L2: Redis (shared, larger). Check L1 → L2 → DB |
| Cache Key Design | Include version, namespace, params. `user:123:profile:v5` |
| Redis Eviction Policies | volatile-lru, allkeys-lru, volatile-ttl, noeviction |
| Cache Hit Rate | Target >95% for read-heavy, >80% for mixed |
| Stale-While-Revalidate | Serve stale content while async fetching fresh |
| Probabilistic Early Expiration | Randomly refresh before TTL expires to spread load |

---

## NEXT TOPIC: `04-AUTH-AUTHZ.md`

> **Study Tip**: Set up Redis locally. Practice: Cache-Aside, Write-Through, TTL + Invalidation, Distributed Lock, Rate Limiter.