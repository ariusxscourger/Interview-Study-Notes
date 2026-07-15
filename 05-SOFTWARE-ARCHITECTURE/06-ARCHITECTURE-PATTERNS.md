# ARCHITECTURE PATTERNS
## Exam-Style Study Notes

**Roadmap Section:** Architecture → CQRS, Eventual Consistency, OOP, MVC/MVP/MVVM, Microservices, ACID/CAP, Serverless, SOLID, TDD, Security, DDD, PKI, Client/Server, OWASP, Auth Strategies, Layered, Reactive Programming, Distributed Systems, Functional Programming, SOA, Working with Data, ETL/Datawarehouses, SQL/NoSQL, SPA/SSR/SSG, Web/Mobile, APIs & Integrations, Microfrontends, Analytics, gRPC, W3C/WHATWG

---

## 1. CQRS (COMMAND QUERY RESPONSIBILITY SEGREGATION)

### WHY? (Problem Statement)

**Traditional CRUD models couple reads and writes** — same model, same database, same queries. This creates:
- **Read/Write conflict**: Optimizing for writes (normalization) hurts reads (joins), and vice versa
- **Scaling mismatch**: Reads often 100:1 vs writes, but must scale together
- **Complexity**: Business logic scattered across repositories, services, controllers
- **Audit gap**: No natural record of *why* state changed

---

### HOW? (Core Mechanism)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CQRS ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   COMMAND SIDE (Write Model)              QUERY SIDE (Read Model)          │
│   ─────────────────────────              ─────────────────────────         │
│                                                                             │
│  ┌──────────────┐                          ┌──────────────┐               │
│  │   Client     │                          │   Client     │               │
│  └──────┬───────┘                          └──────┬───────┘               │
│         │                                       │                           │
│         ▼                                       ▼                           │
│  ┌──────────────┐  Commands          ┌──────────────┐  Queries             │
│  │  API/Controller│ ─────────────────►│  Read API    │ ◄─────────────────  │
│  └──────┬───────┘                     └──────┬───────┘                    │
│         │                                    │                            │
│         ▼                                    ▼                            │
│  ┌──────────────────┐              ┌──────────────────┐                   │
│  │  Command Handlers│              │  Query Handlers  │                   │
│  │  (Use Cases)     │              │  (Projections)   │                   │
│  └────────┬─────────┘              └────────┬─────────┘                   │
│           │                                 │                             │
│           ▼                                 ▼                             │
│  ┌──────────────────┐              ┌──────────────────┐                   │
│  │   DOMAIN MODEL   │              │   READ MODELS    │                   │
│  │  (Aggregates,    │              │  (Denormalized,  │                   │
│  │   Entities,      │              │   Optimized for  │                   │
│  │   Value Objects, │              │   Query Patterns)│                   │
│  │   Domain Events) │              └────────┬─────────┘                   │
│  └────────┬─────────┘                       │                             │
│           │                                 │                             │
│           ▼                                 ▼                             │
│  ┌──────────────────┐              ┌──────────────────┐                   │
│  │  WRITE DATABASE  │              │  READ DATABASES  │                   │
│  │  (Source of      │              │  (PostgreSQL,    │                   │
│  │   Truth, ACID)   │              │   Elasticsearch, │                   │
│  └────────┬─────────┘              │   Redis, etc.)   │                   │
│           │                        └──────────────────┘                   │
│           │ Event Publication                                              │
│           ▼                                                               │
│  ┌──────────────────┐                                                    │
│  │  EVENT BUS       │                                                    │
│  │  (Kafka, Rabbit) │                                                    │
│  └────────┬─────────┘                                                    │
│           │                                                               │
│           ▼                                                               │
│  ┌──────────────────┐                                                    │
│  │  PROJECTIONS     │  Async: Consume events → Update read models       │
│  │  (Event Handlers)│                                                    │
│  └──────────────────┘                                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Implementation Patterns)

#### A. COMMAND SIDE (Write)

```csharp
// COMMAND: Intent to change state
public record CreateOrderCommand(
    CustomerId CustomerId,
    IReadOnlyList<OrderItemData> Items,
    ShippingAddress ShippingAddress
) : IRequest<Result<OrderId>>;

// HANDLER: Validates, executes, persists, publishes
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IDomainEventPublisher _publisher;

    public async Task<Result<OrderId>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        // 1. Create aggregate (domain logic)
        var order = Order.Create(cmd.CustomerId, cmd.Items, cmd.ShippingAddress);
        
        // 2. Persist
        await _repository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync(ct);
        
        // 3. Publish domain events (outbox pattern)
        await _publisher.PublishAsync(order.DomainEvents);
        
        return Result.Success(order.Id);
    }
}

// AGGREGATE ROOT: Encapsulates invariants
public class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderItem> _items = new();
    
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    private Order() { } // EF Core
    
    public static Order Create(CustomerId customerId, 
        IEnumerable<OrderItemData> items, ShippingAddress address)
    {
        var order = new Order 
        { 
            Id = OrderId.New(), 
            CustomerId = customerId, 
            Status = OrderStatus.Draft 
        };
        
        foreach (var item in items)
            order.AddItem(item.ProductId, item.Quantity, item.UnitPrice);
            
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
}
```

#### B. QUERY SIDE (Read)

```csharp
// QUERY: Request for data
public record GetOrderSummaryQuery(OrderId Id) : IRequest<OrderSummaryDto?>;

// HANDLER: Optimized read model
public class GetOrderSummaryHandler : IRequestHandler<GetOrderSummaryQuery, OrderSummaryDto?>
{
    private readonly IOrderReadRepository _readRepo; // Different interface!
    
    public async Task<OrderSummaryDto?> Handle(GetOrderSummaryQuery query, CancellationToken ct)
        => await _readRepo.GetSummaryAsync(query.Id, ct);
}

// READ MODEL: Denormalized, no behavior
public record OrderSummaryDto(
    OrderId Id,
    string CustomerName,
    OrderStatus Status,
    Money Total,
    int ItemCount,
    DateTime CreatedAt,
    DateTime? ShippedAt
);

// READ REPOSITORY: Optimized queries
public interface IOrderReadRepository
{
    Task<OrderSummaryDto?> GetSummaryAsync(OrderId id, CancellationToken ct);
    Task<PagedResult<OrderSummaryDto>> SearchAsync(OrderSearchCriteria criteria, CancellationToken ct);
}

// IMPLEMENTATION: Could be SQL view, Elasticsearch, Redis, materialized view
public class SqlOrderReadRepository : IOrderReadRepository
{
    private readonly ReadDbContext _context;
    
    public async Task<OrderSummaryDto?> GetSummaryAsync(OrderId id, CancellationToken ct)
    {
        return await _context.OrderSummaries
            .Where(s => s.Id == id)
            .Select(s => new OrderSummaryDto(...))
            .FirstOrDefaultAsync(ct);
    }
}
```

#### C. EVENT PROJECTIONS (Sync Read Models)

```csharp
// PROJECTOR: Consumes events → Updates read models
public class OrderProjection : 
    IEventHandler<OrderCreatedEvent>,
    IEventHandler<OrderSubmittedEvent>,
    IEventHandler<OrderItemAddedEvent>,
    IEventHandler<OrderShippedEvent>
{
    private readonly IOrderReadRepository _readRepo;
    private readonly IUnitOfWork _uow;

    public async Task Handle(OrderCreatedEvent evt, CancellationToken ct)
    {
        var summary = new OrderSummaryDto(
            evt.OrderId,
            await GetCustomerName(evt.CustomerId, ct),
            OrderStatus.Draft,
            Money.Zero("USD"),
            0,
            evt.OccurredAt,
            null
        );
        await _readRepo.AddAsync(summary);
        await _uow.SaveChangesAsync(ct);
    }

    public async Task Handle(OrderSubmittedEvent evt, CancellationToken ct)
    {
        var summary = await _readRepo.GetByIdAsync(evt.OrderId, ct);
        summary = summary with { Status = OrderStatus.Submitted };
        await _readRepo.UpdateAsync(summary);
        await _uow.SaveChangesAsync(ct);
    }

    public async Task Handle(OrderItemAddedEvent evt, CancellationToken ct)
    {
        var summary = await _readRepo.GetByIdAsync(evt.OrderId, ct);
        summary = summary with 
        { 
            Total = summary.Total.Add(evt.LineTotal),
            ItemCount = summary.ItemCount + evt.Quantity
        };
        await _readRepo.UpdateAsync(summary);
        await _uow.SaveChangesAsync(ct);
    }
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Full CQRS** | Complex domain, different read/write scaling, audit needed | CQRS for simple CRUD | Over-engineering, eventual consistency confusion |
| **Separate Read Models** | Multiple query patterns (search, dashboard, export) | Single model for all queries | N+1 queries, over-fetching, join explosions |
| **Eventual Consistency** | High write throughput, user accepts slight delay | Synchronous projection updates | Couples write latency to read consistency |
| **Outbox Pattern** | Reliable event publishing | Direct publish in transaction | Dual write problem, message loss |

---

## 2. EVENTUAL CONSISTENCY

### WHY? (Problem Statement)

**Strong consistency (ACID) doesn't scale** across services:
- Distributed transactions (2PC) = blocking, not partition-tolerant
- Single database = coupling, single point of failure
- User experience: "Why is my order pending?" vs "Order confirmed!"

---

### HOW? (Consistency Models)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY SPECTRUM                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STRONG ────────────────────────────────────────────────── EVENTUAL         │
│  ────────                                                                   │
│                                                                             │
│  Linearizable          Sequential          Causal          Eventual         │
│  ─────────────         ─────────────       ─────────       ─────────        │
│  • Single DB           • Same session       • Related        • No order      │
│  • Consensus           • Read your writes    • operations     • Converges    │
│  • CP systems          • Per-session        • CRDTs          • AP systems    │
│  (etcd, ZooKeeper)     • ordering           • (Riak,         • (DynamoDB,    │
│                                                                             │
│  LATENCY: High        LATENCY: Med          LATENCY: Low     LATENCY: Low   │
│  AVAILABILITY: Low    AVAILABILITY: Med     AVAILABILITY: Hi AVAILABILITY:Hi│
│                                                                             │
│  USE FOR:             USE FOR:              USE FOR:         USE FOR:       │
│  - Financial          - User profiles       - Collaborative  - Analytics    │
│    ledgers            - Shopping carts      - editing        - Feeds        │
│  - Inventory          - Session state       - Notifications  - Logs         │
│    reservation        - Preferences         - Chat           - Metrics      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Implementation Patterns)

#### A. SAGA PATTERN (Distributed Transactions)

```csharp
// CHOREOGRAPHY-BASED: Events drive flow
public class OrderSaga
{
    // Order Service emits OrderCreated
    // Payment Service consumes → emits PaymentReserved / PaymentFailed
    // Inventory Service consumes PaymentReserved → emits StockReserved / StockUnavailable
    // Shipping Service consumes StockReserved → emits ShipmentScheduled
    // Order Service consumes all → updates OrderStatus
    
    // COMPENSATION: Reverse events on failure
    // PaymentFailed → OrderCancelled
    // StockUnavailable → PaymentRefunded → OrderCancelled
}

// ORCHESTRATION-BASED: Central coordinator
public class OrderOrchestrator
{
    private readonly IMessageBus _bus;
    private readonly ISagaStateRepository _state;

    public async Task StartOrderAsync(CreateOrderCommand cmd)
    {
        var sagaId = Guid.NewGuid();
        var state = new OrderSagaState 
        { 
            SagaId = sagaId, 
            OrderId = cmd.OrderId, 
            Step = SagaStep.PaymentPending 
        };
        await _state.SaveAsync(state);
        
        await _bus.Send(new ReservePaymentCommand(sagaId, cmd.OrderId, cmd.Total));
    }

    public async Task Handle(PaymentReservedEvent evt)
    {
        var state = await _state.GetAsync(evt.SagaId);
        state.Step = SagaStep.InventoryPending;
        await _state.UpdateAsync(state);
        
        await _bus.Send(new ReserveStockCommand(evt.SagaId, evt.OrderId, evt.Items));
    }

    public async Task Handle(StockReservedEvent evt)
    {
        var state = await _state.GetAsync(evt.SagaId);
        state.Step = SagaStep.ShippingPending;
        await _state.UpdateAsync(state);
        
        await _bus.Send(new ScheduleShipmentCommand(evt.SagaId, evt.OrderId, evt.Address));
    }

    public async Task Handle(PaymentFailedEvent evt)
    {
        // COMPENSATION: Reverse completed steps
        await _bus.Send(new CancelOrderCommand(evt.SagaId, evt.OrderId));
        await _bus.Send(new ReleaseStockCommand(evt.SagaId, evt.OrderId));
    }
}
```

#### B. OUTBOX PATTERN (Reliable Event Publishing)

```csharp
// 1. OUTBOX TABLE (same DB as domain)
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; }           // Assembly-qualified name
    public string Content { get; set; }        // JSON serialized event
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public int RetryCount { get; set; }
    public string? Error { get; set; }
}

// 2. DB CONTEXT: Capture domain events in SaveChanges
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var domainEntities = ChangeTracker.Entries<IHasDomainEvents>()
        .Where(e => e.Entity.DomainEvents.Any())
        .ToList();

    var events = domainEntities.SelectMany(e => e.Entity.DomainEvents).ToList();
    domainEntities.ForEach(e => e.Entity.ClearDomainEvents());

    foreach (var @event in events)
    {
        OutboxMessages.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = @event.GetType().AssemblyQualifiedName!,
            Content = JsonSerializer.Serialize(@event),
            CreatedAt = DateTime.UtcNow
        });
    }

    return await base.SaveChangesAsync(ct);
}

// 3. BACKGROUND PROCESSOR: Publishes events
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var bus = scope.ServiceProvider.GetRequiredService<IMessageBus>();

            var messages = await db.OutboxMessages
                .Where(m => !m.ProcessedAt.HasValue && m.RetryCount < 5)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(stoppingToken);

            foreach (var msg in messages)
            {
                try
                {
                    var @event = JsonSerializer.Deserialize(msg.Content, 
                        Type.GetType(msg.Type)!) as IDomainEvent;
                    await bus.PublishAsync(@event, stoppingToken);
                    msg.ProcessedAt = DateTime.UtcNow;
                }
                catch (Exception ex)
                {
                    msg.RetryCount++;
                    msg.Error = ex.Message;
                }
            }

            await db.SaveChangesAsync(stoppingToken);
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

#### C. CONFLICT RESOLUTION (Eventual Consistency)

```csharp
// LAST-WRITER-WINS (Simple, data loss possible)
public class LwwRegister<T>
{
    public T Value { get; private set; }
    public DateTime Timestamp { get; private set; }
    public string NodeId { get; private set; }

    public void Set(T value, string nodeId, DateTime timestamp)
    {
        if (timestamp > Timestamp || (timestamp == Timestamp && 
            string.CompareOrdinal(nodeId, NodeId) > 0))
        {
            Value = value;
            Timestamp = timestamp;
            NodeId = nodeId;
        }
    }
}

// CRDT: G-COUNTER (Grow-only counter)
public class GCounter
{
    private readonly Dictionary<string, long> _counts = new();
    
    public long Value => _counts.Values.Sum();
    
    public void Increment(string nodeId) 
        => _counts[nodeId] = _counts.GetValueOrDefault(nodeId) + 1;
    
    public void Merge(GCounter other)
    {
        foreach (var kvp in other._counts)
            _counts[kvp.Key] = Math.Max(_counts.GetValueOrDefault(kvp.Key), kvp.Value);
    }
}

// CRDT: LWW-MAP (Last-Writer-Wins Map)
public class LwwMap<TKey, TValue>
{
    private readonly Dictionary<TKey, (TValue Value, DateTime Timestamp, string NodeId)> _map = new();
    
    public void Set(TKey key, TValue value, string nodeId, DateTime timestamp)
    {
        if (!_map.TryGetValue(key, out var existing) || 
            timestamp > existing.Timestamp ||
            (timestamp == existing.Timestamp && 
             string.CompareOrdinal(nodeId, existing.NodeId) > 0))
        {
            _map[key] = (value, timestamp, nodeId);
        }
    }
    
    public bool TryGetValue(TKey key, out TValue value)
    {
        if (_map.TryGetValue(key, out var entry))
        {
            value = entry.Value;
            return true;
        }
        value = default!;
        return false;
    }
}
```

---

## 3. OOP (OBJECT-ORIENTED PROGRAMMING) PRINCIPLES

### SOLID (Already in Technical Skills — Quick Refresher)

| Principle | Essence | Violation Smell |
|-----------|---------|-----------------|
| **S**ingle Responsibility | One reason to change | God class, many dependencies |
| **O**pen/Closed | Extend without modifying | Switch on type, if-else chains |
| **L**iskov Substitution | Subtypes behave as base | Override throws NotImplemented |
| **I**nterface Segregation | Small, focused interfaces | Fat interfaces, unused methods |
| **D**ependency Inversion | Depend on abstractions | `new Concrete()` in business logic |

---

### OOP DESIGN PATTERNS (Gang of Four — Essential for Architects)

| Category | Pattern | Problem Solved | Key Insight |
|----------|---------|----------------|-------------|
| **Creational** | **Factory Method** | Create objects without specifying class | Delegate to subclass |
| | **Abstract Factory** | Families of related objects | Interface for factory |
| | **Builder** | Complex object construction | Step-by-step, fluent |
| | **Prototype** | Clone existing instances | Avoid costly creation |
| | **Singleton** | Single instance (avoid in distributed!) | Private ctor + static access |
| **Structural** | **Adapter** | Incompatible interfaces | Wrapper translates |
| | **Decorator** | Add behavior dynamically | Wrap + delegate |
| | **Facade** | Simplify complex subsystem | Unified high-level API |
| | **Proxy** | Control access to object | Lazy, protection, remote |
| | **Composite** | Tree structures | Uniform treatment |
| **Behavioral** | **Strategy** | Interchangeable algorithms | Encapsulate algorithm |
| | **Observer** | One-to-many notification | Subject + listeners |
| | **Command** | Encapsulate request as object | Queue, undo, log |
| | **Template Method** | Algorithm skeleton + hooks | Base class defines flow |
| | **State** | Object alters behavior by state | Delegates to state obj |
| | **Mediator** | Reduce coupling between objects | Central coordinator |

---

## 4. MVC / MVP / MVVM (UI ARCHITECTURE PATTERNS)

### WHY? (Problem Statement)

**UI logic tangled with business logic** = untestable, unmaintainable, platform-locked.

---

### HOW? (Comparison)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         UI PATTERN COMPARISON                               │
├────────────────┬────────────────┬────────────────┬──────────────────────────┤
│      MVC       │      MVP       │      MVVM      │         MVI              │
├────────────────┼────────────────┼────────────────┼──────────────────────────┤
│                │                │                │                          │
│ Model          │ Model          │ Model          │ Model (State)            │
│    │           │    │           │    │           │    ▲                     │
│    ▼           │    ▼           │    ▼           │    │                     │
│ View ◄──►      │ View ◄──►      │ View ◄──►      │ View │                   │
│ Controller     │ Presenter      │ ViewModel      │ Intent                   │
│                │                │                │    │                     │
│                │                │                │    ▼                     │
│                │                │                │ Reducer                  │
├────────────────┼────────────────┼────────────────┼──────────────────────────┤
│ Controller     │ Presenter      │ ViewModel      │ Reducer (Pure Function)  │
│ handles input, │ handles input, │ binds to View, │ State = reduce(State,    │
│ updates Model, │ updates Model, │ exposes data/  │    Intent)               │
│ selects View   │ updates View   │ commands       │                          │
├────────────────┼────────────────┼────────────────┼──────────────────────────┤
│ View passive,  │ View passive,  │ View binds to  │ View renders State,      │
│ renders Model  │ renders via    │ ViewModel,     │ sends Intent             │
│ data           │ Presenter      │ no code-behind │                          │
├────────────────┼────────────────┼────────────────┼──────────────────────────┤
│ Web (Spring,   │ WinForms,      │ WPF, UWP,      │ React, Flutter,          │
│ ASP.NET MVC,   │ Android (old), │ Xamarin,       │ Redux, Vuex,             │
│ Rails, Django) │ ASP.NET Web    │ Blazor,        │ Jetpack Compose          │
│                │ Forms          │ Angular, Vue   │                          │
├────────────────┼────────────────┼────────────────┼──────────────────────────┤
│ Testable:      │ Testable:      │ Testable:      │ Testable:                │
│ Controller     │ Presenter      │ ViewModel      │ Reducer (pure)           │
│ (mock View)    │ (mock View)    │ (no View)      │                          │
└────────────────┴────────────────┴────────────────┴──────────────────────────┘
```

---

### WHAT? (Modern Variants)

```csharp
// MVVM (Blazor / WPF / MAUI)
public class OrderViewModel : INotifyPropertyChanged
{
    private readonly IOrderService _service;
    private OrderDto? _order;
    
    public OrderDto? Order 
    { 
        get => _order; 
        set { _order = value; OnPropertyChanged(); }
    }
    
    public ICommand SubmitCommand { get; }
    
    public OrderViewModel(IOrderService service)
    {
        _service = service;
        SubmitCommand = new AsyncCommand(SubmitAsync);
    }
    
    private async Task SubmitAsync()
    {
        if (Order == null) return;
        await _service.SubmitAsync(Order.Id);
        Order = Order with { Status = OrderStatus.Submitted };
    }
}

// MVI / REDUX (React / Flutter / Compose)
public record OrderState(
    OrderDto? CurrentOrder,
    bool IsLoading,
    string? Error
);

public record SubmitOrderIntent(OrderId Id) : IIntent;

public static class OrderReducer
{
    public static OrderState Reduce(OrderState state, IIntent intent)
        => intent switch
        {
            SubmitOrderIntent i => state with { IsLoading = true, Error = null },
            SubmitOrderSuccess s => state with 
            { 
                IsLoading = false, 
                CurrentOrder = state.CurrentOrder with { Status = OrderStatus.Submitted }
            },
            SubmitOrderFailure f => state with { IsLoading = false, Error = f.Message },
            _ => state
        };
}
```

---

## 5. MICROSERVICES ARCHITECTURE

### WHY? (Problem Statement)

**Monolith at scale** = deployment coupling, team friction, technology lock-in, blast radius.

---

### HOW? (Decomposition & Patterns)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DECOMPOSITION STRATEGIES:                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. BUSINESS CAPABILITY (DDD Bounded Contexts)                       │   │
│  │    Order │ Payment │ Inventory │ Shipping │ Notification │ Customer │   │
│  │                                                                     │   │
│  │ 2. SUBDOMAIN TYPE                                                   │   │
│  │    Core (competitive advantage)  │  Supporting  │  Generic         │   │
│  │    → Custom, invest              │  → Simple    │  → Buy/Offload   │   │
│  │                                                                     │   │
│  │ 3. TEAM TOPOLOGY (Conway's Law)                                    │   │
│  │    Stream-aligned teams own 1-N services end-to-end                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMUNICATION:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ SYNCHRONOUS (Query/Command)          ASYNCHRONOUS (Event)          │   │
│  │ ─────────────────────────────        ─────────────────────         │   │
│  │ • REST / gRPC                        • Kafka / RabbitMQ            │   │
│  │ • Request-Response                   • Publish-Subscribe           │   │
│  │ • Temporal coupling                  • Temporal decoupling         │   │
│  │ • Read-your-writes                   • Eventual consistency        │   │
│  │ • Use: Queries, immediate actions    • Use: Workflows, notifications│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DATA MANAGEMENT:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • DATABASE PER SERVICE (Polyglot Persistence)                       │   │
│  │ • SAGA for distributed transactions                                 │   │
│  │ • CQRS for read/write scaling                                       │   │
│  │ • EVENT SOURCING for audit/history                                  │   │
│  │ • CDC for cross-service read models                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  INFRASTRUCTURE:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • API GATEWAY (YARP, Kong, Envoy) — Auth, Rate Limit, Routing      │   │
│  │ • SERVICE MESH (Istio, Linkerd) — mTLS, Observability, Resilience  │   │
│  │ • SERVICE DISCOVERY (Consul, etcd, DNS)                             │   │
│  │ • CONFIG MANAGEMENT (Spring Config, etcd, Azure App Config)        │   │
│  │ • OBSERVABILITY (OpenTelemetry, Prometheus, Grafana, Jaeger)       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Implementation Decisions)

| Decision | Options | Recommendation |
|----------|---------|----------------|
| **Service Size** | Nanoservice → Modulith → Macro | **Modular Monolith first**, extract when pain > cost |
| **Communication** | REST, gRPC, GraphQL, Messages | **Sync for queries, Async for commands** |
| **Data** | Shared DB, DB per service, Event Sourcing | **DB per service + Outbox + CDC** |
| **Transactions** | 2PC, Saga (Choreography/Orchestration) | **Saga Orchestration** for complex flows |
| **Deployment** | VM, Container, Serverless, K8s | **Kubernetes** for >20 services |
| **Observability** | Logs, Metrics, Traces | **OpenTelemetry + Prometheus + Jaeger** |

---

### ANTI-PATTERNS TO AVOID

| Anti-Pattern | Symptom | Fix |
|--------------|---------|-----|
| **Distributed Monolith** | Deploy together, fail together, shared DB | Enforce independent deployability |
| **Chatty Services** | 10+ HTTP calls per user request | Aggregate API, GraphQL, CQRS read models |
| **Shared Database** | Schema changes break multiple services | Database per service, CDC for reads |
| **Sync Everything** | Cascading timeouts, low availability | Async for commands, eventual consistency |
| **No Contract Tests** | Breaking changes in production | Pact / Spring Cloud Contract |
| **God Service** | One service does everything | Decompose by bounded context |

---

## 6. ACID / CAP THEOREM

### ACID (Single Database)

| Property | Guarantees |
|----------|------------|
| **Atomicity** | All or nothing |
| **Consistency** | Valid state transitions |
| **Isolation** | Concurrent = serializable |
| **Durability** | Committed = survived crash |

### CAP THEOREM (Correct Framing: PACELC)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CAP / PACELC (The Real Story)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ORIGINAL CAP (Misleading "Pick 2"):                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ You can only guarantee 2 of 3:                                      │   │
│  │  C - Consistency (all nodes see same data)                          │   │
│  │  A - Availability (every request gets response)                     │   │
│  │  P - Partition Tolerance (system works despite network failure)     │   │
│  │                                                                     │   │
│  │ PROBLEM: P is NOT optional — networks WILL partition.               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PACELC (Correct Framing):                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ If Partition (P) → Choose Availability (A) or Consistency (C)      │   │
│  │ Else (E) → Choose Latency (L) or Consistency (C)                   │   │
│  │                                                                     │   │
│  │ SYSTEM CLASSIFICATION:                                              │   │
│  │ • PC/EC: MongoDB, HBase, Redis     (Consistency always)            │   │
│  │ • PA/EL: Cassandra, DynamoDB       (Availability in partition,     │   │
│  │                                     Latency else)                  │   │
│  │ • PA/EC: (Rare)                                                                │   │
│  │ • PC/EL: Cosmos DB (strong consistency mode)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONSISTENCY LEVELS (Strongest → Weakest):                                 │
│  1. LINEARIZABLE / STRONG: Read sees latest write (PostgreSQL, etcd)       │
│  2. SEQUENTIAL: Operations appear in some sequential order                 │
│  3. CAUSAL: Causally related ops ordered (version vectors)                 │
│  4. EVENTUAL: Converges over time (DynamoDB, Cassandra)                    │
│                                                                             │
│  CONFLICT RESOLUTION (Eventual):                                           │
│  • LWW (Last-Writer-Wins) — simple, data loss                              │
│  • CRDTs — mathematically correct merge                                    │
│  • Application-level merge — domain-specific                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. SERVERLESS

### WHY? (Problem Statement)

**Manage servers = undifferentiated heavy lifting.** Serverless = focus on business logic.

---

### HOW? (Patterns & Trade-offs)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SERVERLESS ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FUNCTION-AS-A-SERVICE (FaaS):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Event-driven (HTTP, Queue, Timer, Blob, Stream)                  │   │
│  │ • Stateless, ephemeral (max 15 min typically)                      │   │
│  │ • Auto-scale to zero / thousands                                   │   │
│  │ • Pay per invocation × duration × memory                           │   │
│  │ • Cold start: 10ms-2s (mitigate: provisioned concurrency, snap)    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BACKEND-AS-A-SERVICE (BaaS):                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Auth: Auth0, Firebase Auth, AWS Cognito                          │   │
│  │ • Database: DynamoDB, Firestore, Cosmos DB, Supabase               │   │
│  │ • Storage: S3, Blob Storage, Firebase Storage                      │   │
│  │ • API: API Gateway, AppSync, GraphQL                               │   │
│  │ • Search: Algolia, Elasticsearch Service                           │   │
│  │ • Queue: SQS, EventBridge, Service Bus                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PATTERNS:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. STRANGLER FIG → SERVERLESS: Migrate endpoints one-by-one         │   │
│  │ 2. EVENT-DRIVEN PIPELINE: API → Lambda → DynamoDB → Lambda → SNS   │   │
│  │ 3. FAN-OUT / FAN-IN: One event → parallel processing → aggregate   │   │
│  │ 4. SCHEDULED JOBS: Cron → Lambda (replace cron servers)            │   │
│  │ 5. WEBHOOKS: Receive → Lambda → Queue → Process                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  TRADE-OFFS:                                                                │
│  ┌──────────────────────┬──────────────────────────────────────────────┐   │
│  │ PROS                 │ CONS                                         │   │
│  ├──────────────────────┼──────────────────────────────────────────────┤   │
│  │ No server management │ Cold starts (latency spike)                  │   │
│  │ Auto-scale to zero   │ Vendor lock-in (hard to migrate)             │   │
│  │ Pay per use          │ Debugging harder (distributed, ephemeral)    │   │
│  │ Built-in HA          │ Limits: time, memory, payload, concurrency   │   │
│  │ Fast deploy          │ Testing complexity (local vs cloud)          │   │
│  └──────────────────────┴──────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### DECISION FRAMEWORK

| Use Serverless When | Avoid Serverless When |
|---------------------|----------------------|
| Spiky/unpredictable traffic | Consistent high throughput (cheaper on K8s) |
| Event-driven workloads | Long-running computations (>15 min) |
| Rapid prototyping/MVP | Complex stateful workflows |
| Team lacks ops expertise | Strict data residency / compliance |
| Batch/ETL jobs | Ultra-low latency requirements (<10ms) |

---

## 8. SOLID (See Technical Skills — Cross-Reference)

---

## 9. TDD (TEST-DRIVEN DEVELOPMENT)

### WHY? (Problem Statement)

**Tests after = tests never.** TDD = design tool, not just verification.

---

### HOW? (Red-Green-Refactor Cycle)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TDD CYCLE                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. RED (Fail)                                                              │
│     ├─ Write failing test for desired behavior                             │
│     ├─ Test defines API (consumer-first design)                            │
│     ├─ Run → Red (confirms test works)                                     │
│     └─ Time: 30-60 seconds                                                 │
│                                    │                                        │
│                                    ▼                                        │
│  2. GREEN (Pass)                                                            │
│     ├─ Write MINIMUM code to pass                                          │
│     ├─ No gold-plating, no "future-proofing"                               │
│     ├─ Run → Green                                                         │
│     └─ Time: 1-5 minutes                                                   │
│                                    │                                        │
│                                    ▼                                        │
│  3. REFACTOR (Improve)                                                      │
│     ├─ Clean up: naming, duplication, structure                            │
│     ├─ Keep tests green                                                    │
│     ├─ Apply patterns (Strategy, Factory, etc.)                            │
│     └─ Time: 2-10 minutes                                                  │
│                                    │                                        │
│                                    ▼                                        │
│  ─────────────────────────────────────────────────────────────             │
│  │                    REPEAT FOR NEXT BEHAVIOR                             │
│  ─────────────────────────────────────────────────────────────             │
│                                                                             │
│  TEST PYRAMID (TDD produces):                                              │
│  ┌─────────────────────┐                                                  │
│  │      E2E (5%)       │  Slow, brittle, high confidence                 │
│  ├─────────────────────┤                                                  │
│  │   Integration (15%) │  Service boundaries, DB, external APIs          │
│  ├─────────────────────┤                                                  │
│  │    Unit (80%)       │  Fast, isolated, precise, design feedback      │
│  └─────────────────────┘                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (TDD in Practice)

```csharp
// RED: Test first
[Fact]
public async Task Submit_ValidOrder_ChangesStatusToSubmitted()
{
    // Arrange
    var order = Order.CreateDraft(customerId, items, address);
    
    // Act
    var result = order.Submit();
    
    // Assert
    result.Should().BeSuccess();
    order.Status.Should().Be(OrderStatus.Submitted);
    order.DomainEvents.Should().ContainSingle<e => e is OrderSubmittedEvent>();
}

// GREEN: Minimal implementation
public Result Submit()
{
    if (Status != OrderStatus.Draft) return Result.Failure("Not in draft");
    if (!_items.Any()) return Result.Failure("Empty order");
    
    Status = OrderStatus.Submitted;
    Raise(new OrderSubmittedEvent(Id, CustomerId, Items));
    return Result.Success();
}

// REFACTOR: Extract validation, add guards
public Result Submit()
{
    return Result.SuccessIf(Status == OrderStatus.Draft, "Not in draft")
        .Ensure(_items.Any(), "Empty order")
        .OnSuccess(() => 
        {
            Status = OrderStatus.Submitted;
            Raise(new OrderSubmittedEvent(Id, CustomerId, Items));
        });
}
```

---

### TDD ANTI-PATTERNS

| Anti-Pattern | Symptom | Fix |
|--------------|---------|-----|
| **Test-last** | Tests written after (or never) | Enforce TDD in PR checklist |
| **Testing implementation** | Tests break on refactor | Test behavior, not methods |
| **Slow tests** | Suite > 5 min | Isolate unit tests, mock boundaries |
| **Brittle tests** | Change one thing → 20 tests break | Test via public API, not internals |
| **No refactoring** | Code rots, duplication | Mandate refactor step in cycle |

---

## 10. SECURITY (ARCHITECTURE PERSPECTIVE)

### WHY? (Problem Statement)

**Security is not a feature — it's an architecture quality attribute.** Bolted-on security fails.

---

### HOW? (Security by Design)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECURITY ARCHITECTURE LAYERS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. NETWORK PERIMETER                                                      │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ • WAF (OWASP CRS) • DDoS Protection • Geo-blocking             │   │
│     │ • Private subnets • NAT Gateway • VPC Endpoints                │   │
│     │ • Service Mesh mTLS (Istio, Linkerd)                           │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  2. APPLICATION LAYER                                                          │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ • Input Validation (allowlist) • Output Encoding               │   │
│     │ • Parameterized Queries • CSP Headers • HSTS                   │   │
│     │ • Rate Limiting • Request Size Limits                          │   │
│     │ • Security Headers (Helmet, NWebsec)                           │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  3. AUTHENTICATION / AUTHORIZATION                                           │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ • OAuth 2.0 / OIDC (Auth0, Keycloak, Azure AD)                 │   │
│     │ • JWT with short expiry + refresh tokens                       │   │
│     │ • mTLS for service-to-service                                  │   │
│     │ • RBAC / ABAC (OPA, Casbin)                                    │   │
│     │ • Least privilege (scopes, claims)                             │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  4. DATA PROTECTION                                                          │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ • Encryption at rest (AES-256, KMS)                            │   │
│     │ • Encryption in transit (TLS 1.3)                              │   │
│     │ • Field-level encryption (PII, PCI)                            │   │
│     │ • Key rotation (automated, 90-day)                             │   │
│     │ • Secrets Management (Vault, AWS Secrets Manager, Key Vault)  │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  5. OBSERVABILITY & INCIDENT RESPONSE                                        │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ • Audit Logging (immutable, tamper-evident)                    │   │
│     │ • SIEM Integration (Splunk, Sentinel, Elastic)                │   │
│     │ • Anomaly Detection (ML-based)                                 │   │
│     │ • Incident Response Plan (playbooks, runbooks)                │   │
│     │ • Pen Testing (annual + after major changes)                   │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (OWASP Top 10 Mitigations)

| OWASP 2021 Risk | Architecture Mitigation |
|-----------------|------------------------|
| **A01: Broken Access Control** | Deny-by-default, ABAC/OPA, integration tests for every endpoint |
| **A02: Cryptographic Failures** | TLS 1.3 everywhere, AES-256-GCM, KMS-managed keys, cert rotation |
| **A03: Injection** | Parameterized queries, allowlist validation, ORM, CSP |
| **A04: Insecure Design** | Threat modeling (STRIDE), security requirements in ADRs, design reviews |
| **A05: Security Misconfiguration** | IaC scanning, hardened baselines, config-as-code, drift detection |
| **A06: Vulnerable Components** | SCA in CI (Dependabot, Trivy, Snyk), SBOM, patch SLA |
| **A07: Auth Failures** | MFA, rate limiting, breach detection, passwordless/WebAuthn |
| **A08: Software/Data Integrity** | Signed artifacts, SLSA provenance, supply chain security (Sigstore) |
| **A09: Logging/Monitoring Failures** | Structured audit logs, SIEM, alerting on auth anomalies |
| **A10: SSRF** | Egress controls, allowlist URLs, no user-controlled fetch |

---

## 11. DDD (DOMAIN-DRIVEN DESIGN)

### WHY? (Problem Statement)

**Complex domain + technical complexity = unmaintainable mess.** DDD aligns code with business language.

---

### HOW? (Strategic + Tactical)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DDD BUILDING BLOCKS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STRATEGIC DESIGN (Architecture Level)                                     │
│  ─────────────────────────────────────                                     │
│                                                                             │
│  • UBIQUITOUS LANGUAGE: Shared vocabulary between domain experts & devs   │
│  • BOUNDED CONTEXTS: Explicit boundaries where language/model applies     │
│  • CONTEXT MAP: Relationships between contexts (Partnership, ACL, etc.)    │
│  • DOMAIN EVENTS: Cross-context communication                              │
│                                                                             │
│  TACTICAL DESIGN (Code Level)                                              │
│  ──────────────────────────────                                            │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  ENTITY     │  │ VALUE OBJECT│  │  AGGREGATE  │  │DOMAIN EVENT │       │
│  │ ─────────── │  │ ─────────── │  │ ─────────── │  │ ─────────── │       │
│  │ Identity +  │  │ Immutable,  │  │ Consistency │  │ Something   │       │
│  │ Mutability  │  │ no identity │  │ boundary    │  │ happened    │       │
│  │             │  │             │  │ (root only) │  │             │       │
│  │ Order       │  │ Money       │  │ Order       │  │ OrderSubmit │       │
│  │ Customer    │  │ Address     │  │ (root)      │  │ tedEvent    │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  REPOSITORY │  │  FACTORY    │  │  SERVICE    │  │ SPECIFICATION│      │
│  │ ─────────── │  │ ─────────── │  │ ─────────── │  │ ─────────── │       │
│  │ Persistence │  │ Complex     │  │ Domain logic│  │ Business    │       │
│  │ abstraction │  │ creation    │  │ not fitting │  │ rules as    │       │
│  │ (interface) │  │ logic       │  │ in entity   │  │ objects     │       │
│  │             │  │             │  │             │  │             │       │
│  │ IOrderRepo  │  │ OrderFactory│  │ PricingSvc  │  │ EligibleFor │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Code Patterns)

```csharp
// ENTITY: Identity + Lifecycle
public class Customer : Entity<CustomerId>, IAggregateRoot
{
    public string Name { get; private set; }
    public Email Email { get; private set; }
    private readonly List<Address> _addresses = new();
    public IReadOnlyCollection<Address> Addresses => _addresses.AsReadOnly();
    
    private Customer() { } // EF Core
    
    public static Customer Create(CustomerId id, string name, Email email)
    {
        if (string.IsNullOrWhiteSpace(name)) throw new ArgumentException();
        return new Customer { Id = id, Name = name, Email = email };
    }
    
    public void ChangeEmail(Email newEmail)
    {
        if (Email == newEmail) return;
        Email = newEmail;
        Raise(new CustomerEmailChangedEvent(Id, newEmail));
    }
}

// VALUE OBJECT: Immutable, no identity
public record Email(string Value)
{
    private static readonly Regex EmailRegex = new(@"^[^@\s]+@[^@\s]+\.[^@\s]+$");
    
    public static implicit operator string(Email email) => email.Value;
    
    public static Email Parse(string value)
    {
        if (!EmailRegex.IsMatch(value)) throw new ArgumentException("Invalid email");
        return new Email(value);
    }
}

// AGGREGATE ROOT: Consistency boundary
public class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderItem> _items = new();
    
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    public static Order Create(CustomerId customerId, IEnumerable<OrderItemData> items, 
        ShippingAddress address) => new Order { ... }; // Factory method
    
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
}

// DOMAIN EVENT: Something that happened
public record OrderSubmittedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    IReadOnlyCollection<OrderItem> Items,
    DateTime OccurredAt = default
) : IDomainEvent
{
    public DateTime OccurredAt { get; } = OccurredAt == default ? DateTime.UtcNow : OccurredAt;
}

// REPOSITORY: Persistence abstraction
public interface IOrderRepository
{
    Task<Order?> GetAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    Task UpdateAsync(Order order, CancellationToken ct);
}

// DOMAIN SERVICE: Logic not fitting in entity
public class PricingService
{
    private readonly IDiscountRepository _discounts;
    private readonly ITaxCalculator _tax;
    
    public Money CalculateTotal(Order order, Customer customer)
    {
        var subtotal = order.Items.Sum(i => i.Total);
        var discount = _discounts.GetBestFor(customer, order);
        var taxable = subtotal - discount;
        var tax = _tax.Calculate(taxable, customer.Address);
        return taxable + tax;
    }
}
```

---

## 12. PKI (PUBLIC KEY INFRASTRUCTURE)

### WHY? (Problem Statement)

**Trust at scale requires cryptographic identity.** PKI = certificates, CAs, key management.

---

### HOW? (PKI Components)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PKI ARCHITECTURE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CERTIFICATE AUTHORITY (CA) HIERARCHY:                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │    ROOT CA (Offline, Air-gapped)                                    │   │
│  │    │ Self-signed, 20-30 year validity                               │   │
│  │    │ Issues: Intermediate CA certs                                  │   │
│  │    │ Trust anchor for all validation                                │   │
│  │    ▼                                                                │   │
│  │    INTERMEDIATE CA (Online, HSM)                                    │   │
│  │    │ 5-10 year validity                                             │   │
│  │    │ Issues: Leaf certificates                                      │   │
│  │    │ Can have multiple (per purpose: TLS, Code Sign, Email)        │   │
│  │    ▼                                                                │   │
│  │    LEAF CERTIFICATES (End Entity)                                   │   │
│  │    │ 1-2 year validity                                              │   │
│  │    │ Server (TLS), Client (mTLS), Code Signing, Email (S/MIME)     │   │
│  │    │ Contains: Subject, Public Key, Validity, Extensions           │   │
│  │    │ Signed by: Intermediate CA                                     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CERTIFICATE CONTENTS (X.509):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Version (v3)                                                      │   │
│  │ • Serial Number (unique per CA)                                     │   │
│  │ • Signature Algorithm (SHA-256 + RSA/ECDSA)                        │   │
│  │ • Issuer DN (CA)                                                    │   │
│  │ • Validity Period (NotBefore, NotAfter)                            │   │
│  │ • Subject DN (Entity)                                               │   │
│  │ • Subject Public Key Info (Algorithm + Key)                        │   │
│  │ • Extensions:                                                       │   │
│  │   - Subject Alternative Names (SANs) — DNS, IP, Email, UPN         │   │
│  │   - Key Usage (Digital Signature, Key Encipherment, Cert Sign)     │   │
│  │   - Extended Key Usage (Server Auth, Client Auth, Code Sign)       │   │
│  │   - Basic Constraints (CA: true/false, Path Length)                │   │
│  │   - CRL Distribution Points / OCSP URLs                            │   │
│  │   - Certificate Policies                                            │   │
│  │ • Signature (CA private key over tbsCertificate)                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  VALIDATION PATH:                                                          │
│  Leaf → Intermediate → Root (Trust Anchor)                                 │
│  Checks: Signature, Validity, Revocation (CRL/OCSP), Name Constraints,    │
│  Key Usage, EKU, Path Length, Policy Mapping                               │
│                                                                             │
│  REVOCATION:                                                               │
│  • CRL (Certificate Revocation List) — full list, periodic                │
│  • OCSP (Online Certificate Status Protocol) — real-time, per-cert        │
│  • OCSP Stapling — server staples OCSP response in TLS handshake           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Implementation)

```csharp
// mTLS IN ASP.NET CORE
public static IServiceCollection AddMtls(this IServiceCollection services, 
    IConfiguration config)
{
    services.AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme)
        .AddCertificate(options =>
        {
            options.AllowedCertificateTypes = CertificateTypes.All;
            options.RevocationMode = X509RevocationMode.Online;
            options.RevocationFlag = X509RevocationFlag.EntireChain;
            options.ValidateCertificateUse = true;
            options.ValidateValidity = true;
            
            // Custom validation
            options.Events = new CertificateAuthenticationEvents
            {
                OnCertificateValidated = context =>
                {
                    // Check SAN, OU, custom extensions
                    var cert = context.ClientCertificate;
                    if (!cert.SubjectAlternativeNames.Any(san => 
                        san.Value.EndsWith(".company.internal")))
                    {
                        context.Fail("Invalid SAN");
                    }
                    return Task.CompletedTask;
                }
            };
        });
    
    return services;
}

// CERTIFICATE GENERATION (for dev/test)
public static class CertificateGenerator
{
    public static X509Certificate2 GenerateSelfSigned(string subject, 
        string[] dnsNames, TimeSpan validity)
    {
        using var rsa = RSA.Create(2048);
        var request = new CertificateRequest(
            $"CN={subject}", 
            rsa, 
            HashAlgorithmName.SHA256, 
            RSASignaturePadding.Pkcs1);
        
        request.CertificateExtensions.Add(
            new X509SubjectAlternativeNameExtension(
                dnsNames.Select(n => new DnsName(n)).ToArray()));
        
        request.CertificateExtensions.Add(
            new X509KeyUsageExtension(
                X509KeyUsageFlags.DigitalSignature | X509KeyUsageFlags.KeyEncipherment, 
                critical: true));
        
        request.CertificateExtensions.Add(
            new X509EnhancedKeyUsageExtension(
                new OidCollection { 
                    new Oid("1.3.6.1.5.5.7.3.1"), // Server Auth
                    new Oid("1.3.6.1.5.5.7.3.2")  // Client Auth
                }, 
                critical: false));
        
        return request.CreateSelfSigned(
            DateTimeOffset.UtcNow.AddDays(-1), 
            DateTimeOffset.UtcNow.Add(validity));
    }
}
```

---

## 13. CLIENT / SERVER ARCHITECTURE

### WHY? (Problem Statement)

**Separation of concerns** between presentation (client) and business logic/data (server).

---

### HOW? (Evolution)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLIENT/SERVER EVOLUTION                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. MAINFRAME / TERMINAL (1970s)                                           │
│     Dumb terminal → Central mainframe                                      │
│                                                                             │
│  2. FAT CLIENT (1990s)                                                     │
│     Windows app (VB, Delphi, PowerBuilder) → Database                     │
│     Business logic in client                                               │
│                                                                             │
│  3. THREE-TIER / WEB 1.0 (2000s)                                           │
│     Browser (HTML) → Web Server (ASP, JSP, PHP) → Database                │
│     Server-side rendering                                                  │
│                                                                             │
│  4. AJAX / RICH INTERNET (2005+)                                           │
│     Browser (JS) ↔ REST API → Database                                     │
│     Partial page updates                                                   │
│                                                                             │
│  5. SINGLE PAGE APPLICATIONS (2010+)                                       │
│     Browser (React/Vue/Angular) ↔ REST/GraphQL API → Microservices        │
│     Client-side routing, state management                                  │
│                                                                             │
│  6. EDGE / SERVERLESS (2020+)                                              │
│     CDN Edge (Workers) → API Gateway → Functions → Managed Services       │
│     Compute at edge, zero-origin for static                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Modern Patterns)

| Pattern | Client | Server | Communication |
|---------|--------|--------|---------------|
| **Traditional SSR** | Browser (thin) | MVC/Razor/Pages | Form POST, full reload |
| **SPA + REST** | React/Vue/Angular | REST API (JSON) | Fetch/Axios, JWT |
| **SPA + GraphQL** | Apollo/Relay | GraphQL Gateway | Queries/Mutations |
| **SSR + Hydration** | Next.js/Nuxt/SvelteKit | Node.js + API | Initial HTML + client takeover |
| **Islands Architecture** | Astro/Qwik | Any | Partial hydration |
| **Edge-First** | Cloudflare Workers | Edge Functions | Distributed compute |
| **Backend for Frontend (BFF)** | Per-client API | Aggregation layer | Tailored contracts |

---

## 14. OWASP (See Security Section — Cross-Reference)

---

## 15. AUTH STRATEGIES

### WHY? (Problem Statement)

**Authentication ≠ Authorization.** Multiple strategies for different contexts.

---

### HOW? (Strategy Comparison)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION STRATEGIES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SESSION / COOKIE (Traditional Web)                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Server stores session, sends session ID in HttpOnly cookie       │   │
│  │ • CSRF protection (SameSite, tokens)                               │   │
│  │ • Simple, secure for browser apps                                  │   │
│  │ • Doesn't scale well (sticky sessions or shared store)             │   │
│  │ • Vulnerable to CSRF, XSS (mitigated)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  JWT / BEARER TOKEN (Stateless, Mobile/SPA/API)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Self-contained (claims in payload)                               │   │
│  │ • Stateless server (no session store)                              │   │
│  │ • Short expiry (15-30min) + Refresh token (rotation)               │   │
│  │ • RS256/ES256 (asymmetric) for verification                        │   │
│  │ • Can't revoke easily (use blocklist or short expiry)              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  OAUTH 2.0 / OIDC (Delegated Auth, SSO)                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Authorization Code + PKCE (SPA/Mobile)                           │   │
│  │ • Client Credentials (Service-to-Service)                          │   │
│  │ • Device Code (TV/CLI)                                             │   │
│  │ • Refresh tokens with rotation + reuse detection                   │   │
│  │ • Scopes = coarse permissions, Claims = fine-grained               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MUTUAL TLS (Service-to-Service, Zero Trust)                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Both client + server present certificates                        │   │
│  │ • SPIFFE/SPIRE for workload identity                               │   │
│  │ • Automatic rotation (short-lived certs)                           │   │
│  │ • No secrets in code/config                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  API KEYS (Simple, Partner Integration)                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Long-lived, revocable                                            │   │
│  │ • Prefix for identification (sk_live_..., pk_test_...)             │   │
│  │ • Rate limiting per key                                            │   │
│  │ • HMAC signing for sensitive operations                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PASSWORDLESS / MODERN                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • WebAuthn / FIDO2 (Passkeys) — phishing-resistant                 │   │
│  │ • Magic Links / Email OTP — low friction                            │   │
│  │ • SMS OTP — deprecated (SIM swap risk)                             │   │
│  │ • Push Notification (Duo, Auth0 Guardian)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Implementation)

```csharp
// JWT WITH REFRESH TOKEN ROTATION
public class TokenService
{
    private readonly JwtOptions _options;
    private readonly IRefreshTokenStore _store;
    
    public TokenPair GenerateTokens(User user, IEnumerable<Claim> claims)
    {
        var accessToken = GenerateAccessToken(user, claims);
        var refreshToken = GenerateRefreshToken();
        
        _store.StoreAsync(user.Id, refreshToken, _options.RefreshTokenExpiry);
        
        return new TokenPair(accessToken, refreshToken);
    }
    
    public async Task<TokenPair> RefreshAsync(string refreshToken)
    {
        var stored = await _store.GetAsync(refreshToken);
        if (stored == null || stored.ExpiresAt < DateTime.UtcNow || stored.Revoked)
            throw new SecurityTokenException("Invalid refresh token");
        
        // REUSE DETECTION: If token already used, revoke entire chain
        if (stored.UsedAt.HasValue)
        {
            await _store.RevokeChainAsync(stored.UserId);
            throw new SecurityTokenException("Token reuse detected");
        }
        
        await _store.MarkUsedAsync(refreshToken);
        
        var user = await _userRepo.GetAsync(stored.UserId);
        return GenerateTokens(user, user.Claims);
    }
}

// OAUTH 2.0 AUTHORIZATION CODE + PKCE (SPA)
public class AuthController : ControllerBase
{
    [HttpGet("login")]
    public IActionResult Login(string redirectUri)
    {
        var codeVerifier = Crypto.GenerateCodeVerifier();
        var codeChallenge = Crypto.GenerateCodeChallenge(codeVerifier);
        
        HttpContext.Session.SetString("pkce_verifier", codeVerifier);
        HttpContext.Session.SetString("redirect_uri", redirectUri);
        
        var authUrl = $"{_options.AuthorizationEndpoint}?" +
            $"client_id={_options.ClientId}&" +
            $"redirect_uri={_options.RedirectUri}&" +
            $"response_type=code&" +
            $"scope=openid profile email&" +
            $"code_challenge={codeChallenge}&" +
            $"code_challenge_method=S256&" +
            $"state={Crypto.GenerateState()}";
        
        return Redirect(authUrl);
    }
    
    [HttpGet("callback")]
    public async Task<IActionResult> Callback(string code, string state)
    {
        var codeVerifier = HttpContext.Session.GetString("pkce_verifier");
        
        var tokens = await _tokenClient.RequestAuthorizationCodeTokenAsync(
            new AuthorizationCodeTokenRequest
            {
                Code = code,
                CodeVerifier = codeVerifier,
                RedirectUri = _options.RedirectUri
            });
        
        // Store tokens, create session, redirect
        return Redirect(HttpContext.Session.GetString("redirect_uri") ?? "/");
    }
}
```

---

## 16. LAYERED ARCHITECTURE

### WHY? (Problem Statement)

**Separation of concerns** — each layer has single responsibility, testable in isolation.

---

### HOW? (Classic vs Clean)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LAYERED ARCHITECTURE COMPARISON                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TRADITIONAL N-TIER (Coupled Downward)                                     │
│  ─────────────────────────────────────                                     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PRESENTATION (Controllers, Views)                                   │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ BUSINESS LOGIC (Services, Managers)                                 │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ DATA ACCESS (Repositories, EF Core, ADO.NET)                       │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ DATABASE (SQL Server, PostgreSQL)                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PROBLEMS:                                                                 │
│  • Business logic depends on data access (hard to test)                   │
│  • Database schema drives domain model                                     │
│  • Hard to swap persistence / add new delivery mechanisms                 │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CLEAN / ONION / HEXAGONAL (Dependency Inversion)                          │
│  ─────────────────────────────────────────────────                         │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PRESENTATION (Controllers, gRPC, SignalR, Blazor)                  │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            │                                                 │
│  ┌─────────────────────────┴───────────────────────────────────────────┐   │
│  │ APPLICATION (Use Cases, Commands/Queries, DTOs, Validators)        │   │
│  │ Implements: IUnitOfWork, IEmailService, INotificationService       │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            │                                                 │
│  ┌─────────────────────────┴───────────────────────────────────────────┐   │
│  │ DOMAIN (Entities, VOs, Aggregates, Domain Events, Domain Services,  │   │
│  │         Specifications, Repository Interfaces)                      │   │
│  │ ZERO EXTERNAL DEPENDENCIES                                           │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            │                                                 │
│  ┌─────────────────────────┴───────────────────────────────────────────┐   │
│  │ INFRASTRUCTURE (EF Core, Redis, HTTP Clients, Identity, Jobs,       │   │
│  │              Message Bus, Logging, File Storage)                    │   │
│  │ Implements: Domain Repository Interfaces                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEPENDENCY RULE: Inner layers know NOTHING about outer layers             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 17. REACTIVE PROGRAMMING

### WHY? (Problem Statement)

**Async/await isn't enough** for streams, backpressure, composition of async sequences.

---

### HOW? (Reactive Streams)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REACTIVE PROGRAMMING                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FOUR PRINCIPLES (Reactive Manifesto):                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ RESPONSIVE   │ RESILIENT   │ ELASTIC   │ MESSAGE-DRIVEN              │   │
│  │ ─────────    │ ─────────   │ ───────   │ ────────────               │   │
│  │ Low latency  │ Failure     │ Scale up/ │ Async, non-blocking         │   │
│  │ consistent   │ isolation   │ down      │ backpressure                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  REACTIVE STREAMS SPEC (Publisher/Subscriber):                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Publisher → Subscribe(Subscriber) → Subscription                    │   │
│  │                                                                     │   │
│  │ Subscriber Methods:                                                 │   │
│  │ • onSubscribe(Subscription) — receives subscription                │   │
│  │ • onNext(T) — receives element                                     │   │
│  │ • onError(Throwable) — terminal error                              │   │
│  │ • onComplete() — terminal success                                  │   │
│  │                                                                     │   │
│  │ Subscription Methods:                                              │   │
│  │ • request(long n) — demand signaling (BACKPRESSURE)               │   │
│  │ • cancel() — cancel subscription                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BACKPRESSURE = Consumer controls rate:                                    │
│  Producer ──request(10)──► [buffer] ──onNext(x10)──► Consumer             │
│                            ▲                                                │
│                            │ request(5)                                     │
│                            │                                                │
│  When buffer full → slow down or drop (strategy configurable)             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Libraries & Patterns)

```csharp
// SYSTEM.REACTIVE (Rx.NET)
public class OrderEventProcessor
{
    private readonly IObservable<OrderEvent> _eventStream;
    
    public OrderEventProcessor(IObservable<OrderEvent> eventStream)
    {
        _eventStream = eventStream;
    }
    
    public IDisposable SubscribeToHighValueOrders(
        Action<OrderEvent> onNext, 
        Action<Exception> onError)
    {
        return _eventStream
            .Where(e => e.OrderTotal > 1000)
            .Buffer(TimeSpan.FromSeconds(5), 100) // Batch
            .SelectMany(batch => ProcessBatch(batch)) // FlatMap
            .Retry(3) // Retry on failure
            .Catch<Exception, OrderEvent>(ex => Observable.Empty<OrderEvent>())
            .Subscribe(onNext, onError);
    }
    
    private async Task<IEnumerable<OrderEvent>> ProcessBatch(
        IList<OrderEvent> batch)
    {
        // Process batch...
        return await Task.FromResult(batch.AsEnumerable());
    }
}

// REACTOR (Java/Project Reactor) — Similar concepts
// Flux<T> (0..N) vs Mono<T> (0..1)
// Operators: map, flatMap, filter, zip, combineLatest, window, buffer
```

---

## 18. DISTRIBUTED SYSTEMS

### WHY? (Problem Statement)

**Network is unreliable, latency exists, clocks drift, partitions happen.**

---

### HOW? (Fallacies & Patterns)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FALLACIES OF DISTRIBUTED COMPUTING                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. The network is reliable              → It fails, partitions, drops     │
│  2. Latency is zero                      → 0.5ms (DC) to 100ms (cross-region)│
│  3. Bandwidth is infinite                → Saturates, costs money          │
│  4. The network is secure                → Encrypt, authenticate, authorize│
│  5. Topology doesn't change              → Containers move, DNS changes    │
│  6. There is one administrator           → Multiple teams, orgs            │
│  7. Transport cost is zero               → Serialization, compression      │
│  8. The network is homogeneous           → Multiple protocols, versions    │
│                                                                             │
│  DESIGN FOR FAILURE:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Timeouts on EVERY call (configurable, per dependency)            │   │
│  │ • Retries with exponential backoff + jitter                        │   │
│  │ • Circuit breakers (fail fast, prevent cascade)                    │   │
│  │ • Bulkheads (isolate resource pools)                               │   │
│  │ • Idempotency keys (safe retries)                                  │   │
│  │ • Graceful degradation (fallback, cached data, read-only mode)     │   │
│  │ • Observability (logs, metrics, traces, correlation IDs)           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Patterns)

| Pattern | Purpose | Implementation |
|---------|---------|----------------|
| **Timeout** | Bound wait time | `HttpClient.Timeout`, `Polly.TimeoutAsync` |
| **Retry** | Transient failures | `Polly.RetryAsync` (exp backoff + jitter) |
| **Circuit Breaker** | Stop cascade | `Polly.CircuitBreakerAsync` |
| **Bulkhead** | Resource isolation | `Polly.BulkheadAsync`, semaphores |
| **Idempotency** | Safe retries | Client-generated key, server dedup |
| **Saga** | Distributed transactions | Choreography or Orchestration |
| **Outbox** | Reliable events | DB table + relay |
| **CQRS** | Read/write scaling | Separate models |
| **Event Sourcing** | Audit, replay | Append-only event store |
| **Service Mesh** | Infra concerns | Istio/Linkerd (mTLS, retry, observability) |

---

## 19. FUNCTIONAL PROGRAMMING

### WHY? (Problem Statement)

**Immutability + Pure functions = easier reasoning, testing, concurrency.**

---

### HOW? (Core Concepts)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FUNCTIONAL PROGRAMMING CORE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PURE FUNCTIONS:                                                            │
│  ─────────────                                                              │
│  • Same input → Same output (deterministic)                                │
│  • No side effects (no I/O, no mutation, no global state)                 │
│  • Referential transparency (replace call with result)                     │
│                                                                             │
│  IMMUTABILITY:                                                              │
│  ────────────                                                              │
│  • Data never changes after creation                                       │
│  • "Update" = create new with changes                                      │
│  • Structural sharing for efficiency                                       │
│  • Thread-safe by default                                                  │
│                                                                             │
│  HIGHER-ORDER FUNCTIONS:                                                    │
│  ───────────────────────                                                    │
│  • Functions as arguments (callbacks, strategies)                          │
│  • Functions as return values (factories, decorators)                      │
│  • Composition: f(g(x)) = (f ∘ g)(x)                                       │
│                                                                             │
│  ALGEBRAIC DATA TYPES:                                                      │
│  ───────────────────────                                                    │
│  • Sum Types (Discriminated Unions): type Shape = Circle | Rectangle      │
│  • Product Types (Records/Tuples): type Point = (x: float, y: float)      │
│  • Pattern Matching: exhaustive, compiler-checked                          │
│                                                                             │
│  MONADS (Computational Contexts):                                           │
│  ────────────────────────────────                                           │
│  • Option/Maybe — absence of value                                         │
│  • Result/Either — success or error                                        │
│  • Task/Future — async computation                                         │
│  • List/Sequence — multiple values                                         │
│  • Bind/Map: chain operations preserving context                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (C# Functional Patterns)

```csharp
// RESULT MONAD (Error handling without exceptions)
public abstract record Result<T>
{
    public sealed record Success<T>(T Value) : Result<T>;
    public sealed record Failure<T>(Error Error) : Result<T>;
    
    public static implicit operator Result<T>(T value) => new Success<T>(value);
    public static implicit operator Result<T>(Error error) => new Failure<T>(error);
    
    public Result<TOut> Map<TOut>(Func<T, TOut> f) =>
        this is Success<T> s ? new Success<TOut>(f(s.Value)) : (Result<TOut>)((Failure<T>)this).Error;
    
    public Result<TOut> Bind<TOut>(Func<T, Result<TOut>> f) =>
        this is Success<T> s ? f(s.Value) : (Result<TOut>)((Failure<T>)this).Error;
    
    public T Match<T>(Func<T, T> onSuccess, Func<Error, T> onFailure) =>
        this is Success<T> s ? onSuccess(s.Value) : onFailure(((Failure<T>)this).Error);
}

// OPTION MONAD (Null replacement)
public abstract record Option<T>
{
    public sealed record Some<T>(T Value) : Option<T>;
    public sealed record None<T> : Option<T>;
    
    public static implicit operator Option<T>(T? value) => value is null ? new None<T>() : new Some<T>(value);
    
    public Option<TOut> Map<TOut>(Func<T, TOut> f) =>
        this is Some<T> s ? new Some<TOut>(f(s.Value)) : new None<TOut>();
    
    public Option<TOut> Bind<TOut>(Func<T, Option<TOut>> f) =>
        this is Some<T> s ? f(s.Value) : new None<TOut>();
    
    public T GetOrElse(T fallback) => this is Some<T> s ? s.Value : fallback;
}

// DISCRIMINATED UNION (Pattern matching)
public abstract record PaymentMethod
{
    public sealed record CreditCard(string Token, string Last4, DateTime Expiry) : PaymentMethod;
    public sealed record BankAccount(string AccountId, string Routing, string Last4) : PaymentMethod;
    public sealed record DigitalWallet(string Provider, string Token) : PaymentMethod;
    public sealed record Cash : PaymentMethod;
}

public Money CalculateFee(PaymentMethod method) => method switch
{
    CreditCard c => Money.Percent(2.9, c.Amount) + Money.Fixed(0.30m),
    BankAccount b => Money.Fixed(0.50m),
    DigitalWallet d => Money.Percent(1.5, d.Amount),
    Cash => Money.Zero,
    _ => throw new ArgumentOutOfRangeException() // Exhaustiveness checked!
};

// FUNCTION COMPOSITION
public static class FuncExtensions
{
    public static Func<TIn, TOut> Compose<TIn, TMid, TOut>(
        this Func<TMid, TOut> f, Func<TIn, TMid> g) => x => f(g(x));
    
    public static Func<T, T> Pipe<T>(this T value, params Func<T, T>[] funcs) =>
        funcs.Aggregate(value, (v, f) => f(v));
}

// USAGE
var processOrder = 
    ValidateOrder
    .Compose(CalculatePricing)
    .Compose(ReserveInventory)
    .Compose(ProcessPayment)
    .Compose(EmitEvents);
```

---

## 20. SOA (SERVICE-ORIENTED ARCHITECTURE)

### WHY? (Problem Statement)

**Predecessor to microservices** — coarse-grained, enterprise-focused, ESB-centric.

---

### HOW? (SOA vs Microservices)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SOA vs MICROSERVICES                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SOA (Traditional):                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Coarse-grained services (Enterprise Services)                     │   │
│  │ • ESB (Enterprise Service Bus) — central orchestration             │   │
│  │ • Shared data model / Canonical Data Model                         │   │
│  │ • SOAP / WS-* standards                                            │   │
│  │ • Governance-heavy, centralized                                    │   │
│  │ • Smart endpoints, dumb pipes (ESB does routing, transformation)  │   │
│  │ • Examples: Banking core, Telco OSS/BSS                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MICROSERVICES (Evolution):                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Fine-grained services (Bounded Contexts)                         │   │
│  │ • Decentralized (No ESB) — Smart endpoints, dumb pipes (HTTP/gRPC) │   │
│  │ • Database per service                                             │   │
│  │ • REST / gRPC / Async messaging                                    │   │
│  │ • DevOps, CI/CD, Independent deployability                        │   │
│  │ • Team autonomy (You build it, you run it)                         │   │
│  │ • Examples: Netflix, Amazon, Uber                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  WHEN SOA STILL MAKES SENSE:                                                │
│  • Legacy integration (mainframe, ERP)                                     │
• Enterprise-wide canonical model required                                  │
• Heavy governance/compliance (healthcare, finance)                        │
• Vendor platforms (Oracle SOA Suite, IBM WebSphere, MuleSoft)             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 21. WORKING WITH DATA

### WHY? (Problem Statement)

**Data is the hardest part of distributed systems** — gravity, consistency, evolution.

---

### HOW? (Data Architecture Patterns)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATA ARCHITECTURE PATTERNS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DATABASE PER SERVICE (Microservices):                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Order DB (PostgreSQL)  Payment DB (PostgreSQL)  Inventory (Mongo)  │   │
│  │                                                                     │   │
│  │ Cross-service reads → API Composition / CQRS Read Models           │   │
│  │ Cross-service writes → Saga                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  POLYGLOT PERSISTENCE:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Relational (PostgreSQL) — ACID, complex queries, transactions    │   │
│  │ • Document (MongoDB) — flexible schema, hierarchical data          │   │
│  │ • Key-Value (Redis, DynamoDB) — caching, session, high throughput  │   │
│  │ • Column-Family (Cassandra) — time-series, wide rows, high write   │   │
│  │ • Graph (Neo4j) — relationships, traversals                        │   │
│  │ • Time-Series (InfluxDB, TimescaleDB) — metrics, IoT              │   │
│  │ • Search (Elasticsearch) — full-text, logs                         │   │
│  │ • Blob/Object (S3, Azure Blob) — files, images, videos             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DATA INTEGRATION:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • CDC (Debezium) — Capture DB changes → Kafka                      │   │
│  │ • Event Sourcing — Append-only events → Projections                │   │
│  │ • Dual Write Prevention — Outbox Pattern                           │   │
│  │ • Data Mesh — Domain-owned data products, self-serve               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 22. ETL / DATA WAREHOUSES

### WHY? (Problem Statement)

**Operational DBs ≠ Analytical DBs.** OLTP (transactions) vs OLAP (analytics).

---

### HOW? (Modern Data Stack)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MODERN DATA ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SOURCES → INGESTION → STORAGE → TRANSFORM → SERVE                         │
│                                                                             │
│  1. SOURCES:                                                                │
│     • Operational DBs (CDC via Debezium)                                   │
│     • Event Streams (Kafka, Kinesis)                                       │
│     • SaaS APIs (Salesforce, Stripe, HubSpot via Fivetran/Airbyte)         │
│     • Logs/Metrics (OpenTelemetry, Fluent Bit)                             │
│     • Files (S3, Azure Blob, SFTP)                                         │
│                                                                             │
│  2. INGESTION (ELT preferred over ETL):                                    │
│     • Batch: Airbyte, Fivetran, Stitch (scheduled)                         │
│     • Streaming: Kafka Connect, Flink, Spark Streaming                     │
│     • CDC: Debezium → Kafka → Iceberg/Delta Lake                           │
│                                                                             │
│  3. STORAGE (Data Lakehouse):                                              │
│     • Raw/Bronze: Immutable landing zone (Parquet, Iceberg, Delta)         │
│     • Curated/Silver: Cleaned, typed, partitioned                          │
│     • Business/Gold: Aggregated, modeled (Star/Snowflake schema)           │
│     • Formats: Apache Iceberg, Delta Lake, Apache Hudi (ACID on lake)      │
│     • Engines: Snowflake, BigQuery, Databricks, Redshift, ClickHouse       │
│                                                                             │
│  4. TRANSFORMATION:                                                         │
│     • dbt (SQL-based, version-controlled, tested, documented)              │
│     • Spark/Flink (complex, ML, streaming)                                 │
│     • Python (pandas, Polars) for data science                             │
│                                                                             │
│  5. SERVE:                                                                  │
│     • BI: Tableau, Power BI, Looker, Metabase, Superset                    │
│     • Reverse ETL: Census, Hightouch (warehouse → SaaS)                    │
│     • Data Apps: Streamlit, Evidence, internal tools                       │
│     • APIs: Hasura, PostgREST, GraphQL on warehouse                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Key Technologies)

| Category | Tools |
|----------|-------|
| **Ingestion (Batch)** | Airbyte, Fivetran, Stitch, Meltano |
| **Ingestion (Stream)** | Kafka Connect, Flink, Spark Streaming, RisingWave |
| **CDC** | Debezium, Maxwell, PGLogical |
| **Storage** | S3 + Iceberg/Delta/Hudi, Snowflake, BigQuery, Redshift, ClickHouse |
| **Transform** | dbt (SQL), Spark/PySpark, Flink SQL |
| **Orchestration** | Airflow, Dagster, Prefect, Temporal |
| **Quality** | Great Expectations, Soda, dbt tests |
| **Catalog** | DataHub, Amundsen, Atlas, Unity Catalog |
| **BI** | Tableau, Power BI, Looker, Metabase, Superset, Evidence |
| **Reverse ETL** | Census, Hightouch, RudderStack |

---

## 23. SQL DATABASES

### WHY? (Problem Statement)

**ACID, relational model, mature tooling** — default choice for most services.

---

### HOW? (PostgreSQL Best Practices)

```sql
-- SCHEMA DESIGN
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    status order_status NOT NULL DEFAULT 'draft',
    total_amount NUMERIC(12,2) NOT NULL CHECK (total_amount >= 0),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    shipping_address JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at TIMESTAMPTZ,
    shipped_at TIMESTAMPTZ
);

-- INDEXES FOR QUERY PATTERNS
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_orders_shipping_zip ON orders((shipping_address->>'postalCode'));

-- PARTITIONING (for large tables)
CREATE TABLE orders_2024 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- OPTIMISTIC LOCKING
ALTER TABLE orders ADD COLUMN version BIGINT NOT NULL DEFAULT 1;
-- UPDATE orders SET ..., version = version + 1 WHERE id = ? AND version = ?

-- READ REPLICA QUERY ROUTING (application level)
-- SELECT * FROM orders WHERE ... → replica
-- INSERT/UPDATE/DELETE → primary

-- CONNECTION POOLING (PgBouncer)
-- max_client_conn = 10000, default_pool_size = 25 per CPU
```

---

## 24. NOSQL DATABASES

### WHEN TO USE

| Database | Data Model | Use Case |
|----------|------------|----------|
| **MongoDB** | Document | Flexible schema, hierarchical, rapid iteration |
| **DynamoDB** | Key-Value + Document | Massive scale, single-digit ms, serverless |
| **Cassandra** | Wide Column | Time-series, high write throughput, multi-region |
| **Redis** | In-Memory (Strings, Hash, Set, Sorted Set) | Caching, sessions, rate limiting, pub/sub |
| **Elasticsearch** | Inverted Index | Full-text search, log analytics, observability |
| **Neo4j** | Graph | Relationships, fraud detection, recommendations |
| **InfluxDB/Timescale** | Time-Series | Metrics, IoT, monitoring |

---

### DECISION TREE

```
Need complex joins / transactions / reporting?
    │Yes → PostgreSQL (SQL)
    │No
    ▼
Document/key-value & huge write volume?
    │Yes → DynamoDB / Cassandra
    │No
    ▼
Flexible schema, rapid iteration?
    │Yes → MongoDB
    │No
    ▼
Caching / sessions / rate limiting / pub-sub?
    │Yes → Redis
    │No
    ▼
Full-text search / log analytics?
    │Yes → Elasticsearch / OpenSearch
    │No
    ▼
Relationships / graph traversals?
    │Yes → Neo4j
    │No
    ▼
Time-series / metrics / IoT?
    │Yes → TimescaleDB / InfluxDB
    │No
    ▼
Default → PostgreSQL
```

---

## 25. SPA / SSR / SSG (FRONTEND RENDERING)

### HOW? (Comparison)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RENDERING STRATEGIES                                     │
├──────────────────┬──────────────────┬──────────────────┬───────────────────┤
│      SPA         │      SSR         │      SSG         │       ISR         │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│                  │                  │                  │                   │
│ Client renders   │ Server renders   │ Build-time       │ Stale-while-      │
│ all HTML/JS      │ HTML per request │ generates static │ revalidate        │
│                  │                  │ HTML files       │ (Next.js)         │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ SEO: Poor        │ SEO: Excellent   │ SEO: Excellent   │ SEO: Excellent    │
│ TTFB: Fast       │ TTFB: Slower     │ TTFB: Fastest    │ TTFB: Fast        │
│ FCP: Slower      │ FCP: Fast        │ FCP: Fastest     │ FCP: Fast         │
│ Interactivity:   │ Interactivity:   │ Interactivity:   │ Interactivity:    │
│ After JS loads   │ After hydration  │ After hydration  │ After hydration   │
├──────────────────┼──────────────────┼──────────────────┼───────────────────┤
│ Use: Dashboards, │ Use: Marketing,  │ Use: Blogs, Docs,│ Use: E-commerce,  │
│ Admin, Apps      │ E-commerce, SEO  │ Marketing        │ Dynamic content   │
│                  │ Critical         │                  │                   │
└──────────────────┴──────────────────┴──────────────────┴───────────────────┘
```

---

## 26. WEB / MOBILE / APIs

### API STYLES

| Style | Protocol | Schema | Best For |
|-------|----------|--------|----------|
| **REST** | HTTP/1.1, HTTP/2 | OpenAPI | Public APIs, CRUD, caching |
| **GraphQL** | HTTP/1.1 | SDL | Flexible queries, federation |
| **gRPC** | HTTP/2 | Protobuf | Internal services, streaming |
| **tRPC** | HTTP/1.1 | TypeScript | Type-safe full-stack |
| **WebSocket** | WS/WSS | Custom | Real-time (chat, gaming) |
| **Server-Sent Events** | HTTP | Custom | Server→Client streaming |
| **AsyncAPI** | Kafka/AMQP/MQTT | AsyncAPI | Event-driven |

---

## 27. MICROFRONTENDS

### WHY? (Problem Statement)

**Frontend monolith** = deployment coupling, team friction, tech lock-in.

---

### HOW? (Integration Patterns)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MICROFRONTEND INTEGRATION                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BUILD-TIME (Compile-time):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Module Federation (Webpack 5, Vite, Rsbuild)                     │   │
│  │ • Shared dependencies (React, design system)                       │   │
│  │ • Single bundle, independent deploy via version pinning            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RUN-TIME (Browser):                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Web Components (Custom Elements) — Framework agnostic            │   │
│  │ • Iframes — Strong isolation, SEO/accessibility challenges         │   │
│  │ • Module Federation (Runtime) — Dynamic remoteEntry.js            │   │
│  │ • Single-SPA — Framework-agnostic orchestrator                    │   │
│  │ • Import Maps — Native ES modules, no bundler                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SERVER-SIDE:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Edge-side Includes (ESI) — CDN assembles fragments               │   │
│  │ • Tailor / OpenComponents — Server-side composition                │   │
│  │ • Next.js Multi-Zones / Remix Routes                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  STATE MANAGEMENT:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Shared store (Redux, Zustand) via Module Federation              │   │
│  │ • Custom Events / BroadcastChannel — Decoupled communication      │   │
│  │ • URL as source of truth (router)                                  │   │
│  │ • Backend-for-Frontend (BFF) per microfrontend                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 28. ANALYTICS

### EVENT TRACKING ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ANALYTICS PIPELINE                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CLIENT (Web/Mobile)                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Event SDK (Segment, RudderStack, Snowplow, custom)               │   │
│  │ • Auto-track (page views, clicks) + Custom events                  │   │
│  │ • Identity management (anonymousId → userId on login)             │   │
│  │ • Batching, retry, offline queue                                   │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            │                                                 │
│                            ▼                                                 │
│  COLLECTION (Edge/CDN)                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Edge Workers (Cloudflare, Vercel, Netlify) — enrich, filter     │   │
│  │ • API Gateway — auth, rate limit, schema validation               │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            │                                                 │
│                            ▼                                                 │
│  PROCESSING                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Stream: Kafka → Flink/RisingWave → Real-time dashboards         │   │
│  │ • Batch: S3 → dbt → Warehouse (Snowflake/BigQuery/Redshift)       │   │
│  │ • Enrichment: Join with user/product dims                          │   │
│  │ • Sessionization, Attribution, Funnel computation                  │   │
│  └─────────────────────────┬───────────────────────────────────────────┘   │
│                            │                                                 │
│                            ▼                                                 │
│  CONSUMPTION                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • BI: Tableau, Looker, Metabase, Superset                          │   │
│  │ • Product Analytics: Amplitude, Mixpanel, PostHog                 │   │
│  │ • Reverse ETL: Census, Hightouch → Braze, Salesforce, Braze       │   │
│  │ • ML Features: Feature Store (Feast, Tecton)                      │   │
│  │ • Alerting: Metric monitors (SLO burn rate, anomaly detection)    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 29. gRPC

### WHY? (Problem Statement)

**High-performance, contract-first, polyglot RPC** for internal services.

---

### HOW? (Protocol Buffers + gRPC)

```protobuf
// order.proto
syntax = "proto3";

package orders.v1;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
  rpc WatchOrders(WatchOrdersRequest) returns (stream OrderEvent);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  ShippingAddress shipping_address = 3;
  string idempotency_key = 4;
}

message CreateOrderResponse {
  string order_id = 1;
}

message Order {
  string id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  Money total = 5;
  google.protobuf.Timestamp created_at = 6;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  Money unit_price = 3;
}

message Money {
  int64 amount_cents = 1;
  string currency = 2;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_DRAFT = 1;
  ORDER_STATUS_SUBMITTED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_CANCELLED = 4;
}
```

```csharp
// SERVER IMPLEMENTATION (C#)
public class OrderServiceImpl : OrderService.OrderServiceBase
{
    private readonly IMediator _mediator;
    
    public override async Task<CreateOrderResponse> CreateOrder(
        CreateOrderRequest request, ServerCallContext context)
    {
        var command = new CreateOrderCommand(
            CustomerId.Parse(request.CustomerId),
            request.Items.Select(MapItem),
            MapAddress(request.ShippingAddress),
            request.IdempotencyKey);
        
        var result = await _mediator.Send(command);
        
        return new CreateOrderResponse { OrderId = result.Value.ToString() };
    }
    
    public override async Task<ListOrdersResponse> ListOrders(
        ListOrdersRequest request, ServerCallContext context)
    {
        var query = new GetOrdersQuery(
            CustomerId.Parse(request.CustomerId),
            request.PageSize, request.PageToken);
        
        var result = await _mediator.Send(query);
        
        return new ListOrdersResponse 
        { 
            Orders = { result.Items.Select(MapOrder) },
            NextPageToken = result.NextToken
        };
    }
}

// CLIENT USAGE
var channel = GrpcChannel.ForAddress("https://orders.internal");
var client = new OrderService.OrderServiceClient(channel);

var response = await client.CreateOrderAsync(new CreateOrderRequest
{
    CustomerId = "cust_123",
    Items = { new OrderItem { ProductId = "prod_456", Quantity = 2 } },
    ShippingAddress = new ShippingAddress { ... },
    IdempotencyKey = Guid.NewGuid().ToString()
});
```

---

### gRPC BEST PRACTICES

| Practice | Why |
|----------|-----|
| **Protobuf-first** | Contract before code, breaking change detection (buf) |
| **Version in package** | `orders.v1`, `orders.v2` — parallel deploy |
| **Unary for CRUD, Streaming for real-time** | Match pattern to use case |
| **Deadlines on every call** | Prevent cascade failure |
| **Interceptors for cross-cutting** | Auth, logging, metrics, retries |
| **Connection pooling** | Reuse HTTP/2 connections |
| **Load balancing** | Client-side (gRPC LB) or proxy (Envoy) |

---

## 30. W3C / WHATWG STANDARDS

### KEY STANDARDS FOR ARCHITECTS

| Standard | Body | Relevance |
|----------|------|-----------|
| **HTML Living Standard** | WHATWG | Semantic markup, forms, accessibility |
| **CSS Specifications** | W3C | Layout (Grid, Flexbox), Container Queries, Cascade Layers |
| **Web Components** | W3C | Custom Elements, Shadow DOM, HTML Templates |
| **Service Workers** | W3C | Offline, push, background sync |
| **Web Authentication (WebAuthn)** | W3C | Passwordless, FIDO2, Passkeys |
| **Payment Request API** | W3C | Native browser payment UI |
| **WebAssembly (Wasm)** | W3C | Near-native performance in browser |
| **HTTP/2, HTTP/3** | IETF/W3C | Multiplexing, header compression, QUIC |
| **Content Security Policy (CSP)** | W3C | XSS mitigation, resource control |
| **Permissions Policy** | W3C | Feature gating (camera, geolocation, etc.) |
| **Cross-Origin Resource Sharing (CORS)** | W3C | Secure cross-origin requests |
| **Fetch API / Streams API** | WHATWG | Modern network primitives |
| **WebRTC** | W3C/IETF | Real-time audio/video/data |
| **IndexedDB** | W3C | Client-side structured storage |
| **Web Workers / Worklets** | WHATWG | Off-main-thread computation |
| **Intersection Observer** | W3C | Lazy loading, infinite scroll |
| **Resize Observer** | W3C | Responsive components |
| **Declarative Shadow DOM** | W3C | SSR-friendly Web Components |

---

## PATTERNS & ANTI-PATTERNS SUMMARY

| Category | Pattern | Anti-Pattern |
|----------|---------|--------------|
| **Data** | CQRS, Event Sourcing, Outbox, Saga, CDC | Shared DB, Dual Write, Distributed 2PC |
| **Communication** | Async-first, gRPC, REST, GraphQL | Sync everything, Chatty services |
| **Resilience** | Timeout, Retry, Circuit Breaker, Bulkhead, Idempotency | No timeouts, Infinite retry, No fallback |
| **Consistency** | Eventual (default), Strong (when needed) | Distributed ACID, Global locks |
| **Deployment** | Independent, CI/CD, Feature Flags | Distributed Monolith, Coupled deploys |
| **Observability** | Logs, Metrics, Traces, Correlation IDs | Println debugging, No dashboards |
| **Security** | Zero Trust, mTLS, OAuth2/OIDC, Secrets Mgmt | Shared secrets, No authZ, Long-lived tokens |
| **Frontend** | Microfrontends, SSR/ISR, BFF | Monolith SPA, Direct API calls from UI |
| **Data Platform** | Lakehouse (Iceberg/Delta), dbt, Reverse ETL | Direct DB queries for analytics, Manual ETL |

---

## INTERVIEW QUESTIONS (Top 20)

### 1. CQRS: "When would you use CQRS and what are the trade-offs?"

**Answer:**
```
USE CQRS WHEN:
✓ Read/write patterns differ significantly (100:1 read:write)
✓ Complex domain with rich write model + optimized read models
✓ Need audit trail / temporal queries → Event Sourcing
✓ Different scaling needs (read replicas vs write primary)
✓ Multiple read shapes (search, dashboard, export, mobile)

TRADE-OFFS:
COST: Eventual consistency (read model lags write)
COST: More code (commands, handlers, projections, read models)
COST: Operational complexity (projection lag monitoring, replay)
COST: Schema evolution (event versioning, upcasters)

DON'T USE: Simple CRUD, low scale, strong consistency required everywhere
```

---

### 2. Eventual Consistency: "How do you handle 'read your writes' in eventual consistency?"

**Answer:**
```
STRATEGIES:
1. SYNCHRONOUS PROJECTION: Update read model in same transaction (loses scaling benefit)
2. READ FROM PRIMARY: Route writer's reads to write DB for short window
3. POLLING / WEBSOCKET: Client polls until read model reflects change
4. OPTIMISTIC UI: Show pending state immediately, reconcile on response
5. CQRS WITH SYNC PROJECTIONS: Critical reads sync, others async

IMPLEMENTATION (Option 2):
- After command succeeds, return version/timestamp
- Client includes version in subsequent query
- Read model checks: if version > projection version → fallback to primary
- TTL cache for "recently written" keys
```

---

### 3. Saga: "Choreography vs Orchestration — when to use which?"

**Answer:**
```
CHOREOGRAPHY (Event-driven):
✓ Simple flows (2-3 services)
✓ Services truly autonomous
✓ No central coordinator needed
✗ Implicit flow (hard to trace)
✗ Cyclic dependencies risk
✗ Compensation logic scattered

ORCHESTRATION (Central coordinator):
✓ Complex flows (4+ services, conditional steps)
✓ Explicit flow visibility
✓ Centralized compensation logic
✓ Easier testing/mocking
✗ Coordinator becomes smart (domain logic leak risk)
✗ Single point of failure (mitigate: stateless, replicated)

HYBRID: Orchestrator for complex core, choreography for notifications
```

---

### 4. Microservices: "How do you decompose a monolith?"

**Answer:**
```
1. STRANGLER FIG PATTERN:
   - Identify bounded context (DDD)
   - Create new service alongside monolith
   - Route new traffic to service (feature flag / API GW)
   - Migrate data incrementally (dual write → read new → delete old)
   - Remove from monolith
   - Repeat

2. DECOMPOSITION CRITERIA:
   - Business capability (not technical layer)
   - Team ownership (Conway's Law)
   - Data ownership (who is system of record?)
   - Transactional boundary (what MUST be ACID?)
   - Scaling profile (different load?)
   - Technology heterogeneity needed?

3. DATABASE DECOMPOSITION:
   - Shared DB → Separate schemas → Separate DBs
   - Use CDC (Debezium) for sync during transition
   - Saga for cross-service transactions
```

---

### 5. CAP/PACELC: "Explain PACELC and how it improves on CAP."

**Answer:**
```
CAP MISLEADS: "Pick 2 of 3" — but P (Partition Tolerance) is mandatory in distributed systems.

PACELC (Correct):
- If Partition (P) → Choose Availability (A) or Consistency (C)
- Else (E) → Choose Latency (L) or Consistency (C)

SYSTEMS:
- PC/EC: MongoDB, HBase, Redis (always consistent)
- PA/EL: Cassandra, DynamoDB (available during partition, low latency else)
- PA/EC: Rare
- PC/EL: Cosmos DB (strong consistency mode)

PRACTICAL: During partition, do you error (CP) or serve stale (AP)?
During normal ops, do you wait for quorum (EC) or return local (EL)?
```

---

### 6. Security: "How do you implement zero-trust in microservices?"

**Answer:**
```
ZERO TRUST PRINCIPLES:
1. VERIFY IDENTITY (every request)
   - mTLS via Service Mesh (Istio/Linkerd) — automatic cert rotation
   - SPIFFE/SPIRE for workload identity
   - JWT/OIDC for user-facing APIs

2. LEAST PRIVILEGE (every permission)
   - RBAC/ABAC via OPA/Gatekeeper
   - Service-to-service: scopes/claims
   - Human: Just-in-time access

3. ASSUME BREACH (design for compromise)
   - Network segmentation (micro-segmentation via CNI)
   - Encryption in transit (mTLS) + at rest (KMS)
   - Audit logging (immutable, SIEM)
   - Anomaly detection (ML on auth patterns)

IMPLEMENTATION:
- Service Mesh: mTLS, authZ, observability, resilience
- API Gateway: North-south auth, rate limit, WAF
- Secrets: Vault/Secrets Manager (no secrets in config)
- CI/CD: SLSA provenance, signed artifacts, policy gates
```

---

### 7. DDD: "Explain Aggregate Root and why it's the consistency boundary."

**Answer:**
```
AGGREGATE ROOT:
- Entity that controls access to its aggregate (cluster of entities/VOs)
- ONLY the root can be referenced externally
- All invariants enforced within aggregate boundary
- Single transaction per aggregate (DB row lock or optimistic lock)

EXAMPLE: Order (root) → OrderItems, ShippingAddress (internal)
- External code: order.Submit() not order.Items.Add()
- Order enforces: "Can't submit empty order", "Can't ship cancelled"
- Order raises domain events: OrderSubmitted, OrderShipped

WHY CONSISTENCY BOUNDARY:
- Within aggregate: ACID (single transaction)
- Between aggregates: Eventual consistency (domain events)
- Prevents: Anemic domain, scattered invariants, tight coupling

RULES:
1. Reference aggregates by ID only
2. One aggregate per transaction
3. Small aggregates (performance, contention)
4. Domain events for cross-aggregate consistency
```

---

### 8. Outbox Pattern: "Why is dual write a problem and how does outbox solve it?"

**Answer:**
```
DUAL WRITE PROBLEM:
1. App writes to DB (commit)
2. App publishes event to Kafka (fails — network, broker down)
3. Result: DB says "Order Created", but no event published
   → Downstream services never know → Inconsistency

OUTBOX PATTERN:
1. Single DB transaction:
   - Write domain data (Order)
   - Write event to OUTBOX table (same transaction)
2. Background relay (separate process):
   - Polls outbox table
   - Publishes to Kafka
   - Marks outbox record processed
3. Guarantees: At-least-once delivery (idempotent consumers handle dupes)

IMPLEMENTATION:
- Outbox table: id, type, payload, created_at, processed_at
- Relay: Poll every 100ms, batch 100, ack after publish
- Idempotency: Consumer dedupes by event ID
```

---

### 9. Caching: "Cache-aside vs Write-through vs Write-behind — when to use each?"

**Answer:**
```
CACHE-ASIDE (Lazy Loading):
- Miss → App loads from DB → Populates cache → Returns
- Pro: Simple, cache only hot data
- Con: 3 hops on miss, stale data window
- USE: Most reads, tolerate staleness (profiles, catalog)

READ-THROUGH:
- Miss → Cache loads from DB → Returns
- Pro: App unaware of cache
- Con: Same as cache-aside, more infrastructure
- USE: When cache library supports it

WRITE-THROUGH:
- Write → Cache + DB (sync) → Return
- Pro: Strong consistency, read always fresh
- Con: Write latency = cache + DB
- USE: Read-heavy, write-ok-latency (inventory, pricing)

WRITE-BEHIND (Write-Back):
- Write → Cache only → Async flush to DB
- Pro: Fast writes, absorbs spikes
- Con: Data loss on crash, complex
- USE: Write-heavy, can lose some (analytics, logs)

INVALIDATION IS HARD:
- TTL + Event-driven invalidation (on DB write → publish "invalidate key")
- Never rely on TTL alone for correctness-sensitive data
```

---

### 10. gRPC vs REST: "When would you choose gRPC over REST?"

**Answer:**
```
CHOOSE gRPC WHEN:
✓ Internal service-to-service (polyglot)
✓ High throughput, low latency required
✓ Streaming (server/client/bidirectional)
✓ Strong contract needed (protobuf, breaking change detection)
✓ Generated clients (type-safe, 10+ languages)
✓ HTTP/2 benefits (multiplexing, header compression)

CHOOSE REST WHEN:
✓ Public APIs (browser, mobile, partners)
✓ Caching critical (HTTP semantics, CDN)
✓ Simple CRUD, human-debuggable
✓ Firewall/proxy friendly (HTTP/1.1)
✓ Team lacks protobuf/gRPC expertise
✓ Need HATEOAS / hypermedia

HYBRID: gRPC internal, REST gateway (Envoy gRPC-JSON transcoder) for external
```

---

### 11. Serverless: "What are cold starts and how do you mitigate them?"

**Answer:**
```
COLD START = Function initialization latency (runtime load, dependencies, JIT)
- Typical: 50ms (Node/Python) to 2s (Java/.NET cold)
- Occurs: First invoke after idle, scale from zero, new version deploy

MITIGATIONS:
1. PROVISIONED CONCURRENCY (AWS Lambda) / MIN INSTANCES (Cloud Run)
   - Keep N warm instances always running
   - Cost: Pay for idle

2. SNAPSHOT / INIT OPTIMIZATION
   - AWS Lambda SnapStart (Java) — snapshot after init
   - .NET Native AOT — pre-compiled, no JIT
   - Go/Rust — fast by default

3. KEEP WARM (DIY)
   - Scheduled ping (EventBridge → Lambda every 5 min)
   - Cost: ~1M invocations/month = ~$0.20

4. ARCHITECTURAL
   - Move cold-path logic out of hot path
   - Lazy-load heavy dependencies
   - Use lighter runtime (Node.js, Go, Python vs Java/.NET)

5. ACCEPT FOR NON-CRITICAL PATHS
   - Async workers, batch jobs, webhooks — cold start OK
```

---

### 12. Distributed Tracing: "How do you implement distributed tracing?"

**Answer:**
```
OPENTELEMETRY (OTEL) STANDARD:

1. INSTRUMENTATION:
   - Auto-instrumentation (Java agent, .NET, Node, Go, Python)
   - Manual spans for business logic
   - Context propagation: W3C TraceContext (traceparent, tracestate headers)

2. SPAN STRUCTURE:
   - Trace ID (globally unique)
   - Span ID (unique per operation)
   - Parent Span ID (linkage)
   - Name, Kind (SERVER, CLIENT, PRODUCER, CONSUMER, INTERNAL)
   - Attributes (semantic conventions: http.method, db.statement, etc.)
   - Events (exceptions, logs)
   - Status (OK, ERROR)

3. COLLECTION:
   - OTEL Collector (vendor-neutral)
   - Receivers: OTLP, Jaeger, Zipkin, Prometheus
   - Processors: Batch, Memory Limiter, Tail Sampling
   - Exporters: Jaeger, Zipkin, Tempo, Datadog, Honeycomb, Elastic

4. SAMPLING (Control volume):
   - Head-based (random % at trace start) — simple, may miss errors
   - Tail-based (decide after trace complete) — keep errors, drop success
   - Adaptive (adjust rate based on throughput)

5. SEMANTIC CONVENTIONS:
   - Use standard attribute names (http.method, db.system, messaging.system)
   - Enables vendor-agnostic dashboards/alerts
```

---

### 13. API Gateway: "What does an API Gateway do and why not call services directly?"

**Answer:**
```
API GATEWAY RESPONSIBILITIES (North-South traffic):

1. ROUTING: Path/host/header → Service (with rewriting)
2. AUTHENTICATION: JWT validation, API keys, mTLS termination
3. AUTHORIZATION: RBAC/ABAC (OPA), scope validation
4. RATE LIMITING: Token bucket per client/IP/tenant
5. REQUEST VALIDATION: Schema (OpenAPI), size limits, sanitization
6. TRANSFORMATION: Request/response mapping, aggregation
7. OBSERVABILITY: Logging, metrics, tracing, correlation IDs
8. RESILIENCE: Circuit breaker, timeout, retry, fallback
9. SSL TERMINATION: Cert management, mTLS to services
10. CANNARY/BLUE-GREEN: Traffic splitting, mirroring

WHY NOT DIRECT:
- Clients couple to service topology (ports, hosts, versions)
- No central auth/rate limit/observability
- Can't evolve services without breaking clients
- Cross-cutting concerns duplicated everywhere

PATTERN: BACKEND FOR FRONTEND (BFF)
- Separate gateway per client type (Web, Mobile, Partner)
- Each BFF aggregates, transforms for its client
- Owned by frontend team
```

---

### 14. Event-Driven: "How do you guarantee exactly-once processing?"

**Answer:**
```
EXACTLY-ONCE IS IMPOSSIBLE IN DISTRIBUTED SYSTEMS.
→ Settle for AT-LEAST-ONCE + IDEMPOTENT CONSUMER.

ACHIEVING IDEMPOTENCY:
1. CLIENT GENERATES IDEMPOTENCY KEY (UUID per business operation)
2. SERVER STORES KEY + RESULT (atomic with business logic)
3. DUPLICATE REQUEST → RETURN STORED RESULT

IMPLEMENTATION:
- HTTP: Idempotency-Key header (Stripe pattern)
- Kafka: Producer sends key = idempotency key, enable.idempotence=true
- Consumer: Dedupe via Redis/DB (SETNX key with TTL)

IDEMPOTENCY KEY SCOPE:
- Per user + action (e.g., "user_123:create_order:uuid")
- TTL = business retry window (24-48 hours)

EXACTLY-ONCE DELIVERY (Kafka):
- enable.idempotence=true + acks=all + max.in.flight.requests=5
- Producer: PID + sequence number per partition
- Broker: Deduplicates by PID+seq
- Consumer: Still needs app-level idempotency!
```

---

### 15. Database Sharding: "How do you shard a database and what are the trade-offs?"

**Answer:**
```
SHARDING STRATEGIES:

1. HASH-BASED: shard = hash(key) % N
   ✓ Uniform distribution
   ✗ Resharding = rehash all (painful)
   ✗ No range queries

2. RANGE-BASED: shard = key range (A-M, N-Z)
   ✓ Efficient range queries
   ✗ Hot spots (new users → last shard)
   ✗ Rebalancing complex

3. DIRECTORY-BASED: Lookup table (key → shard)
   ✓ Flexible, easy resharding
   ✗ Extra lookup, SPOF (mitigate: cache + replica)

4. CONSISTENT HASHING: Ring + virtual nodes
   ✓ Minimal reshuffle on add/remove (1/N)
   ✓ Used by DynamoDB, Cassandra, Redis Cluster
   ✗ Slightly uneven, more complex

RESHARDING STRATEGIES:
1. OFFLINE: Stop writes, copy, switch (downtime)
2. ONLINE (Dual-Write): Write to old+new, backfill, verify, switch
3. SHADOW: Mirror reads to new, compare, switch (zero-downtime)

TRADE-OFFS:
- Gains: Horizontal scale, fault isolation, geo-distribution
- Costs: No cross-shard transactions, no JOINs, complex queries, operational burden
- ALTERNATIVE: Read replicas, vertical partitioning, CQRS first
```

---

### 16. Service Mesh: "When do you need a service mesh?"

**Answer:**
```
NEED SERVICE MESH WHEN:
✓ 20+ services with complex communication
✓ mTLS required (compliance, zero trust)
✓ Advanced traffic management (canary, mirroring, fault injection)
✓ Uniform observability (metrics, traces, access logs)
✓ Centralized resilience (retry, timeout, circuit breaker policy)
✓ Multi-cluster / hybrid cloud
✓ Team can't implement cross-cutting concerns consistently

DON'T NEED WHEN:
✗ < 10 services
✗ Simple communication (few synchronous calls)
✗ Team can implement libraries (Polly, OpenTelemetry) consistently
✗ Latency budget extremely tight (sidecar adds 1-2ms)
✗ No platform team to operate mesh

ALTERNATIVES:
- Sidecar-less (Cilium, Istio Ambient) — lower overhead
- Library-based (Polly, gRPC interceptors, OTEL SDKs)
- API Gateway + Library for east-west
```

---

### 17. Microfrontends: "How do you share state between microfrontends?"

**Answer:**
```
STATE SHARING STRATEGIES:

1. URL AS SOURCE OF TRUTH (Preferred)
   - Router owns state (filters, pagination, selected IDs)
   - Microfrontends read from URL, push to URL
   - Shareable, bookmarkable, browser history works

2. CUSTOM EVENTS / BROADCAST CHANNEL
   - window.dispatchEvent(new CustomEvent('cart:updated', {detail: {...}}))
   - BroadcastChannel API for cross-tab
   - Decoupled, no shared dependency

3. SHARED STORE (Module Federation)
   - Expose Redux/Zustand store from host/remote
   - Risk: Tight coupling, version conflicts
   - Use sparingly (auth, theme, cart)

4. BACKEND-FOR-FRONTEND (BFF)
   - Each microfrontend has dedicated BFF
   - BFF aggregates, transforms, owns contract
   - State lives in backend, frontend is thin

5. WEB COMPONENTS (Attributes/Properties)
   - <cart-widget user-id="123"></cart-widget>
   - Reactive via attributeChangedCallback
   - Framework-agnostic

ANTI-PATTERN: Shared Redux store across independently deployed MFEs — deployment coupling
```

---

### 18. Observability: "What are the three pillars and how do you use them?"

**Answer:**
```
THREE PILLARS (Cindy Sridharan):

1. METRICS (Aggregated, numeric, time-series)
   - RED: Rate, Errors, Duration (per service/endpoint)
   - USE: Saturation, Utilization, Errors (USE) for resources
   - SLI/SLO: Availability, Latency, Throughput
   - Tools: Prometheus, Grafana, Datadog
   - Alerting: Symptom-based (SLO burn rate), not cause-based

2. LOGS (Immutable, timestamped, structured)
   - JSON, correlation IDs (trace_id, span_id)
   - Levels: ERROR, WARN, INFO, DEBUG
   - Structured: {timestamp, level, message, trace_id, fields...}
   - Sampling: High-volume DEBUG → sample, ERROR → 100%
   - Tools: Loki, Elasticsearch, CloudWatch, Datadog

3. TRACES (Request flow across services)
   - W3C TraceContext (traceparent header)
   - Spans: Name, Kind, Start/End, Attributes, Events, Links
   - Sampling: Head-based (10%) + Tail-based (keep errors)
   - Tools: Jaeger, Zipkin, Tempo, X-Ray, Zipkin

UNIFIED WORKFLOW:
Dashboard (Metrics) → Alert fires → Click trace ID → View trace (spans) → Click span → View logs
```

---

### 19. Testing: "What's your testing strategy for microservices?"

**Answer:**
```
TEST PYRAMID (Microservices):

1. UNIT TESTS (70-80%)
   - Pure functions, domain logic, validators
   - Fast (<1ms), isolated, no external deps
   - Framework: xUnit, JUnit, Jest, pytest
   - Target: 80%+ coverage on domain

2. INTEGRATION TESTS (15-20%)
   - Service + real dependencies (DB, Kafka, Redis)
   - Testcontainers for ephemeral infra
   - Contract tests (Consumer-driven: Pact)
   - API tests (happy path + 2 error paths per endpoint)

3. CONTRACT TESTS (Critical for microservices)
   - Consumer defines expectations (Pact)
   - Provider verifies against contract
   - Runs in both consumer + provider CI
   - Prevents breaking changes

4. END-TO-END (5-10%)
   - Critical user journeys (happy path only)
   - Staging environment, real data (anonymized)
   - Playwright, Cypress, Selenium
   - Slow, flaky — minimize

5. CHAOS ENGINEERING (Periodic)
   - Kill pods, inject latency, fail DNS
   - Verify resilience, SLOs
   - Tools: Litmus, Chaos Mesh, Gremlin

CI/CD GATES:
- PR: Unit + Contract + Static Analysis
- Merge: Integration + Contract (provider)
- Deploy: Smoke tests in staging
- Post-deploy: Synthetic monitoring
```

---

### 20. Architecture Decision: "How do you document and communicate architectural decisions?"

**Answer:**
```
ARCHITECTURE DECISION RECORDS (ADRs):

FORMAT (Markdown in Git):
# ADR-0042: Use Kafka for Event Backbone

## Status: Accepted (2025-01-15)

## Context
Need reliable, ordered, replayable event transport for order fulfillment.
Requirements: 10K events/sec, 7-year retention, replay for new consumers.

## Decision
Apache Kafka (Confluent Cloud) with Avro schemas in Schema Registry.

## Consequences
Positive:
- Durability (replication factor 3, acks=all)
- Replayability (offset reset, new consumer groups)
- Ordering per partition (key = orderId)
- Schema evolution (BACKWARD compatibility)

Negative:
- Operational complexity (broker, ZooKeeper/KRaft, monitoring)
- Latency (~5ms p99 vs ~1ms HTTP)
- Team learning curve

## Alternatives Considered
1. RabbitMQ — Rejected: No replay, no ordering guarantee
2. AWS EventBridge — Rejected: Vendor lock-in, 14-day retention max
3. gRPC Streaming — Rejected: No durability, no replay

## Implementation Notes
- Topic naming: domain.entity.action (order.created)
- Partitions: 100 (10x brokers)
- Schema Registry: BACKWARD compatibility enforced in CI
- Monitoring: Consumer lag alert (lag > 10K for 5min)

## Related ADRs
- ADR-0012: Avro for Event Serialization
- ADR-0018: Outbox Pattern for Reliable Publishing
- ADR-0031: Correlation ID Standard
```

---

## QUICK REFERENCE: ARCHITECT'S CHEAT SHEET

### Decision Trees (Keep Handy)

**SQL vs NoSQL:**
```
Need complex joins / transactions / reporting? → SQL
    │No
Data is documents/key-value/time-series & huge write volume? → NoSQL
    │No
Strong consistency + relations? → SQL
    │No
Flexible schema, horizontal scale, eventual OK? → NoSQL
```

**Sync vs Async:**
```
Caller needs result NOW to proceed? → Sync (REST/gRPC)
    │No
Slow / retryable / spike-absorbing? → Async (Kafka/RabbitMQ)
```

**Consistency Level:**
```
Financial / cannot lose or show stale? → Strong (Single-leader, Quorum)
    │No
Social / feeds / can converge later? → Eventual + Conflict Resolution
    │No
Causality matters (comments before replies)? → Causal
```

**Build vs Buy:**
```
Commodity (auth, search, queue, email)? → BUY
Strategic differentiator (core algorithm, unique domain)? → BUILD
Hybrid: Build core, Buy commodity (e.g., Build recommendation, Buy Elasticsearch)
```

**Language Choice:**
```
Team expertise + Ecosystem + Cloud support + Performance needs
Default: Org's Technology Radar (Adopt/Trial/Assess/Hold)
```

---

### Architecture Review Checklist

```
☐ Requirements clear? (Functional + NFRs quantified)
☐ ADRs for all structural decisions?
☐ C4 Diagrams (Context, Container, Component)?
☐ Data flow diagram (with trust boundaries)?
☐ Threat model (STRIDE)?
☐ NFR targets (SLOs: latency, availability, throughput)?
☐ Failure modes identified? (What breaks? How detect? How recover?)
☐ Observability designed? (Logs, Metrics, Traces, Alerts)
☐ Security: AuthN, AuthZ, Encryption, Secrets, Audit?
☐ Deployment: CI/CD, Rollback, Canary, Feature Flags?
☐ Runbooks for top 5 failure scenarios?
☐ Team topology matches service boundaries? (Conway)
☐ ADRs linked in repo, reviewed quarterly?
```

---

*This document covers all topics from the roadmap.sh Software Architect roadmap with exam-style depth: WHY/HOW/WHAT, patterns, anti-patterns, code examples, decision frameworks, and interview Q&A.*