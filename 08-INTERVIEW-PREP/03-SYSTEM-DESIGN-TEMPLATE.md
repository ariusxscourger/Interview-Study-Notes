# SYSTEM DESIGN — RUNTIME CHEAT-SHEET
## One-page reference to use DURING practice (pair with 02-SYSTEM-DESIGN.md)

> Keep this open while you practice. The deep *reasoning* lives in `02-SYSTEM-DESIGN.md`; this is the checklist + numbers you reach for mid-design.

---

## THE 50-MINUTE SCRIPT (say these out loud)

```
0–5    "Let me scope this. Functional: ___? Non-functional: latency/availability/
       consistency targets? Scale: roughly how many users / QPS?"
5–10   "Quick math: DAU × actions / 86400 ≈ avg QPS; ×3 peak. Storage = writes ×
       bytes × retention × replicas."
10–20  "API: POST/GET/PATCH for the core flow. Data model: entities + access
       patterns + indexes."
20–35  [DRAW] Client → LB/CDN → API GW → Services → DB/Cache → Queue. Cover
       the primary read AND write path.
35–45  Deep dive (pick 2–3): caching, sharding, consistency, failure.
45–50  "Bottlenecks: ___. Trade-off I made: ___. What I'd do next: ___."
```

---

## NUMBERS TO MEMORIZE

```
1M = 10^6   1B = 10^9   1 KB=10^3 B  1 GB=10^9 B  1 TB=10^12 B
RAM ~100 ns | SSD ~100 µs | DC RPC ~1 ms | cross-region ~50–100 ms
1 app server ≈ 1K QPS simple CRUD | PG primary ≈ 5–10K writes/s | Redis ≈ 100K+ ops/s
avg QPS = DAU × actions/day / 86400      peak ≈ avg × 3
storage = writes/day × bytes × retention × replication(3)
```

---

## DECISION PROMPTS (answer before drawing boxes)

```
SQL or NoSQL?     → default SQL; NoSQL only for scale / flexible schema
Cache?            → read-heavy+stale-ok ⇒ cache-aside+TTL; correctness ⇒ write-through
Sync or async?    → need result now ⇒ sync; slow/retryable/spike ⇒ queue
Consistency?      → money ⇒ strong; feed ⇒ eventual; causal order ⇒ causal
Shard key?        → hash=uniform(hard reshard) | range=hotspots | consistent=minimal move
Single or multi?  → one writer simple; celebrity/multi-region ⇒ multi-leader/quorum
```

---

## COMPONENT MENU (reach for these)

| Need | Use | Watch out for |
|------|-----|---------------|
| Global static/large content | CDN + S3 | invalidation |
| Spread traffic / TLS | Load Balancer (HA pair) | SPOF if single |
| Auth / rate limit / route | API Gateway | extra latency, logic creep |
| Hot reads | Redis (cache-aside) | stampede, staleness |
| Relations / transactions | PostgreSQL | write scaling → shard/replica |
| Huge writes / flexible | DynamoDB / Cassandra | no joins, model by access |
| Decouple / retry / spike | Kafka / RabbitMQ | ordering, ops complexity |
| Full-text / logs | Elasticsearch | not a primary DB |
| Strong consistency coord | etcd / ZooKeeper | not for bulk data |

---

## CACHE, CONSISTENCY, RATE-LIMIT ONE-LINERS

```
Cache-aside:  miss→DB→cache→return; invalidate on write + TTL.
Write-behind: fast writes, risk data loss on crash.
Stampede:     singleflight (one refreshes) | TTL jitter | stale-while-revalidate.
CAP:          partition ⇒ choose C or A; PACELC ⇒ no partition ⇒ choose L or C.
Quorum:       R + W > N  ⇒  strong consistency (leaderless).
Rate limit:   token bucket (bursts OK); Redis + Lua for atomic check+incr.
Idempotency:  client key + server dedupe ⇒ safe retries (payments!).
Outbox:       DB row + event in one tx ⇒ reliable async events.
Saga:         steps + compensating actions for distributed transactions.
Pagination:   cursor (last id/time), NEVER OFFSET at scale.
```

---

## ANTI-PATTERN GATE (check before you finish)

```
[ ] Did I clarify scope before designing?
[ ] Did I estimate QPS/storage and cite it?
[ ] Is my diagram < ~8 boxes and fully defended?
[ ] Did I name a trade-off for every major choice?
[ ] Did I address failure (replica/retry/idempotency/degrade)?
[ ] Did I deep-dive 2–3 areas narrowly (not 7 broadly)?
[ ] Did I end with bottlenecks + next step?
[ ] Did I avoid the "pick 2 of CAP" myth?
```

---

## 12 PRACTICE DESIGNS (in order)

```
Fundamentals:   URL Shortener · Rate Limiter · Pastebin · Distributed ID (Snowflake)
Scale/Async:    News Feed · Notification System · Web Crawler · Distributed Cache
Hard:           Ride-sharing (geo) · Video Streaming · Payments · Search · Autocomplete
```
Self-grade each 0–1 on the 8 gates above; <5 → redo it.
