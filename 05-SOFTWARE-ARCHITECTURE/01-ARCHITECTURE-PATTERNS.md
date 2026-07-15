# SOFTWARE ARCHITECTURE PATTERNS
## Exam-Style Study Notes

---

## TOPIC: Architecture Patterns for Scalable Systems

### WHY? (Problem Statement)

**Monolith Problems at Scale:**
- **Deployment coupling**: Change one feature → deploy entire app
- **Team friction**: 50 developers merging to same codebase
- **Technology lock-in**: Can't adopt new tech for one module
- **Scaling**: Must scale entire app, not just bottleneck
- **Failure blast radius**: Bug in payments takes down catalog

**What Architecture Patterns Solve:**
| Problem | Pattern Solution |
|---------|------------------|
| Deployment independence | Microservices / Modular Monolith |
| Team autonomy | Bounded Contexts / Domain-Driven Design |
| Technology heterogeneity | Polyglot Microservices |
| Independent scaling | Microservices / CQRS |
| Fault isolation | Circuit Breaker / Bulkhead |
| Data consistency | Saga / Event Sourcing / CQRS |

**Real-World Analogy:**
- Monolith = One giant restaurant kitchen (all chefs, all stations, one order ticket)
- Microservices = Food court (independent stalls, shared seating, central ordering)

---

### HOW? (Internal Mechanism)

#### 1. **Architecture Evolution Path**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE SPECTRUM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Modular Monolith ──► Modulith ──► Microservices               │
│       │                  │                  │                   │
│       ▼                  ▼                  ▼                   │
│  Single deploy      Separate modules    Independent services   │
│  Shared DB          Shared DB           Own DB per service     │
│  In-process calls   In-process +        Network calls          │
│                     MediatR             (HTTP/gRPC/Message)    │
│                                                                 │
│  Complexity: Low          Medium              High              │
│  Team Size:  <10           10-50              50+               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. **Communication Patterns**

```
SYNCHRONOUS (Request-Response)          ASYNCHRONOUS (Event-Driven)
┌─────────────────────────────┐         ┌─────────────────────────────┐
│ Client ──HTTP/gRPC──► Service│         │ Service ──Event──► Broker  │
│         ◄───────────────────│         │         ▼                  │
│         Coupled, blocking   │         │    Consumers               │
│         Tight latency       │         │    (decoupled, eventual)   │
└─────────────────────────────┘         └─────────────────────────────┘

Use Sync for:                          Use Async for:
- Queries (need data now)              - Commands (fire-and-forget)
- User-facing operations               - Cross-service workflows
- ACID transactions                    - High throughput
- Simple request/response              - Loose coupling
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Domain-Driven Design (DDD) Building Blocks**

```csharp
// Entity - Identity + Mutability
public class Order : Entity<OrderId>
{
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    public void AddItem(ProductId productId, int quantity, Money price)
    {
        // Business logic in entity
        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null) existing.IncreaseQuantity(quantity);
        else _items.Add(new OrderItem(productId, quantity, price));
    }
}

// Value Object - Immutable, No Identity
public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);
    public Money Add(Money other) 
    {
        if (Currency != other.Currency) throw new InvalidOperationException();
        return new(Amount + other.Amount, Currency);
    }
}

// Aggregate Root - Consistency Boundary
public class Order : AggregateRoot<OrderId>
{
    // Only aggregate root can be referenced externally
    // All changes go through aggregate root methods
    public void Submit() 
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException();
        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmittedEvent(Id, CustomerId, Items));
    }
}

// Domain Event - Something that happened
public record OrderSubmittedEvent(OrderId OrderId, CustomerId CustomerId, 
    IReadOnlyCollection<OrderItem> Items) : IDomainEvent;

// Repository - Persistence Abstraction
public interface IOrderRepository
{
    Task<Order?> GetAsync(OrderId id);
    Task AddAsync(Order order);
    Task UpdateAsync(Order order);
}
```

#### 2. **Clean Architecture Layers**

```
┌─────────────────────────────────────────────────────────────────┐
│                        PRESENTATION                             │
│  (Controllers, Minimal APIs, Blazor, gRPC, SignalR)            │
├─────────────────────────────────────────────────────────────────┤
│                        APPLICATION                              │
│  (Use Cases, Commands/Queries, DTOs, Validators, Handlers)    │
│  Implements: IUnitOfWork, IEmailService, INotificationService  │
├─────────────────────────────────────────────────────────────────┤
│                        DOMAIN                                   │
│  (Entities, Value Objects, Aggregates, Domain Events,          │
│   Domain Services, Specifications, Repository Interfaces)      │
│  ZERO external dependencies                                     │
├─────────────────────────────────────────────────────────────────┤
│                        INFRASTRUCTURE                           │
│  (EF Core, Redis, HTTP Clients, File Storage,                 │
│   Identity, Background Jobs, Message Bus, Logging)             │
│  Implements: Domain Repository Interfaces                      │
└─────────────────────────────────────────────────────────────────┘

Dependency Rule: Inner layers know NOTHING about outer layers
```

#### 3. **CQRS (Command Query Responsibility Segregation)**

```csharp
// COMMAND SIDE (Write Model)
public record CreateOrderCommand(
    CustomerId CustomerId, 
    List<OrderItemDto> Items,
    ShippingAddressDto ShippingAddress
) : IRequest<Result<OrderId>>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IOrderRepository _repo;
    private readonly IUnitOfWork _uow;
    private readonly IDomainEventPublisher _events;
    
    public async Task<Result<OrderId>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items, cmd.ShippingAddress);
        await _repo.AddAsync(order);
        await _uow.SaveChangesAsync(ct);
        
        // Publish domain events
        await _events.PublishAsync(order.DomainEvents);
        
        return Result.Success(order.Id);
    }
}

// QUERY SIDE (Read Model - Optimized for reads)
public record GetOrderDtoQuery(OrderId Id) : IRequest<OrderDto?>;

public class GetOrderDtoHandler : IRequestHandler<GetOrderDtoQuery, OrderDto?>
{
    private readonly IReadOnlyOrderRepository _readRepo; // Different interface!
    
    public async Task<OrderDto?> Handle(GetOrderDtoQuery query, CancellationToken ct)
    {
        return await _readRepo.GetDtoAsync(query.Id);
    }
}

// Read Model (Denormalized, No Behavior)
public class OrderDto
{
    public OrderId Id { get; init; }
    public string CustomerName { get; init; }
    public OrderStatus Status { get; init; }
    public Money Total { get; init; }
    public List<OrderItemDto> Items { get; init; }
    public ShippingAddressDto ShippingAddress { get; init; }
    public DateTime CreatedAt { get; init; }
}
```

#### 4. **Event-Driven Architecture**

```csharp
// Domain Event Publisher
public interface IDomainEventPublisher
{
    Task PublishAsync(IEnumerable<IDomainEvent> events, CancellationToken ct);
}

// Outbox Pattern (Reliable Event Publishing)
public class OutboxEventPublisher : IDomainEventPublisher
{
    private readonly AppDbContext _db;
    private readonly IMessageBus _bus;
    
    public async Task PublishAsync(IEnumerable<IDomainEvent> events, CancellationToken ct)
    {
        foreach (var @event in events)
        {
            var outboxMessage = new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = @event.GetType().AssemblyQualifiedName!,
                Content = JsonSerializer.Serialize(@event),
                CreatedAt = DateTime.UtcNow
            };
            _db.OutboxMessages.Add(outboxMessage);
        }
        await _db.SaveChangesAsync(ct);
    }
}

// Background processor (separate process/thread)
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var bus = scope.ServiceProvider.GetRequiredService<IMessageBus>();
            
            var messages = await db.OutboxMessages
                .Where(m => !m.ProcessedAt.HasValue)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(stoppingToken);
            
            foreach (var msg in messages)
            {
                var @event = JsonSerializer.Deserialize(msg.Content, Type.GetType(msg.Type)!) as IDomainEvent;
                await bus.PublishAsync(@event);
                msg.ProcessedAt = DateTime.UtcNow;
            }
            
            await db.SaveChangesAsync(stoppingToken);
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

#### 5. **Saga Pattern (Distributed Transactions)**

```csharp
// Choreography-based (Events)
public class OrderSaga
{
    // Step 1: Create Order (Pending)
    // Event: OrderCreated → PaymentService.ReservePayment
    // Event: PaymentReserved → InventoryService.ReserveStock
    // Event: StockReserved → ShippingService.ScheduleShipment
    // Event: ShipmentScheduled → OrderConfirmed
    // Compensation: Any failure → Publish compensation events in reverse
}

// Orchestration-based (Central Coordinator)
public class OrderOrchestrator
{
    private readonly IMessageBus _bus;
    
    public async Task StartOrderAsync(CreateOrderCommand cmd)
    {
        var sagaId = Guid.NewGuid();
        var state = new OrderSagaState 
        { 
            SagaId = sagaId, 
            OrderId = cmd.OrderId,
            Step = SagaStep.PaymentPending
        };
        
        await _bus.Send(new ReservePaymentCommand(sagaId, cmd.OrderId, cmd.Total));
    }
    
    public async Task Handle(PaymentReservedEvent evt)
    {
        await _bus.Send(new ReserveStockCommand(evt.SagaId, evt.OrderId, evt.Items));
    }
    
    public async Task Handle(StockReservedEvent evt)
    {
        await _bus.Send(new ScheduleShipmentCommand(evt.SagaId, evt.OrderId, evt.Address));
    }
    
    public async Task Handle(PaymentFailedEvent evt)
    {
        // Compensation: Release stock, cancel order
        await _bus.Send(new ReleaseStockCommand(evt.SagaId, evt.OrderId));
        await _bus.Send(new CancelOrderCommand(evt.SagaId, evt.OrderId));
    }
}
```

#### 6. **API Gateway Pattern**

```csharp
// YARP (Yet Another Reverse Proxy) - .NET Native
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(transforms =>
    {
        transforms.AddRequestTransform(async ctx =>
        {
            // Add correlation ID
            ctx.ProxyRequest.Headers.Add("X-Correlation-ID", 
                ctx.HttpContext.TraceIdentifier);
        });
    });

// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "orders": {
        "ClusterId": "orders-cluster",
        "Match": { "Path": "api/orders/{**catch-all}" },
        "Transforms": [
          { "PathPattern": "api/orders/{**catch-all}" },
          { "RequestHeader": "X-User-ID", "Value": "{claim:sub}" }
        ],
        "AuthorizationPolicy": "Authenticated"
      }
    },
    "Clusters": {
      "orders-cluster": {
        "Destinations": {
          "orders1": { "Address": "http://orders-service:8080" }
        },
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Active": { "Enabled": true, "Interval": "00:00:10", "Path": "/health" }
        }
      }
    }
  }
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Modular Monolith** | Team <10, simple domain | Microservices too early | Distributed complexity without need |
| **DDD Bounded Contexts** | Complex domain, multiple teams | Anemic domain model | Business logic in services, not entities |
| **CQRS** | Read/write scaling different | CQRS everywhere | Complexity for simple CRUD |
| **Event Sourcing** | Audit trail, temporal queries | Event sourcing for all data | Complexity, projection rebuild |
| **Saga** | Long-running distributed transactions | 2PC (Two-Phase Commit) | Doesn't work across services |
| **API Gateway** | Multiple clients, auth, rate limit | Direct service-to-service | Coupling, no cross-cutting concerns |
| **Outbox Pattern** | Reliable event publishing | Direct publish in transaction | Dual write problem, message loss |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Monolith vs Microservices - when to choose which?"

**Answer:**
```
MONOLITH (Start here):
✓ Team < 10 developers
✓ Simple, well-understood domain
✓ Low scaling requirements
✓ Need fast iteration, simple deployment
✓ Transactional consistency important

MICROSERVICES (Evolve to):
✓ Team > 50, multiple independent teams
✓ Complex domain with clear bounded contexts
✓ Different scaling needs per service
✓ Technology heterogeneity required
✓ Independent deployment critical
✓ Fault isolation required

EVOLUTION PATH: Modular Monolith → Strangler Fig → Microservices
```

#### 2. **Design**: "How to split a monolith into microservices?"

```csharp
// STRANGLER FIG PATTERN
// 1. Identify bounded context (e.g., Payments)
// 2. Create new Payments service alongside monolith
// 3. Route new payments traffic to service
// 4. Migrate existing data (dual write → read from new → delete old)
// 5. Remove payments code from monolith
// 6. Repeat

// BOUNDED CONTEXT IDENTIFICATION
// - Ubiquitous Language differences
// - Team ownership boundaries
// - Data ownership (who owns Customer?)
// - Transactional boundaries (what must be ACID?)

// DATABASE DECOMPOSITION
// - Shared DB → Separate schemas → Separate DBs
// - Use CDC (Change Data Capture) for sync during transition
// - Saga for cross-service transactions
```

#### 3. **Code**: "Implement the Outbox Pattern for reliable events"

```csharp
// 1. Outbox Table
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; }
    public string Content { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public int RetryCount { get; set; }
    public string? Error { get; set; }
}

// 2. DbContext SaveChanges Override
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

// 3. Processor (with idempotency)
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
                    var @event = Deserialize(msg);
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

#### 4. **Architecture**: "Design a system for 'Place Order' with: Payment, Inventory, Shipping"

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────►│  API GW     │────►│   Order     │────►│  Payment    │
│             │     │  (YARP)     │     │  Service    │     │  Service    │
└─────────────┘     └─────────────┘     └──────┬──────┘     └──────┬──────┘
                                               │                 │
                                               ▼                 ▼
                                        ┌─────────────┐     ┌─────────────┐
                                        │  Inventory  │     │  Shipping   │
                                        │  Service    │     │  Service    │
                                        └─────────────┘     └─────────────┘
                                               │                 │
                                               └────────┬────────┘
                                                        ▼
                                               ┌─────────────────┐
                                               │  Message Bus    │
                                               │  (Kafka/Rabbit) │
                                               └────────┬────────┘
                                                        │
                    ┌────────────────────────────────────┼────────────────────────────────────┐
                    ▼                                    ▼                                    ▼
             ┌─────────────┐                     ┌─────────────┐                     ┌─────────────┐
             │ Notification│                     │  Analytics  │                     │  Audit Log  │
             │  Service    │                     │  Service    │                     │  Service    │
             └─────────────┘                     └─────────────┘                     └─────────────┘
```

#### 5. **Data Consistency**: "How to handle distributed transactions without 2PC?"

**Answer: SAGA Pattern**
```
CHOREOGRAPHY (Event-driven):
OrderService: Create Order (Pending) → Publish OrderCreated
PaymentService: On OrderCreated → Reserve Payment → Publish PaymentReserved
InventoryService: On PaymentReserved → Reserve Stock → Publish StockReserved
ShippingService: On StockReserved → Schedule → Publish ShipmentScheduled
OrderService: On ShipmentScheduled → Confirm Order

COMPENSATION (Reverse):
PaymentFailed → Publish PaymentFailed
InventoryService: On PaymentFailed → Release Stock
OrderService: On PaymentFailed → Cancel Order

ORCHESTRATION (Central Coordinator):
OrderOrchestrator coordinates steps, handles failures, executes compensations
```

#### 6. **Code**: "Implement Idempotency for API endpoints"

```csharp
// Idempotency Key Header: Idempotency-Key: <guid>
public class IdempotencyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IDistributedCache _cache;
    
    public async Task InvokeAsync(HttpContext context)
    {
        if (!HttpMethods.IsPost(context.Request.Method) && 
            !HttpMethods.IsPut(context.Request.Method) &&
            !HttpMethods.IsPatch(context.Request.Method))
        {
            await _next(context);
            return;
        }
        
        if (!context.Request.Headers.TryGetValue("Idempotency-Key", out var key))
        {
            await _next(context);
            return;
        }
        
        var cacheKey = $"idempotency:{key}";
        var cached = await _cache.GetStringAsync(cacheKey);
        
        if (cached != null)
        {
            var response = JsonSerializer.Deserialize<IdempotentResponse>(cached);
            context.Response.StatusCode = response.StatusCode;
            context.Response.ContentType = response.ContentType;
            await context.Response.WriteAsync(response.Body);
            return;
        }
        
        // Capture response
        var originalBody = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;
        
        await _next(context);
        
        memoryStream.Position = 0;
        var body = await new StreamReader(memoryStream).ReadToEndAsync();
        
        var response = new IdempotentResponse
        {
            StatusCode = context.Response.StatusCode,
            ContentType = context.Response.ContentType,
            Body = body
        };
        
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(response), 
            new DistributedCacheEntryOptions 
            { 
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) 
            });
        
        memoryStream.Position = 0;
        await memoryStream.CopyToAsync(originalBody);
    }
}
```

#### 7. **Scaling**: "Read model scaling with CQRS"

```csharp
// Write Model (Normalized, ACID)
public class Order
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    // ... business methods
}

// Read Model (Denormalized, Optimized for Queries)
public class OrderReadModel
{
    public OrderId Id { get; set; }
    public string CustomerName { get; set; }
    public string CustomerEmail { get; set; }
    public OrderStatus Status { get; set; }
    public Money Total { get; set; }
    public int ItemCount { get; set; }
    public DateTime CreatedAt { get; set; }
    public ShippingAddressDto ShippingAddress { get; set; }
    // No behavior, just data
}

// Projection (Event Handler)
public class OrderProjection : 
    IEventHandler<OrderCreated>,
    IEventHandler<OrderItemAdded>,
    IEventHandler<OrderSubmitted>
{
    private readonly IReadModelRepository<OrderReadModel> _readRepo;
    
    public async Task Handle(OrderCreated evt)
    {
        var model = new OrderReadModel
        {
            Id = evt.OrderId,
            CustomerName = evt.CustomerName,
            CustomerEmail = evt.CustomerEmail,
            Status = OrderStatus.Draft,
            CreatedAt = evt.Timestamp
        };
        await _readRepo.AddAsync(model);
    }
    
    public async Task Handle(OrderSubmitted evt)
    {
        var model = await _readRepo.GetAsync(evt.OrderId);
        model.Status = OrderStatus.Submitted;
        await _readRepo.UpdateAsync(model);
    }
}
```

#### 8. **Resilience**: "Circuit Breaker, Retry, Timeout patterns"

```csharp
// Polly Policies
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .WaitAndRetryAsync(3, 
        retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
        onRetry: (outcome, timespan, retryCount, context) =>
        {
            _logger.LogWarning("Retry {RetryCount} after {Delay}", retryCount, timespan);
        });

var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (ex, duration) => _logger.LogError("Circuit broken for {Duration}", duration),
        onReset: () => _logger.LogInformation("Circuit reset"),
        onHalfOpen: () => _logger.LogInformation("Circuit half-open"));

var timeoutPolicy = Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(10));

// Combined
var resiliencePipeline = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy, timeoutPolicy);

// Usage with HttpClient
builder.Services.AddHttpClient<IOrderService, OrderService>()
    .AddPolicyHandler(resiliencePipeline);

// .NET 8+ Resilience Pipeline (Native)
builder.Services.AddResiliencePipeline("orders", pipeline =>
{
    pipeline.AddRetry(new HttpRetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true
    });
    pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        MinimumThroughput = 10,
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(30)
    });
    pipeline.AddTimeout(TimeSpan.FromSeconds(10));
});
```

#### 9. **Observability**: "Distributed tracing with correlation IDs"

```csharp
// Correlation ID Middleware
public class CorrelationIdMiddleware
{
    private const string HeaderName = "X-Correlation-ID";
    
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var correlationId = context.Request.Headers[HeaderName].FirstOrDefault() 
                           ?? Guid.NewGuid().ToString();
        
        context.Response.Headers[HeaderName] = correlationId;
        
        using (LogContext.PushProperty("CorrelationId", correlationId))
        using (Telemetry.Context.Operation.Id = correlationId)
        {
            // Add to outgoing HttpClient calls
            context.RequestServices.GetRequiredService<IHttpClientFactory>()
                .CreateClient()
                .DefaultRequestHeaders.Add(HeaderName, correlationId);
            
            await next(context);
        }
    }
}

// OpenTelemetry Setup
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddSource("Nanolite.Orders")
        .AddOtlpExporter(opt => opt.Endpoint = new Uri("http://jaeger:4317")));

// Activity Source in Service
private static readonly ActivitySource ActivitySource = new("Nanolite.Orders");

public async Task<Order> CreateAsync(CreateOrderDto dto)
{
    using var activity = ActivitySource.StartActivity("CreateOrder", ActivityKind.Internal);
    activity?.SetTag("customer.id", dto.CustomerId);
    activity?.SetTag("order.itemCount", dto.Items.Count);
    
    // ...
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Distributed Monolith**: Services sharing DB, synchronous calls everywhere
- [ ] **Data Duplication**: CQRS read models eventually consistent - handle in UI
- [ ] **Transaction Boundaries**: Don't span services with ACID transactions
- [ ] **Service Discovery**: Don't hardcode URLs - use service mesh / DNS
- [ ] **Versioning**: API versioning strategy (URL, header, media type)
- [ ] **Testing**: Contract testing (Pact) for service interfaces
- [ ] **Deployment**: Database migrations before service deploy (backward compatible)
- [ ] **Debugging**: Distributed tracing mandatory (OpenTelemetry/Jaeger)
- [ ] **Security**: mTLS between services, zero trust network

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **MediatR Pipeline Behaviors (Cross-cutting)**
```csharp
// Validation Behavior
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    
    public async Task<TResponse> Handle(TRequest request, 
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var failures = _validators
            .Select(v => v.Validate(request))
            .SelectMany(r => r.Errors)
            .Where(e => e != null)
            .ToList();
        
        if (failures.Any())
            throw new ValidationException(failures);
        
        return await next();
    }
}

// Logging Behavior
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    
    public async Task<TResponse> Handle(TRequest request, 
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        _logger.LogInformation("Handling {Request}", typeof(TRequest).Name);
        var response = await next();
        _logger.LogInformation("Handled {Request}", typeof(TRequest).Name);
        return response;
    }
}

// Transaction Behavior
public class TransactionBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly IUnitOfWork _uow;
    
    public async Task<TResponse> Handle(TRequest request, 
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!typeof(TRequest).Name.EndsWith("Query")) // Only commands
        {
            await _uow.BeginTransactionAsync(ct);
            try
            {
                var response = await next();
                await _uow.CommitAsync(ct);
                return response;
            }
            catch
            {
                await _uow.RollbackAsync(ct);
                throw;
            }
        }
        return await next();
    }
}

// Registration
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
```

---

### PRACTICE PROBLEMS

1. **Design**: E-commerce system with: Orders, Payments, Inventory, Shipping, Notifications
2. **Implement**: Modular monolith with MediatR, EF Core, clean architecture
3. **Refactor**: Extract Payments module to microservice using Strangler Fig
4. **Scale**: Add CQRS read models for Order queries
5. **Resilience**: Implement Saga for Order placement with compensation

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Monolith vs Microservices decision | Team size, domain complexity, scaling needs, deployment independence |
| Bounded Context | Ubiquitous language boundary, team ownership, data ownership |
| CQRS | Separate read/write models. Write: normalized, ACID. Read: denormalized, fast |
| Saga vs 2PC | Saga: eventual consistency, compensation. 2PC: blocking, not for microservices |
| Outbox Pattern | Reliable event publishing. Store events in same TX as business data |
| Strangler Fig | Incremental migration: new service alongside old, route traffic gradually |
| API Gateway | Single entry point: auth, rate limit, routing, transformation |
| Circuit Breaker | Fail fast when downstream failing. States: Closed → Open → Half-Open |
| Idempotency | Safe retries. Client sends key, server caches response |
| Distributed Tracing | Correlation ID across services. OpenTelemetry → Jaeger/Zipkin |

---

## NEXT TOPIC: `01-FRONTEND-FUNDAMENTALS/01-HTTP-DNS-BROWSER.md`

> **Study Tip**: Build a modular monolith with: Clean Architecture, MediatR pipeline behaviors, EF Core, Domain Events + Outbox, CQRS read models. Then extract one module as microservice.