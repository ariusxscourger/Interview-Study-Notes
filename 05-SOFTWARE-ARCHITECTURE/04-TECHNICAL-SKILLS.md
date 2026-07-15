# TECHNICAL SKILLS
## Exam-Style Study Notes

**Roadmap Section:** Important Skills to Learn → Documentation, Git, Patterns & Design Principles, Estimate and Evaluate, Trello, Balance, Atlassian Tools, Consult & Coach, Architecture, Communication, Marketing Skills, Actors

---

## 1. DOCUMENTATION

### WHY? (Problem Statement)

**Code is read 10x more than written.** Documentation is the interface between:
- Past you ↔ Future you
- You ↔ Your team
- Team ↔ Other teams
- Engineering ↔ Product/Support/Sales

**Without documentation:**
- Onboarding takes months
- Tribal knowledge = bus factor of 1
- Decisions forgotten, re-litigated
- Incident response = guessing game
- Compliance audits = panic

---

### HOW? (Documentation Strategy: Diátaxis Framework)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DIÁTAXIS: FOUR DOCUMENTATION MODES                       │
├──────────────┬──────────────┬──────────────┬────────────────────────────────┤
│  TUTORIAL    │  HOW-TO      │  REFERENCE   │  EXPLANATION                  │
│  ─────────── │  ─────────   │  ──────────  │  ─────────────                │
│  Learning-   │  Problem-    │  Information-│  Understanding-               │
│  oriented    │  oriented    │  oriented    │  oriented                     │
│              │              │              │                               │
│  "Show me    │  "How do I   │  "What is    │  "Why does this              │
│   how to     │   ...?"      │   the API?"  │   work this way?"            │
│   get        │              │              │                               │
│   started"   │              │              │                               │
├──────────────┼──────────────┼──────────────┼────────────────────────────────┤
│  Step-by-    │  Recipe      │  Complete,   │  Context,                    │
│  step        │  format      │  accurate,   │  background,                 │
│  narrative   │  (Goal →     │  concise     │  design rationale,           │
│  (Build a    │   Steps →    │  (Parameters,│  trade-offs,                 │
│   thing)     │   Result)    │   Returns)   │  alternatives                │
├──────────────┼──────────────┼──────────────┼────────────────────────────────┤
│  New dev     │  Experienced │  API consumers│  Architects,               │
│  onboarding  │  dev solving │  implementing │  senior devs               │
│              │  specific    │  clients      │                               │
│              │  task        │              │                               │
└──────────────┴──────────────┴──────────────┴────────────────────────────────┘
```

---

### WHAT? (Documentation Types & Standards)

#### A. ARCHITECTURE DECISION RECORDS (ADRs)

```markdown
# ADR-0042: Use Event-Driven Choreography for Order Fulfillment

## Status: Accepted (2025-01-15)

## Context
Order fulfillment spans Payment, Inventory, Shipping, Notification services.
Current synchronous REST calls create temporal coupling and cascade failures.

## Decision
Adopt event-driven choreography with Kafka as event backbone.
Order Service emits `OrderCreated` → downstream services react independently.

## Consequences
### Positive
- Services decoupled temporally
- Failure isolation (one down ≠ all down)
- Natural audit trail for compliance
- Scales horizontally per service

### Negative
- Eventual consistency (order status not immediate)
- Distributed tracing required (correlation IDs)
- Schema evolution discipline (Schema Registry)
- Operational complexity (Kafka cluster)

## Alternatives Considered
1. **Synchronous Orchestration** — Rejected: central coupling, SPOF
2. **Saga Orchestrator** — Rejected: over-engineered for current complexity
3. **CDC from Order DB** — Rejected: leaks DB schema as contract

## Implementation Notes
- Outbox pattern for reliable event publishing
- Avro schemas in Confluent Schema Registry (BACKWARD compatibility)
- Correlation ID propagation via Kafka headers (W3C TraceContext)
- Dead letter topics + alerting for poison messages
```

#### B. RUNBOOKS (Operational Procedures)

```markdown
# Runbook: Order Service High Latency

## Symptoms
- p99 latency > 500ms (SLO: 200ms)
- Alert: `order_service_latency_p99_5m > 500ms`

## Diagnosis Steps
1. Check dashboard: `Order Service - Golden Signals`
   - [ ] CPU / Memory / Disk normal?
   - [ ] Request rate spike?
   - [ ] Error rate increase?
   - [ ] Downstream latency (Payment, Inventory, Kafka)?

2. If downstream latency:
   - Check Payment Service dashboard
   - Check Inventory Service dashboard
   - Check Kafka consumer lag (`kafka_consumergroup_lag`)

3. If DB latency:
   - `pg_stat_activity` — long-running queries?
   - Connection pool exhaustion?
   - Missing index? (check `pg_stat_statements`)

## Remediation
| Cause | Action | Owner |
|-------|--------|-------|
| Payment downstream | Circuit breaker: fail fast, queue orders | On-call |
| Kafka consumer lag | Scale consumer pods, check processing time | On-call |
| DB query regression | Rollback migration, add index, analyze plan | DBA |
| Connection pool | Increase pool size, fix leak | Dev team |

## Escalation
- 15 min: Page secondary on-call
- 30 min: Page team lead + architect
- 60 min: Incident commander, stakeholder comms

## Post-Incident
- Create incident report (template in `/incidents`)
- Add action items to sprint (prevent recurrence)
- Update runbook if gaps found
```

#### C. API DOCUMENTATION (OpenAPI + Examples)

```yaml
# openapi.yaml
openapi: 3.1.0
info:
  title: Order Service API
  version: 1.3.0
  description: |
    Manages customer orders. See [Architecture](https://wiki/internal/arch/order-service).
    
    **Authentication:** Bearer token (OAuth2, `orders:write` scope)
    **Rate Limit:** 1000 req/min per client (429 on exceed)
    **Idempotency:** `Idempotency-Key` header required for POST/PUT

servers:
  - url: https://api.prod.company.com/orders/v1
    description: Production
  - url: https://api.staging.company.com/orders/v1
    description: Staging

paths:
  /orders:
    post:
      summary: Create new order
      operationId: createOrder
      security: [{ OAuth2: ["orders:write"] }]
      parameters:
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
            examples:
              standard:
                summary: Standard order
                value:
                  customerId: "cust_abc123"
                  items:
                    - productId: "prod_xyz"
                      quantity: 2
                  shippingAddress:
                    line1: "123 Main St"
                    city: "Seattle"
                    state: "WA"
                    postalCode: "98101"
                    country: "US"
      responses:
        '201':
          description: Order created
          headers:
            Location:
              schema: { type: string, format: uri }
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '400':
          $ref: '#/components/responses/ValidationError'
        '409':
          $ref: '#/components/responses/IdempotencyConflict'
        '429':
          $ref: '#/components/responses/RateLimited'
```

#### D. ARCHITECTURE DIAGRAMS (C4 Model + Mermaid)

```mermaid
# System Context (C4 Level 1)
C4Context
title System Context - Order Management

Person(customer, "Customer", "Places orders via web/mobile")
Person(support, "Support Agent", "Views/modifies orders")
System(oms, "Order Management", "Core order lifecycle")
System_Ext(payment, "Payment Gateway", "Stripe/Adyen")
System_Ext(erp, "ERP System", "SAP - Inventory & Shipping")
System_Ext(email, "Email Service", "SendGrid")

Rel(customer, oms, "Places/Views orders", "HTTPS/JSON")
Rel(support, oms, "Manages orders", "HTTPS/JSON")
Rel(oms, payment, "Authorizes/captures", "HTTPS/JSON")
Rel(oms, erp, "Reserves stock, creates shipment", "SOAP/HTTP")
Rel(oms, email, "Sends confirmations", "HTTPS/JSON")
```

```mermaid
# Container Diagram (C4 Level 2)
C4Container
title Container - Order Management System

Container(api_gw, "API Gateway", "YARP", "Routing, Auth, Rate Limit")
Container(order_api, "Order API", ".NET 8 / ASP.NET Core", "REST + gRPC")
Container(order_worker, "Order Worker", ".NET 8 / BackgroundService", "Async processing")
ContainerDb(pg, "PostgreSQL 16", "Primary DB", "Orders, Events (Outbox)")
ContainerQueue(kafka, "Apache Kafka", "Event Backbone", "Order events, Integration events")
ContainerCache(redis, "Redis Cluster", "Caching", "Hot orders, Idempotency keys")

Rel(api_gw, order_api, "Routes /orders/*", "HTTPS")
Rel(order_api, pg, "Reads/Writes", "ADO.NET / EF Core")
Rel(order_api, redis, "Cache/Idempotency", "StackExchange.Redis")
Rel(order_api, kafka, "Publishes events", "Confluent.Kafka")
Rel(order_worker, kafka, "Consumes events", "Confluent.Kafka")
Rel(order_worker, pg, "Reads/Writes", "ADO.NET / EF Core")
```

---

### DOCUMENTATION GOVERNANCE

| Document Type | Owner | Review Cadence | Location |
|---------------|-------|----------------|----------|
| ADRs | Architect | On decision + quarterly | `/docs/adr/` (Git) |
| Runbooks | Team + SRE | Post-incident + quarterly | `/docs/runbooks/` (Git) |
| API Specs | API Owner | On change (CI gate) | `/docs/api/` + Portal |
| Architecture Diagrams | Architect | Quarterly | `/docs/arch/` (Mermaid in Git) |
| Onboarding Guide | EM + Tech Lead | Per hire + quarterly | `/docs/onboarding/` |
| Decision Log | Architect | Continuous | ADR index |

**CI Enforcement:**
- ADR template validation (required fields)
- OpenAPI spec linting (`spectral`)
- Diagram syntax check (`mermaid-cli`)
- Link checking (`markdown-link-check`)
- Freshness check (warn if > 90 days unreviewed)

---

## 2. GIT (VERSION CONTROL MASTERY)

### WHY? (Problem Statement)

Git is the **source of truth** for:
- Code + Infrastructure + Configuration + Documentation
- Audit trail (who, what, when, why)
- Collaboration (parallel work, review, merge)
- Deployment trigger (GitOps)
- Rollback capability

**Poor Git hygiene =** merge conflicts, lost work, unreadable history, broken bisect, compliance failures.

---

### HOW? (Branching Strategy: Trunk-Based Development)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TRUNK-BASED DEVELOPMENT                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   MAIN (protected, always deployable)                                       │
│   │                                                                         │
│   ├─── feat/order-validation (short-lived, <2 days)                        │
│   │       │                                                                 │
│   │       ├── commit: "Add quantity validation"                            │
│   │       ├── commit: "Fix edge case: negative quantity"                   │
│   │       └── PR #1234 → review → squash merge → delete branch             │
│   │                                                                         │
│   ├─── feat/payment-retry (short-lived)                                     │
│   │       └── PR #1235 → review → squash merge                              │
│   │                                                                         │
│   └─── fix/inventory-null-ref (hotfix, <4 hours)                           │
│           └── PR #1236 → review → squash merge                              │
│                                                                             │
│   TAGS: v1.3.0, v1.3.1, v1.4.0-rc.1                                        │
│                                                                             │
│   RULES:                                                                    │
│   ✅ Branch lifetime < 2 days (prefer hours)                               │
│   ✅ Small PRs (< 400 lines changed)                                       │
│   ✅ CI passes before merge (tests, lint, security, contract)              │
│   ✅ Squash merge (clean history, one commit per feature)                  │
│   ✅ Delete branch after merge                                             │
│   ❌ No long-lived release branches                                        │
│   ❌ No merge commits (except explicit revert)                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Git Practices & Commands)

#### A. COMMIT MESSAGE CONVENTION (Conventional Commits)

```bash
# Format: <type>(<scope>): <subject>
#         <blank line>
#         <body>
#         <blank line>
#         <footer>

# Types: feat, fix, docs, style, refactor, perf, test, chore, revert
# Scope: module, component, or area affected

# Examples:
feat(orders): add quantity validation to order items

Validate that quantity > 0 and <= maxStockPerOrder.
Returns 400 with validation error details.

Closes #123
```

```bash
fix(payment): handle network timeout on authorization

Increased HTTP timeout from 30s to 60s for payment provider calls.
Added retry with exponential backoff (3 attempts) for 5xx errors.

Fixes #456
Co-authored-by: Jane Dev <jane@company.com>
```

```bash
refactor(inventory)!: extract stock reservation to separate service

BREAKING CHANGE: InventoryService.ReserveStock() now returns Result<ReservationId>
instead of throwing. Callers must handle Result pattern.

Migration guide: https://wiki/internal/migration/inventory-v2
```

#### B. USEFUL GIT ALIASES (`.gitconfig`)

```ini
[alias]
    # Logging
    lg = log --graph --pretty=format:'%C(yellow)%h%Creset %C(blue)%ad%Creset %s %C(green)(%an)%Creset %C(red)%d%Creset' --date=short
    lg1 = log --oneline --graph --decorate --all
    
    # Status & Diff
    st = status -sb
    ds = diff --stat
    dw = diff --word-diff
    
    # Branching
    co = checkout
    cob = checkout -b
    br = branch -vv
    brd = branch -d
    brD = branch -D
    
    # Staging
    aa = add -A
    ap = add -p
    unstage = reset HEAD --
    
    # Committing
    cm = commit -m
    ca = commit --amend --no-edit
    fixup = commit --fixup
    squash = commit --squash
    
    # History Rewriting
    rbi = rebase -i
    rb = rebase
    rbc = rebase --continue
    rba = rebase --abort
    
    # Remote
    pu = push -u origin HEAD
    pf = push --force-with-lease
    pr = pull --rebase
    
    # Cleanup
    clean-branches = "!git branch --merged | grep -v '\\*\\|main\\|master' | xargs -n 1 git branch -d"
    
    # Search
    find = "!git log --all --grep="
    find-code = "!git log -p --all -S "
    
    # Stash
    sl = stash list
    sp = stash push -m
    sa = stash apply
    sd = stash drop
```

#### C. INTERACTIVE REBASE WORKFLOW

```bash
# Before PR: Clean up commit history
git fetch origin
git rebase -i origin/main

# In editor, mark commits:
# pick  abc123  feat: add validation
# fixup def456  fix: typo in validation
# fixup ghi789  fix: edge case
# squash jkl012  docs: update comments
# 
# Result: Single clean commit "feat(orders): add quantity validation"
```

#### D. GIT HOOKS (`.git/hooks/pre-commit` + `commit-msg`)

```bash
#!/bin/bash
# .git/hooks/pre-commit
# Runs: lint, format, test (fast), secret scan

set -e

# 1. Format check (fail if not formatted)
dotnet format --verify-no-changes --verbosity quiet
# OR: cargo fmt --check
# OR: gofmt -l . | grep -q . && exit 1

# 2. Lint
dotnet build --no-restore -v q  # or: golangci-lint run, eslint .

# 3. Fast tests (unit only, < 30s)
dotnet test --filter "Category=Unit" --no-build -v q

# 4. Secret scan (gitleaks, trufflehog)
gitleaks detect --source . --no-git --verbose
```

```bash
#!/bin/bash
# .git/hooks/commit-msg
# Enforces Conventional Commits

commit_msg=$(cat "$1")
pattern='^(feat|fix|docs|style|refactor|perf|test|chore|revert)(\(.+\))?: .{1,72}'

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ Invalid commit message format"
    echo "Expected: <type>(<scope>): <subject>"
    echo "Types: feat, fix, docs, style, refactor, perf, test, chore, revert"
    exit 1
fi
```

---

## 3. PATTERNS & DESIGN PRINCIPLES

### WHY? (Problem Statement)

**Patterns are reusable solutions to recurring problems.** They provide:
- Shared vocabulary ("Use Circuit Breaker here")
- Proven approaches (avoid reinventing)
- Communication efficiency
- Quality baseline

**Principles guide decisions when patterns don't fit.**

---

### HOW? (Pattern Categories)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DESIGN PATTERN CATEGORIES                                │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│   CREATIONAL    │   STRUCTURAL    │   BEHAVIORAL    │   ARCHITECTURAL       │
│   (Object       │   (Composition) │   (Interaction) │   (System-Level)      │
│    Creation)    │                 │                 │                       │
├─────────────────┼─────────────────┼─────────────────┼───────────────────────┤
│ • Factory       │ • Adapter       │ • Observer      │ • Circuit Breaker     │
│ • Builder       │ • Decorator     │ • Strategy      │ • Bulkhead            │
│ • Singleton*    │ • Facade        │ • Command       │ • Retry/Timeout       │
│ • Prototype     │ • Proxy         │ • State         │ • Rate Limiter        │
│ • Pool          │ • Composite     │ • Template      │ • Cache-Aside         │
│                 │ • Bridge        │ • Mediator      │ • Saga                │
│                 │ • Flyweight     │ • Visitor       │ • CQRS                │
│                 │                 │ • Chain of Resp │ • Event Sourcing      │
│                 │                 │                 │ • Outbox              │
│                 │                 │                 │ • API Gateway         │
│                 │                 │                 │ • Service Mesh        │
│                 │                 │                 │ • Strangler Fig       │
└─────────────────┴─────────────────┴─────────────────┴───────────────────────┘

* Singleton: Avoid in distributed systems. Use DI container scoped lifestyle.
```

---

### WHAT? (Essential Patterns for Architects)

#### A. RESILIENCE PATTERNS (Must Know)

```csharp
// CIRCUIT BREAKER (Polly)
var circuitBreaker = Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (ex, ts) => logger.LogWarning("Circuit opened for {Duration}", ts),
        onReset: () => logger.LogInformation("Circuit closed"),
        onHalfOpen: () => logger.LogInformation("Circuit half-open"));

// BULKHEAD (SemaphoreSlim for concurrency limit)
var bulkhead = Policy.BulkheadAsync<HttpResponseMessage>(
    maxParallelization: 10,
    maxQueuingActions: 20,
    onBulkheadRejectedAsync: ctx => 
        Task.FromResult(new HttpResponseMessage(HttpStatusCode.TooManyRequests)));

// RETRY WITH JITTER
var retry = Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => (int)r.StatusCode >= 500)
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => 
            TimeSpan.FromSeconds(Math.Pow(2, attempt)) 
            + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000)),
        onRetry: (outcome, ts, attempt) => 
            logger.LogWarning("Retry {Attempt} after {Delay}: {Error}", attempt, ts, outcome.Exception?.Message));

// COMPOSE: Wrap HttpClient
var policyWrap = Policy.WrapAsync(retry, circuitBreaker, bulkhead);

var response = await policyWrap.ExecuteAsync(async () => 
    await httpClient.PostAsync("/api/orders", content));
```

#### B. CQRS + EVENT SOURCING (Architecture Patterns)

```csharp
// COMMAND SIDE (Write Model)
public record CreateOrderCommand(
    CustomerId CustomerId,
    IReadOnlyList<OrderItemData> Items,
    ShippingAddress Address
) : IRequest<Result<OrderId>>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IDomainEventPublisher _publisher;

    public async Task<Result<OrderId>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items, cmd.Address);
        await _repository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync(ct);
        await _publisher.PublishAsync(order.DomainEvents);
        return Result.Success(order.Id);
    }
}

// QUERY SIDE (Read Model - Optimized)
public record GetOrderSummaryQuery(OrderId Id) : IRequest<OrderSummaryDto?>;

public class GetOrderSummaryHandler : IRequestHandler<GetOrderSummaryQuery, OrderSummaryDto?>
{
    private readonly IOrderReadRepository _readRepo; // Different interface!

    public async Task<OrderSummaryDto?> Handle(GetOrderSummaryQuery query, CancellationToken ct)
        => await _readRepo.GetSummaryAsync(query.Id, ct);
}

// READ MODEL (Denormalized, No Behavior)
public record OrderSummaryDto(
    OrderId Id,
    string CustomerName,
    OrderStatus Status,
    Money Total,
    int ItemCount,
    DateTime CreatedAt,
    DateTime? ShippedAt
);

// EVENT SOURCING (Append-Only Event Store)
public interface IEventStore
{
    Task AppendAsync(StreamName stream, IEnumerable<EventData, IEnumerable<EventData>, ExpectedVersion);
    Task<IReadOnlyList<ResolvedEvent>> ReadAsync(StreamName, StreamPosition, int);
    Task SubscribeAsync(StreamName, Func<ResolvedEvent, Task>, CancellationToken);
}
```

#### C. DDD BUILDING BLOCKS (Domain Patterns)

```csharp
// ENTITY: Identity + Lifecycle
public class Order : Entity<OrderId>, IAggregateRoot
{
    private readonly List<OrderItem> _items = new();
    private readonly List<IDomainEvent> _events = new();

    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _events.AsReadOnly();

    private Order() { } // EF Core

    public static Order Create(CustomerId customerId, IEnumerable<OrderItemData> items, ShippingAddress address)
    {
        var order = new Order { Id = OrderId.New(), CustomerId = customerId, Status = OrderStatus.Draft };
        foreach (var item in items) order.AddItem(item.ProductId, item.Quantity, item.UnitPrice);
        order.SetShippingAddress(address);
        order.Raise(new OrderCreatedEvent(order.Id, order.CustomerId, order.Items));
        return order;
    }

    public void AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null) existing.IncreaseQuantity(quantity);
        else _items.Add(new OrderItem(productId, quantity, unitPrice));
    }

    public Result Submit()
    {
        if (Status != OrderStatus.Draft) return Result.Failure("Not in draft");
        if (!_items.Any()) return Result.Failure("Empty order");
        Status = OrderStatus.Submitted;
        Raise(new OrderSubmittedEvent(Id, CustomerId, Items));
        return Result.Success();
    }

    protected void Raise(IDomainEvent @event) => _events.Add(@event);
    public void ClearEvents() => _events.Clear();
}

// VALUE OBJECT: Immutable, No Identity
public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);
    public Money Add(Money other) 
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Currency mismatch");
        return new(Amount + other.Amount, Currency);
    }
}

// AGGREGATE ROOT: Consistency Boundary
// Only aggregate root referenced externally
// All changes go through aggregate root methods
// Invariants enforced within boundary

// DOMAIN EVENT: Something That Happened
public record OrderSubmittedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    IReadOnlyCollection<OrderItem> Items,
    DateTime OccurredAt = default
) : IDomainEvent
{
    public DateTime OccurredAt { get; } = OccurredAt == default ? DateTime.UtcNow : OccurredAt;
}
```

---

### DESIGN PRINCIPLES (Decision Compass)

| Principle | Essence | When to Apply |
|-----------|---------|---------------|
| **SOLID** | S: Single Responsibility, O: Open/Closed, L: Liskov, I: Interface Segregation, D: Dependency Inversion | Class/module design |
| **DRY** | Don't Repeat Yourself — but prefer duplication over wrong abstraction | Code reuse decisions |
| **KISS** | Keep It Simple, Stupid — simplest solution that works | Always |
| **YAGNI** | You Aren't Gonna Need It — don't build for hypothetical futures | Feature scoping |
| **Conway's Law** | Org structure = System structure | Team/service boundaries |
| **Law of Demeter** | Talk only to immediate friends | Coupling reduction |
| **Tell, Don't Ask** | Command objects, don't query state then decide | OOP vs procedural |
| **Explicit Dependencies** | Constructor injection, no service locator | DI container setup |
| **Fail Fast** | Validate early, crash loudly on invariant violation | Error handling |
| **Immutability** | Prefer immutable data, pure functions | Concurrency, testing |
| **Composition over Inheritance** | Build behavior by combining, not extending | Reuse strategy |
| **Bounded Context** | Explicit context boundaries, ubiquitous language | Microservices, modules |

---

## 4. ESTIMATE AND EVALUATE

### WHY? (Problem Statement)

**Estimation is not prediction — it's risk reduction.** Good estimates:
- Reveal unknowns early
- Drive scope conversations
- Enable planning & commitment
- Build stakeholder trust

**Bad estimates =** death marches, broken promises, lost credibility.

---

### HOW? (Estimation Techniques)

#### A. BACK-OF-ENVELOPE (System Design)

```bash
# QPS from DAU
avg_qps = DAU * actions_per_user_per_day / 86400
peak_qps = avg_qps * peak_factor (typically 3-10x)

# Example: 1M DAU, 10 actions/day
avg = 1,000,000 * 10 / 86,400 ≈ 115 QPS
peak = 115 * 5 = 575 QPS

# Storage
daily_writes = DAU * writes_per_user
storage_per_year = daily_writes * bytes_per_record * 365 * replication_factor

# Example: 100M tweets/day, 500 bytes, 3 replicas
= 100M * 500 * 365 * 3 ≈ 55 TB/year (before compression)

# Capacity Rules of Thumb
# Stateless service: ~1K QPS per instance (simple CRUD, cached)
# PostgreSQL primary: ~5-10K writes/sec before sharding
# Redis: ~100K ops/sec per node
# Kafka: ~1MB/sec per partition (produce+consume)
```

#### B. SOFTWARE PROJECT ESTIMATION

| Technique | When | Process |
|-----------|------|---------|
| **T-Shirt Sizes** | Early discovery | XS/S/M/L/XL → map to days (1/3/5/10/20) |
| **Story Points** | Sprint planning | Fibonacci (1,2,3,5,8,13,21) — relative complexity |
| **PERT / Three-Point** | High uncertainty | (Optimistic + 4×Most Likely + Pessimistic) / 6 |
| **Monte Carlo** | Release forecasting | Simulate 10K runs with velocity distribution |
| **Reference Class** | Similar past work | "Last payment integration took 3 weeks" |

#### C. EVALUATION CRITERIA (Architecture Trade-offs)

```
DECISION MATRIX: Choose Event Broker

| Criterion          | Weight | Kafka | RabbitMQ | NATS | EventBridge |
|--------------------|--------|-------|----------|------|-------------|
| Throughput         | 5      | 5     | 3        | 4    | 4           |
| Latency            | 4      | 3     | 4        | 5    | 3           |
| Ordering           | 5      | 5     | 2        | 3    | 2           |
| Replay             | 5      | 5     | 2        | 2    | 3           |
| Operational Cost   | 3      | 2     | 3        | 4    | 5           |
| Team Expertise     | 4      | 4     | 3        | 2    | 3           |
| Vendor Lock-in     | 2      | 5     | 5        | 5    | 1           |
| WEIGHTED SCORE     |        | 4.3   | 3.1      | 3.6  | 3.0         |

★ Kafka selected — highest score on critical criteria (throughput, ordering, replay)
```

---

## 5. TOOLS (ATLASSIAN / TRELLO / JIRA / CONFLUENCE)

### WHY? (Problem Statement)

**Tools amplify process — they don't replace it.** Good tooling:
- Makes the right thing easy
- Makes the wrong thing visible
- Reduces cognitive load
- Enables automation

---

### HOW? (Tool Configuration Standards)

#### A. JIRA WORKFLOW (Standardized)

```
BACKLOG → REFINED → READY → IN PROGRESS → IN REVIEW → DONE
                ↑         │                  │
                │         ▼                  ▼
                │    BLOCKED ←──────────────┘
                │         │
                └─────────┘ (clarification needed)

STATUSES:
- Backlog: Unprioritized, unrefined
- Refined: Acceptance criteria clear, estimated, dependencies identified
- Ready: Sprint-ready, no blockers, committed by team
- In Progress: Actively worked (WIP limit applies)
- In Review: PR open, awaiting review
- Done: Merged, deployed, verified in staging
- Blocked: External dependency, waiting on info

TRANSITIONS REQUIRE:
- Ready → In Progress: Assign to self, move to sprint
- In Progress → In Review: PR opened, CI passing
- In Review → Done: Approved + merged + deployed
- Any → Blocked: Comment with blocker reason + owner
```

#### B. CONFLUENCE SPACE STRUCTURE

```
📁 TEAM SPACE
├── 📁 Architecture
│   ├── ADR Index
│   ├── System Context Diagrams
│   ├── Technology Radar
│   └── Standards & Patterns
├── 📁 Runbooks
│   ├── Index (by service)
│   ├── Incident Response
│   └── Deployment Procedures
├── 📁 Onboarding
│   ├── Day 1 Checklist
│   ├── Dev Environment Setup
│   ├── Codebase Tour
│   └── Team Contacts
├── 📁 Processes
│   ├── Sprint Ceremonies
│   ├── Release Process
│   ├── Incident Management
│   └── Security Review
└── 📁 Decisions (Meeting Notes)
    ├── Architecture Reviews
    ├── Sprint Retrospectives
    └── Planning Sessions
```

---

## 6. BALANCE (WORK-LIFE / TECHNICAL)

### WHY? (Problem Statement)

**Sustainable pace = better architecture.** Burnout leads to:
- Shortcut decisions (technical debt)
- Reduced code quality
- Knowledge silos (no pairing/mentoring)
- Turnover (loss of institutional memory)

---

### HOW? (Balance Practices)

| Dimension | Practice | Anti-Pattern |
|-----------|----------|--------------|
| **Work Hours** | Core hours (10-3), async communication, no weekend deploys | "Hero culture", late-night fixes |
| **On-Call** | Rotation (1 week), max 2 incidents/week, compensation time | Same people always, no backup |
| **Learning** | 10% time (Friday), conference budget, internal tech talks | Zero learning time |
| **Technical Debt** | 20% sprint capacity, debt register, paydown sprints quarterly | "We'll fix it later" |
| **Meetings** | No-meeting Wednesdays, async updates, 25/50 min default | Calendar tetris, 8hr meeting days |
| **Vacation** | Minimum 3 weeks/year, mandatory disconnect, coverage plan | Rollover, checking email on leave |

---

## 7. CONSULT & COACH

### WHY? (Problem Statement)

**Architects multiply team capability.** Consulting & coaching:
- Transfers knowledge (reduces bus factor)
- Improves decision-making at all levels
- Builds engineering culture
- Prevents architecture by committee

---

### HOW? (Coaching Models)

#### A. GROW MODEL (Coaching Conversations)

```
G - GOAL:       "What do you want to achieve?" (specific, measurable)
R - REALITY:    "What's happening now?" (facts, not interpretations)
O - OPTIONS:    "What could you do?" (brainstorm, no judgment)
W - WILL:       "What will you do?" (commitment, timeline, support needed)

Example:
Goal: "Reduce order service latency p99 from 400ms to 200ms"
Reality: "Current p99 400ms, DB queries 60%, cache hit 40%"
Options: "1) Add read replicas 2) Improve cache strategy 3) Denormalize 4) CQRS"
Will: "Try cache TTL + write-through first (2 weeks), measure, then decide"
```

#### B. FEEDBACK FRAMEWORK (SBI)

```
SITUATION: "In yesterday's architecture review..."
BEHAVIOR:  "...you interrupted the security engineer twice when they raised concerns..."
IMPACT:    "...the team didn't hear the threat model, and we missed a data residency requirement."

vs.

SITUATION: "In yesterday's review..."
BEHAVIOR:  "...you asked clarifying questions about the event schema..."
IMPACT:    "...the team caught a breaking change before merge. Thank you."
```

#### C. TEACHING TECHNIQUES

| Technique | When | How |
|-----------|------|-----|
| **Pair Programming** | Complex feature, knowledge transfer | Driver/navigator, rotate every 30 min |
| **Architecture Kata** | Team learning | 90-min design exercise, present & critique |
| **Code Review Teaching** | PR reviews | "Why this pattern?" not "Change this" |
| **Decision Shadowing** | Junior architects | Include in ADR process, explain reasoning |
| **Incident Retrospective** | Post-incident | Blameless, focus on system, not person |

---

## 8. COMMUNICATION

### WHY? (Problem Statement)

**Architecture is communication.** The best design fails if:
- Stakeholders don't understand it
- Teams implement differently
- Decisions aren't socialized
- Trade-offs aren't articulated

---

### HOW? (Communication Patterns)

#### A. AUDIENCE-ADAPTED MESSAGING

| Audience | What They Need | Format | Language |
|----------|----------------|--------|----------|
| **Executive** | Business outcome, cost, risk, timeline | 1-pager, dashboard | Business value, ROI |
| **Product** | User impact, dependencies, timeline | User journey, sequence diagram | Features, milestones |
| **Engineering** | Technical design, contracts, NFRs | C4 diagrams, ADRs, API specs | Technical, precise |
| **Operations** | Deploy, monitor, scale, debug | Runbooks, dashboards, alerts | Procedural, actionable |
| **Security/Compliance** | Data flow, controls, evidence | DFD, threat model, audit trail | Regulatory, risk |

#### B. ARCHITECTURE PRESENTATION STRUCTURE

```
1. CONTEXT (30 sec)
   - Business problem + constraints
   - "We need to handle 10x Black Friday traffic"

2. DECISION (1 min)
   - "We chose event-driven choreography with Kafka"
   - One sentence, memorable

3. ALTERNATIVES CONSIDERED (1 min)
   - "Considered sync orchestration (rejected: coupling) and CDC (rejected: leaky)"

4. TRADE-OFFS (2 min)
   - "Gain: independence, resilience | Cost: eventual consistency, ops complexity"

5. IMPLEMENTATION PLAN (1 min)
   - "Phase 1: Outbox pattern (2 sprints) → Phase 2: Extract fulfillment (3 sprints)"

6. RISKS & MITIGATIONS (1 min)
   - "Risk: Schema evolution | Mitigation: Schema Registry + contract tests"

7. QUESTIONS / NEXT STEPS
```

#### C. WRITTEN COMMUNICATION STANDARDS

- **BLUF** (Bottom Line Up Front) — decision/ask in first paragraph
- **Executive Summary** — for docs > 2 pages
- **Visuals First** — diagrams before text
- **Glossary** — define domain terms
- **Action Items** — owner, due date, status

---

## 9. MARKETING SKILLS (INTERNAL)

### WHY? (Problem Statement)

**"Build it and they will come" fails internally too.** Architects must:
- Sell vision to leadership (funding)
- Recruit teams to adopt standards (adoption)
- Advocate for quality (priority)
- Celebrate wins (momentum)

---

### HOW? (Internal Marketing)

| Activity | Purpose | Example |
|----------|---------|---------|
| **Architecture Newsletter** | Monthly visibility | "This month: 3 services migrated, latency -40%, 2 ADRs" |
| **Tech Talks / Brown Bags** | Knowledge sharing | "Deep dive: How we reduced Kafka lag 90%" |
| **Architecture Office Hours** | Accessibility | "Fridays 2-3pm, bring any design question" |
| **Decision Announcements** | Transparency | "ADR-0042 accepted: Event-driven fulfillment. See wiki." |
| **Win Celebrations** | Momentum | "Order service hit 99.99% availability! 🎉" |
| **RFC Process** | Inclusive decisions | "RFC-012: New auth model — comments due Friday" |

---

## 10. ACTORS (STAKEHOLDER MANAGEMENT)

### WHY? (Problem Statement)

**Every architecture decision affects people.** Mapping actors ensures:
- Right people consulted
- Concerns addressed early
- Buy-in secured
- No surprises

---

### HOW? (Actor Map)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STAKEHOLDER MAP (RACI)                              │
├─────────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│ ACTOR               │ RESPONS. │ ACCOUNT. │ CONSULT. │ INFORM.  │ CONCERNS │
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ CTO / VP Eng        │          │    A     │    C     │    I     │ Strategy,│
│                     │          │          │          │          │ Budget   │
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Product Manager     │    R     │          │    A     │    I     │ Features,│
│                     │          │          │          │          │ Timeline │
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Architect (You)     │    R     │    A     │          │    I     │ Quality, │
│                     │          │          │          │          │ Standards│
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Tech Lead           │    R     │          │    C     │    I     │ Delivery,│
│                     │          │          │          │          │ Team     │
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Developers          │    R     │          │    C     │    I     │ Code,    │
│                     │          │          │          │          │ Tests    │
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ SRE / Platform      │    R     │          │    C     │    I     │ Ops,     │
│                     │          │          │          │          │ Scale    │
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Security            │          │          │    C     │    I     │ Compliance,│
│                     │          │          │          │          │ Threats  │
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Data Engineering    │          │          │    C     │    I     │ Analytics,│
│                     │          │          │          │          │ Pipelines│
├─────────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Support             │          │          │    C     │    I     │ Debug,   │
│                     │          │          │          │          │ Runbooks │
└─────────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

R = Responsible (does the work)
A = Accountable (decision owner, only ONE)
C = Consulted (two-way, before decision)
I = Informed (one-way, after decision)
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Context | Anti-Pattern | Why Avoid |
|---------|---------|--------------|-----------|
| **Diátaxis Documentation** | All documentation | "Docs in Confluence" (unstructured) | Unfindable, outdated, inconsistent |
| **Trunk-Based Development** | Team > 3 | Long-lived feature branches | Merge hell, delayed integration |
| **Conventional Commits** | All repos | Free-form messages | Unreadable history, no changelog gen |
| **ADR for Decisions** | Structural choices | Verbal/Slack decisions | Lost context, re-litigation |
| **C4 Diagrams** | Architecture comms | Box-and-arrow only | Ambiguous, not scalable |
| **GROW Coaching** | 1:1s, mentoring | Giving answers | Dependency, no growth |
| **SBI Feedback** | Performance, reviews | "You're good/bad" | Non-actionable, defensive |
| **RACI for Decisions** | Cross-team | "Everyone decide" | No ownership, endless debate |
| **BLUF Communication** | All written | Mystery novel structure | Time wasted, missed point |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. Documentation: "How do you keep architecture documentation current?"

**Answer:**
```
1. DOCS AS CODE — Markdown in Git, reviewed in PR, CI-validated
2. ADR PROCESS — Every structural decision = ADR (template, required fields)
3. DIAGRAMS IN CODE — Mermaid/PlantUML in repo, rendered in CI
4. OWNERSHIP — Every doc has owner + review cadence (quarterly)
5. FRESHNESS CHECKS — CI fails if ADR > 90 days unreviewed
6. INCIDENT-DRIVEN — Post-incident: "What docs would have helped?"
7. ONBOARDING TEST — New hire reads docs, fixes confusion → PR
```

---

### 2. Git: "Explain your branching strategy and why."

**Answer:**
```
TRUNK-BASED DEVELOPMENT:
- Short-lived branches (< 2 days, < 400 lines)
- Squash merge to main (clean history, bisect works)
- Feature flags for incomplete work
- Main always deployable

WHY:
- Continuous integration (real CI)
- Small batches = less risk, faster feedback
- No merge hell, no stale branches
- Enables CI/CD, GitOps

EXCEPTIONS:
- Release tags (v1.3.0) for rollback reference
- Critical hotfix branch (cherry-pick to main after)
```

---

### 3. Patterns: "When would you use Circuit Breaker vs Retry vs Bulkhead?"

**Answer:**
```
RETRY: Transient failures (network blip, GC pause, brief lock)
- Config: Exponential backoff + jitter, max 3 attempts
- Don't retry: 4xx, non-idempotent, timeout already near

CIRCUIT BREAKER: Persistent downstream failure (service down, overloaded)
- States: Closed (normal) → Open (fail fast) → Half-open (probe)
- Config: 5 failures in 10s → open 30s → probe
- Prevents cascade, gives downstream recovery time

BULKHEAD: Resource isolation (thread pool, connections, memory)
- Limits blast radius of one failing dependency
- Config: Max 10 concurrent calls, queue 20
- Complements: Circuit breaker stops calls, bulkhead limits concurrent

COMPOSE: Retry → Circuit Breaker → Bulkhead (inner to outer)
```

---

### 4. Estimation: "How do you estimate a project with high uncertainty?"

**Answer:**
```
1. BREAK DOWN — Decompose to smallest verifiable units
2. SPIKE — Time-box (2-3 days) riskiest unknowns, produce data
3. THREE-POINT — Optimistic / Most Likely / Pessimistic per item
4. MONTE CARLO — Simulate 10K runs with velocity distribution
5. BUFFER — Add 20-30% for unknown-unknowns (not in line items)
6. RANGE — Communicate as "6-10 weeks (80% confidence)" not "8 weeks"
7. RE-ESTIMATE — Every sprint, update with actuals

KEY: Estimate in ranges, commit to dates only for near-term.
```

---

### 5. Coaching: "How do you help a senior developer grow into an architect?"

**Answer:**
```
GROWTH PATH (12-18 months):

PHASE 1: EXPAND SCOPE (3-6 mo)
- Lead architecture for team-level features
- Write first ADRs (with review)
- Present at architecture review
- Mentor 1-2 juniors

PHASE 2: CROSS-TEAM (6-12 mo)
- Drive multi-team initiative (Strangler Fig, platform capability)
- Own technology radar entry (evaluate, recommend)
- Facilitate Event Storming / ADR sessions
- Incident commander for sev-1

PHASE 3: STRATEGIC (12-18 mo)
- Define target state architecture (3-year)
- Influence technology radar (Adopt/Trial)
- Coach other tech leads
- Architecture Board member

COACHING:
- Monthly 1:1 (GROW model)
- Shadow on decisions (explain reasoning)
- Feedback on ADRs, presentations (SBI)
- Stretch assignments with safety net
```

---

### 6. Communication: "How do you present a complex architectural decision to executives?"

**Answer:**
```
EXECUTIVE ONE-PAGER (Max 1 page):

TITLE: Event-Driven Order Fulfillment — Reduce Black Friday Risk

CONTEXT: Current sync architecture failed at 3x load last year. 
         45 min downtime = $2.3M lost revenue.

DECISION: Migrate to async event-driven (Kafka) over 2 quarters.

INVESTMENT: 3 engineers × 6 months = $450K
            + Kafka managed service $120K/yr

RETURN:     99.99% availability target (vs 99.5%)
            Horizontal scaling (no vertical limits)
            Foundation for real-time features (tracking, alerts)

RISKS:      Team learning curve (mitigation: training + pair programming)
            Operational complexity (mitigation: managed Kafka, runbooks)

ALTERNATIVES REJECTED:
- Vertical scaling: $2M/yr, hard limit at 5x
- Sync orchestration: Same coupling, added SPOF

ASK: Approve Q1-Q2 investment. Decision needed by Jan 15 for procurement.

VISUAL: [C4 Context Diagram showing before/after]
```

---

### 7. Tools: "How do you manage technical debt in Jira?"

**Answer:**
```
DEBT AS FIRST-CLASS WORK ITEMS:

1. LABEL: `tech-debt` + `area:payments` + `severity:high`
2. TEMPLATE:
   - Description: What, where, why
   - Impact: Performance, risk, velocity drag
   - Effort: Story points (estimated)
   - Risk of not fixing: Incident likelihood, compliance gap
   - Acceptance criteria: Measurable (latency < X, coverage > Y%)
3. PRIORITIZATION:
   - Quarterly debt sprint (20% capacity)
   - High severity + high impact = next sprint
   - Link to incidents (traceability)
4. VISIBILITY:
   - Dashboard: Debt by area, severity, age
   - Sprint review: Debt paid down / added
   - Architecture Board: Top 10 debt items quarterly
```

---

### 8. Balance: "How do you prevent burnout in your team?"

**Answer:**
```
SYSTEMIC APPROACHES:

1. WIP LIMITS — Kanban WIP per person (max 2), per team (max 8)
2. NO-MEETING WEDNESDAYS — Deep work protected
3. ON-CALL ROTATION — 1 week, max 2 pages/week, comp time mandatory
4. 20% LEARNING TIME — Fridays, tracked, celebrated
5. DEPLOY FREEZE — No deploys Fri/weekend without VP approval
6. RETROSPECTIVE ACTIONS — Must have owner + due date, tracked
7. VACATION POLICY — Min 3 weeks, coverage plan required, no email
8. MANAGER 1:1s — Monthly, include "energy check" (1-10 scale)

MEASURE: eNPS quarterly, voluntary turnover, incident after-hours
```

---

### 9. Stakeholders: "How do you handle a Product Manager pushing for a deadline that compromises architecture?"

**Answer:**
```
RACI CLARITY + TRADE-OFF TRANSPARENCY:

1. ACKNOWLEDGE — "I understand the business need for Q1 launch."
2. QUANTIFY TRADE-OFFS — "If we skip the outbox pattern:
   - Risk: 5% order loss during payment outage
   - Cost: Manual reconciliation ~$50K/incident
   - Time to fix later: 3x current effort"
3. OFFER OPTIONS —
   A) Full pattern (6 weeks, 99.99% reliability)
   B) Partial — sync with compensation queue (4 weeks, 99.9%)
   C) Tech debt — sync now, outbox in Q2 (3 weeks, 99.5% + debt)
4. DECISION RIGHTS — PM owns scope/date (A), Architect owns quality (C)
   - If PM chooses C, document in ADR: "Accepted debt, revisit Q2"
5. ESCALATION — If safety/compliance, Architect = Accountable (A)
```

---

### 10. Marketing: "How do you drive adoption of a new architectural standard?"

**Answer:**
```
ADOPTION CURVE STRATEGY:

1. PAVED ROAD — Make standard the easiest path
   - Template repo (git clone → working service in 5 min)
   - CI pipeline pre-configured
   - Libraries published to internal registry
   - Documentation with copy-paste examples

2. EARLY ADOPTERS — Recruit 1-2 teams for pilot
   - Co-design with them
   - Celebrate their success publicly
   - Fix friction points before broad rollout

3. RFC PROCESS — Broad input, clear decision
   - 2-week comment period
   - Architecture Board decides
   - Communicate: "ADR-042 accepted, effective Sprint 12"

4. MIGRATION PATH — No big bang
   - New services: Must use standard
   - Existing: Migrate on next major touch
   - Exception process (RACI) for legitimate cases

5. MEASURE & ITERATE — Dashboard
   - % services compliant
   - Time to onboard new service
   - Developer satisfaction (survey)
```

---

### 11. Actors: "How do you identify all stakeholders for a new platform capability?"

**Answer:**
```
STAKEHOLDER DISCOVERY CHECKLIST:

DIRECT USERS:
☐ Application teams (consumers)
☐ Platform team (operators)
☐ Security team (reviewers)
☐ Data team (downstream analytics)

INDIRECT USERS:
☐ Support (debugging)
☐ Compliance (audit)
☐ Finance (cost allocation)
☐ Legal (contracts, data residency)

AFFECTED PARTIES:
☐ Teams sharing infrastructure (blast radius)
☐ Upstream providers (API changes)
☐ Downstream consumers (breaking changes)
☐ On-call (new alerts, runbooks)

DECISION MAKERS:
☐ Architecture Board (standards)
☐ VP Engineering (funding)
☐ CTO (strategy alignment)

PROCESS:
1. Brainstorm with team (15 min)
2. Validate with EMs (anyone missing?)
3. Map RACI (see Actor Map template)
4. Schedule consultations (C) before decision
5. Communicate decision to (I) after
```

---

### 12. Patterns: "Explain the Outbox Pattern and why it's critical for microservices."

**Answer:**
```
PROBLEM: Dual write — DB commit succeeds, message publish fails
         → Inconsistent state (DB says order created, but no event)

OUTBOX PATTERN:
1. Single DB transaction:
   - Write business data (Order)
   - Write event to OUTBOX table (same transaction)
2. Background process (relay):
   - Polls outbox table (unpublished)
   - Publishes to message broker
   - Marks outbox record published
3. Idempotent consumers handle duplicates

WHY CRITICAL:
- Atomicity: DB + Event = single transaction
- Reliability: No message loss (DB durability)
- Ordering: Events published in DB commit order
- Simplicity: Application code unchanged

IMPLEMENTATION:
- Table: id, event_type, payload, created_at, published_at
- Index: WHERE published_at IS NULL ORDER BY created_at
- Polling interval: 100ms-1s (latency vs load)
- Cleanup: TTL or periodic delete published > 7 days
```

---

### 13. Estimation: "Your team consistently underestimates. How do you fix it?"

**Answer:**
```
CALIBRATION LOOP:

1. TRACK — Record estimate vs actual for every story (Jira: Original Estimate vs Time Spent)
2. ANALYZE — Monthly: "We estimate 1.5x actual" or "Frontend 2x, Backend 1.2x"
3. ADJUST — Apply team-specific multiplier to future estimates
4. DECOMPOSE — Large stories (>5 pts) = "not understood, break down"
5. SPIKE — Unknowns = time-boxed research, not guess
6. RETROSPECTIVE — "What surprised us this sprint?" → add to estimation checklist

CHECKLIST ADDITIONS FROM PAST SURPRISES:
☐ Migration script tested on prod-scale data?
☐ Third-party API rate limits verified?
☐ Cross-team dependency confirmed?
☐ Rollback plan documented?
☐ Observability (logs/metrics/alerts) included?
☐ Documentation updated?

RESULT: Team calibration typically converges in 3-4 sprints.
```

---

### 14. Coaching: "A developer proposes a complex framework for a simple problem. How do you respond?"

**Answer:**
```
CURIOSITY OVER JUDGMENT (GROW):

GOAL: "What problem are you solving?"
REALITY: "What have you considered? What's the simplest thing that could work?"
OPTIONS: "What would v1 look like without the framework? What if we used library X?"
WILL: "Let's prototype the simple version today. If it hurts, we evolve."

PRINCIPLES:
- YAGNI: "We don't need plugin architecture for one integration"
- KISS: "Standard library HTTP client handles this in 20 lines"
- TEACHING MOMENT: "Here's how I'd evaluate: [criteria]"
- SAFE TO FAIL: "Try simple first. Complexity is reversible; simplicity discovered is gold."

OUTCOME: Developer learns evaluation criteria, not just "no."
```

---

### 15. Communication: "How do you document a decision that stakeholders disagreed with?"

**Answer:**
```
ADR WITH DISSENT DOCUMENTED:

# ADR-0087: Use Kubernetes for ML Model Serving

## Status: Accepted (2025-03-15)

## Context
Need to serve 50 ML models, 10K RPS, sub-100ms p99.

## Decision
Deploy models on KServe (Kubernetes) with GPU node pool.

## Consequences
### Positive
- Autoscaling, rolling updates, model versioning built-in
- Team has K8s expertise

### Negative
- Operational complexity (mitigated: platform team owns cluster)
- Cold start for GPU (mitigated: minReplicas=1 per model)

## Alternatives Considered
1. **SageMaker Endpoints** (AWS managed)
   - Pro: Fully managed, no ops
   - Con: Vendor lock-in, 30% higher cost, limited custom runtime
   - **Dissent: Data team preferred** — "Faster iteration, less infra"

2. **Triton + VMs** (Self-managed)
   - Pro: Full control, lower unit cost
   - Con: High ops burden, manual scaling

## Decision Rationale
Architecture Board voted 4-1 for KServe. 
Dissent (Data Team Lead): "SageMaker reduces time-to-model by 50%."
Board: "Long-term portability + team capability > short-term speed.
         Revisit in 12 months (ADR review)."

## Review Date: 2026-03-15
```

---

## SUMMARY: ARCHITECT'S TOOLKIT CHECKLIST

- [ ] **Documentation**: ADR template, C4 diagrams, runbook template, API spec standard
- [ ] **Git**: Trunk-based, conventional commits, hooks (lint/test/secret-scan), aliases
- [ ] **Patterns**: Resilience (CB, Retry, Bulkhead), CQRS/ES, DDD, Outbox, Saga
- [ ] **Principles**: SOLID, DRY/KISS/YAGNI, Conway, Demeter, Tell-Don't-Ask
- [ ] **Estimation**: Three-point, Monte Carlo, spikes, calibration loop, buffers
- [ ] **Tools**: Jira workflow, Confluence structure, dashboard standards
- [ ] **Balance**: WIP limits, no-meeting day, on-call rotation, learning time
- [ ] **Coaching**: GROW model, SBI feedback, pairing, architecture katas
- [ ] **Communication**: BLUF, audience-adapted, executive one-pager, visual-first
- [ ] **Marketing**: Newsletter, tech talks, office hours, RFC process, celebrations
- [ ] **Actors**: RACI map, stakeholder discovery, consultation before decision