# DEPENDENCY INJECTION & MIDDLEWARE
## Exam-Style Study Notes

---

## TOPIC: Dependency Injection & Middleware in ASP.NET Core

### WHY? (Problem Statement)

**Before DI (Manual Dependency Management):**
```csharp
// Tight coupling, hard to test, singleton-like lifecycle
public class OrderController : Controller
{
    private readonly SqlOrderRepository _repo = new SqlOrderRepository(); // Hardcoded!
    private readonly EmailService _email = new EmailService();
    
    public IActionResult Create(OrderDto dto)
    {
        _repo.Save(dto);  // Can't swap for testing
        _email.Send(...); // Can't mock
    }
}
```

**Problems Solved by DI:**
| Problem | DI Solution |
|---------|-------------|
| Tight coupling | Depend on abstractions (interfaces) |
| Hard to test | Mock interfaces in unit tests |
| Lifecycle management | Container handles Transient/Scoped/Singleton |
| Configuration scattering | Centralized registration in `Program.cs` |
| Circular dependencies | Detected at startup |

**Real-World Analogy:**
- Without DI: You build your own car engine, tires, electronics
- With DI: You specify "I need a car" - factory provides assembled car with swappable parts

---

### HOW? (Internal Mechanism)

#### 1. **DI Container Registration & Resolution**

```csharp
// Program.cs - Registration Phase
var builder = WebApplication.CreateBuilder(args);

// ServiceCollection implements IServiceCollection
builder.Services.AddTransient<IEmailService, EmailService>();
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();

// Build Phase - Creates ServiceProvider
var app = builder.Build();  // Internally: _serviceProvider = _services.BuildServiceProvider();

// Resolution Phase - Per Request (Scoped)
public class OrderController : ControllerBase
{
    private readonly IOrderRepository _repo;  // Injected via constructor
    
    public OrderController(IOrderRepository repo)  // Constructor Injection
    {
        _repo = repo;
    }
}
```

**Container Internals:**
```
ServiceCollection (Registration)
    │
    ├─► ServiceDescriptor[]
    │       ├─ ServiceType: IOrderRepository
    │       ├─ ImplementationType: SqlOrderRepository
    │       ├─ Lifetime: Scoped
    │       └─ Factory: null (or Func<IServiceProvider, T>)
    │
    ▼
ServiceProvider (Resolution)
    │
    ├─► Root Scope (Singleton services)
    │
    └─► Request Scope (Scoped services) ──► Disposed at request end
            │
            ├─► IOrderRepository → SqlOrderRepository (same instance per request)
            ├─► DbContext → AppDbContext (same instance per request)
            └─► IEmailService → EmailService (new per request if Transient)
```

#### 2. **Middleware Pipeline Execution**

```
Request
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  Middleware 1 (UseExceptionHandler)                         │
│    try { await _next(context); }                            │
│    catch { HandleError(); }                                 │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  Middleware 2 (UseHttpsRedirection)                         │
│    if (HTTP) { Redirect to HTTPS; return; }                 │
│    await _next(context);                                    │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  Middleware 3 (UseRouting)                                  │
│    // Matches endpoint, populates Endpoint property         │
│    await _next(context);                                    │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  Middleware 4 (UseAuthentication)                           │
│    // Validates JWT, sets User principal                    │
│    await _next(context);                                    │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  Middleware 5 (UseAuthorization)                            │
│    // Checks policies, returns 403 if fail                  │
│    await _next(context);                                    │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  ENDPOINT (Controller/Minimal API)                          │
│    // Executes action, returns result                       │
└─────────────────────────────────────────────────────────────┘
  │
  ▼ (Response flows back UP through middleware)
Response
```

**Middleware Delegate Signature:**
```csharp
public delegate Task RequestDelegate(HttpContext context);

// Custom middleware implementation
public class CustomMiddleware
{
    private readonly RequestDelegate _next;
    
    public CustomMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context, IMyService scopedService)
    {
        // BEFORE next middleware (Request)
        context.Request.Headers["X-Custom-Header"] = "Value";
        
        try
        {
            await _next(context);  // Call next middleware
        }
        finally
        {
            // AFTER next middleware (Response) - runs even on exception
            context.Response.Headers["X-Response-Time"] = "123ms";
        }
    }
}

// Extension for clean registration
public static class CustomMiddlewareExtensions
{
    public static IApplicationBuilder UseCustomMiddleware(this IApplicationBuilder builder)
        => builder.UseMiddleware<CustomMiddleware>();
}
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Service Lifetimes - CRITICAL FOR INTERVIEWS**

| Lifetime | Creation | Disposal | Scope | Use Cases |
|----------|----------|----------|-------|-----------|
| **Transient** | Every injection | End of request | Per injection | Lightweight, stateless services |
| **Scoped** | Once per request | End of request | Per HTTP request | DbContext, UnitOfWork, Repositories |
| **Singleton** | First request | App shutdown | Application | Configuration, Cache, Heavy services |

**Visual:**
```
Request 1                    Request 2
─────────────────            ─────────────────
Singleton:  [A] ────────────── [A] (SAME instance)
Scoped:     [B] ──── (end)     [C] (NEW per request)
Transient:  [D] [E] [F]        [G] [H] (NEW per injection)
```

**Common Pitfall - Captive Dependency:**
```csharp
// WRONG - Singleton capturing Scoped
builder.Services.AddSingleton<IOrderService>(sp => 
    new OrderService(sp.GetRequiredService<AppDbContext>())); // DbContext is Scoped!

// CORRECT - Make OrderService Scoped
builder.Services.AddScoped<IOrderService, OrderService>();

// OR - Use IServiceScopeFactory in Singleton
builder.Services.AddSingleton<IBackgroundTask>(sp => 
    new BackgroundTask(sp.GetRequiredService<IServiceScopeFactory>()));
```

#### 2. **Advanced Registration Patterns**

```csharp
// Factory Pattern
builder.Services.AddScoped<IRepository>(sp => 
{
    var ctx = sp.GetRequiredService<AppDbContext>();
    return new EfRepository(ctx);
});

// Multiple Implementations (Named/Keyed Services - .NET 8+)
builder.Services.AddKeyedSingleton<IPaymentProcessor, StripeProcessor>("stripe");
builder.Services.AddKeyedSingleton<IPaymentProcessor, PayPalProcessor>("paypal");

// Injection with key
public class OrderService
{
    public OrderService(
        [FromKeyedServices("stripe")] IPaymentProcessor stripe,
        [FromKeyedServices("paypal")] IPaymentProcessor paypal) { }
}

// Decorator Pattern
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.Decorate<IOrderRepository, CachedOrderRepository>();

// Generic Registration
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
builder.Services.AddScoped(typeof(IQueryHandler<,>), typeof(QueryHandler<,>));

// Conditional Registration
if (builder.Environment.IsDevelopment())
    builder.Services.AddSingleton<IEmailService, DevEmailService>();
else
    builder.Services.AddSingleton<IEmailService, SendGridEmailService>();

// Options Pattern (Configuration Binding)
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("Jwt"));
builder.Services.AddSingleton(resolver => 
    resolver.GetRequiredService<IOptions<JwtSettings>>().Value);
```

#### 3. **Middleware Essentials**

**Built-in Middleware Order (Critical):**
```csharp
var app = builder.Build();

// 1. Exception Handling (EARLY - catches everything after)
if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();
else
    app.UseExceptionHandler("/error");

// 2. HTTPS & Security
app.UseHttpsRedirection();
app.UseHsts();  // HTTP Strict Transport Security

// 3. Static Files (before routing)
app.UseStaticFiles();

// 4. Routing (MUST be before Auth)
app.UseRouting();

// 5. CORS (after routing, before auth)
app.UseCors("AllowAll");

// 6. Authentication (IDENTIFIES user)
app.UseAuthentication();

// 7. Authorization (AUTHORIZES user)
app.UseAuthorization();

// 8. Custom Middleware
app.UseRequestLogging();
app.UseRateLimiting();

// 9. Endpoints (LAST)
app.MapControllers();
app.MapHealthChecks("/health");
```

**Writing Custom Middleware:**
```csharp
// 1. Inline (Simple)
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next(context);
    sw.Stop();
    context.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
});

// 2. Class-based (Complex, DI support)
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;
    
    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var requestId = context.TraceIdentifier;
        using (_logger.BeginScope(new Dictionary<string, object> { ["RequestId"] = requestId }))
        {
            _logger.LogInformation("Request: {Method} {Path}", context.Request.Method, context.Request.Path);
            
            try
            {
                await _next(context);
                _logger.LogInformation("Response: {StatusCode}", context.Response.StatusCode);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unhandled exception");
                throw;
            }
        }
    }
}

// 3. Factory-based (for per-request dependencies)
public class TenantMiddleware
{
    private readonly RequestDelegate _next;
    
    public TenantMiddleware(RequestDelegate next) => _next = next;
    
    public async Task InvokeAsync(HttpContext context, ITenantService tenantService)
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault();
        if (!string.IsNullOrEmpty(tenantId))
        {
            await tenantService.SetCurrentTenantAsync(tenantId);
        }
        await _next(context);
    }
}
```

#### 4. **IApplicationBuilder Extensions**

```csharp
// MapWhen - Conditional Middleware
app.MapWhen(ctx => ctx.Request.Path.StartsWithSegments("/api"), api =>
{
    api.UseAuthentication();
    api.UseAuthorization();
    api.MapControllers();
});

// Map - Sub-pipeline
app.Map("/admin", admin =>
{
    admin.UseMiddleware<AdminAuthMiddleware>();
    admin.MapControllers();
});

// UseWhen - Branch and rejoin
app.UseWhen(
    ctx => ctx.Request.Path.StartsWithSegments("/health"),
    branch => branch.UseMiddleware<HealthCheckMiddleware>()
);

// Run - Terminal Middleware (short-circuits)
app.Run(async context => 
{
    await context.Response.WriteAsync("Hello World");
});
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Constructor Injection** | Always for required deps | Property Injection | Hidden dependencies, null refs |
| **Scoped DbContext** | Per request | Singleton DbContext | Thread safety, memory leaks |
| **IServiceScopeFactory** | Background services | Capturing Scoped in Singleton | Captive dependency |
| **Middleware for Cross-Cutting** | Logging, Auth, Errors | Putting in Controllers | Violates SRP, duplication |
| **Options Pattern** | Configuration | `IConfiguration` everywhere | Hard to test, no validation |
| **Decorator via DI** | Caching, Logging, Retry | Manual wrapping | Boilerplate, inconsistent |

---

### INTERVIEW QUESTIONS (Top 10)

#### 1. **Conceptual**: "Explain the difference between Transient, Scoped, and Singleton"

**Answer:**
```
Transient: New instance EVERY time injected. Lightweight, stateless.
Scoped: SAME instance within ONE HTTP request. DbContext, UnitOfWork.
Singleton: SAME instance for ENTIRE APP LIFETIME. Config, Cache, heavy services.

Key Rule: Singleton CANNOT inject Scoped (captive dependency).
Scoped CAN inject Singleton (fine).
Transient CAN inject anything.
```

#### 2. **Code**: "Fix this captive dependency bug"

```csharp
// BUGGY CODE
builder.Services.AddSingleton<IUserService>(sp => 
    new UserService(sp.GetRequiredService<AppDbContext>())); // DbContext is Scoped!

// FIX 1: Make UserService Scoped
builder.Services.AddScoped<IUserService, UserService>();

// FIX 2: Use IServiceScopeFactory in Singleton
builder.Services.AddSingleton<IUserService>(sp => 
{
    var scopeFactory = sp.GetRequiredService<IServiceScopeFactory>();
    return new UserService(scopeFactory); // Service creates scope internally
});

// UserService implementation for Fix 2:
public class UserService : IUserService
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public UserService(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;
    
    public async Task<User> GetUserAsync(int id)
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        return await db.Users.FindAsync(id);
    }
}
```

#### 3. **Design**: "Where does Middleware go in the pipeline?"

**Answer:**
```
ORDER MATTERS (Request flows DOWN, Response flows UP):

1. Exception Handling (UseExceptionHandler) - FIRST, catches all
2. HTTPS Redirection / HSTS
3. Static Files (UseStaticFiles) - Before routing
4. Routing (UseRouting) - Matches endpoint
5. CORS (UseCors) - After routing, before auth
6. Authentication (UseAuthentication) - Identifies user
7. Authorization (UseAuthorization) - Checks permissions
8. Custom Middleware (Logging, Rate Limit)
9. Endpoints (MapControllers, MapGet) - LAST

Mnemonic: "Exception → HTTPS → Static → Route → CORS → AuthN → AuthZ → Custom → Endpoints"
```

#### 4. **Code**: "Write middleware that adds correlation ID to all logs"

```csharp
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationIdHeader = "X-Correlation-ID";
    
    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;
    
    public async Task InvokeAsync(HttpContext context, ILogger<CorrelationIdMiddleware> logger)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault() 
                           ?? Guid.NewGuid().ToString();
        
        context.Response.Headers[CorrelationIdHeader] = correlationId;
        
        using (logger.BeginScope(new Dictionary<string, object> 
        { 
            ["CorrelationId"] = correlationId,
            ["RequestPath"] = context.Request.Path.Value
        }))
        {
            logger.LogInformation("Request started");
            try
            {
                await _next(context);
                logger.LogInformation("Request completed: {StatusCode}", context.Response.StatusCode);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Request failed");
                throw;
            }
        }
    }
}

// Register EARLY in pipeline
app.UseMiddleware<CorrelationIdMiddleware>();
```

#### 5. **Performance**: "Why avoid `IServiceProvider.GetService()` in middleware?"

**Answer:**
```
Service Locator Pattern (Anti-pattern):
- Hidden dependencies (not in constructor)
- Harder to test
- Runtime errors vs compile-time
- Breaks DI container's lifetime management

Correct: Constructor Injection in Middleware
public class MyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMyService _service;  // Injected at startup (Singleton/Scoped)
    
    public MyMiddleware(RequestDelegate next, IMyService service)
    {
        _next = next;
        _service = service;  // Resolved once per middleware lifetime
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // For Scoped services, use InvokeAsync parameter injection:
        // public async Task InvokeAsync(HttpContext context, IScopedService scoped)
    }
}
```

#### 6. **Debugging**: "Middleware runs twice - why?"

**Common Causes:**
```csharp
// 1. Double registration
app.UseMiddleware<LoggingMiddleware>();
app.UseMiddleware<LoggingMiddleware>();  // Oops!

// 2. Branching with Map/MapWhen without rejoining
app.Map("/api", api => api.UseMiddleware<LoggingMiddleware>());
app.UseMiddleware<LoggingMiddleware>();  // Runs for non-/api too

// 3. Terminal middleware (Run) before your middleware
app.Run(async ctx => await ctx.Response.WriteAsync("Hello"));
app.UseMiddleware<LoggingMiddleware>();  // NEVER RUNS

// 4. Next() called twice in custom middleware
await _next(context);
await _next(context);  // BUG!
```

#### 7. **Architecture**: "Implement a retry middleware with Polly"

```csharp
public class ResilienceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ResilienceMiddleware> _logger;
    
    public ResilienceMiddleware(RequestDelegate next, ILogger<ResilienceMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var pipeline = new ResiliencePipelineBuilder()
            .AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromSeconds(1),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>()
            })
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio = 0.5,
                MinimumThroughput = 10,
                SamplingDuration = TimeSpan.FromSeconds(30),
                BreakDuration = TimeSpan.FromMinutes(1)
            })
            .Build();
        
        await pipeline.ExecuteAsync(async token =>
        {
            await _next(context);
        }, context.RequestAborted);
    }
}
```

#### 8. **Code**: "Register a generic decorator for caching"

```csharp
// Decorator
public class CachedRepository<T> : IRepository<T> where T : class
{
    private readonly IRepository<T> _inner;
    private readonly IDistributedCache _cache;
    
    public CachedRepository(IRepository<T> inner, IDistributedCache cache)
    {
        _inner = inner;
        _cache = cache;
    }
    
    public async Task<T?> GetByIdAsync(int id)
    {
        var key = $"{typeof(T).Name}:{id}";
        var cached = await _cache.GetStringAsync(key);
        if (cached != null) return JsonSerializer.Deserialize<T>(cached);
        
        var entity = await _inner.GetByIdAsync(id);
        if (entity != null)
            await _cache.SetStringAsync(key, JsonSerializer.Serialize(entity));
        return entity;
    }
    
    // ... other methods delegate to _inner
}

// Registration (Scrutor library)
builder.Services.Scan(scan => scan
    .FromAssemblyOf<IRepository>()
    .AddClasses(classes => classes.AssignableTo(typeof(IRepository<>)))
    .AsImplementedInterfaces()
    .WithScopedLifetime());

builder.Services.Decorate(typeof(IRepository<>), typeof(CachedRepository<>));
```

#### 9. **Security**: "Middleware for API Key validation"

```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _config;
    private const string ApiKeyHeader = "X-API-Key";
    
    public ApiKeyMiddleware(RequestDelegate next, IConfiguration config)
    {
        _next = next;
        _config = config;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Skip for health checks
        if (context.Request.Path.StartsWithSegments("/health"))
        {
            await _next(context);
            return;
        }
        
        if (!context.Request.Headers.TryGetValue(ApiKeyHeader, out var providedKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new { error = "API Key required" });
            return;
        }
        
        var validKey = _config["ApiKey"];
        if (!CryptographicOperations.FixedTimeEquals(
            System.Text.Encoding.UTF8.GetBytes(providedKey!),
            System.Text.Encoding.UTF8.GetBytes(validKey!)))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new { error = "Invalid API Key" });
            return;
        }
        
        await _next(context);
    }
}

// Register BEFORE UseAuthentication
app.UseMiddleware<ApiKeyMiddleware>();
app.UseAuthentication();
```

#### 10. **Testing**: "Unit test middleware in isolation"

```csharp
[Fact]
public async Task CorrelationIdMiddleware_AddsHeader()
{
    // Arrange
    var middleware = new CorrelationIdMiddleware(next: (ctx) => Task.CompletedTask);
    var context = new DefaultHttpContext();
    context.Response.Body = new MemoryStream();
    
    var loggerFactory = LoggerFactory.Create(b => b.AddConsole());
    var logger = loggerFactory.CreateLogger<CorrelationIdMiddleware>();
    
    // Act
    await middleware.InvokeAsync(context, logger);
    
    // Assert
    Assert.True(context.Response.Headers.ContainsKey("X-Correlation-ID"));
    var correlationId = context.Response.Headers["X-Correlation-ID"].First();
    Assert.True(Guid.TryParse(correlationId, out _));
}

// Test with TestServer (Integration)
[Fact]
public async Task ApiKeyMiddleware_Returns401_WhenMissingKey()
{
    var builder = new WebHostBuilder()
        .ConfigureServices(s => s.AddSingleton<IConfiguration>(new ConfigurationBuilder()
            .AddInMemoryCollection(new[] { new KeyValuePair<string, string>("ApiKey", "secret") })
            .Build()))
        .Configure(app => 
        {
            app.UseMiddleware<ApiKeyMiddleware>();
            app.Run(ctx => ctx.Response.WriteAsync("OK"));
        });
    
    var server = new TestServer(builder);
    var client = server.CreateClient();
    
    var response = await client.GetAsync("/api/test");
    
    Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Middleware Order**: `UseAuthentication` before `UseAuthorization` - ALWAYS
- [ ] **Scoped in Singleton**: Use `IServiceScopeFactory` not direct injection
- [ ] **Transient Disposal**: Container disposes Transient services (if IDisposable) at scope end
- [ ] **Middleware Constructor**: Runs ONCE at startup - don't inject Scoped here
- [ ] **InvokeAsync Parameters**: Scoped services via method injection, not constructor
- [ ] **Short-circuiting**: Not calling `_next` = terminal middleware
- [ ] **Exception Handling**: Use try/finally for response modification
- [ ] **Response Headers**: Must set before writing body
- [ ] **CORS**: Must be after Routing, before Auth
- [ ] **Health Checks**: Map before Auth middleware if public
- [ ] **IApplicationLifetime**: For startup/shutdown events
- [ ] **Host vs WebApplication**: `IHost` for background services, `WebApplication` for HTTP

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Program.cs Template**
```csharp
var builder = WebApplication.CreateBuilder(args);

// Configuration
builder.Configuration.AddJsonFile("appsettings.Local.json", optional: true);

// Services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddDbContext<AppDbContext>(opt => 
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
builder.Services.AddAutoMapper(typeof(Program));

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt => { /* config */ });

builder.Services.AddAuthorization(opt => 
{
    opt.AddPolicy("Admin", p => p.RequireRole("Admin"));
});

builder.Services.AddCors(opt => opt.AddDefaultPolicy(p => 
    p.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod()));

builder.Services.AddRateLimiter(opt => 
{
    opt.AddFixedWindowLimiter("api", o => 
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
    });
});

var app = builder.Build();

// Pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseExceptionHandler("/error");
app.UseHsts();
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

#### 2. **Background Service with Scoped Dependencies**
```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderProcessingService> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromMinutes(5);
    
    public OrderProcessingService(IServiceScopeFactory scopeFactory, ILogger<OrderProcessingService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();
            
            try
            {
                await orderService.ProcessPendingOrdersAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Order processing failed");
            }
            
            await Task.Delay(_interval, stoppingToken);
        }
    }
}

// Registration
builder.Services.AddHostedService<OrderProcessingService>();
```

#### 3. **Fluent Validation Middleware**
```csharp
public class ValidationMiddleware
{
    private readonly RequestDelegate _next;
    
    public ValidationMiddleware(RequestDelegate next) => _next = next;
    
    public async Task InvokeAsync(HttpContext context, IValidatorFactory validatorFactory)
    {
        var endpoint = context.GetEndpoint();
        if (endpoint?.Metadata.GetMetadata<ValidateAttribute>() is not ValidateAttribute attr)
        {
            await _next(context);
            return;
        }
        
        var validator = validatorFactory.GetValidator(attr.DtoType);
        if (validator == null)
        {
            await _next(context);
            return;
        }
        
        // Read body (rewindable)
        context.Request.EnableBuffering();
        var body = await new StreamReader(context.Request.Body).ReadToEndAsync();
        context.Request.Body.Position = 0;
        
        var dto = JsonSerializer.Deserialize(body, attr.DtoType);
        var result = await validator.ValidateAsync(new ValidationContext(dto));
        
        if (!result.IsValid)
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(result.ToDictionary());
            return;
        }
        
        await _next(context);
    }
}

[AttributeUsage(AttributeTargets.Method)]
public class ValidateAttribute : Attribute
{
    public Type DtoType { get; }
    public ValidateAttribute(Type dtoType) => DtoType = dtoType;
}

// Usage on Minimal API
app.MapPost("/api/orders", async (CreateOrderDto dto, IOrderService svc) => 
{
    return await svc.CreateAsync(dto);
})
.WithMetadata(new ValidateAttribute(typeof(CreateOrderDto)));
```

---

### PRACTICE PROBLEMS

1. **Design**: Create a middleware pipeline for multi-tenant SaaS (tenant resolution, isolation, per-tenant config)
2. **Debug**: Middleware modifies response body but client receives empty - diagnose
3. **Performance**: Implement a cached singleton service that refreshes config every 5 minutes
4. **Security**: Write middleware that prevents XXE attacks on XML endpoints
5. **Testing**: Unit test a middleware that depends on `IHttpContextAccessor`

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| DI Lifetimes: Transient vs Scoped vs Singleton | Transient: per injection. Scoped: per request. Singleton: per app |
| Captive Dependency | Singleton injecting Scoped service → bug. Fix: ScopeFactory or make Scoped |
| Middleware Order (Request) | Exception → HTTPS → Static → Routing → CORS → AuthN → AuthZ → Custom → Endpoints |
| Middleware Constructor vs InvokeAsync | Constructor: startup (Singleton). InvokeAsync: per request (Scoped via params) |
| Short-circuit Middleware | Not calling `_next(context)` - terminal |
| Service Locator Anti-pattern | `GetService()` in code instead of constructor injection |
| IServiceScopeFactory Use Case | Background services needing Scoped services (DbContext) |
| Decorator Pattern in DI | `services.Decorate<IRepository, CachedRepository>()` (Scrutor) |
| UseRouting vs UseEndpoints | .NET 6+: UseRouting + MapControllers. Legacy: UseRouting + UseEndpoints |
| Health Checks Public | Map before Auth middleware: `app.MapHealthChecks("/health").AllowAnonymous()` |

---

## NEXT TOPIC: `05-AUTH-IDENTITY.md`

> **Study Tip**: Build a minimal API with: custom middleware (logging + correlation ID), DI with all 3 lifetimes, background service using ScopeFactory, and API key auth middleware. Test each in isolation.