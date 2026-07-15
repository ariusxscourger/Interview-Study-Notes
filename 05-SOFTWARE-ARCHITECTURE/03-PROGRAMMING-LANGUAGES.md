# PROGRAMMING LANGUAGES
## Exam-Style Study Notes

**Roadmap Section:** Programming Languages → Java/Kotlin/Scala, Python, Ruby, Go, JavaScript/TypeScript, .NET Framework Based, How to Code

---

## 1. LANGUAGE ECOSYSTEM OVERVIEW

### WHY? (Problem Statement)

**Language choice is an architectural decision** with long-term consequences:
- Team hiring & retention (skills market)
- Ecosystem maturity (libraries, tooling, community)
- Performance characteristics (latency, throughput, memory)
- Operational complexity (runtime, deployment, observability)
- Long-term maintenance (vendor support, version lifecycle)

> **Rule:** Default to organization's approved languages (Technology Radar). Deviate only with ADR documenting specific technical justification.

---

### HOW? (Language Selection Framework)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANGUAGE SELECTION DECISION TREE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  START: What is the PRIMARY workload?                                       │
│                                                                             │
│  ├─► HIGH-PERFORMANCE / LOW-LATENCY / EMBEDDED                             │
│  │     ├─► Rust / C++ / Go (systems programming)                           │
│  │     └─► Requires: Memory safety, no GC pauses, bare metal               │
│  │                                                                         │
│  ├─► DATA SCIENCE / ML / SCIENTIFIC COMPUTING                              │
│  │     ├─► Python (ecosystem: NumPy, Pandas, PyTorch, TensorFlow)         │
│  │     └─► Julia (emerging, high performance)                              │
│  │                                                                         │
│  ├─► ENTERPRISE BACKEND / MICROSERVICES                                     │
│  │     ├─► Java / Kotlin (Spring Boot, Quarkus) — mature, talent, JVM      │
│  │     ├─► C# / .NET 8+ (ASP.NET Core) — cloud-native, performance, AOT    │
│  │     ├─► Go — simple, fast startup, concurrency, cloud-native            │
│  │     └─► Node.js / TypeScript — full-stack, high I/O, team unification  │
│  │                                                                         │
│  ├─► FRONTEND / FULL-STACK                                                 │
│  │     ├─► TypeScript (React, Vue, Angular, Next.js) — type safety         │
│  │     └─► JavaScript (legacy, small teams)                                │
│  │                                                                         │
│  ├─► SCRIPTING / AUTOMATION / GLUE                                        │
│  │     ├─► Python (readability, libraries)                                 │
│  │     ├─► Bash / PowerShell (ops, CI/CD)                                  │
│  │     └─► TypeScript (Node.js scripts, type-safe)                         │
│  │                                                                         │
│  ├─► FUNCTIONAL / CONCURRENT / DISTRIBUTED                                 │
│  │     ├─► Scala (Akka, Spark, functional + OOP, JVM)                     │
│  │     ├─► F# (.NET functional, domain modeling)                          │
│  │     └─► Elixir (BEAM, fault-tolerance, Phoenix)                        │
│  │                                                                         │
│  └─► LEGACY / MAINFRAME / SPECIALIZED                                      │
│        ├─► COBOL (banking, mainframe)                                      │
│        ├─► ABAP (SAP)                                                      │
│        └─► PL/SQL, T-SQL (database logic)                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Language Profiles for Architecture Decisions)

#### JAVA / KOTLIN / SCALA (JVM ECOSYSTEM)

| Aspect | Java 21+ | Kotlin | Scala 3 |
|--------|----------|--------|---------|
| **Paradigm** | OOP, growing functional | Multi-paradigm (OOP + Functional) | Functional-first, OOP |
| **Best For** | Enterprise microservices, Spring ecosystem | Android, Spring Boot, coroutines | Data engineering (Spark), Akka, complex domain |
| **Startup** | Slow (improving with CRaC) | Similar to Java | Similar to Java |
| **Memory** | Higher (JVM overhead) | Similar to Java | Similar to Java |
| **Concurrency** | Virtual Threads (21+), CompletableFuture | Coroutines (structured concurrency) | Futures, Akka, ZIO, Cats Effect |
| **Type System** | Nominal, verbose | Null-safe, type inference, sealed classes | Advanced (HKT, GADT, type classes) |
| **Ecosystem** | Massive (Maven Central) | Growing, 100% Java interop | Strong in data/FP |
| **Talent Pool** | Largest | Growing rapidly | Niche (FP, data) |
| **Cloud Native** | Spring Boot 3, Quarkus, Micronaut | Spring Boot, Ktor, Micronaut | Akka, http4s, ZIO HTTP |
| **AOT/Native** | GraalVM Native Image | GraalVM Native Image | GraalVM Native Image |

**Architecture Guidance:**
- **Java:** Default for enterprise JVM. Spring Boot 3 + Virtual Threads = high throughput.
- **Kotlin:** Preferred for new JVM services. Null safety, coroutines, less boilerplate.
- **Scala:** Only when FP benefits outweigh complexity (Spark, Akka, complex domain logic).

---

#### PYTHON

| Aspect | Details |
|--------|---------|
| **Paradigm** | Multi-paradigm (OOP, functional, procedural) |
| **Best For** | Data science, ML/AI, scripting, automation, prototyping, backend (FastAPI/Django) |
| **Performance** | Slow (interpreted, GIL). Use: NumPy/Pandas (C), asyncio, multiprocessing, Cython, PyPy |
| **Async** | asyncio (single-threaded concurrency), uvloop for performance |
| **Type Safety** | Optional (mypy, pyright, pydantic). Gradual typing. |
| **Packaging** | pip/uv, poetry, conda. Virtual environments required. |
| **Deployment** | Containers (slim images), serverless (Lambda layers), ZipApp |
| **Observability** | OpenTelemetry, structlog, prometheus-client |
| **Talent** | Largest pool (data, ML, web, scripting) |

**Architecture Guidance:**
- ✅ ML pipelines, data processing, API backends (FastAPI), automation
- ❌ High-frequency trading, real-time gaming, memory-constrained embedded
- **Pattern:** Offload compute to Rust/Go/C++ extensions (pyo3, maturin)

---

#### RUBY

| Aspect | Details |
|--------|---------|
| **Paradigm** | Pure OOP, dynamic, metaprogramming |
| **Best For** | Rapid web development (Rails), scripting, DevOps tooling |
| **Performance** | Slow (MRI). YJIT (3.3+) improves significantly. |
| **Concurrency** | Threads (GIL), Fibers, Async gems (Sidekiq, Async) |
| **Ecosystem** | Rails (opinionated), Gems, strong conventions |
| **Talent** | Shrinking but experienced. High productivity per dev. |

**Architecture Guidance:**
- ✅ Internal tools, admin panels, rapid MVP, Shopify-scale (with investment)
- ❌ High-throughput microservices, ML, systems programming
- **Pattern:** Rails API mode + Sidekiq for async. Consider migration path to Go/Java for scale.

---

#### GO (GOLANG)

| Aspect | Details |
|--------|---------|
| **Paradigm** | Procedural, composition over inheritance, CSP concurrency |
| **Best For** | Cloud-native services, CLI tools, infrastructure, high-throughput APIs |
| **Performance** | Excellent (compiled, lightweight, fast GC). ~10x Python. |
| **Concurrency** | Goroutines (green threads) + Channels (CSP). First-class. |
| **Deployment** | Single static binary. Scratch/distroless containers. |
| **Type System** | Static, simple, no generics (until 1.18), no sum types. |
| **Error Handling** | Explicit (if err != nil). No exceptions. |
| **Ecosystem** | Standard library rich. Modules (go.mod). Growing. |
| **Talent** | Growing fast. Cloud-native default. |

**Architecture Guidance:**
- ✅ Microservices, API gateways, service mesh sidecars, CLI, operators
- ❌ Complex domain modeling (no generics pre-1.18, no algebraic types), GUI
- **Pattern:** Standard library first. Chi/Gin/Echo for HTTP. gRPC + Protobuf.

---

#### JAVASCRIPT / TYPESCRIPT

| Aspect | JavaScript | TypeScript |
|--------|------------|------------|
| **Paradigm** | Multi-paradigm, prototype-based | JS + Static types |
| **Best For** | Frontend (browser), Full-stack (Node.js), Serverless | All JS + Large apps, teams |
| **Runtime** | V8 (Node, Deno, Bun), SpiderMonkey, JavaScriptCore | Compiles to JS |
| **Async** | Promises, async/await, Event Loop (single-threaded) | Same |
| **Type Safety** | None (JIT optimizations) | Structural, gradual, powerful inference |
| **Ecosystem** | NPM (largest), fragmentation | DefinitelyTyped, first-class in frameworks |
| **Tooling** | ESLint, Prettier, Jest, Vitest | tsc, ESLint+TS, Vitest, tsx |
| **Deployment** | Node.js, Deno, Bun, Cloudflare Workers, Vercel, Lambda | Same |

**Architecture Guidance:**
- **TypeScript:** Default for all new Node.js/frontend. Strict mode. Enforce `strict: true`.
- **Node.js:** High I/O concurrency. Worker threads for CPU-bound.
- **Bun/Deno:** Modern alternatives (built-in TS, faster, secure defaults).
- **Pattern:** Shared types between frontend/backend (tRPC, GraphQL Codegen, OpenAPI).

---

#### .NET (C# / F#)

| Aspect | C# 12+ / .NET 8+ | F# 8+ / .NET 8+ |
|--------|------------------|-----------------|
| **Paradigm** | OOP, growing functional | Functional-first, OOP interop |
| **Best For** | Enterprise, cloud-native, Azure, Blazor, MAUI | Domain modeling, data pipelines, scripting |
| **Performance** | Excellent (Tiered PGO, AOT, Native AOT) | Same runtime |
| **Startup** | Fast (Native AOT: <1ms, ~5MB) | Similar |
| **Concurrency** | async/await, TPL, Channels, Virtual Threads (planned) | Async workflows, MailboxProcessor |
| **Type System** | Rich (records, patterns, nullable refs) | Algebraic data types, units of measure, type providers |
| **Ecosystem** | NuGet (massive), Microsoft-supported | NuGet, strong in data/finance |
| **Cloud** | Azure first-class, AWS Lambda, Container Apps | Same |
| **Talent** | Large enterprise pool | Niche (finance, science, domain-driven) |

**Architecture Guidance:**
- **C#:** Default for .NET. ASP.NET Core Minimal APIs + Native AOT = cloud-native.
- **F#:** Domain modeling (DDD), financial, data-intensive. Use with C# for APIs.
- **Pattern:** Clean Architecture, Vertical Slices, MediatR, Wolverine, MassTransit.

---

## 2. HOW TO CODE (CROSS-CUTTING CONCERNS)

### WHY? (Problem Statement)

**Language syntax is 10% of the job.** The other 90% is:
- Writing code that is testable, maintainable, observable
- Managing dependencies, versions, supply chain
- Ensuring security, performance, reliability
- Enabling team collaboration (reviews, standards, knowledge sharing)

---

### HOW? (Universal Coding Practices)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    UNIVERSAL CODING PRINCIPLES                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. SOLID PRINCIPLES (apply in every language)                             │
│     ├─ S: Single Responsibility — One reason to change                     │
│     ├─ O: Open/Closed — Extend without modifying                           │
│     ├─ L: Liskov Substitution — Subtypes behave like base                  │
│     ├─ I: Interface Segregation — Small, focused interfaces                │
│     └─ D: Dependency Inversion — Depend on abstractions, not concretions   │
│                                                                             │
│  2. CLEAN CODE PRACTICES                                                    │
│     ├─ Meaningful names (Intent-revealing, pronounceable, searchable)      │
│     ├─ Small functions (<20 lines, one level of abstraction)               │
│     ├─ No side effects in queries (CQS)                                     │
│     ├─ Explicit error handling (Result types, not exceptions for flow)     │
│     ├─ Immutable by default (records, data classes, readonly)              │
│     └─ Composition over inheritance                                        │
│                                                                             │
│  3. TESTABILITY BY DESIGN                                                  │
│     ├─ Pure functions (same input → same output, no side effects)          │
│     ├─ Dependency Injection (constructor injection)                        │
│     ├─ Test doubles: Fakes > Mocks > Stubs                                 │
│     ├─ Test pyramid: Unit (70%) → Integration (20%) → E2E (10%)            │
│     └─ Property-based testing (Hypothesis, FsCheck, fast-check)            │
│                                                                             │
│  4. OBSERVABILITY INSTRUMENTATION                                          │
│     ├─ Structured logging (JSON, correlation IDs)                          │
│     ├─ Metrics (RED: Rate, Errors, Duration)                               │
│     ├─ Distributed tracing (W3C TraceContext)                              │
│     └─ Health checks (liveness, readiness, startup)                        │
│                                                                             │
│  5. SECURITY BY DEFAULT                                                    │
│     ├─ Input validation (allowlist, not denylist)                          │
│     ├─ Parameterized queries (never string concat SQL)                     │
│     ├─ Secrets in vault (never config/files)                               │
│     ├─ Dependency scanning (SCA) in CI                                     │
│     └─ Least privilege (RBAC, service accounts)                            │
│                                                                             │
│  6. PERFORMANCE AWARENESS                                                  │
│     ├─ Profile before optimizing                                           │
│     ├─ Understand allocations (GC pressure)                                │
│     ├─ Connection pooling (DB, HTTP, gRPC)                                 │
│     ├─ Caching strategy (cache-aside, TTL, invalidation)                   │
│     └─ Pagination (cursor, not offset)                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Language-Specific Best Practices)

#### JAVA / KOTLIN
```kotlin
// Kotlin: Records (data classes), Sealed classes, Result type
sealed interface Result<out T> {
    data class Success<out T>(val value: T) : Result<T>
    data class Failure(val error: Error) : Result<Nothing>
}

// Virtual threads (Java 21+) for high-throughput blocking I/O
Executors.newVirtualThreadPerTaskExecutor().submit { blockingCall() }

// Structured concurrency (JEP 453+)
StructuredTaskScope.open().use { scope ->
    val user = scope.fork { userService.get(id) }
    val orders = scope.fork { orderService.getForUser(id) }
    scope.join().throwIfFailed()
    UserWithOrders(user.get(), orders.get())
}
```

#### GO
```go
// Explicit error handling, no exceptions
func GetOrder(ctx context.Context, id string) (*Order, error) {
    var order Order
    err := db.QueryRowContext(ctx, "SELECT ...", id).Scan(&order.ID, ...)
    if err != nil {
        return nil, fmt.Errorf("get order %s: %w", id, err) // wrap context
    }
    return &order, nil
}

// Concurrency with errgroup
func FetchUserWithOrders(ctx context.Context, userID string) (*User, []Order, error) {
    g, ctx := errgroup.WithContext(ctx)
    var user *User
    var orders []Order
    
    g.Go(func() error {
        var err error
        user, err = userSvc.Get(ctx, userID)
        return err
    })
    g.Go(func() error {
        var err error
        orders, err = orderSvc.ListByUser(ctx, userID)
        return err
    })
    
    if err := g.Wait(); err != nil {
        return nil, nil, err
    }
    return user, orders, nil
}
```

#### TYPESCRIPT
```typescript
// Discriminated unions for state modeling
type OrderStatus =
  | { status: 'Draft'; items: OrderItem[] }
  | { status: 'Submitted'; submittedAt: Date; paymentId: string }
  | { status: 'Shipped'; trackingNumber: string; shippedAt: Date }
  | { status: 'Cancelled'; reason: string; cancelledAt: Date };

// Result type for explicit error handling
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

async function submitOrder(cmd: SubmitOrderCommand): Promise<Result<OrderId>> {
  const validation = validate(cmd);
  if (!validation.ok) return { ok: false, error: validation.error };
  
  try {
    const order = await orderRepo.save(Order.create(cmd));
    await eventBus.publish(order.events);
    return { ok: true, value: order.id };
  } catch (e) {
    return { ok: false, error: new OrderSubmitError(e) };
  }
}

// Zod for runtime validation + types
const SubmitOrderSchema = z.object({
  customerId: z.string().uuid(),
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive(),
  })).min(1),
  shippingAddress: AddressSchema,
});
type SubmitOrderCommand = z.infer<typeof SubmitOrderSchema>;
```

#### C# / .NET
```csharp
// Records for immutable data
public record OrderId(Guid Value) : IComparable<OrderId>;
public record Money(decimal Amount, string Currency);

// Result pattern (OneOf or custom)
public abstract record Result<T> {
    public sealed record Success<T>(T Value) : Result<T>;
    public sealed record Failure<T>(Error Error) : Result<T>;
}

// Domain-driven, testable
public class Order : AggregateRoot<OrderId> {
    private readonly List<OrderItem> _items = new();
    
    public static Result<Order> Create(CreateOrderCommand cmd) {
        if (cmd.Items.Count == 0) 
            return Result.Failure<Order>(Errors.Order.Empty);
        
        var order = new Order(OrderId.New(), cmd.CustomerId);
        foreach (var item in cmd.Items) 
            order.AddItem(item.ProductId, item.Quantity, item.Price);
        
        order.Raise(new OrderCreatedEvent(order.Id, order.CustomerId, order.Items));
        return Result.Success(order);
    }
    
    public Result Submit() {
        if (Status != OrderStatus.Draft) 
            return Result.Failure(Errors.Order.AlreadySubmitted);
        Status = OrderStatus.Submitted;
        Raise(new OrderSubmittedEvent(Id, CustomerId, Items));
        return Result.Success();
    }
}

// Minimal API with Results
app.MapPost("/orders", async (CreateOrderCommand cmd, IMediator mediator) => {
    var result = await mediator.Send(cmd);
    return result.Match(
        id => Results.Created($"/orders/{id}", new { id }),
        error => Results.BadRequest(new { error.Message })
    );
});
```

#### PYTHON
```python
# Type hints + Pydantic for validation
from pydantic import BaseModel, Field
from typing import Optional
from dataclasses import dataclass
from enum import Enum

class OrderStatus(str, Enum):
    DRAFT = "Draft"
    SUBMITTED = "Submitted"
    SHIPPED = "Shipped"
    CANCELLED = "Cancelled"

class OrderItem(BaseModel):
    product_id: str = Field(..., min_length=1)
    quantity: int = Field(..., gt=0)
    unit_price: Decimal = Field(..., gt=0)

class CreateOrderCommand(BaseModel):
    customer_id: str
    items: list[OrderItem] = Field(..., min_length=1)
    shipping_address: Address

# Result pattern
@dataclass(frozen=True)
class Result(Generic[T]):
    value: Optional[T] = None
    error: Optional[Exception] = None
    
    @property
    def ok(self) -> bool:
        return self.error is None
    
    @staticmethod
    def success(value: T) -> 'Result[T]':
        return Result(value=value)
    
    @staticmethod
    def failure(error: Exception) -> 'Result[T]':
        return Result(error=error)

# FastAPI with dependency injection
@router.post("/orders", response_model=OrderResponse)
async def create_order(
    cmd: CreateOrderCommand,
    mediator: Mediator = Depends(get_mediator)
) -> OrderResponse:
    result = await mediator.send(cmd)
    if not result.ok:
        raise HTTPException(400, detail=str(result.error))
    return OrderResponse(id=result.value)
```

---

## 3. POLYGLOT ARCHITECTURE

### WHEN TO USE MULTIPLE LANGUAGES

| Scenario | Languages | Rationale |
|----------|-----------|-----------|
| **ML Pipeline + API** | Python (ML) + Go/Java (API) | Python for ecosystem, Go/Java for serving |
| **Data Processing + Web** | Scala/Spark + TypeScript/React | Spark on JVM, React on Node |
| **Infrastructure + App** | Go (CLI, operators) + C# (API) | Go for tooling, C# for business logic |
| **Real-time + Batch** | Elixir (Phoenix Channels) + Python (ETL) | BEAM for concurrency, Python for data |

### POLYGLOT GUARDRAILS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    POLYGLOT ARCHITECTURE RULES                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. MAX 3 LANGUAGES IN PRODUCTION                                          │
│     └─ Core (1), Specialized (1-2), Scripting (1)                          │
│                                                                             │
│  2. SHARED CONTRACTS, NOT SHARED CODE                                      │
│     ├─ Protobuf / Avro schemas in Schema Registry                          │
│     ├─ OpenAPI specs for REST                                              │
│     ├─ AsyncAPI for event-driven                                           │
│     └─ NO shared libraries across languages                                │
│                                                                             │
│  3. CONSISTENT OPERATIONAL BASELINE                                        │
│     ├─ Same logging format (JSON + correlation ID)                         │
│     ├─ Same metrics format (Prometheus)                                    │
│     ├─ Same tracing (W3C TraceContext)                                     │
│     ├─ Same health check contract (/health/live, /health/ready)            │
│     └─ Same deployment pipeline (container → registry → k8s)               │
│                                                                             │
│  4. LANGUAGE-SPECIFIC EXPERTISE REQUIRED                                   │
│     └─ Don't adopt a language without team capability                      │
│                                                                             │
│  5. MIGRATION PATH FOR LEGACY                                              │
│     └─ Strangler Fig: new language for new services, legacy frozen         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Context | Anti-Pattern | Why Avoid |
|---------|---------|--------------|-----------|
| **Technology Radar** | Org-wide language strategy | Ad-hoc language adoption | Sprawl, skill dilution, support burden |
| **Language per Bounded Context** | Microservices | One language for everything | Wrong tool for job, forced patterns |
| **Shared Contracts (Protobuf/Avro)** | Polyglot integration | Shared client libraries | Coupling, version lockstep |
| **Gradual Typing (TS, Python, Kotlin)** | Evolving codebase | No types / Full types from day 1 | Velocity vs safety balance |
| **Native AOT / GraalVM** | Serverless, CLI, constrained | JIT-only in cold-start paths | Cold start latency, memory |
| **Structured Concurrency** | High-concurrency services | Thread-per-request / callback hell | Resource leaks, cancellation |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. "How do you choose a programming language for a new service?"

**Answer:**
```
1. CHECK TECHNOLOGY RADAR — Is it Adopt/Trial? If Hold → need exception ADR
2. MAP WORKLOAD TO LANGUAGE STRENGTHS:
   - CPU-bound, low-latency → Rust, Go, C++
   - I/O-bound, high concurrency → Go, Node.js, Java (virtual threads), .NET
   - Data/ML → Python, Scala
   - Enterprise domain → Java/Kotlin, C#, F#
   - Full-stack team → TypeScript
3. EVALUATE OPERATIONAL FACTORS:
   - Team expertise (learning curve vs productivity)
   - Hiring market (can we staff it?)
   - Ecosystem maturity (libraries, tooling, security)
   - Cloud provider support (Lambda, Cloud Run, AKS/EKS/GKE)
   - Observability story (OpenTelemetry, profiling)
4. PROTOTYPE RISKIEST AREA (spike, 1-2 days)
5. DOCUMENT IN ADR with trade-offs
```

---

### 2. "Java vs Go vs .NET for microservices — compare."

**Answer:**
| Dimension | Java (Spring Boot) | Go | .NET 8+ |
|-----------|-------------------|-----|---------|
| **Startup** | Slow (~2-5s), improving with CRaC | Instant (~ms) | Fast (~100ms), Native AOT ~1ms |
| **Memory** | Higher (JVM heap + metaspace) | Low (5-10MB base) | Medium, Native AOT very low |
| **Throughput** | Excellent (virtual threads) | Excellent (goroutines) | Excellent (async, channels) |
| **Ecosystem** | Massive (Spring, everything) | Growing, cloud-native | Strong (Microsoft + community) |
| **Talent** | Largest pool | Growing fast | Large enterprise pool |
| **Deployment** | JAR + JRE / Native Image | Single binary | Single file / Native AOT |
| **Observability** | Micrometer, OpenTelemetry | Native expvar, OTEL | Built-in, OpenTelemetry |
| **Best For** | Complex domain, enterprise | Cloud-native, infra, high-scale | Azure, enterprise, full-stack |

**Decision:** Team expertise > benchmarks. All three handle 10K+ RPS easily.

---

### 3. "Why TypeScript over JavaScript for backend?"

**Answer:**
```
TYPESCRIPT ADVANTAGES:
✅ Compile-time error detection (refactoring confidence)
✅ Self-documenting code (types = living documentation)
✅ IDE intelligence (autocomplete, go-to-definition, rename)
✅ Shared types with frontend (tRPC, GraphQL Codegen, OpenAPI)
✅ Catches null/undefined (strictNullChecks)
✅ Discriminated unions for state machines
✅ Gradual adoption (allowJs, checkJs)

COSTS:
⚠️ Build step (tsc, esbuild, swc) — mitigated by tsx/bun in dev
⚠️ Learning curve for JS developers — ~2 weeks to productivity
⚠️ Over-typing (fighting the compiler) — use type inference, avoid any

VERDICT: Default for ALL new Node.js services. Strict mode. No 'any' without eslint-disable comment + TODO.
```

---

### 4. "How do you handle polyglot microservices communication?"

**Answer:**
```
CONTRACT-FIRST, NOT CODE-FIRST:

1. SCHEMA REGISTRY (Confluent / Apicurio)
   - Protobuf for gRPC (required)
   - Avro for Kafka events
   - Schema evolution rules: BACKWARD compatibility default

2. API SPECIFICATIONS
   - OpenAPI 3.1 for REST (generates clients, validators)
   - AsyncAPI for event-driven
   - GraphQL Schema (if using GraphQL)

3. CLIENT GENERATION (CI pipeline)
   - buf generate (Protobuf → Go, Java, TS, C#, Python, Rust)
   - openapi-generator / orval (OpenAPI → TS, Java, Kotlin, Go, C#)
   - NO hand-written clients

4. COMPATIBILITY TESTING
   - Consumer-driven contracts (Pact) for REST
   - Schema Registry compatibility checks in CI (breaking change = fail)
   - Contract tests in each service's pipeline

5. OPERATIONAL PARITY
   - All services: /health/live, /health/ready, /metrics (Prometheus)
   - All logs: JSON, correlation-id, trace-id, span-id
   - All tracing: W3C TraceContext headers
   - All deploy: Same pipeline, same base image strategy
```

---

### 5. "What's your approach to dependency management and supply chain security?"

**Answer:**
```
MULTI-LAYER DEFENSE:

1. LOCKFILES ALWAYS COMMITTED
   - package-lock.json / pnpm-lock.yaml / go.mod+go.sum / Cargo.lock
   - pom.xml + dependency:lock (Maven) / packages.lock.json (.NET)

2. AUTOMATED UPDATES WITH TESTS
   - Dependabot / Renovate → PRs with changelog + test results
   - Group: security (immediate), patch (weekly), minor (monthly), major (manual)

3. VULNERABILITY SCANNING (CI + Registry)
   - CI: Trivy / Grype / Snyk / OWASP Dependency Check on every PR
   - Registry: Harbor / JFrog Xray / GitHub Container Registry scanning
   - Fail build on CRITICAL/HIGH (configurable grace period)

4. ALLOWLIST / DENYLIST POLICIES
   - License compliance (FOSSA, ClearlyDefined)
   - Prohibited packages (known malicious, abandoned)
   - Approved registries only (private proxy: Artifactory, Nexus, GitHub Packages)

5. SBOM GENERATION
   - CycloneDX / SPDX on every build (Syft, Trivy, Microsoft SBOM Tool)
   - Stored in artifact registry for audit

6. REPRODUCIBLE BUILDS
   - Fixed base images (digest, not tag)
   - BuildKit / Nix / Bazel for hermetic builds
   - SLSA Level 3 target (provenance, tamper-proof)
```

---

### 6. "How do you ensure code quality across multiple languages?"

**Answer:**
```
UNIFIED QUALITY GATES (per language, same pipeline stage):

LINTING (fail fast):
├─ Java/Kotlin: SpotBugs, Checkstyle, ktlint, Detekt
├─ Go: golangci-lint (all linters)
├─ .NET: dotnet format --verify-no-changes, Roslynator, SonarAnalyzer
├─ TypeScript: ESLint (typescript-eslint, unused-imports), Prettier
├─ Python: Ruff (replaces flake8, black, isort), mypy (strict)
└─ Ruby: RuboCop, StandardRB

STATIC ANALYSIS (CI):
├─ SonarCloud / SonarQube (all languages) — Quality Gate
├─ CodeQL (GitHub Advanced Security) — Security queries
├─ Semgrep — Custom rules, secret detection

TESTING (minimum thresholds):
├─ Unit: ≥ 80% line, ≥ 70% branch (per service)
├─ Integration: Critical paths (API contracts, DB migrations)
├─ Contract: Consumer-driven (Pact) for REST, Schema Registry for events
├─ Mutation: Stryker (TS), Pitest (Java), GoMutesting (Go) — ≥ 60% score

SECURITY (every PR):
├─ SAST: CodeQL, Semgrep
├─ SCA: Trivy, OWASP Dependency Check
├─ Secrets: TruffleHog, GitLeaks, ggshield
└─ Container: Trivy, Grype (base image + final image)

QUALITY GATE (merge blocked if):
- Any CRITICAL/HIGH vulnerability
- Sonar Quality Gate failed
- Test coverage below threshold
- Linting errors
- Secret detected
```

---

### 7. "Explain structured concurrency and why it matters."

**Answer:**
```
PROBLEM: Traditional concurrency leaks resources, loses errors, no cancellation.

UNSTRUCTURED (BAD):
```go
go func() { doWork() }()  // Fire and forget — no handle, no error, no cancel
```
```java
executor.submit(() -> doWork()); // Future ignored — leak, no cancellation
```

STRUCTURED CONCURRENCY (GOOD):
- **Parent waits for all children** (scope)
- **Errors propagate** (first failure cancels siblings)
- **Cancellation flows down** (context/token)
- **No orphaned tasks**

LANGUAGE EXAMPLES:
```go
// Go: errgroup
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error { return svcA.Call(ctx) })
g.Go(func() error { return svcB.Call(ctx) })
if err := g.Wait(); err != nil { return err } // First error, others cancelled
```

```java
// Java 21+: StructuredTaskScope
try (var scope = StructuredTaskScope.open()) {
    var a = scope.fork(() -> svcA.call());
    var b = scope.fork(() -> svcB.call());
    scope.join().throwIfFailed();
    return new Result(a.get(), b.get());
}
```

```typescript
// TypeScript: Promise.allSettled + AbortController
const controller = new AbortController();
const results = await Promise.allSettled([
    svcA.call({ signal: controller.signal }),
    svcB.call({ signal: controller.signal }),
]);
const firstError = results.find(r => r.status === 'rejected')?.reason;
if (firstError) { controller.abort(firstError); throw firstError; }
```

```csharp
// C#: TaskGroup (preview) or custom
await Task.WhenAll(
    svcA.CallAsync(ct),
    svcB.CallAsync(ct)
).ConfigureAwait(false);
// With custom TaskGroup: automatic cancellation on first exception
```

BENEFITS:
✅ No resource leaks (goroutines, threads, connections)
✅ Predictable error handling (first failure wins)
✅ Cancellation propagates automatically
✅ Observable (scope visible in debugger/profiler)
✅ Testable (deterministic completion)
```

---

### 8. "How do you handle error handling across languages consistently?"

**Answer:**
```
PRINCIPLE: ERRORS ARE VALUES, NOT EXCEPTIONS (for expected failures)

CROSS-LANGUAGE ERROR MODEL:
{
  "code": "ORDER_NOT_FOUND",        // Machine-readable, stable
  "message": "Order 123 not found", // Human-readable
  "details": { "orderId": "123" },  // Structured context
  "retryable": false,               // Client behavior hint
  "traceId": "abc-123"              // Observability
}

LANGUAGE IMPLEMENTATIONS:
┌─────────────┬──────────────────────────────────────────────────────────┐
│ Language    │ Pattern                                                  │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Go          │ error wrapping (fmt.Errorf %w), sentinel errors,         │
│             │ custom error types with Error() + Unwrap()               │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Java/Kotlin │ Result<T> sealed interface (Success/Failure),            │
│             │ avoid checked exceptions for domain errors               │
├─────────────┼──────────────────────────────────────────────────────────┤
│ C#          │ Result<T> (OneOf or custom),                             │
│             │ ProblemDetails (RFC 7807) for HTTP APIs                  │
├─────────────┼──────────────────────────────────────────────────────────┤
│ TypeScript  │ Result<T, E> discriminated union,                        │
│             │ Zod for validation → typed errors                        │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Python      │ Result[T] dataclass,                                     │
│             │ Pydantic for validation, custom exceptions for bugs      │
└─────────────┴──────────────────────────────────────────────────────────┘

HTTP MAPPING (consistent):
- 400: Validation error (code: VALIDATION_ERROR)
- 401: Unauthenticated (code: UNAUTHENTICATED)
- 403: Forbidden (code: PERMISSION_DENIED)
- 404: Not found (code: NOT_FOUND)
- 409: Conflict (code: ALREADY_EXISTS, CONCURRENT_MODIFICATION)
- 422: Business rule violation (code: BUSINESS_RULE_VIOLATION)
- 429: Rate limited (code: RATE_LIMITED, retry-after header)
- 500: Internal (code: INTERNAL, traceId for support)
- 503: Unavailable (code: UNAVAILABLE, retry-after)

RULE: Expected failures (validation, not found, business rules) = Result types / 4xx
      Unexpected failures (bugs, infrastructure) = Exceptions / 5xx
      NEVER use exceptions for control flow.
```

---

### 9. "When would you use Native AOT / GraalVM Native Image?"

**Answer:**
```
USE NATIVE AOT WHEN:
✅ Serverless / FaaS (AWS Lambda, Azure Functions, Cloud Run)
   - Cold start < 10ms vs 100-500ms (JIT)
   - Memory < 50MB vs 200-500MB
   - Pay-per-invocation cost reduction

✅ CLI Tools (distributed to users)
   - Single binary, no runtime install
   - Instant startup

✅ Kubernetes Sidecars / Operators
   - Many replicas, memory-constrained
   - Fast scaling (pod ready in ms)

✅ Edge / IoT Gateways
   - ARM, limited RAM, no JIT warmup

✅ Short-lived Batch Jobs
   - Startup dominates runtime

AVOID NATIVE AOT WHEN:
❌ Long-running services (JIT wins after warmup)
❌ Heavy reflection / dynamic code (Spring, Hibernate, serialization)
   - Requires configuration, increases build time
❌ Large apps (>50MB native) — build time, debuggability
❌ Team lacks Native Image expertise (debugging is harder)
❌ Libraries not compatible (JNI, Unsafe, finalizers)

STRATEGY:
1. Default: JIT (standard JVM / .NET runtime)
2. Profile: Identify cold-start sensitive paths
3. Migrate incrementally: CLI → Lambda → Sidecars → Core services
4. Test: Compare p99 latency, memory, throughput at steady state
```

---

### 10. "How do you manage language versions and upgrades?"

**Answer:**
```
VERSION STRATEGY (per language on Technology Radar):

LTS FIRST:
├─ Java: LTS only (11, 17, 21, 25...) — upgrade within 6 months of release
├─ .NET: LTS only (6, 8, 10...) — upgrade within 3 months
├─ Node.js: Active LTS (even versions: 18, 20, 22...)
├─ Go: Latest 2 versions (1.22, 1.23) — upgrade within 1 month
├─ Python: Latest 3.x (3.11, 3.12, 3.13...) — upgrade within 3 months
└─ TypeScript: Latest stable — upgrade with major framework updates

UPGRADE PROCESS:
1. TRACK: Dependabot/Renovate PRs for runtime version bumps
2. TEST: CI matrix runs against current + next version
3. SPIKE: Upgrade one low-risk service first
4. ROLLOUT: Canary → Staged → All (feature flag runtime if needed)
5. DEPRECATE: Remove old version from base images after migration

BASE IMAGES:
- Pin to digest (sha256:...), not tag
- Update base image weekly (automated PR)
- Distroless / Chainguard / Wolfi for minimal attack surface

EXAMPLE (Dockerfile):
FROM eclipse-temurin:21-jre-alpine@sha256:abc123... AS runtime
# NOT: FROM eclipse-temurin:21-jre-alpine
```

---

### 11. "What's your stance on monorepo vs polyrepo for polyglot code?"

**Answer:**
```
MONOREPO (Recommended for most orgs < 500 devs):
✅ Atomic commits across languages (API + client + infra)
✅ Shared tooling (lint, test, build, release)
✅ Easy code sharing (types, schemas, utils)
✅ Single CI pipeline, unified versioning
✅ Cross-language refactoring (IDE support)
✅ Simplified dependency management

TOOLS: Nx, Turborepo, Bazel, Earthly, Dagger

POLYREPO (When):
✅ Distinct teams, independent release cycles
✅ Open source / external contributors
✅ Regulatory isolation (PCI, GDPR separate repos)
✅ Massive scale (Google, Meta — custom tooling)

HYBRID (Common):
- Core platform: Monorepo (infra, shared libs, platform services)
- Product domains: Separate repos per domain (bounded contexts)
- Shared contracts: Published packages (npm, NuGet, Maven, Go modules)

DECISION FACTORS:
- Team topology (Conway's Law)
- Release cadence alignment
- Tooling investment capacity
- Security/compliance boundaries
```

---

### 12. "How do you do code reviews across unfamiliar languages?"

**Answer:**
```
REVIEWER GUIDELINES (Language-Agnostic):

1. FOCUS ON UNIVERSALS (80% of value):
   - Architecture fit (layering, boundaries, dependencies)
   - Error handling (Result types, not exceptions for flow)
   - Observability (logs, metrics, traces, correlation IDs)
   - Security (validation, secrets, SQL injection, XSS)
   - Testing (unit coverage, integration, contract tests)
   - Performance (N+1, pooling, caching, pagination)
   - Naming & clarity (intent-revealing, no abbreviations)

2. USE TOOLING FOR LANGUAGE SPECIFICS:
   - Linters catch style, bugs, anti-patterns
   - Static analysis catches security, complexity
   - Tests verify behavior

3. PAIR / MOB PROGRAMMING for unfamiliar areas:
   - "I don't know Go well — can we pair on this PR?"
   - Knowledge transfer > gatekeeping

4. CHECKLIST PER LANGUAGE (append to PR template):
   - Go: error handling, context propagation, defer cleanup
   - Java: null safety, resource try-with-resources, virtual threads
   - TypeScript: strict mode, no any, Zod validation, no console.log
   - C#: nullable refs, async/await, Result pattern, DI
   - Python: type hints, mypy clean, pydantic validation, async context

5. ARCHITECTURE REVIEW (separate from code review):
   - New service? New language? → Architecture Decision Record
   - Cross-cutting concerns (auth, observability, config) → Platform team review
```

---

### 13. "How do you handle shared types between frontend (TS) and backend (Go/Java/C#)?"

**Answer:**
```
SINGLE SOURCE OF TRUTH: PROTOBUF / OPENAPI SCHEMA

WORKFLOW:
1. DEFINE CONTRACT (protobuf for gRPC, OpenAPI for REST)
   ```protobuf
   // order.proto
   message Order {
     string id = 1;
     string customer_id = 2;
     repeated OrderItem items = 3;
     OrderStatus status = 4;
     google.protobuf.Timestamp created_at = 5;
   }
   ```

2. GENERATE CLIENTS (CI Pipeline):
   ```yaml
   # .github/workflows/generate-clients.yml
   - uses: bufbuild/buf-generate-action@v1
     with:
       input: proto
       outputs: |
         plugin: go
         out: gen/go
         plugin: typescript
         out: gen/ts
         plugin: csharp
         out: gen/csharp
   ```

3. PUBLISH PACKAGES:
   - Go: go.mod replace / private module proxy
   - TypeScript: npm package (@org/api-client)
   - C#: NuGet package (internal feed)
   - Java: Maven artifact (internal Nexus)

4. VERSIONING:
   - Schema version in package version (v1.2.0 → proto v1)
   - Breaking change = major version bump
   - Deprecation policy: 6 months overlap

5. CONSUMER USAGE:
   ```typescript
   // Frontend (TypeScript)
   import { Order, OrderServiceClient } from '@org/api-client';
   const client = new OrderServiceClient('https://api.example.com');
   const order = await client.getOrder({ id: '123' });
   // Fully typed!
   ```

ANTI-PATTERN: Hand-written clients, copy-pasted types, drift.
```

---

### 14. "How do you evaluate a new language for adoption?"

**Answer:**
```
EVALUATION FRAMEWORK (4-Week Spike):

WEEK 1: TECHNICAL FEASIBILITY
├─ Build hello-world service (API + DB + Observability)
├─ Implement critical path (auth, config, health, tracing)
├─ Load test: 10K RPS, measure p99, memory, CPU
├─ Deploy to staging (container, k8s, serverless)
└─ Debug production-like issue (logging, profiling, core dump)

WEEK 2: ECOSYSTEM & OPERATIONS
├─ Library availability (DB driver, HTTP client, messaging, config)
├─ Testing story (unit, integration, contract, property-based)
├─ CI/CD integration (lint, test, build, scan, deploy)
├─ Dependency management (lockfiles, updates, vulnerabilities)
├─ Observability maturity (OpenTelemetry, profiling, debugging)
└─ Team ramp-up (pair programming, code kata, mob review)

WEEK 3: PRODUCTION READINESS
├─ Run canary (5% traffic) for 1 week
├─ Incident simulation (kill pod, latency injection, error injection)
├─ On-call experience (alerts, runbooks, debugging)
├─ Cost analysis (CPU, memory, network vs current)
└─ Rollback procedure tested

WEEK 4: DECISION
├─ Scorecard (Technical, Operational, Team, Strategic)
├─ ADR with: Recommendation, Risks, Mitigations, Rollback Plan
├─ Architecture Board Review
└─ If APPROVED → Add to Radar (Trial), create paved road template

SCORECARD WEIGHTS:
- Technical Fit: 30%
- Operational Maturity: 25%
- Team Capability: 20%
- Strategic Alignment: 15%
- Cost: 10%

THRESHOLD: ≥ 75% weighted score + no veto from Platform/Security
```

---

### 15. "How do you handle language-specific build tools in a unified CI?"

**Answer:**
```
UNIFIED CI PIPELINE (Dagger / Earthly / GitHub Actions composite):

PRINCIPLE: Each language uses ITS NATIVE TOOLCHAIN, orchestrated by CI.

BUILD STAGE (parallel per language):
```yaml
jobs:
  build-go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-go@v5
        with: { go-version-file: 'go.mod' }
      - run: go build -ldflags="-s -w" ./...
      - run: go test -race -coverprofile=coverage.out ./...
      - uses: actions/upload-artifact@v4
        with: { name: go-binary, path: ./bin/ }

  build-dotnet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet restore
      - run: dotnet build -c Release --no-restore
      - run: dotnet test --no-build -c Release --collect:"XPlat Code Coverage"
      - uses: actions/upload-artifact@v4
        with: { name: dotnet-artifacts, path: ./artifacts/ }

  build-typescript:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint && pnpm typecheck
      - run: pnpm test --coverage
      - run: pnpm build
      - uses: actions/upload-artifact@v4
        with: { name: ts-dist, path: ./dist/ }
```

CONTAINER STAGE (unified):
```dockerfile
# Multi-stage, language-appropriate base
FROM golang:1.23-alpine AS go-builder
COPY --from=build-go /workspace/bin/app /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS dotnet-builder
COPY --from=build-dotnet /workspace/artifacts/publish /app

FROM node:20-alpine AS ts-builder
COPY --from=build-typescript /workspace/dist /app

# Final: distroless per language or unified scratch
FROM gcr.io/distroless/static-debian12
COPY --from=go-builder /app /app-go
COPY --from=dotnet-builder /app /app-dotnet
# ... each service gets its own final image
```

QUALITY GATES (all languages, same stage):
- SonarCloud analysis (multi-language)
- Trivy scan (filesystem + language-specific)
- License check (FOSSA / ClearlyDefined)
- SBOM generation (Syft → CycloneDX)

KEY: CI orchestrates; each language owns its build logic.
```

---

## LANGUAGE QUICK REFERENCE CARD

| Need | First Choice | Alternative | Avoid |
|------|--------------|-------------|-------|
| High-throughput API | Go, Java 21+, .NET 8+ | Rust, Node.js | Python, Ruby |
| ML/Data Pipeline | Python, Scala | Julia, F# | Go, JS |
| CLI Tool | Go, Rust | Python (with uv), .NET AOT | Node.js (cold start) |
| Full-Stack Web | TypeScript (Next.js/Remix) | C# (Blazor), Java (Vaadin) | Raw JS, PHP |
| Event-Driven System | Java/Kotlin (Kafka Streams), Go | .NET (Wolverine), Rust | Python (GIL) |
| Domain Modeling | F#, Scala, Kotlin | TypeScript, C# | Go, Java (pre-records) |
| Serverless | Go, Rust, .NET AOT, Node.js | Python, Java (SnapStart) | Ruby, PHP |
| Legacy Migration | Same language (strangler) | .NET/Java (enterprise) | Rewrite in new language |

---

## SUMMARY: ARCHITECT'S LANGUAGE CHECKLIST

- [ ] Technology Radar current? (Adopt/Trial/Assess/Hold)
- [ ] ADR for every non-standard language choice?
- [ ] Max 3 languages in production?
- [ ] Shared contracts (Protobuf/OpenAPI) not shared code?
- [ ] Operational parity (logging, metrics, tracing, health)?
- [ ] Lockfiles committed, Dependabot/Renovate configured?
- [ ] Vulnerability scanning in CI + Registry?
- [ ] SBOM on every build?
- [ ] LTS versions pinned, upgrade cadence defined?
- [ ] Base images pinned to digest, updated weekly?
- [ ] Team has expertise or ramp-up plan for each language?
- [ ] Code quality gates unified across languages?
- [ ] Cross-language debugging story (correlation IDs)?