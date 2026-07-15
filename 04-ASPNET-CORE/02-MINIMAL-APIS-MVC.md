# MINIMAL APIs vs CONTROLLERS
## Exam-Style Study Notes

---

## TOPIC: Minimal APIs vs Controllers in ASP.NET Core

### WHY? (Problem Statement)

**Traditional Controllers (MVC/Web API):**
- Class-based, attribute routing (`[HttpGet]`, `[Route]`)
- Ceremony: Controller base class, action methods, model binding attributes
- Good for: Complex APIs with many related endpoints, teams familiar with MVC
- **Pain points**: Boilerplate, harder to test in isolation, larger surface area

**Minimal APIs (.NET 6+):**
- Function-based, lambda/delegate endpoints
- Lightweight, no controller class needed
- Built for: Microservices, simple CRUD, serverless, high-performance APIs
- **Benefits**: Less code, faster startup, better AOT compilation, easier testing

**Real-World Analogy:**
- Controllers = Traditional restaurant with waiters, menus, courses
- Minimal APIs = Food truck - order, get food, leave. Fast, focused.

---

### HOW? (Internal Mechanism)

#### Minimal API Request Handling Flow:
```
Request → Routing (EndpointDataSource) → EndpointFilterPipeline → Delegate → Response
                │
                ├─ MapGet("/api/users", handler)
                ├─ MapPost("/api/users", handler)
                └─ MapMethods("/api/users", ["PUT", "PATCH"], handler)
```

**Endpoint Registration:**
```csharp
// Internally creates RouteEndpoint with RequestDelegate
app.MapGet("/api/users", async (AppDbContext db) => 
{
    return await db.Users.ToListAsync();
});

// Compiles to:
var endpoint = new RouteEndpoint(
    requestDelegate: async (HttpContext ctx) => { /* handler */ },
    routePattern: RoutePatternFactory.Parse("/api/users"),
    order: 0
);
```

**Model Binding in Minimal APIs:**
- **Automatic** from: Route params, Query string, Headers, Body (JSON)
- **Source generators** create binding code at compile time (AOT-friendly)
- **TryParse** pattern for custom types

```csharp
// Automatic binding
app.MapGet("/api/users/{id:int}", (int id, string? name, CancellationToken ct) => ...);

// Custom binder
public class PaginationParams
{
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    
    public static ValueTask<PaginationParams?> BindAsync(HttpContext ctx, ParameterInfo param)
    {
        var page = int.TryParse(ctx.Request.Query["page"], out var p) ? p : 1;
        var size = int.TryParse(ctx.Request.Query["pageSize"], out var s) ? s : 10;
        return ValueTask.FromResult<PaginationParams?>(new() { Page = page, PageSize = size });
    }
}
```

---

### WHAT? (Key Concepts & APIs)

#### Minimal API Patterns

**1. Basic CRUD:**
```csharp
var users = app.MapGroup("/api/users").WithTags("Users");

users.MapGet("/", async (AppDbContext db) => await db.Users.ToListAsync());
users.MapGet("/{id}", async (int id, AppDbContext db) => 
    await db.Users.FindAsync(id) is User u ? Results.Ok(u) : Results.NotFound());
users.MapPost("/", async (User user, AppDbContext db) => 
{
    db.Users.Add(user);
    await db.SaveChangesAsync();
    return Results.Created($"/api/users/{user.Id}", user);
});
users.MapPut("/{id}", async (int id, User input, AppDbContext db) => 
{
    var user = await db.Users.FindAsync(id);
    if (user is null) return Results.NotFound();
    user.Name = input.Name;
    user.Email = input.Email;
    await db.SaveChangesAsync();
    return Results.Ok(user);
});
users.MapDelete("/{id}", async (int id, AppDbContext db) => 
{
    var user = await db.Users.FindAsync(id);
    if (user is null) return Results.NotFound();
    db.Users.Remove(user);
    await db.SaveChangesAsync();
    return Results.NoContent();
});
```

**2. Route Groups & Organization:**
```csharp
var api = app.MapGroup("/api").RequireAuthorization();

var products = api.MapGroup("/products").WithTags("Products");
products.MapGet("/", GetProducts);
products.MapPost("/", CreateProduct).AddEndpointFilter<ValidationFilter<CreateProductDto>>();

var admin = api.MapGroup("/admin").RequireAuthorization("AdminPolicy");
admin.MapGet("/stats", GetStats);
```

**3. Endpoint Filters (Cross-cutting concerns):**
```csharp
// Validation Filter
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var dto = context.Arguments.OfType<T>().FirstOrDefault();
        if (dto is not null)
        {
            var validator = context.HttpContext.RequestServices.GetService<IValidator<T>>();
            if (validator is not null)
            {
                var result = await validator.ValidateAsync(dto);
                if (!result.IsValid)
                    return Results.ValidationProblem(result.ToDictionary());
            }
        }
        return await next(context);
    }
}

// Usage
app.MapPost("/api/users", CreateUser)
   .AddEndpointFilter<ValidationFilter<CreateUserDto>>();

// Built-in filters
app.MapGet("/api/users", GetUsers)
   .AddEndpointFilter(async (ctx, next) => 
   {
       // Logging, timing, caching
       var sw = Stopwatch.StartNew();
       var result = await next(ctx);
       sw.Stop();
       ctx.HttpContext.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
       return result;
   });
```

**4. Typed Results (Results<T>):**
```csharp
// Explicit return types for OpenAPI/Swagger
app.MapGet("/api/users/{id}", async (int id, AppDbContext db) => 
{
    var user = await db.Users.FindAsync(id);
    return user is not null 
        ? TypedResults.Ok(user) 
        : TypedResults.NotFound();
})
.Produces<User>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);
```

**5. OpenAPI/Swagger Integration:**
```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { Title = "My API", Version = "v1" });
    // XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    c.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));
});

// Annotations on endpoints
app.MapGet("/api/users/{id}", GetUser)
   .WithName("GetUser")
   .WithSummary("Gets a user by ID")
   .WithDescription("Returns a single user")
   .Produces<User>(200)
   .Produces(404);
```

#### Controller Patterns (Still Relevant)

**When to Use Controllers:**
- Complex business logic with many related actions
- Teams familiar with MVC patterns
- Need for `ControllerBase` utilities (`File()`, `Challenge()`, `View()`)
- Gradual migration from ASP.NET Framework

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _service;
    public UsersController(IUserService service) => _service = service;

    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<UserDto>), 200)]
    public async Task<ActionResult<IEnumerable<UserDto>>> GetAll()
        => Ok(await _service.GetAllAsync());

    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(UserDto), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<UserDto>> GetById(int id)
    {
        var user = await _service.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user);
    }

    [HttpPost]
    [ProducesResponseType(typeof(UserDto), 201)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<UserDto>> Create(CreateUserDto dto)
    {
        var user = await _service.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Minimal APIs** | Microservices, simple CRUD, high perf | Large controllers with 20+ actions | Harder to organize, discoverability |
| **Route Groups** | Organize by resource/feature | Flat MapGet/MapPost everywhere | Unmaintainable, no shared middleware |
| **Typed Results** | Public APIs, Swagger generation | `return Results.Ok()` everywhere | No OpenAPI metadata, poor DX |
| **Endpoint Filters** | Validation, logging, auth | Duplicating try/catch in every handler | Violates DRY, inconsistent errors |
| **Controller + Minimal** | Migration, hybrid apps | Mixing in same file randomly | Confusion, inconsistent patterns |
| **IResult Return** | Explicit status codes | `ActionResult<T>` in Minimal APIs | Not idiomatic, less control |

**Hybrid Approach (Recommended for Migration):**
```csharp
// Controllers for complex domains
app.MapControllers();  // Maps attribute-routed controllers

// Minimal APIs for simple/horizontal concerns
app.MapGet("/health", () => Results.Ok(new { Status = "Healthy" }));
app.MapGet("/version", () => Results.Ok(new { Version = "1.0.0" }));
```

---

### INTERVIEW QUESTIONS (Top 8)

#### 1. **Conceptual**: "When would you choose Minimal APIs over Controllers?"

**Answer:**
```
Minimal APIs:
✓ Microservices, serverless (cold start matters)
✓ Simple CRUD, few endpoints
✓ AOT compilation, Native AOT
✓ Team prefers functional style
✓ High throughput, low latency needs

Controllers:
✓ Complex domain logic, many related actions
✓ Team experienced with MVC
✓ Need ControllerBase features (View, File, Challenge)
✓ Gradual migration from ASP.NET Framework
✓ Better discoverability for large APIs
```

#### 2. **Code**: "Implement a Minimal API with validation, auth, and OpenAPI docs"

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

var api = app.MapGroup("/api").RequireAuthorization();

api.MapPost("/orders", async (CreateOrderDto dto, IOrderService service, IValidator<CreateOrderDto> validator) =>
{
    var result = await validator.ValidateAsync(dto);
    if (!result.IsValid) return Results.ValidationProblem(result.ToDictionary());
    
    var order = await service.CreateAsync(dto);
    return Results.Created($"/api/orders/{order.Id}", order);
})
.WithName("CreateOrder")
.Produces<OrderDto>(201)
.ProducesValidationProblem()
.Produces(401)
.AddEndpointFilter<ValidationFilter<CreateOrderDto>>(); // Alternative to inline validation
```

#### 3. **Design**: "How do you organize 50+ Minimal API endpoints?"

**Answer:**
```
1. Route Groups by Domain
   var orders = app.MapGroup("/api/orders").WithTags("Orders");
   var payments = app.MapGroup("/api/payments").WithTags("Payments");

2. Extension Methods per Feature
   public static class OrderEndpoints
   {
       public static void MapOrderEndpoints(this IEndpointRouteBuilder app) { ... }
   }
   // In Program.cs: app.MapOrderEndpoints();

3. Vertical Slice Architecture
   Each feature = folder with Endpoints, DTOs, Handlers, Validators

4. Shared Filters via Extension
   public static RouteHandlerBuilder WithValidation<T>(this RouteHandlerBuilder b) 
       where T : class => b.AddEndpointFilter<ValidationFilter<T>>();
```

#### 4. **Debugging**: "Minimal API returns 404 but route looks correct"

**Checklist:**
- [ ] `app.MapControllers()` vs `MapGet` - not mixing incorrectly
- [ ] Route group prefix applied correctly
- [ ] HTTP method matches (MapGet vs MapPost)
- [ ] Parameter binding - `{id:int}` vs `{id}` vs `[FromQuery]`
- [ ] `UseRouting()` / `UseEndpoints()` not needed in .NET 6+ (automatic)
- [ ] Check `app.MapGroup("/api")` - trailing slash matters for sub-paths

#### 5. **Performance**: "Why are Minimal APIs faster for cold start?"

**Answer:**
- **No Controller instantiation** - no reflection, no `ControllerActionDescriptor` creation
- **Source generators** - model binding code at compile time, not runtime reflection
- **Smaller dependency graph** - fewer services to resolve
- **AOT-friendly** - no dynamic code generation
- **Benchmarks**: ~40% faster startup, ~20% lower memory

#### 6. **Code**: "Write an endpoint filter for API key authentication"

```csharp
public class ApiKeyFilter : IEndpointFilter
{
    private const string ApiKeyHeader = "X-Api-Key";
    
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        if (!context.HttpContext.Request.Headers.TryGetValue(ApiKeyHeader, out var providedKey))
            return Results.Unauthorized();
        
        var config = context.HttpContext.RequestServices.GetRequiredService<IConfiguration>();
        var validKey = config["ApiKey"];
        
        if (!CryptographicOperations.FixedTimeEquals(
            System.Text.Encoding.UTF8.GetBytes(providedKey!),
            System.Text.Encoding.UTF8.GetBytes(validKey!)))
        {
            return Results.Unauthorized();
        }
        
        return await next(context);
    }
}

// Usage
app.MapGet("/api/admin/stats", GetStats)
   .AddEndpointFilter<ApiKeyFilter>()
   .RequireAuthorization(); // Can combine with JWT auth
```

#### 7. **System Design**: "Minimal API vs Controller for a high-traffic e-commerce checkout"

**Answer:**
```
Checkout = Critical path, complex flow
→ Use Controllers for:
  - Multiple related actions (GET cart, POST checkout, GET confirmation, POST webhook)
  - Complex model binding (nested DTOs, multiple payment methods)
  - ControllerBase utilities (Challenge for 3DS redirect, File for receipt PDF)
  - Better testability with mock ControllerContext

Minimal APIs for:
  - Health checks, webhooks, simple callbacks
  - High-throughput read endpoints (product catalog, search)
```

#### 8. **Migration**: "Migrate this Controller to Minimal API"

```csharp
// Before (Controller)
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll([FromQuery] PaginationParams p)
        => Ok(await _service.GetAllAsync(p));

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var p = await _service.GetByIdAsync(id);
        return p is null ? NotFound() : Ok(p);
    }
}

// After (Minimal API)
var products = app.MapGroup("/api/products").WithTags("Products");

products.MapGet("/", async (PaginationParams p, IProductService service) 
    => Results.Ok(await service.GetAllAsync(p)))
    .Produces<IEnumerable<ProductDto>>();

products.MapGet("/{id:int}", async (int id, IProductService service) 
{
    var p = await service.GetByIdAsync(id);
    return p is null ? Results.NotFound() : Results.Ok(p);
})
.Produces<ProductDto>()
.Produces(404);
```

---

### GOTCHAS & EDGE CASES

- [ ] **Parameter Binding Priority**: Route > Query > Header > Body (JSON)
- [ ] **CancellationToken** - Automatically bound, use in all async handlers
- [ ] **Route Constraints** - `{id:int}`, `{id:guid}`, `{slug:regex(^[a-z-]+$)}`
- [ ] **File Uploads** - `IFormFile` / `IFormFileCollection` auto-bound in Minimal APIs
- [ ] **Response Caching** - `app.UseResponseCaching()` + `[ResponseCache]` attribute (controllers) or custom middleware
- [ ] **Rate Limiting** - .NET 7+ `AddRateLimiter`, `MapRateLimiter`
- [ ] **ProblemDetails** - Use `Results.Problem()` for RFC 7807 compliance
- [ ] **AOT Compatibility** - Avoid `DynamicCode`, use source generators, `IEndpointFilter` over reflection
- [ ] **OpenAPI** - Minimal APIs need explicit `.Produces()` / `.ProducesProblem()` for good docs

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Vertical Slice Endpoint Pattern**
```csharp
// Features/Orders/CreateOrderEndpoint.cs
public static class CreateOrderEndpoint
{
    public static void MapCreateOrder(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (CreateOrderCommand command, ISender sender) =>
        {
            var result = await sender.Send(command);
            return result.IsSuccess ? Results.Created($"/api/orders/{result.Value}", result.Value)
                                    : Results.BadRequest(result.Error);
        })
        .WithName("CreateOrder")
        .Produces<OrderDto>(201)
        .ProducesValidationProblem()
        .RequireAuthorization();
    }
}

// In Program.cs
app.MapCreateOrder();
app.MapGetOrder();
app.MapUpdateOrder();
```

#### 2. **Result Pattern with Minimal API**
```csharp
public static class ResultExtensions
{
    public static IResult ToHttpResult<T>(this Result<T> result)
        => result.IsSuccess ? Results.Ok(result.Value) : Results.Problem(result.Error);

    public static IResult ToCreatedResult<T>(this Result<T> result, string location)
        => result.IsSuccess ? Results.Created(location, result.Value) : Results.Problem(result.Error);
}

// Usage
app.MapPost("/api/users", async (CreateUserDto dto, IUserService service) =>
{
    var result = await service.CreateAsync(dto);
    return result.ToCreatedResult($"/api/users/{result.Value?.Id}");
});
```

#### 3. **FluentValidation Integration**
```csharp
// Validator
public class CreateUserValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Password).NotEmpty().MinimumLength(8);
    }
}

// Registration
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Endpoint Filter (reusable)
public class FluentValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (validator is null) return await next(context);
        
        var dto = context.Arguments.OfType<T>().FirstOrDefault();
        if (dto is null) return await next(context);
        
        var result = await validator.ValidateAsync(dto);
        return result.IsValid ? await next(context) : Results.ValidationProblem(result.ToDictionary());
    }
}
```

---

### PRACTICE PROBLEMS

1. **Code**: Build a Minimal API for a Todo app with: CRUD, filtering, pagination, validation, auth
2. **Design**: Convert a 15-action Controller to Minimal APIs - what stays, what changes?
3. **Debug**: Minimal API returns 415 Unsupported Media Type - fix content type handling
4. **Performance**: Benchmark Controller vs Minimal API for 10K req/s - analyze differences
5. **System Design**: Design a webhook receiver using Minimal APIs with signature verification

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Minimal API parameter binding order | Route → Query → Header → Body (JSON) |
| How to add validation to Minimal API | `AddEndpointFilter<ValidationFilter<T>>()` or inline validator |
| Minimal API vs Controller: cold start | Minimal API ~40% faster (no controller activation, source gen binding) |
| Route Group purpose | Shared prefix, middleware, tags, auth - `app.MapGroup("/api")` |
| TypedResults vs Results | TypedResults: OpenAPI metadata. Results: runtime only |
| IEndpointFilter use cases | Validation, logging, timing, caching, auth - cross-cutting concerns |
| AOT-friendly Minimal API | Source generators, no reflection, `IEndpointFilter`, typed results |
| ProblemDetails in Minimal API | `Results.Problem(statusCode: 400, title: "Validation failed", detail: ...)` |

---

## NEXT TOPIC: `03-EF-CORE.md`

> **Study Tip**: Build a small Minimal API project (5 endpoints) from memory. Then add: validation filter, auth, OpenAPI docs, result pattern.