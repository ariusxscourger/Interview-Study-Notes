# SYSTEM DESIGN INTERVIEW TEMPLATE
## Exam-Style Study Notes

---

## TOPIC: System Design Interview Structure & Template

### WHY? (Problem Statement)

**System Design interviews are open-ended:**
- No single correct answer
- Evaluated on: Process, Trade-offs, Scalability, Communication
- 45-60 minutes for a massive system

**What This Template Solves:**
| Problem | Template Provides |
|---------|------------------|
| Where to start? | Step-by-step framework |
| Running out of time? | Time-boxed sections |
| Missing key areas? | Checklist of all areas |
| Rambling | Structured communication |

---

### HOW? (The 6-Step Framework)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM DESIGN INTERVIEW (45-60 min)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 1: REQUIREMENTS (5-10 min)          STEP 2: API DESIGN (5-10 min)   │
│  ─────────────────────────────────         ─────────────────────────────    │
│  • Clarify functional requirements         • REST/GraphQL/gRPC              │
│  • Define non-functional (NFR)             • Request/Response contracts     │
│  • Back-of-envelope calculations           • Error codes & versioning       │
│  • Constraints & assumptions               • Rate limiting                  │
│                                                                             │
│  STEP 3: DATA MODEL (10 min)              STEP 4: HIGH-LEVEL DESIGN (10m) │
│  ───────────────────────────────          ─────────────────────────────     │
│  • Entities & relationships               • Service decomposition           │
│  • SQL vs NoSQL choice                    • Communication patterns          │
│  • Partitioning strategy                  • Data flow diagram               │
│  • Indexing strategy                      • External dependencies           │
│                                                                             │
│  STEP 5: DEEP DIVES (10-15 min)         STEP 6: SCALE & TRADE-OFFS (5m)   │
│  ───────────────────────────────         ─────────────────────────────      │
│  Pick 2-3 areas:                          • Bottlenecks & solutions        │
│  • Caching strategy                       • Consistency vs Availability    │
│  • Sharding/partitioning                  • CAP/PACELC trade-offs          │
│  • Caching invalidation                   • Cost optimization              │
│  • Consistency models                     • Failure scenarios              │
│  • Replication                            • Monitoring & alerting          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Detailed Templates)

#### **STEP 1: REQUIREMENTS GATHERING**

```
FUNCTIONAL REQUIREMENTS:
─────────────────────────────────────────────────────────────────────────────
□ Core features (What does the system DO?)
□ User flows (Who does WHAT?)
□ Edge cases (What CAN go wrong?)

NON-FUNCTIONAL REQUIREMENTS (NFR):
─────────────────────────────────────────────────────────────────────────────
□ Availability: 99.9% / 99.99% / 99.999%
□ Latency: p50, p95, p99 targets
□ Throughput: QPS, peak vs average
□ Consistency: Strong / Eventual / Causal
□ Durability: Data loss tolerance
□ Scalability: Horizontal vs Vertical
□ Security: AuthZ, encryption, audit
□ Compliance: GDPR, HIPAA, PCI-DSS

BACK-OF-ENVELOPE ESTIMATION:
─────────────────────────────────────────────────────────────────────────────
□ Daily Active Users (DAU)
□ Peak QPS = DAU * actions_per_user / 86400 * peak_factor
□ Storage: Daily data * retention_days * replication_factor
□ Bandwidth: QPS * avg_request_size * 8 bits
□ Memory: Working set size + cache

CONSTRAINTS:
─────────────────────────────────────────────────────────────────────────────
□ Budget limits
□ Team size & expertise
□ Timeline
□ Legacy systems integration
□ Geographic distribution
```

#### **STEP 2: API DESIGN**

```
REST API CONTRACT:
─────────────────────────────────────────────────────────────────────────────
POST   /api/v1/resources           → Create
GET    /api/v1/resources           → List (pagination, filtering, sorting)
GET    /api/v1/resources/{id}      → Get by ID
PATCH  /api/v1/resources/{id}      → Partial update
PUT    /api/v1/resources/{id}      → Full replace
DELETE /api/v1/resources/{id}      → Delete

QUERY PARAMETERS:
?page=1&limit=20&sort=-createdAt&filter[status]=active

ERROR RESPONSE (RFC 7807):
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Email already exists",
  "instance": "/api/v1/users",
  "errors": [
    { "field": "email", "code": "unique", "message": "Email already registered" }
  ]
}

VERSIONING: /api/v1/ or Accept: application/vnd.example.v1+json
```

#### **STEP 3: DATA MODEL**

```
ENTITY RELATIONSHIP:
─────────────────────────────────────────────────────────────────────────────
User 1──* Post *──1 Category
  │
  └──* Comment
       │
       └──* Like

SCHEMA DESIGN:
─────────────────────────────────────────────────────────────────────────────
SQL (PostgreSQL):
────────────────
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  title VARCHAR(500) NOT NULL,
  content TEXT,
  published BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published_created ON posts(published, created_at DESC);

NoSQL (DynamoDB - Single Table Design):
────────────────────────────────────────
PK: USER#<user_id>     SK: PROFILE          → User profile
PK: USER#<user_id>     SK: POST#<post_id>   → Posts
PK: POST#<post_id>     SK: COMMENT#<id>     → Comments
GSI1PK: CATEGORY#<id>  GSI1SK: CREATED_AT   → Category queries

PARTITIONING STRATEGY:
─────────────────────────────────────────────────────────────────────────────
□ Horizontal (Sharding): By user_id, tenant_id, geographic
□ Vertical: Separate tables: Hot vs Cold data (archival)
□ Time-based: Daily/Monthly partitions for time-series
```

#### **STEP 4: HIGH-LEVEL ARCHITECTURE**

```
ARCHITECTURE DIAGRAM:
─────────────────────────────────────────────────────────────────────────────
                         ┌─────────────┐
                         │   Client    │
                         └──────┬──────┘
                                │
                    ┌───────────▼───────────┐
                    │   API Gateway / LB    │
                    │  (Auth, Rate Limit)   │
                    └───────────┬───────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│  User Service │       │  Post Service │       │Comment Service│
│  (Auth, Profile)       │  (CRUD, Feed) │       │ (Comments)    │
└───────┬───────┘       └───────┬───────┘       └───────┬───────┘
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│  User DB      │       │  Post DB      │       │ Comment DB    │
│  (Primary)    │       │  (Sharded)    │       │ (Replicated)  │
└───────────────┘       └───────────────┘       └───────────────┘
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                ▼
                    ┌───────────────────────┐
                    │   Message Broker      │
                    │   (Kafka/RabbitMQ)    │
                    └───────────┬───────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│ Notification  │       │  Analytics    │       │  Search       │
│   Service     │       │   Service     │       │  Service      │
└───────────────┘       └───────────────┘       └───────────────┘

COMMUNICATION PATTERNS:
─────────────────────────────────────────────────────────────────────────────
□ Synchronous: REST/gRPC (Request-Response, real-time needs)
□ Asynchronous: Message Queue (Decoupling, reliability, peak shaving)
□ Event Streaming: Kafka (Audit log, event sourcing, replay)
□ Pub/Sub: Notifications, cache invalidation
```

#### **STEP 5: DEEP DIVE AREAS (Pick 2-3)**

```
CACHING STRATEGY:
─────────────────────────────────────────────────────────────────────────────
□ Cache-Aside (Lazy Loading)
  - On miss: Load from DB → Cache → Return
  - TTL: 5min-1hr for reference, 1-5min for volatile
  
□ Write-Through
  - Write to Cache → DB (sync)
  - Strong consistency, higher latency
  
□ Write-Behind (Write-Back)
  - Write to Cache → Async flush to DB
  - Low latency, risk of data loss
  
□ Refresh-Ahead
  - Async refresh before TTL expiry
  - Prevents cache stampede
  
□ Cache Keys: entity:{id}, list:{filter}:{page}
□ Invalidation: TTL + Event-driven (on write)
□ Cache Stampede: Probabilistic early refresh / Mutex
□ Multi-level: L1 (Local/Redis) → L2 (Redis Cluster) → DB

SHARDING STRATEGY:
─────────────────────────────────────────────────────────────────────────────
□ Hash-based: hash(key) % N → Uniform, but resharding hard
□ Range-based: Key ranges per shard → Range queries easy, hot spots
□ Directory-based: Lookup service → Flexible, extra hop
□ Consistent Hashing: Minimal reshuffle on add/remove

CONSISTENCY MODELS:
─────────────────────────────────────────────────────────────────────────────
□ Strong: Linearizable (CP systems, financial)
□ Sequential: Operations appear in order
□ Causal: Cause→Effect preserved (social feeds)
□ Eventual: High availability, conflict resolution (AP systems)
□ Read Your Writes: Session consistency
□ Monotonic Reads: Never go back in time

REPLICATION:
─────────────────────────────────────────────────────────────────────────────
□ Single-Leader: Simple, write bottleneck
□ Multi-Leader: Geo-distributed, conflict resolution
□ Leaderless: Dynamo-style, quorum reads/writes (R+W>N)
□ Synchronous: Strong consistency, higher latency
□ Asynchronous: Low latency, potential data loss
```

---

### QUICK REFERENCE CARD

| Component | When to Use | Key Trade-off |
|-----------|-------------|---------------|
| **Load Balancer** | Distribute traffic | Single point of failure (need HA) |
| **CDN** | Static assets, global users | Cache invalidation complexity |
| **API Gateway** | Auth, rate limit, routing | Extra hop, latency |
| **Message Queue** | Async, decoupling | Complexity, ordering |
| **Redis** | Cache, sessions, rate limit | Memory limit, persistence |
| **PostgreSQL** | Relational, ACID, complex queries | Write scaling limits |
| **DynamoDB/Cassandra** | High write, simple queries | No joins, data modeling |
| **Kafka** | Event streaming, audit, replay | Operational complexity |
| **Elasticsearch** | Full-text search, logs | Not primary DB |
| **CDN** | Static assets, edge compute | Cost, cache invalidation |

---

### INTERVIEW QUESTIONS (System Design Classics)

| Problem | Key Challenges | Approach |
|---------|---------------|----------|
| **Design URL Shortener** | ID generation, collision, analytics | Base62, consistent hashing, Redis |
| **Design Twitter** | Fan-out (push vs pull), timeline | Push for celebs, pull for users |
| **Design Instagram** | Photo storage, feed generation | CDN, object storage, fan-out |
| **Design WhatsApp** | Real-time, offline, encryption | WebSocket, push notifications, e2e |
| **Design YouTube** | Video encoding, streaming, CDN | Transcoding pipeline, adaptive bitrate |
| **Design Uber/Lyft** | Real-time matching, geo-indexing | Quadtrees, geohash, WebSocket |
| **Design Dropbox** | File sync, conflict resolution | Chunking, delta sync, versioning |
| **Design TinyURL** | ID generation, redirect, analytics | Base62, DB sharding, Redis cache |
| **Design Rate Limiter** | Token bucket, sliding window | Redis sorted sets, Lua scripts |
| **Design Notification System** | Multi-channel, preferences, retry | Queue per channel, dead letter |

---

### PRACTICE PROBLEMS

1. **Design Twitter Timeline** - Push vs Pull, fan-out, caching
2. **Design Instagram Feed** - Ranking, pagination, media storage
3. **Design WhatsApp** - Real-time, delivery guarantees, groups
4. **Design YouTube** - Transcoding, adaptive streaming, CDN
5. **Design Uber Matching** - Geo-indexing, real-time, surge pricing
6. **Design Distributed Cache** - Consistent hashing, eviction, HA
7. **Design Search Engine** - Inverted index, ranking, distributed
8. **Design Payment System** - Idempotency, reconciliation, fraud

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| System Design Steps | 1. Requirements 2. API 3. Data Model 4. Architecture 5. Deep Dives 6. Scale |
| Back-of-Envelope QPS | DAU × actions / 86400 × peak_factor (2-10x) |
| Storage Estimate | Daily_write × retention_days × replication_factor |
| Cache-Aside | Miss → DB → Cache → Return. TTL + Invalidation on write. |
| Write-Through | Write Cache → DB sync. Consistent, slower writes. |
| Write-Behind | Write Cache → Async DB. Fast, risk data loss. |
| Cache Stampede Fix | Probabilistic early refresh / Mutex / Lease |
| Sharding: Hash vs Range | Hash=uniform, hard reshard. Range=easy queries, hot spots. |
| Consistent Hashing | Minimal reshuffle on add/remove, virtual nodes |
| CAP Theorem | Pick 2: Consistency, Availability, Partition Tolerance |
| PACELC | If P→A/C, Else→L/C. Latency vs Consistency when no partition. |
| Strong Consistency | Linearizable. Single leader, quorum reads/writes. |
| Eventual Consistency | High availability. Conflict resolution: LWW, CRDT, merge. |
| Read/Write Quorum | R + W > N for strong consistency. Dynamo-style. |
| Leader Election | Raft/Paxos, lease-based, etcd/Consul/ZooKeeper |
| Circuit Breaker | Fail fast, prevent cascade. States: Closed/Open/Half-Open. |
| Rate Limiting | Token Bucket, Sliding Window, Leaky Bucket |
| Saga Pattern | Choreography (events) vs Orchestration (central) |
| Idempotency Keys | Client-generated, server deduplicates. Safe retries. |
| Outbox Pattern | DB transaction + message in same TX. Reliable events. |
| CQRS | Separate read/write models. Optimize each independently. |
| Event Sourcing | Store events, not state. Replay for state. Audit trail. |

---

## NEXT TOPIC: `04-NEGOTIATION.md`

> **Study Tip**: Practice drawing architecture diagrams in 5 minutes. Explain trade-offs aloud. Time-box each section.