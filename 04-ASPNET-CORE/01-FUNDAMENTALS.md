# ASP.NET CORE FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: ASP.NET Core Fundamentals

### WHY? (Problem Statement)

**Before ASP.NET Core:**
- **ASP.NET Framework** was Windows-only, tied to IIS, heavy (System.Web.dll ~30MB)
- No cross-platform support (Linux containers, macOS dev)
- Monolithic pipeline - couldn't opt out of unused features
- No built-in DI, configuration, logging abstractions
- Slow startup, high memory footprint

**What ASP.NET Core Solves:**
| Problem | Solution |
|---------|----------|
| Windows-only | Cross-platform (Windows, Linux, macOS) via .NET Core/.NET 5+ |
| Heavy runtime | Modular, pay-for-play (NuGet packages) |
| Tied to IIS | Kestrel web server + reverse proxy (IIS, Nginx, Apache) |
| No built-in DI | Built-in DI container (Microsoft.Extensions.DependencyInjection) |
| XML config hell | Flexible configuration (JSON, env vars, cmd args, Azure Key Vault) |
| Tight coupling | Middleware pipeline (composable, orderable) |
| No unified logging | Microsoft.Extensions.Logging abstraction |

**Real-World Analogy:** 
- Old ASP.NET = Buying a fully furnished house you can't modify
- ASP.NET Core = Buying land + modular prefab rooms you assemble as needed

---

### HOW? (Internal Mechanism)

#### 1. **Startup Process (Program.cs → WebApplication)**

```csharp
// Program.cs (Minimal API style - .NET 6+)
var builder = WebApplication.CreateBuilder(args);  // 1. Create builder
// 2. Configure services (DI container setup)
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(...));
var app = builder.Build();  // 3. Build WebApplication (immutable after this)
// 4. Configure middleware pipeline (ORDER MATTERS!)
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();  // 5. Start Kestrel, block until shutdown
```

**Internal Flow:**
```
WebApplication.CreateBuilder(args)
    │
    ├─► Creates WebApplicationBuilder
    │       ├─ ConfigurationManager (loads appsettings.json, env vars, etc.)
    │       ├─ ServiceCollection (DI container)
    │       ├─ LoggingBuilder (ILoggingBuilder)
    │       ├─ WebHostBuilder (configures Kestrel, IIS, URLs)
    │       └─ HostBuilder (generic host for background services)
    │
    └─► builder.Build()
            │
            ├─► Builds ServiceProvider from ServiceCollection
            ├─► Builds WebApplication (implements IApplicationBuilder, IEndpointRouteBuilder)
            ├─► Configures default middleware (DeveloperExceptionPage, etc. in Dev)
            └─► Returns WebApplication (sealed pipeline ready to Run()
```

#### 2. **Middleware Pipeline (Request Processing)**

```
Request ──► Middleware 1 ──► Middleware 2 ──► ... ──► Endpoint ──► Response
           (Terminal?)       (Terminal?)                (Controller/Minimal API)
           │                 │
           ▼                 ▼
      Next() or          Next() or
      Short-circuit      Short-circuit
```

**Key Principle:** Middleware executes **in order added** (FIFO) for request, **reverse order** for response.

```csharp
// Custom Middleware Pattern
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;
    
    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // BEFORE next middleware (Request)
        _logger.LogInformation("Request: {Method} {Path}", context.Request.Method, context.Request.Path);
        var sw = Stopwatch.StartNew();
        
        try 
        {
            await _next(context);  // CALL NEXT MIDDLEWARE
        }
        finally
        {
            // AFTER next middleware (Response)
            sw.Stop();
            _logger.LogInformation("Response: {StatusCode} in {Elapsed}ms", 
                context.Response.StatusCode, sw.ElapsedMilliseconds);
        }
    }
}

// Extension method for clean registration
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder builder)
        => builder.UseMiddleware<RequestLoggingMiddleware>();
}

// Usage in Program.cs - ORDER MATTERS!
app.UseRequestLogging();  // Early - logs everything
app.UseHttpsRedirection();
app.UseAuthentication();   // Must be before Authorization
app.UseAuthorization();
```

#### 3. **Dependency Injection (Built-in Container)**

**Service Lifetimes - CRITICAL FOR INTERVIEWS:**

| Lifetime | Created | Disposed | Use Case | Thread Safety |
|----------|---------|----------|----------|---------------|
| **Transient** | Every request | End of request | Lightweight, stateless | Must be thread-safe |
| **Scoped** | Once per request | End of request | DB contexts, Unit of Work | Per-request scope |
| **Singleton** | First request | App shutdown | Caches, config, heavy services | Must be thread-safe |

```csharp
// Registration patterns
builder.Services.AddTransient<IEmailService, EmailService>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();

// Factory registration (for complex creation)
builder.Services.AddScoped<IRepository>(sp => 
{
    var context = sp.GetRequiredService<AppDbContext>();
    return new EfRepository(context);
});

// Generic registration
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));

// Keyed services (.NET 8+)
builder.Services.AddKeyedSingleton<IPaymentProcessor, StripeProcessor>("stripe");
builder.Services.AddKeyedSingleton<IPaymentProcessor, PayPalProcessor>("paypal");
// Injection: [FromKeyedServices("stripe")] IPaymentProcessor processor
```

**Service Resolution Flow:**
```
Controller ctor requests IEmailService
    │
    ▼
ServiceProvider.GetService<IEmailService>()
    │
    ├─► Checks Transient: Creates new instance
    ├─► Checks Scoped: Returns from current IServiceScope
    └─► Checks Singleton: Returns cached instance
```

#### 4. **Configuration System**

**Priority Order (Highest Wins):**
1. Command line args (`--Key=Value`)
2. Environment variables (`KEY=Value` or `KEY__SUBKEY=Value`)
3. User secrets (`dotnet user-secrets`)
4. `appsettings.{Environment}.json`
5. `appsettings.json`
6. Default values in code

```csharp
// Strongly-typed configuration (Options Pattern)
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("JwtSettings"));
builder.Services.Configure<DatabaseSettings>(builder.Configuration.GetSection("Database"));

// Validation (.NET 8+)
builder.Services.AddOptions<JwtSettings>()
    .BindConfiguration("JwtSettings")
    .ValidateDataAnnotations()
    .ValidateOnStart();  // Fail fast at startup

// Access in services
public class JwtTokenService
{
    private readonly JwtSettings _settings;
    public JwtTokenService(IOptions<JwtSettings> options) => _settings = options.Value;
    
    // Or IOptionsSnapshot for scoped reload
    // Or IOptionsMonitor for singleton with change notifications
}
```

**appsettings.json:**
```json
{
  "JwtSettings": {
    "SecretKey": "your-256-bit-secret",
    "Issuer": "NanoliteTech",
    "Audience": "NanoliteTechClients",
    "ExpiryMinutes": 60
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Database=...;User=...;Password=..."
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

#### 5. **Kestrel Web Server**

```csharp
// Program.cs - Kestrel configuration
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5000);  // HTTP
    options.ListenAnyIP(5001, listenOptions =>  // HTTPS
    {
        listenOptions.UseHttps("cert.pfx", "password");
    });
    
    // Limits
    options.Limits.MaxRequestBodySize = 10_000_000; // 10MB
    options.Limits.MaxConcurrentConnections = 1000;
    options.Limits.MaxConcurrentUpgradedConnections = 1000;
    options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
    options.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(30);
});
```

---

### WHAT? (Key Concepts & APIs)

#### Essential Namespaces
```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
```

#### Core Interfaces to Know
| Interface | Purpose | Common Implementations |
|-----------|---------|------------------------|
| `IApplicationBuilder` | Build middleware pipeline | `WebApplication` |
| `IServiceCollection` | Register DI services | `ServiceCollection` |
| `IServiceProvider` | Resolve services | Built-in container |
| `IConfiguration` | Access config values | `ConfigurationRoot` |
| `IHostEnvironment` | Environment info | `HostingEnvironment` |
| `ILogger<T>` | Structured logging | `Logger<T>` |
| `IOptions<T>` | Typed config access | `OptionsManager<T>` |
| `IHostedService` | Background services | `BackgroundService` |

#### Minimal APIs (.NET 6+) - Modern Approach
```csharp
var app = builder.Build();

// MapGet, MapPost, MapPut, MapDelete, MapMethods
app.MapGet("/api/products", async (AppDbContext db) => 
    await db.Products.ToListAsync());

app.MapGet("/api/products/{id}", async (int id, AppDbContext db) =>
{
    var product = await db.Products.FindAsync(id);
    return product is not null ? Results.Ok(product) : Results.NotFound();
});

app.MapPost("/api/products", async (Product product, AppDbContext db) =>
{
    db.Products.Add(product);
    await db.SaveChangesAsync();
    return Results.Created($"/api/products/{product.Id}", product);
});

// Route groups
var api = app.MapGroup("/api").RequireAuthorization();
api.MapGet("/admin/stats", GetAdminStats);

// Filters
app.MapGet("/api/public", () => "Public")
   .AddEndpointFilter(async (context, next) =>
   {
       // Pre-processing
       var result = await next(context);
       // Post-processing
       return result;
   });
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Options Pattern** | Typed config with validation | `Configuration["Key"]` everywhere | No validation, magic strings, hard to test |
| **Scoped DbContext** | Per-request EF Core | Singleton DbContext | Thread-safety, concurrency bugs, memory leaks |
| **Middleware for Cross-Cutting** | Logging, auth, errors | Putting logic in controllers | Violates SRP, duplicates code |
| **Minimal APIs** | Simple CRUD, microservices | Controllers for everything | Overhead for simple endpoints |
| **Result Pattern** | Error handling without exceptions | Exceptions for control flow | Performance, clarity |
| **IHostedService** | Background jobs | ThreadPool.QueueUserWorkItem | No DI, no lifecycle management |

**Result Pattern Example (Interview Favorite):**
```csharp
public record Result<T>(bool IsSuccess, T? Value, string? Error);
public record Result(bool IsSuccess, string? Error);

public static class ResultExtensions
{
    public static Result<T> Ok<T>(T value) => new(true, value, null);
    public static Result<T> Fail<T>(string error) => new(false, default, error);
    
    public static async Task<Result<T>> Bind<T, TNext>(
        this Task<Result<T>> task, 
        Func<T, Task<Result<TNext>>> func)
    {
        var result = await task;
        return result.IsSuccess ? await func(result.Value) : Result<TNext>.Fail(result.Error!);
    }
}

// Usage in Minimal API
app.MapPost("/api/orders", async (CreateOrderDto dto, IOrderService service) =>
{
    var result = await service.CreateOrderAsync(dto);
    return result.IsSuccess ? Results.Ok(result.Value) : Results.BadRequest(result.Error);
});
```

---

### INTERVIEW QUESTIONS (Top 10)

#### 1. **Conceptual**: "Explain the ASP.NET Core middleware pipeline. How does request/response flow?"

**Answer:**
```
Request enters → Middleware 1 (pre) → Middleware 2 (pre) → ... → Endpoint
Response exits ← Middleware 1 (post) ← Middleware 2 (post) ← ... ← Endpoint

Each middleware can:
1. Process request BEFORE calling _next(context)
2. Call _next(context) to pass to next middleware
3. Process response AFTER _next(context) returns
4. SHORT-CIRCUIT by NOT calling _next (terminal middleware)

Order of app.UseXxx() calls = Request order
Reverse order = Response order
```

**Follow-up:** "Where does `UseAuthentication` go relative to `UseAuthorization`?"
→ **AuthN before AuthZ** - must identify user before checking permissions.

---

#### 2. **Code**: "What's wrong with this code?"

```csharp
// PROBLEM: Captured scoped service in singleton
builder.Services.AddSingleton<IEmailService>(sp => 
{
    var db = sp.GetRequiredService<AppDbContext>(); // SCOPED!
    return new EmailService(db);
});
```

**Answer:** **Captive Dependency** - Singleton captures Scoped service. DbContext lives forever, causing:
- Memory leaks
- Concurrency issues (DbContext not thread-safe)
- Stale data

**Fix:** Make EmailService Scoped, or use `IServiceScopeFactory` to create scope inside singleton.

---

#### 3. **Design**: "Design a rate-limiting middleware for ASP.NET Core"

**Answer:**
```csharp
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;
    private readonly RateLimitOptions _options;
    
    public RateLimitingMiddleware(RequestDelegate next, IMemoryCache cache, IOptions<RateLimitOptions> options)
    {
        _next = next;
        _cache = cache;
        _options = options.Value;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var key = $"ratelimit:{context.Connection.RemoteIpAddress}:{context.Request.Path}";
        var count = _cache.GetOrCreate(key, entry => 
        {
            entry.AbsoluteExpirationRelativeToNow = _options.Window;
            return 0;
        });
        
        if (count >= _options.MaxRequests)
        {
            context.Response.StatusCode = 429;
            context.Response.Headers["Retry-After"] = _options.Window.TotalSeconds.ToString();
            await context.Response.WriteAsync("Rate limit exceeded");
            return;
        }
        
        _cache.Set(key, count + 1);
        await _next(context);
    }
}
```

**Follow-ups:**
- How to distribute across servers? → Redis distributed cache
- How to customize per user/endpoint? → Key by user ID + path
- Sliding window vs fixed window? → Sliding more accurate, use SortedSet in Redis

---

#### 4. **Debugging**: "Controller works locally but returns 404 in production"

**Checklist:**
- [ ] `app.MapControllers()` called?
- [ ] Route template matches? (`[Route("api/[controller]")]`)
- [ ] HTTPS redirection before routing?
- [ ] Reverse proxy (Nginx/IIS) stripping path prefix?
- [ ] `UsePathBase` needed for sub-path deployment?
- [ ] Controller compiled? (`dotnet publish` includes all?)

---

#### 5. **System Design**: "How does ASP.NET Core handle 100K concurrent connections?"

**Answer:**
- **Kestrel** = libuv (or managed sockets) - async I/O, no thread per connection
- **ThreadPool** - processes completions, not connections
- **Configure Kestrel limits:** `MaxConcurrentConnections`, `MaxRequestBodySize`
- **Scale out:** Multiple Kestrel instances behind load balancer
- **Async all the way:** `await` everywhere, no `.Result`/`.Wait()`
- **Connection pooling:** DB, HTTP client, Redis

---

### GOTCHAS & EDGE CASES

- [ ] **Captive Dependency** - Singleton injecting Scoped
- [ ] **Middleware Order** - Auth before AuthZ, Exception handling early
- [ ] **DbContext Lifetime** - Scoped per request, not Singleton
- [ ] **IOptions vs IOptionsSnapshot vs IOptionsMonitor** - Know the difference
- [ ] **Kestrel vs IIS** - IIS is reverse proxy, Kestrel is app server
- [ ] **Endpoint Routing** - `UseRouting`/`UseEndpoints` vs Minimal APIs
- [ ] **Async Suffix** - Always `async Task`, not `async void` (except event handlers)
- [ ] **ConfigureAwait(false)** - In libraries, not in ASP.NET Core (has SynchronizationContext)
- [ ] **ProblemDetails** - Use `Results.Problem()` for RFC 7807 compliance
- [ ] **Health Checks** - `AddHealthChecks()`, `MapHealthChecks("/health")`

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Program.cs Template (.NET 8 Minimal API)**
```csharp
var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddAutoMapper(typeof(Program));
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt => opt.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        ValidAudience = builder.Configuration["Jwt:Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
    });
builder.Services.AddAuthorization();
builder.Services.AddCors(opt => opt.AddDefaultPolicy(p => p.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod()));

var app = builder.Build();

// Pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### 2. **Global Exception Handling Middleware**
```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    
    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation failed");
            await WriteProblemDetails(context, StatusCodes.Status400BadRequest, ex.Message, ex.Errors);
        }
        catch (NotFoundException ex)
        {
            _logger.LogInformation(ex, "Resource not found");
            await WriteProblemDetails(context, StatusCodes.Status404NotFound, ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await WriteProblemDetails(context, StatusCodes.Status500InternalServerError, "An error occurred");
        }
    }
    
    private static async Task WriteProblemDetails(HttpContext context, int status, string title, object? errors = null)
    {
        context.Response.StatusCode = status;
        context.Response.ContentType = "application/problem+json";
        var problem = new ProblemDetails
        {
            Status = status,
            Title = title,
            Extensions = { ["errors"] = errors }
        };
        await context.Response.WriteAsJsonAsync(problem);
    }
}
```

#### 3. **Background Service with DI Scope**
```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderProcessingService> _logger;
    
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
                _logger.LogError(ex, "Error processing orders");
            }
            
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}

// Registration
builder.Services.AddHostedService<OrderProcessingService>();
```

---

### PRACTICE PROBLEMS

1. **LeetCode Style**: Implement a custom `IEndpointFilter` for request validation
2. **System Design**: Design a distributed rate limiter using ASP.NET Core + Redis
3. **Code Review**: Find 5 issues in a Program.cs with mixed middleware order, captive dependencies, no health checks
4. **Debugging**: Given a 500 error with no logs, trace through middleware pipeline to find missing `UseExceptionHandler`

---

### ANKI FLASHCARDS (Create these)

| Front | Back |
|-------|------|
| ASP.NET Core middleware execution order | Request: FIFO (order added). Response: LIFO (reverse order) |
| Service lifetimes: Transient vs Scoped vs Singleton | Transient: per request. Scoped: per HTTP request. Singleton: app lifetime |
| Captive Dependency | Singleton capturing Scoped service → memory leaks, thread safety issues |
| IOptions vs IOptionsSnapshot vs IOptionsMonitor | IOptions: singleton, no reload. IOptionsSnapshot: scoped, reload per request. IOptionsMonitor: singleton, supports change notifications |
| Kestrel vs IIS | Kestrel: cross-platform web server. IIS: Windows reverse proxy. Use together in production |
| Minimal API vs Controllers | Minimal: lightweight, functional. Controllers: classes, attributes, better for complex APIs |
| UseAuthentication vs UseAuthorization order | Authentication FIRST (identify user), then Authorization (check permissions) |
| ProblemDetails (RFC 7807) | Standard error format: type, title, status, detail, instance |

---

## NEXT TOPIC: `02-MINIMAL-APIS-MVC.md`

> **Study Tip**: Write the WHY/HOW/WHAT for Minimal APIs vs Controllers from memory, then verify against this note structure.