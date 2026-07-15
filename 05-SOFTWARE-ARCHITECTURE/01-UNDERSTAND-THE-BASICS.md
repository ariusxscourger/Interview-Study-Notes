# UNDERSTAND THE BASICS
## Exam-Style Study Notes

**Roadmap Section:** Understand the Basics → Application Architecture, Solution Architecture, Enterprise Architecture

---

## 1. APPLICATION ARCHITECTURE

### WHY? (Problem Statement)

**Application Architecture** defines the internal structure of a single deployable unit (service, monolith, mobile app). It answers: *How is this application organized internally?*

**Without it:**
- Spaghetti code — everything references everything
- Untestable — business logic tangled with infrastructure
- Rigid — changing one feature breaks three others
- Onboarding nightmare — new developers can't find where things live

---

### HOW? (Architectural Styles for Applications)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION ARCHITECTURE STYLES                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. LAYERED (N-TIER)                                                        │
│  ─────────────────                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Presentation  │  Business Logic  │  Data Access  │  Database      │   │
│  │  (Controllers) │  (Services)      │  (Repositories)│  (SQL/NoSQL)  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ✓ Simple, familiar                                                         │
│  ✗ Tight coupling (upper layers depend on lower)                           │
│  ✗ Hard to test business logic in isolation                                │
│  Best for: Simple CRUD apps, small teams                                   │
│                                                                             │
│  2. CLEAN ARCHITECTURE (Onion / Hexagonal / Ports & Adapters)             │
│  ─────────────────────────────────────────────────────────────────         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  PRESENTATION (Controllers, Minimal APIs, gRPC, Blazor)            │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  APPLICATION (Use Cases, Commands/Queries, DTOs, Validators)       │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  DOMAIN (Entities, Value Objects, Aggregates, Domain Events,       │   │
│  │         Domain Services, Specifications, Repository Interfaces)    │   │
│  │  ◄──────────────────────────────────────────────────────────       │   │
│  │  ZERO external dependencies                                         │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  INFRASTRUCTURE (EF Core, Redis, HTTP Clients, Identity,           │   │
│  │                 Message Bus, Logging, Background Jobs)             │   │
│  │  Implements: Domain Repository Interfaces                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  Dependency Rule: Inner layers know NOTHING about outer layers             │
│  ✓ Testable, flexible, framework-independent                              │
│  ✗ More upfront structure, learning curve                                 │
│  Best for: Complex domain, long-lived apps, multiple delivery mechanisms  │
│                                                                             │
│  3. MODULAR MONOLITH                                                       │
│  ────────────────────────                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Shared Host Process                                                 │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │   │
│  │  │  Module A  │ │  Module B  │ │  Module C  │ │  Shared    │       │   │
│  │  │ (Feature)  │ │ (Feature)  │ │ (Feature)  │ │ Kernel     │       │   │
│  │  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘       │   │
│  │        │              │              │             │               │   │
│  │        └──────────────┼──────────────┼─────────────┘               │   │
│  │                       ▼              ▼                             │   │
│  │              ┌─────────────────────────────┐                       │   │
│  │              │     Shared Database         │                       │   │
│  │              │   (separate schemas/tables) │                       │   │
│  │              └─────────────────────────────┘                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ✓ Simple deployment, ACID transactions, in-process calls                 │
│  ✓ Clear module boundaries, extractable to microservices later            │
│  ✗ Shared DB = coupling risk, single process = scaling limit              │
│  Best for: Teams < 10, well-understood domain, start here                 │
│                                                                             │
│  4. VERTICAL SLICE ARCHITECTURE                                           │
│  ──────────────────────────────────                                        │
│  Organize by FEATURE not LAYER                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Feature: Orders                                                    │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │   │
│  │  │  API        │ │  Command    │ │  Query      │ │  Domain     │  │   │
│  │  │  Endpoints  │ │  Handlers   │ │  Handlers   │ │  Model      │  │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  │   │
│  │  Feature: Payments                                                  │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │   │
│  │  │  API        │ │  Command    │ │  Query      │ │  Domain     │  │   │
│  │  │  Endpoints  │ │  Handlers   │ │  Handlers   │ │  Model      │  │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  ✓ High cohesion, low coupling, easy to extract                           │
│  ✗ Some duplication (mediator, validation) per slice                      │
│  Best for: Feature teams, evolving domains                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Core Application Architecture Patterns)

#### 1. CLEAN ARCHITECTURE IN CODE

```csharp
// DOMAIN LAYER (Zero dependencies)
namespace Ordering.Domain;

public record OrderId(Guid Value) : EntityId(Value);

public class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderItem> _items = new();
    
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public Money Total { get; private set; }

    private Order() { } // EF Core

    public static Order Create(CustomerId customerId, 
        IEnumerable<OrderItemData> items, ShippingAddress address)
    {
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
            Total = Money.Zero("USD")
        };
        
        foreach (var item in items)
        {
            order.AddItem(item.ProductId, item.Quantity, item.UnitPrice);
        }
        
        order.AddDomainEvent(new OrderCreatedEvent(order.Id, order.CustomerId, order.Items));
        return order;
    }

    public void AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null)
        {
            existing.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(new OrderItem(productId, quantity, unitPrice));
        }
        RecalculateTotal();
    }

    public Result Submit()
    {
        if (Status != OrderStatus.Draft) 
            return Result.Failure("Order already submitted");
        if (!_items.Any()) 
            return Result.Failure("Cannot submit empty order");
            
        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmittedEvent(Id, CustomerId, Items));
        return Result.Success();
    }
}

// APPLICATION LAYER (Use Cases)
namespace Ordering.Application;

public record CreateOrderCommand(
    CustomerId CustomerId,
    List<OrderItemDto> Items,
    ShippingAddressDto ShippingAddress
) : IRequest<Result<OrderId>>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IDomainEventPublisher _events;

    public async Task<Result<OrderId>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items, cmd.ShippingAddress);
        
        await _repository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync(ct);
        
        await _events.PublishAsync(order.DomainEvents);
        
        return Result.Success(order.Id);
    }
}

// INFRASTRUCTURE LAYER (Implements Domain Interfaces)
namespace Ordering.Infrastructure;

public class EfOrderRepository : IOrderRepository
{
    private readonly OrderingDbContext _context;
    
    public async Task<Order?> GetAsync(OrderId id) 
        => await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
    
    public async Task AddAsync(Order order) 
        => await _context.Orders.AddAsync(order);
    
    public Task UpdateAsync(Order order) 
        => Task.FromResult(_context.Orders.Update(order));
}
```

---

#### 2. CROSS-CUTTING CONCERNS IN APPLICATION ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CROSS-CUTTING CONCERNS HANDLING                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CONCERN              │  WHERE IT LIVES          │  HOW IMPLEMENTED         │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Logging              │  Infrastructure          │  Serilog + structured    │
│                       │  (Presentation for req)  │  logging, correlation ID │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Validation           │  Application (Fluent     │  Pipeline behavior in    │
│                       │   Validation) + Domain   │  MediatR, domain guards  │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Authorization        │  Application (policies)  │  Policy-based, claims    │
│                       │  + Presentation (attrs)  │  + resource-based        │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Transactions         │  Application (UnitOfWork)│  EF Core SaveChanges,    │
│                       │  + Infrastructure        │  outbox pattern          │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Caching              │  Infrastructure          │  Redis, Cache-Aside,     │
│                       │  (Application defines    │  Write-Through, TTL +    │
│                       │   cache keys)            │  event invalidation      │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Observability        │  Infrastructure          │  OpenTelemetry, metrics, │
│                       │  + all layers            │  traces, health checks   │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Idempotency          │  Application             │  Idempotency key in      │
│                       │  (command handlers)      │  request header, dedupe  │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Rate Limiting        │  Presentation (API GW)   │  YARP / middleware,      │
│                       │                          │  token bucket per user   │
│  ─────────────────────┼──────────────────────────┼───────────────────────── │
│  Resilience           │  Infrastructure          │  Polly: retry, circuit   │
│                       │  (HTTP clients)          │  breaker, timeout,       │
│                       │                          │  bulkhead                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. SOLUTION ARCHITECTURE

### WHY? (Problem Statement)

**Solution Architecture** designs an end-to-end system that delivers a business capability. It spans multiple applications/services and answers: *How do these pieces work together to solve a business problem?*

**Without it:**
- Integration chaos — services talk in incompatible ways
- Data inconsistency — no clear ownership, duplicate data
- Operational blindness — can't trace a request across services
- Scaling surprises — one service's load kills another

---

### HOW? (Solution Architecture Process)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SOLUTION ARCHITECTURE PROCESS                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. UNDERSTAND THE BUSINESS CAPABILITY                                     │
│     ├─ Business process map (value stream)                                 │
│     ├─ Actors & personas (who does what)                                   │
│     ├─ Functional requirements (use cases)                                 │
│     ├─ Non-functional requirements (SLAs, compliance, scale)               │
│     └─ Constraints (budget, timeline, existing systems, team skills)       │
│                                    │                                        │
│                                    ▼                                        │
│  2. DEFINE SOLUTION BOUNDARIES                                             │
│     ├─ Context diagram (C4 Level 1) — system + external actors            │
│     ├─ Identify bounded contexts (DDD) — where language changes           │
│     ├─ Define data ownership — who owns Customer? Order? Product?         │
│     └─ Identify integration points — sync vs async, contracts             │
│                                    │                                        │
│                                    ▼                                        │
│  3. DESIGN COMPONENT TOPOLOGY                                              │
│     ├─ Container diagram (C4 Level 2) — services, DBs, queues, UIs        │
│     ├─ Communication patterns per flow                                     │
│     │   ├─ Query/Read  → Sync (REST/gRPC) or CQRS read model             │
│     │   ├─ Command/Write → Async (events) or Sync (if immediate needed)   │
│     │   └─ Workflow → Saga (orchestrated or choreographed)                │
│     ├─ Data management strategy                                            │
│     │   ├─ Database per service (polyglot persistence)                    │
│     │   ├─ Shared reference data (cache / read replicas / CDC)            │
│     │   └─ Event sourcing for audit-critical domains                      │
│     └─ Infrastructure topology                                             │
│         ├─ API Gateway (auth, rate limit, routing)                        │
│         ├─ Service Mesh (mTLS, observability, resilience)                 │
│         ├─ Event Backbone (Kafka / Event Grid / Service Bus)              │
│         └─ Observability Stack (logs, metrics, traces)                    │
│                                    │                                        │
│                                    ▼                                        │
│  4. SPECIFY QUALITY ATTRIBUTES (NFRs)                                      │
│     ├─ Latency budgets per critical path                                   │
│     ├─ Availability targets (99.9% vs 99.99% → cost difference)           │
│     ├─ Throughput requirements (peak QPS, burst handling)                 │
│     ├─ Consistency model per data domain (strong/eventual/causal)         │
│     ├─ Security zones & trust boundaries                                   │
│     ├─ Disaster recovery (RPO/RTO per component)                          │
│     └─ Observability SLIs (what to alert on)                              │
│                                    │                                        │
│                                    ▼                                        │
│  5. VALIDATE & ITERATE                                                     │
│     ├─ Architecture Decision Records (ADRs) for each major choice         │
│     ├─ Threat model (STRIDE)                                               │
│     ├─ Cost model (infrastructure + operational)                          │
│     ├─ Prototype / spike risky areas                                       │
│     └─ Review with stakeholders (tech + business)                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Solution Architecture Artifacts)

#### 1. CONTEXT DIAGRAM (C4 Level 1)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTEXT DIAGRAM: ORDER MANAGEMENT                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐                                                           │
│   │  Customer   │──────►│                    ┌─────────────────────┐        │
│   │  (Web App)  │       │                    │   ORDER MANAGEMENT  │        │
│   └─────────────┘       │                    │      SOLUTION       │        │
│                         │                    │                     │        │
│   ┌─────────────┐       │  ┌─────────────┐  │  ┌─────────────┐   │        │
│   │  Customer   │──────►│  │  API GW     │──►│  │   Order     │   │        │
│   │  (Mobile)   │       │  │  (YARP)     │  │  │  Service    │   │        │
│   └─────────────┘       │  └──────┬──────┘  │  └──────┬──────┘   │        │
│                         │         │         │         │          │        │
│   ┌─────────────┐       │  ┌──────┴──────┐  │  ┌──────┴──────┐   │        │
│   │  Support    │──────►│  │  Message    │──┼─►│  Payment    │   │        │
│   │  Agent UI   │       │  │  Broker     │  │  │  Service    │   │        │
│   └─────────────┘       │  │  (Kafka)    │  │  └─────────────┘   │        │
│                         │  └─────────────┘  │  ┌─────────────┐   │        │
│   ┌─────────────┐       │         │         │  │ Inventory   │   │        │
│   │  Warehouse  │◄──────┤         │         │  │  Service    │   │        │
│   │  System     │       │         ▼         │  └─────────────┘   │        │
│   └─────────────┘       │  ┌─────────────┐  │  ┌─────────────┐   │        │
│                         │  │  Shipping   │◄─┼──│ Notification│   │        │
│   ┌─────────────┐       │  │  Service    │  │  │  Service    │   │        │
│   │  Payment    │──────►│  └─────────────┘  │  └─────────────┘   │        │
│   │  Gateway    │       │                   │                    │        │
│   └─────────────┘       │                   └────────────────────┘        │
│                         │                                                   │
│   ┌─────────────┐       │                                                   │
│   │  Email/SMS  │◄──────┤                                                   │
│   │  Providers  │       │                                                   │
│   └─────────────┘       │                                                   │
│                         └───────────────────────────────────────────────────┘
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2. CONTAINER DIAGRAM (C4 Level 2)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTAINER DIAGRAM: ORDER MANAGEMENT                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  EXTERNAL SYSTEMS                                                           │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │  Payment GW  │ │  Email Prov  │ │  SMS Prov    │ │  Warehouse   │       │
│  │  (Stripe)    │ │  (SendGrid)  │ │  (Twilio)    │ │  (ERP)       │       │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘       │
│         │                │                │                │                │
│         ▼                ▼                ▼                ▼                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         SOLUTION BOUNDARY                             │  │
│  │                                                                       │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │  │
│  │  │   Web App   │    │  Mobile App │    │ Support UI  │               │  │
│  │  │  (React)    │    │  (React N)  │    │  (Blazor)   │               │  │
│  │  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘               │  │
│  │         │                  │                  │                       │  │
│  │         └──────────────────┼──────────────────┘                       │  │
│  │                            ▼                                           │  │
│  │                   ┌─────────────────┐                                  │  │
│  │                   │   API Gateway   │                                  │  │
│  │                   │     (YARP)      │                                  │  │
│  │                   │  Auth │ Rate L  │                                  │  │
│  │                   │  Route│ Valid   │                                  │  │
│  │                   └────────┬────────┘                                  │  │
│  │                            │                                           │  │
│  │         ┌──────────────────┼──────────────────┐                       │  │
│  │         ▼                  ▼                  ▼                       │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │  │
│  │  │   Order     │    │  Payment    │    │ Inventory   │               │  │
│  │  │  Service    │    │  Service    │    │  Service    │               │  │
│  │  │  (.NET 8)   │    │  (.NET 8)   │    │  (.NET 8)   │               │  │
│  │  │  PostgreSQL │    │  PostgreSQL │    │  PostgreSQL │               │  │
│  │  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘               │  │
│  │         │                  │                  │                       │  │
│  │         └──────────────────┼──────────────────┘                       │  │
│  │                            ▼                                           │  │
│  │                   ┌─────────────────┐                                  │  │
│  │                   │  Message Broker │                                  │  │
│  │                   │     (Kafka)     │                                  │  │
│  │                   │  Topics:        │                                  │  │
│  │                   │  order.*        │                                  │  │
│  │                   │  payment.*      │                                  │  │
│  │                   │  inventory.*    │                                  │  │
│  │                   └────────┬────────┘                                  │  │
│  │                            │                                           │  │
│  │         ┌──────────────────┼──────────────────┐                       │  │
│  │         ▼                  ▼                  ▼                       │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │  │
│  │  │  Shipping   │    │Notification │    │  Analytics  │               │  │
│  │  │  Service    │    │  Service    │    │  (Flink/    │               │  │
│  │  │  (.NET 8)   │    │  (.NET 8)   │    │   Spark)    │               │  │
│  │  └─────────────┘    └─────────────┘    └─────────────┘               │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3. SEQUENCE DIAGRAM: KEY FLOW (Order Placement)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SEQUENCE: PLACE ORDER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Customer    API GW     Order Svc    Kafka      Payment Svc   Inventory Svc │
│    │          │          │            │            │             │
│    │──POST /orders                                 │             │
│    │          │                                    │             │
│    │          │──Validate & Create Order           │             │
│    │          │    (Draft, emit OrderCreated)      │             │
│    │          │                                    │             │
│    │          │──────────────────►                 │             │
│    │          │      OrderCreated Event            │             │
│    │          │                                    │             │
│    │          │◄──────────────────                 │             │
│    │          │    201 Created (OrderId)           │             │
│    │◄─────────│                                    │             │
│    │          │                                    │             │
│    │          │                                    │             │
│    │          │                │                   │             │
│    │          │                │──OrderCreated      │             │
│    │          │                │                    │             │
│    │          │                │                    │             │
│    │          │                │◄───PaymentReserved │             │
│    │          │                │                    │             │
│    │          │                │                    │──ReserveStock│
│    │          │                │                    │             │
│    │          │                │                    │◄──StockReserved│
│    │          │                │                    │             │
│    │          │                │◄───StockReserved   │             │
│    │          │                │                    │             │
│    │          │                │──OrderConfirmed    │             │
│    │          │                │                    │             │
│    │          │                │◄───OrderConfirmed  │             │
│    │          │                │                    │             │
│    │          │                │◄───OrderConfirmed  │             │
│    │          │                │                    │             │
│    │          │                │──ShipmentScheduled │             │
│    │          │                │                    │             │
│    │          │                │◄───ShipmentSched.  │             │
│    │          │                │                    │             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. ENTERPRISE ARCHITECTURE

### WHY? (Problem Statement)

**Enterprise Architecture (EA)** aligns technology strategy with business strategy across the entire organization. It answers: *How do we govern technology decisions at scale across hundreds of applications and teams?*

**Without it:**
- Technology sprawl — 50 databases, 20 languages, 10 CI/CD tools
- Integration nightmare — no standards, point-to-point spaghetti
- Security gaps — inconsistent auth, data leakage, compliance failures
- Waste — duplicate systems, unused licenses, shadow IT
- Inability to execute strategy — "We want to be cloud-native" but 80% on-prem

---

### HOW? (EA Frameworks & Process)

#### TOGAF ADM (Architecture Development Method)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TOGAF ADM CYCLE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│        ┌─────────────────┐                                                 │
│        │   PRELIMINARY   │  Architecture principles, framework,           │
│        │   (Framework)   │  stakeholders, governance model               │
│        └────────┬────────┘                                                 │
│                 │                                                          │
│                 ▼                                                          │
│        ┌─────────────────┐                                                 │
│        │     PHASE A     │  Business architecture: capabilities,          │
│        │  ARCHITECTURE   │  value streams, organization, stakeholders    │
│        │    VISION       │                                                 │
│        └────────┬────────┘                                                 │
│                 │                                                          │
│        ┌────────┴────────┐                                                 │
│        ▼                ▼                                                 │
│  ┌─────────────┐  ┌─────────────┐                                        │
│  │   PHASE B   │  │   PHASE C   │  (can run in parallel)                 │
│  │  BUSINESS   │  │  INFORMATION│                                        │
│  │ARCHITECTURE │  │  SYSTEMS    │                                        │
│  │             │  │ARCHITECTURE │                                        │
│  └──────┬──────┘  └──────┬──────┘                                        │
│         │                │                                                │
│         └────────┬───────┘                                                │
│                  ▼                                                         │
│        ┌─────────────────┐                                                 │
│        │   PHASE D       │  Technology architecture: platforms,           │
│        │  TECHNOLOGY     │  infrastructure, networks, standards          │
│        │ARCHITECTURE     │                                                 │
│        └────────┬────────┘                                                 │
│                 │                                                          │
│                 ▼                                                          │
│        ┌─────────────────┐                                                 │
│        │   PHASE E       │  Opportunities & solutions: gap analysis,      │
│        │  OPPORTUNITIES  │  work packages, migration planning            │
│        │   & SOLUTIONS   │                                                 │
│        └────────┬────────┘                                                 │
│                 │                                                          │
│                 ▼                                                          │
│        ┌─────────────────┐                                                 │
│        │   PHASE F       │  Migration planning: projects, sequence,       │
│        │  MIGRATION      │  risks, resources                              │
│        │  PLANNING       │                                                 │
│        └────────┬────────┘                                                 │
│                 │                                                          │
│                 ▼                                                          │
│        ┌─────────────────┐                                                 │
│        │   PHASE G       │  Implementation governance: compliance,        │
│        │ IMPLEMENTATION  │  architecture contracts, fitness functions    │
│        │  GOVERNANCE     │                                                 │
│        └────────┬────────┘                                                 │
│                 │                                                          │
│                 ▼                                                          │
│        ┌─────────────────┐                                                 │
│        │   PHASE H       │  Architecture change management:               │
│        │  CHANGE         │  monitor, assess, trigger new cycle           │
│        │ MANAGEMENT      │                                                 │
│        └─────────────────┘                                                 │
│                                                                             │
│        ┌─────────────────────────────────────────────────────────────┐    │
│        │  CONTINUOUS: Requirements Management (feeds all phases)     │    │
│        └─────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### EA Domains & Artifacts

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE ARCHITECTURE DOMAINS                          │
├──────────────────────────┬──────────────────────────┬──────────────────────┤
│    BUSINESS ARCHITECTURE │  INFORMATION ARCHITECTURE│  APPLICATION ARCH    │
│    ───────────────────── │  ──────────────────────  │  ─────────────────   │
│    • Business Strategy   │  • Data Governance       │  • Application       │
│    • Capability Model    │  • Master Data Mgmt      │    Portfolio         │
│    • Value Streams       │  • Reference Data        │  • Integration       │
│    • Business Processes  │  • Analytics & BI        │    Architecture      │
│    • Organization Map    │  • Data Privacy (GDPR)   │  • Standards &       │
│    • Stakeholder Map     │  • Metadata Mgmt         │    Patterns          │
│                          │  • Data Lineage          │  • Lifecycle Mgmt    │
│                          │                          │  • Rationalization   │
├──────────────────────────┼──────────────────────────┼──────────────────────┤
│    TECHNOLOGY ARCHITECTURE                                                         │
│    ──────────────────────                                                         │
│    • Infrastructure Strategy (Cloud, On-prem, Hybrid)                             │
│    • Platform Standards (K8s, Serverless, DBaaS, Message Brokers)                │
│    • Network Topology (VPC, Peering, Transit Gateway, Service Mesh)              │
│    • Security Baseline (Zero Trust, mTLS, Secrets, Identity)                     │
│    • Observability Stack (Logs, Metrics, Traces, Alerting)                       │
│    • Developer Platform (IDP: CI/CD, Templates, Docs, Self-Service)              │
│    • Vendor Management (Strategic vs Tactical, Lock-in Assessment)               │
│    • Cost Optimization (FinOps, Right-sizing, Reserved Instances)                │
├──────────────────────────┴──────────────────────────┴──────────────────────────┤
│                         CROSS-CUTTING CONCERNS                                   │
│    • Security Architecture (SABSA)    • Compliance & Audit                     │
│    • Enterprise Integration (ESB, API Gateway, Event Mesh)                     │
│    • Architecture Governance (Review Boards, Fitness Functions, ADRs)          │
│    • Talent & Skills Strategy                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (EA Deliverables & Governance)

#### Key EA Deliverables

| Deliverable | Purpose | Cadence | Owner |
|-------------|---------|---------|-------|
| **Technology Radar** | Adopt/Trial/Assess/Hold for technologies | Quarterly | EA Team |
| **Application Portfolio** | Inventory: cost, risk, health, fit | Annual + continuous | EA + App Owners |
| **Architecture Principles** | Guardrails for decision-making | Review annually | CTO / EA Lead |
| **Standards & Patterns** | Approved: languages, frameworks, clouds, DBs | As needed | Architecture Board |
| **Target State Architecture** | 3-5 year vision | Annual | EA + Business |
| **Roadmap & Work Packages** | Initiatives, sequencing, dependencies | Quarterly | EA + PMO |
| **Compliance Reports** | Standards adherence, exceptions | Quarterly / audit | EA + Security |
| **Architecture Decision Records** | Decision log with rationale | Continuous | All Architects |

#### Architecture Governance Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE GOVERNANCE MODEL                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ENTERPRISE ARCHITECTURE BOARD (Monthly)                                    │
│  ─────────────────────────────────────                                      │
│  Chair: CTO / Chief Architect                                               │
│  Members: Domain Architects, Security, Infra, Data, PMO                    │
│  Decisions: Standards, exceptions, strategic direction, funding            │
│                                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  DOMAIN ARCHITECTURE REVIEWS (Bi-weekly per domain)                         │
│  ───────────────────────────────────────────                                │
│  Chair: Domain Architect                                                    │
│  Members: Tech Leads, Security Champion, Product                           │
│  Scope: Solution designs, ADRs, fitness function results                   │
│                                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  AUTOMATED FITNESS FUNCTIONS (Every CI/CD run)                              │
│  ───────────────────────────────────────────                                │
│  • ArchUnit / NetArchTest — layer rules, banned deps                       │
│  • Dependency freshness — CVE scanning, version lag                        │
│  • Contract tests — Pact consumer-driven contracts                         │
│  • Schema compatibility — Confluent Schema Registry                        │
│  • Security — SAST/DAST/SCA, secrets scanning                              │
│  • Cost — Infracost in PR, budget alerts                                   │
│                                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  EXCEPTION PROCESS                                                          │
│  ─────────────────                                                          │
│  1. Team submits exception request (risk, mitigation, timeline)            │
│  2. Domain Architect reviews, recommends                                    │
│  3. EA Board approves/denies with conditions                               │
│  4. Exception tracked with expiry date & remediation plan                  │
│  5. Auto-escalate if not remediated by expiry                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Context | Anti-Pattern | Why Avoid |
|---------|---------|--------------|-----------|
| **C4 Model for Diagrams** | All architecture levels | Ad-hoc boxes & arrows | Inconsistent, not scalable, stakeholders confused |
| **ADR for Decisions** | Every significant choice | Verbal decisions, Confluence pages | Decisions lost, no rationale, can't revisit |
| **Fitness Functions in CI** | Governance at scale | Manual architecture reviews | Slow, inconsistent, doesn't scale |
| **Domain-Driven Bounded Contexts** | Solution architecture | Anemic services, shared DB | Distributed monolith, coupling, team friction |
| **API Gateway + Service Mesh** | Solution infrastructure | Direct service-to-service | No auth/rate limit/observability at edge |
| **Technology Radar** | EA technology strategy | "Use whatever you want" | Sprawl, skill dilution, support burden |
| **Application Portfolio Rationalization** | EA cost/health | Never retire apps | Zombie apps, security risk, wasted spend |
| **Strategic vs Tactical Decisions** | EA governance | All decisions tactical | Short-term fixes create long-term debt |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. Application Architecture: "Walk me through Clean Architecture layers and dependency rule."

**Answer:**
```
Four layers, dependency points INWARD:

1. DOMAIN (center) — Entities, VOs, Aggregates, Domain Events, Repository Interfaces
   ZERO external deps. Pure business logic. Most stable.

2. APPLICATION — Use Cases (Commands/Queries), DTOs, Validators, Handlers
   Depends on Domain. Orchestrates domain objects. No infrastructure.

3. INFRASTRUCTURE — EF Core, Redis, HTTP Clients, Identity, Message Bus
   Implements Domain interfaces (IOrderRepository). Depends on Application.

4. PRESENTATION — Controllers, Minimal APIs, gRPC, Blazor, SignalR
   Depends on Application. Entry points. Thin — just serialization/validation.

DEPENDENCY RULE: Inner knows nothing about outer.
- Domain has NO reference to EF Core, MediatR, ASP.NET
- Application has NO reference to PostgreSQL, Redis, Kafka
- Infrastructure IMPLEMENTS interfaces defined in Domain/Application

WHY: Testability (mock interfaces), Flexibility (swap infra), 
     Stability (domain changes least), Team autonomy
```

---

### 2. Application Architecture: "Modular Monolith vs Microservices — when to choose?"

**Answer:**
```
MODULAR MONOLITH (Start here):
✓ Team < 10 developers
✓ Domain well-understood, stable boundaries
✓ Low scaling requirements (or uniform scaling)
✓ Need ACID transactions across modules
✓ Simple deployment, operational maturity low
✓ Fast iteration, shared database OK

MICROSERVICES (Evolve to):
✓ Team > 50, multiple independent teams
✓ Clear bounded contexts (different languages, owners)
✓ Different scaling profiles per service
✓ Technology heterogeneity required
✓ Independent deployment critical
✓ Fault isolation required
✓ Organization has DevOps/platform maturity

EVOLUTION PATH:
Modular Monolith → Strangler Fig → Microservices

KEY: Extract service ONLY when pain > cost of extraction
```

---

### 3. Solution Architecture: "How do you decompose a system into services?"

**Answer:**
```
1. IDENTIFY BOUNDED CONTEXTS (DDD)
   - Ubiquitous Language differences (Customer means different things)
   - Team ownership boundaries (who owns what)
   - Data ownership (who is system of record for Customer?)
   - Transactional boundaries (what MUST be ACID?)

2. MAP BUSINESS CAPABILITIES
   - Value stream map → capabilities
   - Each capability = candidate service
   - Capabilities stable, processes change

3. ANALYZE COUPLING
   - High cohesion within, low coupling between
   - Data coupling (shared tables = bad)
   - Temporal coupling (must deploy together = bad)
   - Behavioral coupling (chatty calls = bad)

4. DEFINE SERVICE CONTRACTS
   - API contracts (OpenAPI/Protobuf)
   - Event contracts (Avro/Protobuf in Schema Registry)
   - SLOs per service (latency, availability, throughput)

5. VALIDATE WITH TEAM TOPOLOGY
   - One team per service (or service per team)
   - Team has all skills to own service end-to-end
   - Cognitive load manageable (< 7 services/team)

ANTI-PATTERNS:
❌ Nanoservices (too small, chatty, operational hell)
❌ Shared database (defeats independence)
❌ Distributed monolith (deploy together, fail together)
❌ Anemic services (no behavior, just CRUD over DB)
```

---

### 4. Solution Architecture: "Sync vs Async communication — when to use which?"

**Answer:**
```
SYNCHRONOUS (REST/gRPC) — Use for:
✓ Queries — caller needs data NOW to proceed
✓ User-facing operations — latency critical
✓ ACID transactions — immediate consistency required
✓ Simple request/response — low coupling
✓ Read-your-writes — confirmation needed

ASYNCHRONOUS (Kafka/RabbitMQ) — Use for:
✓ Commands — fire-and-forget, work can be delayed
✓ Cross-service workflows — Saga orchestration/choreography
✓ High throughput — absorb spikes, decouple producer/consumer
✓ Loose coupling — services don't need to know each other
✓ Eventual consistency acceptable — most business operations

DECISION TREE:
Caller needs result NOW? → Sync
   │
   └─No → Work is slow/retryable/spiky? → Async
              │
              └─No → Simple notification? → Async (fire-and-forget)
                     │
                     └─Yes → Sync (but ask: can we make it async?)

HYBRID (Real systems):
- Order placement: Sync for validation + order creation
- Payment: Sync for authorization (immediate)
- Fulfillment: Async (inventory, shipping, notification)
- Analytics: Async (event stream to warehouse)
```

---

### 5. Solution Architecture: "How do you handle data consistency across services?"

**Answer:**
```
CONSISTENCY SPECTRUM (choose per business operation):

1. STRONG CONSISTENCY (ACID)
   - Single database, local transactions
   - Use: Financial ledger, inventory reservation
   - Pattern: Keep in same service / same DB

2. EVENTUAL CONSISTENCY (BASE) — DEFAULT for distributed
   - Write local, publish event, others react
   - Use: Order → Payment → Inventory → Shipping
   - Pattern: Saga (choreography or orchestration)

3. SAGA CHOREOGRAPHY
   - Services emit/consume events directly
   - OrderCreated → Payment.Reserve → Inventory.Reserve → Shipping.Schedule
   - Compensation: PaymentFailed → Inventory.Release → Order.Cancel
   - Good for: Simple workflows, few services

4. SAGA ORCHESTRATION
   - Central coordinator (state machine) tells participants what to do
   - OrderOrchestrator: ReservePayment → ReserveStock → ScheduleShipment
   - Compensation built into orchestrator
   - Good for: Complex workflows, many services, visibility needed

5. OUTBOX PATTERN (Reliable event publishing)
   - DB write + event in SAME transaction
   - Background processor publishes events
   - Solves: "DB updated but event lost" (dual-write problem)

6. EVENT SOURCING
   - Store events, not current state
   - Rebuild state by replaying events
   - Full audit trail, temporal queries, debugging
   - Use: Audit-critical domains (financial, regulatory)

7. CQRS (Command Query Responsibility Segregation)
   - Separate write model (commands) from read model (queries)
   - Write: Domain model, validation, events
   - Read: Denormalized projections, optimized for queries
   - Use: Complex read patterns, different scaling needs

CHOICE GUIDE:
- Financial/Inventory reservation → Strong (same service)
- Order processing → Saga (orchestrated)
- Audit required → Event Sourcing
- Complex reads → CQRS
- Everything else → Eventual + Outbox
```

---

### 6. Solution Architecture: "Design an API Gateway — what does it do?"

**Answer:**
```
API GATEWAY = Single entry point for all clients

CORE RESPONSIBILITIES:
┌─────────────────────────────────────────────────────────────────┐
│  ROUTING              │  Path-based, host-based, header-based   │
│  AUTHENTICATION       │  JWT validation, API keys, mTLS        │
│  AUTHORIZATION        │  RBAC/ABAC policies, scope validation  │
│  RATE LIMITING        │  Token bucket per user/IP/tenant       │
│  REQUEST VALIDATION   │  Schema validation, size limits        │
│  TRANSFORMATION       │  Request/response mapping, aggregation │
│  OBSERVABILITY        │  Logging, metrics, tracing, correlation│
│  RESILIENCE           │  Circuit breaker, retry, timeout       │
│  LOAD BALANCING       │  Round-robin, least connections        │
│  SSL TERMINATION      │  Cert management, mTLS to services     │
└─────────────────────────────────────────────────────────────────┘

TECHNOLOGY CHOICES:
- YARP (.NET native, high perf, config-driven)
- Kong (plugin ecosystem, Lua scripting)
- NGINX Plus / Envoy (high perf, service mesh sidecar)
- AWS API Gateway / Azure API Management (managed)

PATTERN: BACKEND FOR FRONTEND (BFF)
- Separate gateway per client type (Web, Mobile, Partner)
- Each BFF aggregates, transforms for its client
- Avoids "one API fits all" over-fetching/under-fetching
```

---

### 7. Solution Architecture: "What is a Service Mesh and when do you need it?"

**Answer:**
```
SERVICE MESH = Infrastructure layer for service-to-service communication

WHAT IT PROVIDES:
┌─────────────────────────────────────────────────────────────────┐
│  TRAFFIC MANAGEMENT  │  Load balancing, routing, retries,       │
│                      │  timeouts, circuit breaking, fault inj   │
│  SECURITY            │  mTLS (auto cert rotation), authZ,       │
│                      │  peer authentication, authorization      │
│  OBSERVABILITY       │  Distributed tracing, metrics, logs      │
│                      │  (no code changes — sidecar injects)     │
│  RESILIENCE          │  Retry budgets, deadlines, bulkheads     │
└─────────────────────────────────────────────────────────────────┘

POPULAR: Istio, Linkerd, Consul Connect, AWS App Mesh

WHEN YOU NEED IT:
✓ 20+ services with complex communication
✓ Need mTLS everywhere (compliance)
✓ Advanced traffic shifting (canary, blue/green)
✓ Multi-cluster / hybrid cloud
✓ Organization standardizes on mesh

WHEN YOU DON'T:
✗ < 10 services (complexity > benefit)
✗ Simple communication patterns
✗ Team can't operate mesh (dedicated platform team needed)
✗ Latency budget extremely tight (sidecar adds ~1-2ms)

ALTERNATIVE: Library-based (gRPC interceptors, Polly, custom middleware)
```

---

### 8. Enterprise Architecture: "Explain TOGAF ADM cycle."

**Answer:**
```
TOGAF ADM = Architecture Development Method (iterative cycle)

PHASES:
A. ARCHITECTURE VISION — Scope, stakeholders, business context, approval
B. BUSINESS ARCHITECTURE — Capabilities, value streams, org, processes
C. INFORMATION SYSTEMS ARCHITECTURE
   C1. Data Architecture — Governance, master data, analytics, privacy
   C2. Application Architecture — Portfolio, integration, standards
D. TECHNOLOGY ARCHITECTURE — Platforms, infrastructure, networks, security
E. OPPORTUNITIES & SOLUTIONS — Gap analysis, work packages, migration
F. MIGRATION PLANNING — Projects, sequence, risks, resources
G. IMPLEMENTATION GOVERNANCE — Compliance, contracts, fitness functions
H. CHANGE MANAGEMENT — Monitor, assess, trigger new cycle

CONTINUOUS: Requirements Management (feeds all phases)

KEY INSIGHT: ADM is NOT waterfall. It's iterative:
- Cycle at enterprise level (annual)
- Cycle at solution level (per initiative)
- Cycle at project level (per delivery)

ARTIFACTS PER PHASE: Architecture Vision, Building Blocks, 
Gap Analysis Report, Migration Plan, Architecture Contracts
```

---

### 9. Enterprise Architecture: "How do you govern technology choices at scale?"

**Answer:**
```
TECHNOLOGY RADAR (ThoughtWorks model):
┌──────────────┬────────────────────────────────────────────────────┐
│ ADOPT        │ Proven, standard, default choice                   │
│              │ (e.g., .NET 8, PostgreSQL, Kafka, Kubernetes)     │
├──────────────┼────────────────────────────────────────────────────┤
│ TRIAL        │ Evaluating for specific use case                   │
│              │ (e.g., Rust for high-perf service, ClickHouse)    │
├──────────────┼────────────────────────────────────────────────────┤
│ ASSESS       │ Promising, not ready for production               │
│              │ (e.g., WebAssembly server-side, new DB)           │
├──────────────┼────────────────────────────────────────────────────┤
│ HOLD         │ Deprecated, avoid for new work                     │
│              │ (e.g., .NET Framework 4.8, legacy message queue)  │
└──────────────┴────────────────────────────────────────────────────┘

GOVERNANCE MECHANISMS:
1. ARCHITECTURE REVIEW BOARD — Monthly, approves exceptions
2. AUTOMATED FITNESS FUNCTIONS — In CI/CD (ArchUnit, dependency check)
3. PAVED ROAD / GOLDEN PATH — Templates, scaffolds, approved stacks
4. SELF-SERVICE PLATFORM — IDP: developers get approved stack by default
5. EXCEPTION PROCESS — Documented, time-boxed, remediation required

PRINCIPLE: "Make the right thing the easy thing."
```

---

### 10. Enterprise Architecture: "Application Portfolio Rationalization — how?"

**Answer:**
```
GOAL: Reduce cost, risk, complexity by retiring/consolidating apps

PROCESS:
1. INVENTORY — Discover all apps (CMDB, surveys, APM, network scans)
   For each: Owner, users, cost, tech stack, criticality, health

2. ASSESS — Score each app:
   ┌─────────────────┬──────────────┬──────────────┬──────────────┐
   │ DIMENSION       │ WEIGHT       │ METRIC       │ SCORE 1-5    │
   ├─────────────────┼──────────────┼──────────────┼──────────────┤
   │ Business Fit    │ 30%          │ Strategic?   │              │
   │ Technical Health│ 25%          │ Debt, bugs   │              │
   │ Cost            │ 20%          │ TCO/year     │              │
   │ Risk            │ 15%          │ Security, SPOF│             │
   │ Agility         │ 10%          │ Change lead  │              │
   └─────────────────┴──────────────┴──────────────┴──────────────┘

3. CATEGORIZE:
   - INVEST — High fit, high health → modernize, enhance
   - TOLERATE — Low fit, high health → maintain, plan sunset
   - MIGRATE — High fit, low health → refactor/replatform
   - RETIRE — Low fit, low health → sunset, extract data

4. EXECUTE — Roadmap with dependencies, funding, change mgmt

METRICS:
- % apps in Invest/Migrate (target > 70%)
- Annual cost reduction from retirements
- Average app age (target < 7 years)
- Security vulnerabilities per app (target 0 critical)
```

---

### 11. Cross-Cutting: "How do you handle observability across all levels?"

**Answer:**
```
OBSERVABILITY STACK (OpenTelemetry standard):

┌─────────────────────────────────────────────────────────────────────────────┐
│                         THREE PILLARS + CONTINUOUS PROFILING                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  METRICS (Prometheus)           LOGS (Loki/ELK)         TRACES (Tempo/Jaeger)│
│  ──────────────────────         ────────────────         ────────────────── │
│  • RED metrics                  • Structured JSON        • W3C TraceContext │
│    Rate, Errors, Duration       • Correlation IDs        • Span per RPC     │
│  • USE metrics                  • Log levels             • Parent/child     │
│    Utilization, Saturation,     • Sampling               • Attributes       │
│    Errors                       • Retention policies     • Sampling         │
│  • Business KPIs                • Alerting on logs       • Service map      │
│    Orders/min, Revenue/hr       • PII redaction          • Critical path    │
│  • SLI/SLO tracking             • Cost optimization      • Latency analysis │
│                                                                             │
│  CONTINUOUS PROFILING (Pyroscope/Parca)                                     │
│  • CPU, Memory, Goroutines, Lock contention                                 │
│  • Always-on, low overhead                                                  │
│  • Flame graphs per service                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

IMPLEMENTATION PER LEVEL:
├── APPLICATION — Structured logs, custom metrics, manual spans
├── SOLUTION — Distributed traces across services, service map, SLO dashboards
├── ENTERPRISE — Centralized stack, unified dashboards, alerting standards,
│                cost allocation, compliance auditing

KEY PRINCIPLES:
✓ Instrument once, use everywhere (OpenTelemetry SDK)
✓ Correlation IDs everywhere (trace-id, span-id in every log)
✓ Sample intelligently (head-based + tail-based for errors)
✓ Alert on SYMPTOMS (SLO breach) not CAUSES (CPU high)
✓ Dashboards per persona (Dev, SRE, Product, Exec)
```

---

### 12. Cross-Cutting: "Security architecture — Zero Trust principles."

**Answer:**
```
ZERO TRUST = "Never trust, always verify"

CORE PRINCIPLES:
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. VERIFY IDENTITY           │  Every request authenticated               │
│     • mTLS for service-to-service                                            │
│     • OAuth2/OIDC for users                                                  │
│     • SPIFFE/SPIRE for workload identity                                     │
│  2. LEAST PRIVILEGE           │  Minimal access required                   │
│     • RBAC/ABAC policies                                                     │
│     • Just-in-time access                                                    │
│     • Service accounts per service (no shared)                              │
│  3. ASSUME BREACH             │  Design for compromise                     │
│     • Micro-segmentation (network policies)                                 │
│     • Encryption in transit (TLS 1.3) + at rest (AES-256)                  │
│     • Audit logging on all access                                           │
│  4. CONTINUOUS VALIDATION     │  Trust is not permanent                    │
│     • Short-lived certs (SPIFFE rotates hourly)                             │
│     • Continuous authorization (OPA/Gatekeeper)                            │
│     • Device posture checks                                                 │
└─────────────────────────────────────────────────────────────────────────────┘

ARCHITECTURE PATTERNS:
├── API GATEWAY — North-south traffic (auth, rate limit, WAF)
├── SERVICE MESH — East-west traffic (mTLS, authZ, observability)
├── SECRETS MANAGEMENT — Vault, AWS Secrets Manager, Azure Key Vault
├── POLICY ENGINE — OPA/Gatekeeper for admission + runtime authZ
├── SECURITY SCANNING — SAST/DAST/SCA in CI, container scanning
└── INCIDENT RESPONSE — Runbooks, forensics, key rotation procedures

COMPLIANCE MAPPING:
- SOC2 → Access control, encryption, audit logging
- GDPR → Data minimization, consent, right to erasure, DPIA
- PCI-DSS → Network segmentation, encryption, vulnerability mgmt
- HIPAA → Access control, audit controls, transmission security
```

---

### 13. Cross-Cutting: "How do you manage technical debt at enterprise scale?"

**Answer:**
```
TECH DEBT MANAGEMENT FRAMEWORK:

1. MAKE IT VISIBLE
   ├── Tech Debt Registry (Jira/GitHub Projects) — every item tracked
   ├── Categorize: Code, Architecture, Infrastructure, Process, Security
   ├── Score: Impact (1-5) × Likelihood (1-5) × Effort to fix (1-5)
   ├── Link to business outcomes (incidents, velocity, cost, risk)

2. ALLOCATE BUDGET
   ├── "20% rule" — 20% capacity for debt reduction (configurable)
   ├── Dedicated sprints (quarterly) for large items
   ├── Boy Scout Rule — leave code better than you found it (daily)

3. PRIORITIZE BY RISK
   ├── CRITICAL — Security vulns, data loss risk, single points of failure
   ├── HIGH — Frequent incidents, blocks features, high change failure
   ├── MEDIUM — Slows velocity, onboarding pain, scaling limits
   ├── LOW — Cosmetic, minor duplication, outdated comments

4. PREVENT NEW DEBT
   ├── Architecture Fitness Functions in CI (ArchUnit, dependency rules)
   ├── Definition of Done includes: tests, docs, observability, no TODOs
   ├── Code review checklist: "Does this add debt? Can we avoid it?"
   ├── ADR required for deviations from standards

5. MEASURE & REPORT
   ├── Debt Ratio (SonarQube) — remediation cost / development cost
   ├── Debt Index trend (monthly)
   ├── Velocity impact (story points lost to debt)
   ├── Incident correlation (% incidents caused by known debt)

GOVERNANCE:
- Quarterly Debt Review (Architecture Board)
- Debt burn-down in sprint reviews
- Executive dashboard: Debt trend, risk exposure, investment ROI
```

---

### 14. Strategy: "How do you align architecture with business strategy?"

**Answer:**
```
ARCHITECTURE STRATEGY = Business Strategy translated to Technical Decisions

PROCESS:
1. UNDERSTAND BUSINESS STRATEGY
   ├── OKRs / KPIs for next 1-3 years
   ├── Market position (cost leader, differentiation, niche)
   ├── Growth targets (users, revenue, geographies)
   ├── M&A plans, partnerships, divestitures
   ├── Regulatory changes (new markets, data laws)

2. DERIVE ARCHITECTURAL DRIVERS
   ┌─────────────────────┬─────────────────────────────────────────────┐
   │ BUSINESS GOAL       │ ARCHITECTURAL IMPLICATION                   │
   ├─────────────────────┼─────────────────────────────────────────────┤
   │ "Expand to EU"      │ Multi-region, GDPR, data residency,        │
   │                     │ local payment providers, latency budgets   │
   │ "10x users in 2yr"  │ Horizontal scaling, stateless, caching,    │
   │                     │ event-driven, sharding strategy            │
   │ "Faster features"   │ Modular architecture, feature flags,       │
   │                     │ self-service platform, CI/CD maturity      │
   │ "Reduce cost 20%"   │ FinOps, right-sizing, serverless,         │
   │                     │ data tiering, reserved instances           │
   │ "Acquire startup"   │ Integration patterns, API gateway,        │
   │                     │ data migration, identity federation        │
   └─────────────────────┴─────────────────────────────────────────────┘

3. CREATE TARGET STATE ARCHITECTURE
   ├── 3-year vision (where we want to be)
   ├── Current state assessment (gaps)
   ├── Roadmap: Initiatives, sequencing, dependencies, funding
   ├── Quick wins (3-6 months) + Strategic bets (1-3 years)

4. GOVERN EXECUTION
   ├── Architecture Review Board approves initiatives
   ├── Fitness functions ensure compliance
   ├── Quarterly business reviews: "Are we on track?"
   ├── Adjust based on market feedback
```

---

### 15. Leadership: "How do you build architecture culture in engineering org?"

**Answer:**
```
ARCHITECTURE CULTURE = "Architecture is everyone's responsibility"

1. ARCHITECTS AS ENABLERS (not gatekeepers)
   ├── Pair program with teams on complex features
   ├── Build platform tools that make right thing easy
   ├── Office hours / Slack channel for architecture questions
   ├── Write decision records WITH teams, not FOR teams

2. DECENTRALIZE DECISIONS
   ├── Architecture principles = guardrails, not rules
   ├── Teams make tactical decisions within boundaries
   ├── Escalate only cross-cutting / strategic decisions
   ├── Document team-level ADRs in repo (not central wiki)

3. INVEST IN PLATFORM
   ├── Internal Developer Platform (IDP) — self-service everything
   ├── Paved roads: templates, scaffolds, approved libraries
   ├── Shared infrastructure: CI/CD, observability, auth, secrets
   ├── Platform team as product team (customers = developers)

4. MEASURE WHAT MATTERS
   ├── DORA metrics (deployment freq, lead time, CFR, MTTR)
   ├── Architecture Fitness Function pass rate
   ├── Developer experience (survey, NPS)
   ├── Time to onboard new developer

5. CELEBRATE GOOD ARCHITECTURE
   ├── "Architecture Wins" in all-hands (debt paid, service extracted)
   ├── Blameless postmortems — focus on system, not person
   ├── Architecture Katas / Dojos — practice design skills
   ├── Mentor program: Senior ↔ Junior architecture pairing

ANTI-PATTERNS:
❌ Architecture Review Board as approval gate (slows delivery)
❌ "Architect" title without coding (ivory tower)
❌ Centralized standards team disconnected from delivery
❌ Diagrams in Confluence that no one updates
❌ Architecture as "phase" before coding (waterfall)
```