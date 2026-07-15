# SYSTEM DESIGN INTERVIEW
## A Book-Level Guide to Thinking, Not Memorizing

---

> **Read this first.** A system design interview has *no correct answer*. The interviewer is not checking whether you built the same system they have in mind. They are watching a senior engineer think out loud: clarify scope, make reasonable assumptions, sketch a design that addresses the core, go deep where it matters, and reason about trade-offs and failure. This guide teaches that *process* and the *building blocks* you combine during it. The sample designs at the end are vehicles for the reasoning — study the *decisions*, not the diagrams.

---

## TABLE OF CONTENTS

1. [What Interviewers Actually Grade](#1-what-interviewers-actually-grade)
2. [The 50-Minute Framework](#2-the-50-minute-framework)
3. [Step 1 — Requirements & Scope](#3-step-1--requirements--scope)
4. [Step 2 — Back-of-Envelope Estimation](#4-step-2--back-of-envelope-estimation)
5. [Step 3 — API & Data Model](#5-step-3--api--data-model)
6. [Step 4 — High-Level Architecture](#6-step-4--high-level-architecture)
7. [Step 5 — Deep Dives (the part that gets you hired)](#7-step-5--deep-dives)
8. [The Building-Block Menu (when to reach for what)](#8-the-building-block-menu)
9. [Decision Trees (SQL vs NoSQL, cache, sync vs async, consistency)](#9-decision-trees)
10. [End-to-End Design Walkthroughs](#10-end-to-end-design-walkthroughs)
    - 10.1 URL Shortener
    - 10.2 News Feed (Push vs Pull)
    - 10.3 Rate Limiter
    - 10.4 Notification System
11. [Common Mistakes That Sink Candidates](#11-common-mistakes-that-sink-candidates)
12. [Practice Plan & Question Bank](#12-practice-plan--question-bank)

---

## 1. WHAT INTERVIEWERS ACTUALLY GRADE

Different companies weight these differently, but the buckets are stable. Know them so you spend effort where it's scored.

| Dimension | What it looks like | How to show it |
|-----------|--------------------|----------------|
| **Requirements clarity** | Did you pin down scope before designing? | Ask 2–3 sharp clarifying questions up front; state assumptions. |
| **Estimation & justification** | Can you size the system and let numbers drive choices? | Do a quick back-of-envelope; cite it when picking a DB/cache. |
| **High-level design** | Does the sketch solve the core problem? | Draw boxes + data flow; cover the primary read/write paths. |
| **Depth** | Can you go 2 levels deep on a real bottleneck? | Pick 2–3 areas; discuss algorithms, data structures, failure. |
| **Trade-offs** | Do you know the cost of your choices? | For every major choice say what you gave up and why. |
| **Failure handling** | What breaks and how do you survive it? | Mention replicas, retries, idempotency, degradation. |
| **Communication** | Can a colleague follow your thinking? | Narrate; check in ("shall I go deeper on caching?"). |

**The single most important sentence in this guide:** *"There is no right answer — only defensible answers with acknowledged trade-offs."* A candidate who names a simple design, explains its limits, and proposes the next step beats a candidate who silently draws a 40-box microservice monolith.

---

## 2. THE 50-MINUTE FRAMEWORK

Time-box yourself. Running out of time on deep dives is the most common failure.

```
0–5 min    STEP 1  Requirements & scope (functional, non-functional, constraints)
5–10 min   STEP 2  Back-of-envelope (QPS, storage) — justifies everything after
10–20 min  STEP 3  API + data model (high level)
20–35 min  STEP 4  High-level architecture (boxes + data flow) — DRAW IT
35–45 min  STEP 5  Deep dives (2–3 areas the interviewer steers you to)
45–50 min  Bottlenecks, trade-offs, failure, "what I'd do next"
```

**Golden rule:** Spend the first 10 minutes *narrowing*, not building. A tight scope beats a broad one you can't finish. If the problem is huge ("design YouTube"), immediately propose a slice: "I'll focus on upload + serve of a single video; skip recommendations and monetization unless we have time."

---

## 3. STEP 1 — REQUIREMENTS & SCOPE

Before any box is drawn, lock down three things. Ask, don't assume.

**Functional requirements** — what the system *does*.
- Core use cases: "A user shortens a URL"; "a reader is redirected."
- Explicitly ask: "Is this read-heavy or write-heavy?" "Do we need real-time or is eventual OK?" "Custom aliases? Analytics? Expiry?"

**Non-functional requirements (NFR)** — the *-ilities*. Name the ones that matter:
- **Availability**: 99.9% (≈9h downtime/yr) vs 99.99% (≈1h/yr). Drives replication + multi-AZ.
- **Latency**: p50/p95/p99 targets (e.g., "redirect < 50ms p99").
- **Durability**: "Can we lose data?" (payments: no; analytics: maybe).
- **Consistency**: strong vs eventual (see §9.4).
- **Scale**: this is where Step 2 begins.

**Constraints** — the real world:
- Team size, existing tech, budget, geo-distribution, compliance (GDPR/HIPAA/PCI).

> **Why this matters:** A URL shortener and a payment system have opposite consistency needs. The interviewer watches whether you *ask which one you're building* before committing.

---

## 4. STEP 2 — BACK-OF-ENVELOPE ESTIMATION

Numbers turn opinions into decisions. You do **not** need precision — within 2–4× is fine. The point is to know whether you're designing for 1K QPS or 1M QPS.

**Reference numbers (memorize these):**
```
1 million  = 10^6        1 billion = 10^9
1 KB = 10^3 B   1 MB = 10^6 B   1 GB = 10^9 B   1 TB = 10^12 B

Latency (rough, from Bran Eich / systems lore):
  L1 cache ............ ~0.5 ns
  Main memory (RAM) ... ~100 ns
  SSD random read ..... ~50–150 µs
  Same datacenter RPC . ~0.5–2 ms
  Cross-region RPC .... ~50–100 ms
  Disk/network is 10^3–10^6× slower than RAM — that's WHY we cache.
```

**QPS from DAU** (the most common calc):
```
avg QPS  ≈  DAU × actions_per_user_per_day / 86,400
peak QPS ≈  avg QPS × peak_factor        (peak_factor usually 2–10×, use 3× by default)
```
Example: 1M DAU, 10 actions/day → 10M / 86,400 ≈ **115 QPS avg**, ~**350 QPS peak**.

**Storage:**
```
total = writes_per_day × bytes_per_record × retention_days × replication_factor
```
Example: 100M tweets/day × 500 B × 365 × 3 replicas ≈ **55 TB** over a year (before compression).

**Rule of thumb for capacity:**
- One stateless app server comfortably handles **~1K QPS** of simple CRUD (more if cached, less if CPU-heavy).
- A single PostgreSQL primary handles **~5–10K writes/sec** before you must shard.
- A single Redis node handles **~100K+ ops/sec**.

> **Use the numbers aloud:** "At 40K QPS reads, a single DB can't keep up, so I'll put Redis in front and shard the DB by user_id." That sentence is what gets you points.

---

## 5. STEP 3 — API & DATA MODEL

**API:** Prefer REST for clarity unless the problem screams RPC (internal, low-latency, streaming → gRPC) or graph (deeply connected, client-driven fetches → GraphQL). Show the *primary* endpoints only — the ones the core flow uses.

```
POST   /api/v1/urls         body: {url, custom_alias?}   → {short_code}
GET    /api/v1/{short_code} → 301/302 redirect to long url
GET    /api/v1/urls/{code}/stats                          → analytics
```

**Error contract:** pick a consistent shape (RFC 7807 problem+json is a good signal):
```json
{ "type": "…/not-found", "title": "Short code not found",
  "status": 404, "detail": "No mapping for 'abc123'" }
```

**Data model:** Choose SQL or NoSQL deliberately (see §9.1). Sketch the key entities and the *access patterns* — for NoSQL the access pattern drives the table, not the ER diagram. Show indexes that back your queries. Example (PostgreSQL for URL shortener):
```sql
CREATE TABLE urls (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(10) UNIQUE NOT NULL,
  long_url    TEXT NOT NULL,
  owner_id    BIGINT,
  created_at  TIMESTAMPTZ DEFAULT now(),
  expires_at  TIMESTAMPTZ
);
CREATE INDEX idx_short_code ON urls(short_code);   -- the redirect lookup
```

---

## 6. STEP 4 — HIGH-LEVEL ARCHITECTURE

Draw a simple component diagram with **data flow**, not a microservice org chart. Cover the primary read and write paths end to end.

```
        Client
          │
          ▼
   ┌──────────────┐
   │ API Gateway  │  (TLS, auth, rate limit, request validation)
   └──────┬───────┘
          │
          ▼
   ┌──────────────┐      ┌──────────────┐
   │  App / Write │─────►│  Database    │  (source of truth)
   │  Service     │      └──────────────┘
   └──────┬───────┘            │
          │                    ▼
          ▼              ┌──────────────┐
   ┌──────────────┐      │   Cache      │  (Redis: hot reads)
   │  Analytics   │◄─────│   (Redis)    │
   │  (async via  │  log  └──────────────┘
   │   Kafka)     │
   └──────────────┘
```

**Keep it honest:** every box you draw, you should be able to explain *why it exists* and *what happens when it dies*. A diagram you can't defend is worse than no diagram.

---

## 7. STEP 5 — DEEP DIVES

This is where offers are made. The interviewer will point at one box ("talk to me about your cache") and you go deep. Have the following ready for *any* system:

1. **Caching** — strategy, what's cached, TTL, invalidation, stampede (§8.2).
2. **Database scaling** — replication, sharding key, resharding pain (§8.3).
3. **Consistency** — what level, how you achieve it, what you sacrifice (§9.4).
4. **Failure** — what breaks, retries, idempotency, DLQ, degradation.
5. **Hotspots / bottlenecks** — the celebrity problem, single-writer, thundering herd.

> Go *narrow and deep*, not *broad and shallow*. "Let me go deep on the redirect hot path and caching" lands far better than touching seven boxes for 30 seconds each.

---

## 8. THE BUILDING-BLOCK MENU

Know every block below well enough to say *when to use it and what it costs*. This is your vocabulary during the interview.

### 8.1 Edge & Routing
- **CDN**: caches static/large content at the edge (images, video, JS). *Cost:* invalidation complexity. *Use:* anything user-facing and read-heavy.
- **Load Balancer (L4/L7)**: spreads traffic; needs to be HA itself (active-active pair). *Use:* in front of every service tier.
- **API Gateway**: auth, rate limiting, routing, request validation in one place. *Cost:* extra hop/latency, a place to couple logic.

### 8.2 Caching (Redis)
Four strategies — know the trade-off, not just the names:

| Strategy | Write path | Read on miss | Consistency | Use when |
|----------|-----------|--------------|-------------|----------|
| **Cache-aside** | App writes DB; cache invalidated/updated | App loads DB→cache | Eventual (stale window) | Most reads; tolerates staleness |
| **Read-through** | — | Cache loads DB itself | Eventual | Simpler app code, stable data |
| **Write-through** | Cache + DB written together (sync) | Always hits cache | Stronger | Read-heavy, write-ok-latency |
| **Write-behind** | Cache writes, flushes async | Always hits cache | Risk of loss on crash | Write-heavy, can lose some data |

**Invalidation** is the hard part. Prefer **TTL + event-driven invalidation** (on DB write, publish "user X changed" → consumers evict `user:X`). Never rely on TTL alone for correctness-sensitive data.

**Cache stampede / thundering herd:** when a hot key expires, thousands of requests hit the DB at once. Fixes: (a) **request coalescing / singleflight** (one request refreshes, others wait), (b) **probabilistic early expiration** (random TTL jitter), (c) **stale-while-revalidate** (serve stale, refresh in background).

### 8.3 Database Scaling
- **Read replicas**: primary takes writes, async-replicated to replicas for reads. *Cost:* replication lag (ms to seconds) → "read-your-writes" anomalies (you update, then read your old value from a lagging replica). *Fix:* route the writer's own reads to primary, or use synchronous-ish consistency for critical reads.
- **Sharding**: split data across nodes by a shard key.
  - *Hash sharding* (`hash(key) % N`): uniform, but resharding = rehash everything (painful).
  - *Range sharding*: good range queries, but **hot spots** (newest users all land on one shard).
  - *Consistent hashing* (virtual nodes on a ring): adding/removing a node reshuffles only `1/N` of data. Standard for distributed caches (DynamoDB, Cassandra).
- **Vertical vs horizontal partitioning**: separate hot/cold data (e.g., archive old rows to cheap storage).

### 8.4 Replication Topologies & Consistency
- **Single-leader**: one primary takes writes, replicas follow. Simple, but the leader is a write bottleneck and a SPOF.
- **Multi-leader**: writes accepted in multiple regions. *Cost:* conflict resolution (last-write-wins, CRDTs). *Use:* multi-region write tolerance.
- **Leaderless (Dynamo-style)**: any node serves reads/writes; use a **quorum**: write to `W` nodes, read from `R` nodes, require **R + W > N** for strong consistency. *Cost:* reconciling divergent versions on read.

**CAP — the correct framing.** The popular "pick 2 of 3" is misleading. *Partition tolerance isn't optional* — networks do partition, so in practice you must survive partitions. The real statement:
> **During a network partition, you must choose Consistency (C) or Availability (A) — you cannot have both.** When there is *no* partition, you instead trade off **Latency (L) vs Consistency (C)**.

That second half is **PACELC**: **P**artition → **A** or **C**; **E**lse → **L** or **C**. Use this instead of the broken "pick 2" line — interviewers notice.

**Consistency levels (strong → weak):**
- *Linearizable / Strong*: reads always see the latest write (CP systems: PostgreSQL, etcd, ZooKeeper).
- *Causal*: causally-related ops preserve order (good for social feeds).
- *Eventual*: replicas converge over time; reads may be stale (AP systems: DynamoDB, Cassandra). Conflict resolution: LWW, CRDTs, or app-level merge.

### 8.5 Async & Messaging
- **Message queue (Kafka, RabbitMQ)**: decouples producer/consumer, absorbs spikes, enables retry. *Cost:* operational complexity, eventual processing, ordering caveats (Kafka preserves order *within a partition*).
- **Use async when:** work is slow (email, transcoding), benefits from retry (notifications), or must survive a downstream outage (queue buffers it).
- **Outbox pattern**: write DB change + event in *one* transaction, then publish — solves "updated DB but failed to send event" (dual-write inconsistency).
- **Saga**: long-running multi-service transaction split into steps with compensating actions (vs distributed 2PC, which doesn't scale).

### 8.6 Search, Objects, Specialized
- **Elasticsearch / OpenSearch**: inverted-index full-text search and log analytics. *Not* a primary DB.
- **Object storage (S3) + CDN**: the only sane place for images/video/files. Generate thumbnails/transcodes asynchronously.
- **Idempotency**: client sends a unique key; server dedupes. Makes retries safe (essential for payments).
- **Observability**: metrics (Prometheus), logs, traces (OpenTelemetry) — name *what you'd alert on* (p99 latency, error rate, queue depth).

---

## 9. DECISION TREES

When the interviewer asks "why this?", walk a tree out loud.

### 9.1 SQL vs NoSQL
```
Need complex joins / transactions / reporting? ──yes──► SQL (PostgreSQL/MySQL)
   │no
Data is文档/key-value/time-series & huge write volume? ──yes──► NoSQL
   │no
Strong consistency + relations ──yes──► SQL
   │no
Flexible schema, horizontal scale, eventual OK ──yes──► NoSQL (DynamoDB/Cassandra/Mongo)
```
Rule: **default to SQL** unless scale or schema flexibility forces NoSQL. SQL can shard and replicate; don't over-engineer early.

### 9.2 Cache strategy
```
Correctness-critical (balances, inventory)? ──yes──► write-through or no cache
   │no
Read-heavy, stale-tolerant (profiles, feeds)? ──yes──► cache-aside + TTL
   │no
Write-heavy, can lose some? ──yes──► write-behind
```

### 9.3 Sync vs Async
```
Caller needs the result NOW to proceed? ──yes──► sync (REST/gRPC)
   │no
Slow / retryable / spike-absorbing (email, video, analytics)? ──yes──► async (queue)
```

### 9.4 Consistency level
```
Financial / cannot lose or show stale? ──yes──► Strong (single-leader, quorum)
   │no
Social / feeds / can converge later? ──yes──► Eventual + conflict resolution
   │no
Causality matters (comments before replies)? ──yes──► Causal
```

---

## 10. END-TO-END DESIGN WALKTHROUGHS

These teach *reasoning*. For each: scope, estimate, the key decision, the bottleneck, and the trade-off.

### 10.1 Design a URL Shortener (bit.ly)

**Scope:** shorten (write), redirect (read → original URL), optional custom alias + analytics + expiry. Read-heavy (~100:1).

**Estimate:** 10B redirects/month ≈ **3.8K QPS avg**, peak ~**40K QPS**. 100M new URLs/month ≈ 40 QPS writes. Storage ≈ 50 GB/month.

**Key decision — the short code:** We need a unique, short, URL-safe code. Options:
- *Auto-increment ID → base62*: simple, but a central counter is a bottleneck/SPOF and resharding is painful.
- *Hash the long URL (e.g., MD5 → base62)*: no counter, but **collisions** happen; need a retry/unique-check.
- *Distributed sequence (Snowflake: timestamp + node + seq, 64-bit)*: no central bottleneck, time-ordered. → **base62 of a Snowflake ID**. 7 chars = 62⁷ ≈ 3.5×10¹² codes — plenty.

**Redirect hot path (the read, 99% of traffic):**
```
GET /{code} → API GW → check Redis (TTL 24h+) → hit: return 302
                              miss: query DB, cache, return 302
            → unknown code: Bloom filter first (reject without DB hit)
```
Use **302** (not 301) if you need per-click analytics — 301 is browser-cached and won't count. Trade-off: 302 costs a request each time.

**Analytics:** do **not** block the redirect. Fire an event to Kafka → stream processor → warehouse. Async keeps redirect fast.

**Bottleneck & fix:** DB under 40K QPS reads. Solved by Redis in front (most codes cached, since popular URLs dominate) + read replicas + bloom filter for negatives. **Trade-off:** cache staleness vs speed — acceptable because URLs rarely change.

---

### 10.2 Design a News Feed (Twitter timeline)

**Scope:** post a tweet; view your home timeline (tweets from people you follow). Read-heavy; latency-sensitive.

**The core decision — fan-out: push vs pull vs hybrid.**

| Approach | How | Pro | Con |
|----------|-----|-----|-----|
| **Pull (on read)** | On open, fetch each followed user's recent tweets, merge/sort | Simple; follows take effect instantly; no extra storage | Expensive for users following many (merge 1000 timelines per open); high read latency |
| **Push (on write)** | On tweet, write to every follower's timeline store | Read is O(1) fetch; fast timeline | Write amplification: a celebrity with 10M followers = 10M writes per tweet |
| **Hybrid** | Push to normal followers; *pull* celebrity tweets on read and merge | Best of both | More complex; needs a "celebrity" threshold |

**Real systems use hybrid.** Define a follower-count threshold (e.g., >100K = celebrity). When a celebrity tweets, you don't push to 10M timelines; instead, on read you fetch the celebrity's tweets directly and merge with your pushed timeline. *This single decision is the heart of the design* — be able to explain it with the celebrity example.

**Storage:** each user's timeline = a Redis sorted set (score = timestamp), capped at ~1000 entries. Media/thumbnails precomputed async.

**Deep dive — ranking & pagination:** Timelines are rarely pure-chronological; ranking by relevance needs a scoring service. Paginate with a **cursor** (last seen timestamp/id), never `LIMIT/OFFSET` at scale (offset gets slower as you page deep).

---

### 10.3 Design a Rate Limiter

**Scope:** limit requests per user/IP (e.g., 100 req/min). Must be **distributed** (many app nodes share one limit) and **atomic**.

**Algorithms — know the trade-offs:**

| Algorithm | Behavior | Pro | Con |
|-----------|----------|-----|-----|
| Fixed window counter | Count per fixed interval | Simple | Allows 2× burst across a boundary |
| Sliding window log | Store timestamps, count in window | Precise | Memory-heavy at scale |
| Sliding window counter | Weighted avg of prev + curr window | Good approximation, low memory | Slightly imprecise |
| **Token bucket** | Tokens refill at rate; burst up to capacity | Smooth, allows controlled bursts | Needs refill math |
| Leaky bucket | Fixed outflow rate | Smooths bursts, protects downstream | Rejects bursts outright |

**Default recommendation: token bucket** (allows reasonable bursts, simple to reason about).

**Distributed correctness:** counters must live in shared storage (**Redis**), and check-and-increment must be **atomic** — use a **Lua script** so two concurrent requests can't both pass. Don't use local server memory (each node has its own count → limit is N× too high).

**Edge cases to name:** use Redis's time, not local clocks (skew breaks windows); return **429 + `Retry-After`**; handle the thundering-herd of a reset; per-user vs per-IP (IP shares across NAT/users).

---

### 10.4 Design a Notification System

**Scope:** send email/SMS/push; retry on failure; respect user preferences; track delivery. ~10M/day.

**Architecture:**
```
API → Notification Service → per-channel Queues (email/SMS/push)
                            → Channel Workers (SendGrid / Twilio / FCM+APNs)
                            → Retry w/ exponential backoff → Dead Letter Queue
                                                ↘ Delivery tracking + analytics
```

**Key decisions:**
- **Provider abstraction**: one interface, many impls (SendGrid, Twilio, FCM). Swap providers without touching logic.
- **Retry + backoff**: 1m, 5m, 15m, 1h, 6h; after max → **DLQ** for manual/alert.
- **Idempotency**: a notification can be retried safely only if deduplicated by a client key — otherwise a retry double-sends. This is *exactly-once delivery is impossible in distributed systems; settle for at-least-once + idempotent consumer.*
- **Push specifics**: mobile needs a **device-token registry**; tokens expire/rotate → must refresh. Use FCM/APNs, not raw WebSocket, for mobile (battery/connectivity).
- **Preferences**: user channel opt-ins stored separately; checked before enqueue.

**Bottleneck & fix:** a downstream provider outage shouldn't lose notifications → the **queue buffers** them; workers back off. **Trade-off:** at-least-once delivery means a rare duplicate unless idempotency keys are enforced.

---

## 11. COMMON MISTAKES THAT SINK CANDIDATES

| Mistake | Why it kills you | The fix |
|---------|------------------|---------|
| Jumping to a solution before requirements | Looks like you can't scope | Ask 2–3 clarifying Qs; state assumptions |
| No back-of-envelope math | Choices look arbitrary | Spend 5 min on QPS/storage; cite it |
| Designing for 1B users on problem 1 | Over-engineering; runs out of time | Right-size; say "at this scale, X" |
| Microservice explosion | Can't explain 40 boxes | 5–8 meaningful components, fully defended |
| "Pick 2 of CAP" | Technically wrong; signals shallow knowledge | Use PACELC framing (§8.4) |
| Stating a choice with no trade-off | Looks like memorization | Always say what you gave up |
| Ignoring failure | Systems always fail; so should designs | Replicas, retries, idempotency, degradation |
| Talking only, not drawing | Interviewer can't follow | Draw the data flow early |
| Going broad & shallow in deep dives | No signal of depth | Pick 2–3 areas, go deep |
| No "what's next" | Looks like you think it's done | End with bottlenecks + next step |

**The mindset trap:** candidates think silence is failure and fill it with jargon. Better to *think out loud slowly* — "here's what I'm weighing…" — than to rattle off components. The interviewer is rating the thinking.

---

## 12. PRACTICE PLAN & QUESTION BANK

**Weekly cadence:** 2 designs/week, timed at 50 min, drawn on a whiteboard/paper (not code). Record yourself; listen for "um" and missing trade-offs.

**Progression:**
- **Weeks 1–2 (fundamentals):** URL Shortener, Rate Limiter, Pastebin, Key-Value Store.
- **Weeks 3–4 (scale & async):** News Feed, Notification System, Web Crawler, Distributed Cache.
- **Weeks 5–6 (hard):** Ride-sharing (geo), Video streaming (YouTube), Payments (idempotency), Search engine (inverted index), Autocomplete (trie + top-k).

**Question bank by theme:**
| Theme | Questions |
|-------|-----------|
| Storage / IDs | URL shortener, distributed ID generator (Snowflake), Pastebin |
| Real-time / geo | Ride-sharing, chat (WhatsApp/Slack), live comments |
| Media | YouTube, Instagram, Dropbox (file sync) |
| Async / pipelines | Notification system, web crawler, log aggregation |
| Consistency / money | Payment system, distributed lock, leader election |
| Search | Autocomplete, typeahead, search engine |

**Self-grading after each practice:** Did I (1) clarify scope, (2) estimate, (3) draw a coherent flow, (4) deep-dive 2 areas with trade-offs, (5) address failure, (6) state next steps? Score 0–1 each; <5 means redo that design.

---

### ANKI FLASHCARDS (reference)

| Front | Back |
|-------|------|
| System design time budget | 5 req → 5 est → 10 API/data → 15 arch → 10 deep → 5 trade-offs |
| QPS from DAU | DAU × actions/day / 86400; peak ×3 |
| CAP correct framing | During partition choose C or A; PACELC adds L-vs-C when no partition |
| SQL vs NoSQL default | Default SQL; NoSQL only for scale/schema flexibility |
| Cache-aside | Miss→DB→cache→return; TTL + event invalidation |
| Write-behind risk | Async DB flush → data loss on crash |
| Cache stampede fix | Singleflight, TTL jitter, stale-while-revalidate |
| Consistent hashing | Ring + virtual nodes; only 1/N reshuffles on add/remove |
| R+W>N | Quorum reads/writes → strong consistency (leaderless) |
| Rate limiter atomicity | Redis + Lua script (check+incr must be atomic) |
| Token bucket | Allows controlled bursts; smooth default |
| Feed fan-out | Hybrid: push to followers, pull celebrities (threshold) |
| Feed pagination | Cursor (last id/time), not OFFSET |
| Idempotency | Client key + server dedupe → safe retries |
| Outbox pattern | DB + event in one tx → reliable async events |
| Saga | Compensating steps for distributed transactions |
| Exactly-once delivery | Impossible distributed; use at-least-once + idempotent consumer |
| Read replica lag | ms–sec; route writer's reads to primary |

---

> **Final wisdom:** You will not design the "perfect" system in 50 minutes, and you're not supposed to. You will show a thoughtful engineer who scopes, estimates, draws a coherent flow, goes deep where pressed, names trade-offs, and plans for failure. That person gets the offer. You've prepared — now think out loud and enjoy the conversation.
