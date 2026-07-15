# DATABASES - NOSQL FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: NoSQL Databases

### WHY? (Problem Statement)

**When SQL Isn't Enough:**
- Horizontal scaling (sharding) difficult with ACID
- Schema rigidity for rapidly changing data
- High write throughput (time-series, logs)
- Complex hierarchical/graph data
- Geo-distributed low-latency reads

**What NoSQL Solves:**
| Problem | NoSQL Solution |
|---------|----------------|
| Horizontal scale | Automatic sharding, replication |
| Flexible schema | Dynamic/document model |
| High write throughput | LSM trees, append-only |
| Specialized queries | Graph traversals, time-series |
| Global distribution | Multi-region active-active |

**Real-World Analogy:**
- SQL = Filing cabinet (structured, indexed, relationships)
- Key-Value = Coat check (fast, simple, no queries)
- Document = Binder of forms (semi-structured, nested)
- Column-Family = Warehouse shelves (columnar, time-series)
- Graph = Social network map (relationships first)

---

### HOW? (Internal Mechanisms)

#### 1. **CAP Theorem (Brewer's Theorem)**

```
        CONSISTENCY
           /\
          /  \
         /    \
        /      \
   AVAILABLE  PARTITION TOLERANCE
```

| System | CP (Consistency + Partition) | AP (Availability + Partition) |
|--------|------------------------------|-------------------------------|
| **MongoDB** (default) | ✓ | |
| **Redis** (cluster) | ✓ | |
| **Cassandra** | | ✓ |
| **DynamoDB** | | ✓ |
| **PostgreSQL** | ✓ | |

**PACELC Extension:** If Partition → C or A. Else (no partition) → Latency or Consistency.

#### 2. **Storage Engines**

```
LSM Tree (Log-Structured Merge)          B-Tree
─────────────────────────                ─────────
Write: Append to MemTable (RAM)          Write: In-place update
Flush: SSTable (immutable, sorted)       Read: Tree traversal
Compaction: Merge SSTables               Balance: Rebalance nodes
Read: Check MemTable → SSTables          Write amplification: Higher
Optimized for: Writes                    Optimized for: Reads

Used by: Cassandra, RocksDB, DynamoDB    Used by: PostgreSQL, MySQL, MongoDB (WiredTiger)
```

---

### WHAT? (Database Types & Usage)

#### 1. **Document Databases (MongoDB)**

```javascript
// Flexible Schema
{
  _id: ObjectId("..."),
  name: "Alice",
  email: "alice@example.com",
  roles: ["user", "premium"],
  address: {
    street: "123 Main St",
    city: "NYC",
    zip: "10001"
  },
  orders: [
    { orderId: "ORD-1", total: 99.99, items: [...] },
    { orderId: "ORD-2", total: 149.50, items: [...] }
  ],
  createdAt: ISODate("2024-01-15"),
  metadata: { source: "web", campaign: "summer2024" }
}

// Indexes
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ "address.city": 1, "address.zip": 1 });
db.users.createIndex({ roles: 1 }); // Multi-key index
db.users.createIndex({ createdAt: -1 }); // TTL
db.users.createIndex({ name: "text", email: "text" }); // Text search

// Aggregation Pipeline
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2024-01-01") } } },
  { $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "user" } },
  { $unwind: "$user" },
  { $group: { 
      _id: "$user.email", 
      total: { $sum: "$total" },
      count: { $sum: 1 },
      avgOrder: { $avg: "$total" }
  }},
  { $sort: { total: -1 } },
  { $limit: 10 }
]);

// Transactions (Replica Set / Sharded Cluster)
const session = client.startSession();
await session.withTransaction(async () => {
  await users.updateOne({ _id: userId }, { $inc: { balance: -amount } }, { session });
  await orders.insertOne({ userId, items, total: amount }, { session });
  await inventory.updateMany({ _id: { $in: itemIds } }, { $inc: { reserved: 1 } }, { session });
});
```

#### 2. **Key-Value Stores (Redis)**

```bash
# Strings
SET user:1001 '{"name":"Alice","email":"alice@example.com"}' EX 3600
GET user:1001
MGET user:1001 user:1002
INCR counter:pageviews

# Hashes (Objects)
HSET user:1001 name "Alice" email "alice@example.com" role "admin"
HGETALL user:1001
HMGET user:1001 name email

# Lists (Queues/Stacks)
LPUSH queue:emails '{"to":"bob@x.com","subject":"Welcome"}'
RPOP queue:emails
LRANGE queue:emails 0 -1

# Sets (Unique)
SADD tags:post:123 "redis" "nosql" "performance"
SINTER tags:post:123 tags:post:456
SISMEMBER tags:post:123 "redis"

# Sorted Sets (Leaderboards)
ZADD leaderboard:game1 1500 "alice" 2300 "bob" 1800 "charlie"
ZRANGE leaderboard:game1 0 9 WITHSCORES REV  # Top 10
ZRANK leaderboard:game1 "alice"

# Pub/Sub
SUBSCRIBE notifications:user:1001
PUBLISH notifications:user:1001 '{"type":"message","data":{...}}'

# Streams (Kafka-like)
XADD events:orders * userId 1001 amount 99.99 status "created"
XREAD COUNT 10 BLOCK 5000 STREAMS events:orders $

# Lua Scripts (Atomic)
EVAL "local current = redis.call('GET', KEYS[1]) 
      if current then return redis.call('INCR', KEYS[1]) else return redis.call('SET', KEYS[1], 1) end" 1 counter:api
```

#### 3. **Wide-Column / Column-Family (Cassandra/DynamoDB)**

```sql
-- Cassandra CQL
CREATE KEYSPACE nanolite 
WITH REPLICATION = {'class': 'NetworkTopologyStrategy', 'dc1': 3, 'dc2': 3};

CREATE TABLE users (
    user_id UUID,
    email TEXT,
    name TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id)
) WITH default_time_to_live = 0;

CREATE TABLE user_posts (
    user_id UUID,
    post_id TIMEUUID,
    title TEXT,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy'};

-- Query by user, recent posts
SELECT * FROM user_posts WHERE user_id = ? LIMIT 10;

-- Materialized View (auto-maintained)
CREATE MATERIALIZED VIEW posts_by_tag AS
SELECT tag, post_id, user_id, title, created_at
FROM posts
WHERE tag IS NOT NULL AND post_id IS NOT NULL AND user_id IS NOT NULL
PRIMARY KEY (tag, created_at, post_id)
WITH CLUSTERING ORDER BY (created_at DESC);
```

```javascript
// DynamoDB (Single Table Design)
const table = "nanolite-entities";

await docClient.put({
  TableName: table,
  Item: {
    PK: "USER#alice@example.com",
    SK: "PROFILE",
    email: "alice@example.com",
    name: "Alice",
    role: "admin",
    createdAt: new Date().toISOString(),
    GSI1PK: "ROLE#admin",    // Global Secondary Index
    GSI1SK: "USER#alice@example.com"
  }
}).promise();

// Query by email (PK)
await docClient.query({
  TableName: table,
  KeyConditionExpression: "PK = :pk AND SK = :sk",
  ExpressionAttributeValues: { ":pk": "USER#alice@example.com", ":sk": "PROFILE" }
}).promise();

// Query all admins (GSI)
await docClient.query({
  TableName: table,
  IndexName: "GSI1",
  KeyConditionExpression: "GSI1PK = :pk",
  ExpressionAttributeValues: { ":pk": "ROLE#admin" }
}).promise();

// Transaction
await docClient.transactWrite({
  TransactItems: [
    { Put: { TableName: table, Item: orderItem } },
    { Update: { TableName: table, Key: { PK: "INVENTORY#123", SK: "STOCK" }, UpdateExpression: "SET quantity = quantity - :qty" } },
    { Put: { TableName: table, Item: { PK: "USER#alice", SK: `ORDER#${orderId}`, ... } } }
  ]
}).promise();
```

#### 4. **Graph Database (Neo4j)**

```cypher
// Nodes & Relationships
CREATE (alice:User {name: "Alice", email: "alice@example.com"})
CREATE (bob:User {name: "Bob", email: "bob@example.com"})
CREATE (post:Post {title: "Hello Graph", content: "Graph DBs are cool"})
CREATE (alice)-[:AUTHORED]->(post)
CREATE (bob)-[:LIKED {timestamp: datetime()}]->(post)
CREATE (alice)-[:FOLLOWS {since: datetime()}]->(bob)

// Queries
// Friends of friends
MATCH (u:User {email: "alice@example.com"})-[:FOLLOWS]->()-[:FOLLOWS]->(fof)
RETURN fof.name, fof.email

// Shortest path
MATCH (a:User {name: "Alice"}), (b:User {name: "Charlie"})
MATCH path = shortestPath((a)-[:FOLLOWS*]-(b))
RETURN path

// Recommendations
MATCH (u:User {email: "alice@example.com"})-[:FOLLOWS]->(f)-[:AUTHORED]->(p:Post)
WHERE NOT (u)-[:LIKED]->(p)
RETURN p, COUNT(DISTINCT f) as mutualFollowers
ORDER BY mutualFollowers DESC LIMIT 5

// Indexes
CREATE INDEX FOR (u:User) ON (u.email)
CREATE INDEX FOR (p:Post) ON (p.createdAt)
```

#### 5. **Time-Series (InfluxDB/TimescaleDB)**

```sql
-- TimescaleDB (PostgreSQL Extension)
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id UUID NOT NULL,
    cpu_usage DOUBLE PRECISION,
    memory_usage DOUBLE PRECISION,
    temperature DOUBLE PRECISION
);

SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '1 day');

-- Compression
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id'
);
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- Continuous Aggregates
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
       device_id,
       AVG(cpu_usage) AS avg_cpu,
       MAX(memory_usage) AS max_memory
FROM metrics
GROUP BY bucket, device_id;

-- Query
SELECT time_bucket('5 minutes', time) AS bucket,
       AVG(cpu_usage)
FROM metrics
WHERE time > NOW() - INTERVAL '1 hour'
  AND device_id = 'device-123'
GROUP BY bucket
ORDER BY bucket DESC;
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Right DB for job** | One DB for everything | Wrong tool = pain |
| **Denormalize for reads** | Normalize everything | Slow queries, joins |
| **Single table design (DynamoDB)** | Multiple tables | More requests, complexity |
| **Embedding (1-to-few)** | Embedding 1-to-many | Document size limits, duplication |
| **Referencing (1-to-many)** | Deep nesting | Multiple queries |
| **TTL for expiration** | Manual cleanup | Forgotten data, bloat |
| **Idempotent writes** | Blind writes | Duplication |
| **Circuit breaker** | No fallback | Cascade failure |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "When to choose MongoDB vs PostgreSQL?"

```
MONGODB:
✓ Rapidly changing schema
✓ Nested/hierarchical data (comments, configs)
✓ Horizontal scale from day 1
✓ JSON-native development
✓ Geo-spatial queries
✗ Complex transactions
✗ Strong consistency required
✗ Ad-hoc analytical queries

POSTGRESQL:
✓ Relational data, complex joins
✓ ACID transactions
✓ Ad-hoc queries, reporting
✓ Strong consistency
✓ Mature ecosystem, tooling
✗ Schema migrations painful
✗ Horizontal scale complex
```

#### 2. **Code**: "Redis Cache Invalidation Strategy"

```typescript
class CacheService {
  constructor(private redis: Redis) {}

  // Cache-Aside Pattern
  async getOrSet<T>(key: string, factory: () => Promise<T>, ttl = 300): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached);
    
    const value = await factory();
    await this.redis.setex(key, ttl, JSON.stringify(value));
    return value;
  }

  // Write-Through (cache updated on write)
  async set(key: string, value: any, ttl = 300): Promise<void> {
    await this.redis.setex(key, ttl, JSON.stringify(value));
  }

  // Invalidate on update
  async invalidate(pattern: string): Promise<void> {
    let cursor = '0';
    do {
      const [nextCursor, keys] = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      if (keys.length > 0) await this.redis.del(...keys);
      cursor = nextCursor;
    } while (cursor !== '0');
  }

  // Cache Stampede Protection (using Redis lock)
  async getOrSetWithLock<T>(key: string, factory: () => Promise<T>, ttl = 300): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached);

    const lockKey = `lock:${key}`;
    const lock = await this.redis.set(lockKey, '1', 'NX', 'EX', 10);
    if (!lock) {
      // Wait and retry
      await new Promise(r => setTimeout(r, 50));
      return this.getOrSetWithLock(key, factory, ttl);
    }

    try {
      const value = await factory();
      await this.redis.setex(key, ttl, JSON.stringify(value));
      return value;
    } finally {
      await this.redis.del(lockKey);
    }
  }
}
```

#### 3. **Design**: "Model a Social Feed in DynamoDB (Single Table)"

```
TABLE: SocialFeed

PK                    | SK                    | GSI1PK      | GSI1SK          | Data
────────────────────────────────────────────────────────────────────────────────
USER#alice            | PROFILE               | USER#alice  | PROFILE         | {name, avatar, bio}
USER#alice            | POST#uuid-1           | POST#uuid-1 | USER#alice      | {content, image, likes: 5, createdAt}
USER#alice            | POST#uuid-2           | POST#uuid-2 | USER#alice      | {...}
POST#uuid-1           | COMMENT#uuid-c1               | COMMENT#c1  | POST#uuid-1     | {user: bob, text: "Nice!", createdAt}
POST#uuid-1           | COMM-c2               | COMMENT#c2  | POST#uuid-1     | {...}
POST#uuid-1           | LIKE#bob              | LIKE#bob    | POST#uuid-1     | {user: bob, createdAt}
USER#alice            | FOLLOW#bob            | FOLLOW#bob  | USER#alice      | {createdAt}
USER#alice            | FOLLOW#charlie        | FOLLOW#charlie | USER#alice   | {...}

QUERIES:
1. Get user profile: PK=USER#alice, SK=PROFILE
2. User's posts: PK=USER#alice, SK begins_with POST#
3. Post comments: PK=POST#uuid-1, SK begins_with COMMENT#
4. Post likes: PK=POST#uuid-1, SK begins_with LIKE#
5. User's followers: GSI1PK=FOLLOW#bob (GSI1)
6. Feed: Query GSI1 for FOLLOW#* → get users → batch get their POSTs
```

#### 4. **Performance**: "Cassandra Data Modeling for Time-Series"

```sql
-- Sensor data: High write, time-range queries
CREATE TABLE sensor_readings (
    sensor_id UUID,
    day DATE,              -- Partition key (bucket by day)
    timestamp TIMESTAMP,   -- Clustering key
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    PRIMARY KEY ((sensor_id, day), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy'}
  AND default_time_to_live = 2592000;  -- 30 days TTL

-- Queries
-- Recent readings for sensor
SELECT * FROM sensor_readings 
WHERE sensor_id = ? AND day = ? 
ORDER BY timestamp DESC LIMIT 100;

-- Range query across days (requires ALLOW FILTERING or separate queries)
-- Better: Application queries each day partition
```

#### 5. **Architecture**: "Polyglot Persistence Architecture"

```
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY                              │
└────────────────────────────┬────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  PostgreSQL   │    │    MongoDB    │    │     Redis     │
│  (Users,      │    │  (Catalog,    │    │  (Sessions,   │
│   Orders,     │    │   Content,    │    │   Cache,      │
│   Payments)   │    │   Logs)       │    │   Rate Limit) │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Cassandra   │    │  Elasticsearch│    │   Kafka       │
│  (Events,     │    │  (Search,     │    │  (Event Bus,  │
│   Metrics)    │    │   Analytics)  │    │   Async)      │
└───────────────┘    └───────────────┘    └───────────────┘
```

---

### GOTCHAS & EDGE CASES

- [ ] **MongoDB 16MB document limit** - Use GridFS or references
- [ ] **Redis single-threaded** - Don't run expensive Lua scripts
- [ ] **Cassandra partition size** - Keep < 100MB, < 100k rows
- [ ] **DynamoDB 400KB item limit** - Split large items
- [ ] **Redis memory eviction** - Configure `maxmemory-policy`
- [ ] **Cassandra tombstones** - TTLs create tombstones, run repair
- [ ] **DynamoDB hot partitions** - Distribute keys, avoid sequential
- [ ] **Graph DB supernodes** - High-degree nodes kill traversals
- [ ] **Eventual consistency reads** - Don't read own writes immediately
- [ ] **Index overhead** - Each index slows writes, uses memory

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Redis Distributed Lock**
```typescript
async function acquireLock(redis: Redis, key: string, ttl = 30): Promise<boolean> {
  const result = await redis.set(`lock:${key}`, process.env.HOSTNAME, 'NX', 'EX', ttl);
  return result === 'OK';
}

async function releaseLock(redis: Redis, key: string): Promise<void> {
  // Only delete if we own it (Lua for atomicity)
  const script = `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
    else
      return 0
    end
  `;
  await redis.eval(script, 1, `lock:${key}`, process.env.HOSTNAME);
}

// Usage
async function withLock<T>(key: string, fn: () => Promise<T>): Promise<T> {
  const lock = await acquireLock(redis, key);
  if (!lock) throw new Error('Could not acquire lock');
  try {
    return await fn();
  } finally {
    await releaseLock(redis, key);
  }
}
```

#### 2. **MongoDB Change Streams (Real-time)**
```typescript
const pipeline = [
  { $match: { 'fullDocument.userId': userId, operationType: { $in: ['insert', 'update'] } } }
];

const changeStream = db.collection('orders').watch(pipeline);

changeStream.on('change', (change) => {
  if (change.operationType === 'insert') {
    notifyUser(userId, 'New order', change.fullDocument);
  } else if (change.operationType === 'update') {
    notifyUser(userId, 'Order updated', change.fullDocument);
  }
});

changeStream.on('error', (err) => {
  // Resume token for recovery
  console.error('Change stream error:', err);
});
```

#### 3. **DynamoDB Pagination**
```typescript
async function queryAllPages(params: QueryCommandInput): Promise<any[]> {
  const items: any[] = [];
  let lastEvaluatedKey: Record<string, any> | undefined;
  
  do {
    const result = await docClient.query({
      ...params,
      ExclusiveStartKey: lastEvaluatedKey,
    }).promise();
    
    items.push(...(result.Items || []));
    lastEvaluatedKey = result.LastEvaluatedKey;
  } while (lastEvaluatedKey);
  
  return items;
}
```

---

### PRACTICE PROBLEMS

1. **Design** a rate limiter using Redis (sliding window, fixed window, token bucket)
2. **Implement** a distributed session store with Redis Cluster
3. **Model** a product catalog with variants in MongoDB (embedded vs referenced)
4. **Build** a real-time leaderboard with Redis Sorted Sets
5. **Query** time-series data in TimescaleDB with continuous aggregates

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| CAP Theorem | Can only have 2 of 3: Consistency, Availability, Partition Tolerance |
| PACELC | If P → C or A, Else → Latency or Consistency |
| LSM Tree vs B-Tree | LSM: write-optimized (Cassandra, RocksDB). B-Tree: read-optimized (PostgreSQL) |
| MongoDB Sharding | Partition by shard key, chunk migration, config servers |
| Redis Data Types | String, Hash, List, Set, Sorted Set, Stream, Bitmap, HyperLogLog |
| Redis Persistence | RDB (snapshots), AOF (append-only), both recommended |
| DynamoDB Single Table | One table, composite keys, GSIs for access patterns |
| Cassandra Partition | (Partition Key, Clustering Key) - partition key = distribution |
| Graph DB Use Case | Relationships first: social, fraud, recommendations, network |
| Time-Series DB | Hypertable (TimescaleDB), compression, continuous aggregates |

---

## NEXT TOPIC: `03-CACHING.md`

> **Study Tip**: Set up local Redis, MongoDB, Cassandra via Docker. Practice CRUD + queries for each.