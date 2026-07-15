# CACHING IN ASP.NET CORE
## Exam-Style Study Notes

---

## TOPIC: Caching Strategies & Implementation

### WHY? (Problem Statement)

**Without Caching:**
- Every request hits database → High latency, high DB load
- Repeated computations → Wasted CPU
- External API calls → Rate limits, latency, costs
- No scalability → Can't handle traffic spikes

**What Caching Solves:**
| Problem | Cache Solution |
|---------|----------------|
| DB read load | In-memory / Distributed cache |
| Slow computations | Computed result caching |
| External API calls | Response caching |
| Latency | Sub-millisecond reads |
| Scalability | Reduce DB connections |

**Real-World Analogy:**
- No cache = Going to library for every fact you need
- Cache = Keeping reference books on your desk

---

### HOW? (Internal Mechanism)

#### 1. **Cache Abstraction Layers**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ICacheService (Your Interface)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐    ┌──────────────────────────────────┐  │
│  │  MemoryCache     │    │  DistributedCache (Redis)        │  │
│  │  (IMemoryCache)  │    │  (IDistributedCache)             │  │
│  ├──────────────────┤    ├──────────────────────────────────┤  │
│  │ • In-process     │    │ • Cross-process                  │  │
│  │ • Single server  │    │ • Multi-server                   │  │
│  │ • Fastest        │    │ • Survives restarts              │  │
│  │ • Lost on restart│    │ • Network latency                │  │
│  │ • No serialization│   │ • Serialization required         │  │
│  └──────────────────┘    └──────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Hybrid Cache (Memory + Distributed)                     │  │
│  │  • L1: MemoryCache (fast, local)                         │  │
│  │  • L2: Redis (shared, persistent)                        │  │
│  │  • Invalidation: Pub/Sub for L1 sync                     │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. **MemoryCache Internals**

```csharp
// Registration
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024 * 1024 * 100; // 100MB
    options.CompactionPercentage = 0.2;    // Compact when 80% full
    options.ExpirationScanFrequency = TimeSpan.FromMinutes(1);
});

// Usage
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly AppDbContext _db;
    
    public ProductService(IMemoryCache cache, AppDbContext db)
    {
        _cache = cache;
        _db = db;
    }
    
    public async Task<ProductDto> GetProductAsync(int id)
    {
        // TryGetValue pattern (atomic check + get)
        if (_cache.TryGetValue($"product:{id}", out ProductDto cached))
            return cached;
        
        // Cache miss - load from DB
        var product = await _db.Products.FindAsync(id);
        if (product == null) return null;
        
        var dto = _mapper.Map<ProductDto>(product);
        
        // Set with options
        _cache.Set($"product:{id}", dto, new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
            SlidingExpiration = TimeSpan.FromMinutes(2),
            Size = 1024, // For size limit
            Priority = CacheItemPriority.Normal,
            PostEvictionCallbacks = new List<PostEvictionCallbackRegistration>
            {
                new(key, value, reason, state) => 
                    _logger.LogDebug("Evicted: {Key}, Reason: {Reason}", key, reason)
            }
        });
        
        return dto;
    }
}
```

#### 3. **Distributed Cache (Redis) Internals**

```csharp
// Registration
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "Nanolite:";
});

// Or with more control
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
{
    var config = ConfigurationOptions.Parse(builder.Configuration.GetConnectionString("Redis"));
    config.AbortOnConnectFail = false;
    config.ConnectRetry = 3;
    config.ConnectTimeout = 5000;
    return ConnectionMultiplexer.Connect(config);
});

builder.Services.AddDistributedRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "Nanolite:";
});

// Usage (byte[] based)
public class DistributedProductService
{
    private readonly IDistributedCache _cache;
    private readonly AppDbContext _db;
    private readonly IMapper _mapper;
    
    public async Task<ProductDto> GetProductAsync(int id)
    {
        var key = $"product:{id}";
        var cached = await _cache.GetAsync(key);
        
        if (cached != null)
            return JsonSerializer.Deserialize<ProductDto>(cached);
        
        var product = await _db.Products.FindAsync(id);
        if (product == null) return null;
        
        var dto = _mapper.Map<ProductDto>(product);
        var serialized = JsonSerializer.SerializeToUtf8Bytes(dto);
        
        await _cache.SetAsync(key, serialized, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
            SlidingExpiration = TimeSpan.FromMinutes(5)
        });
        
        return dto;
    }
    
    public async Task InvalidateProductAsync(int id)
    {
        await _cache.RemoveAsync($"product:{id}");
    }
}
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Cache Patterns**

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Cache-Aside** | App checks cache, loads on miss | General purpose |
| **Read-Through** | Cache loads data automatically | Transparent caching |
| **Write-Through** | Write to cache + DB simultaneously | Consistency critical |
| **Write-Behind** | Write to cache, async flush to DB | High write throughput |
| **Refresh-Ahead** | Pre-load before expiry | Predictable access patterns |

#### 2. **Cache-Aside Pattern (Most Common)**

```csharp
public class CacheAsideService<T> where T : class
{
    private readonly IDistributedCache _cache;
    private readonly IRepository<T> _repo;
    private readonly JsonSerializerOptions _jsonOptions = new() { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };
    
    public async Task<T> GetOrAddAsync(string key, Func<Task<T>> factory, TimeSpan? ttl = null)
    {
        var cached = await _cache.GetAsync(key);
        if (cached != null)
            return JsonSerializer.Deserialize<T>(cached, _jsonOptions);
        
        var value = await factory();
        if (value != null)
        {
            var serialized = JsonSerializer.SerializeToUtf8Bytes(value, _jsonOptions);
            await _cache.SetAsync(key, serialized, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = ttl ?? TimeSpan.FromMinutes(10)
            });
        }
        return value;
    }
    
    public async Task RemoveAsync(string key) => await _cache.RemoveAsync(key);
    
    public async Task RefreshAsync(string key) => await _cache.RefreshAsync(key);
}
```

#### 3. **Response Caching (HTTP Level)**

```csharp
// Controller
[HttpGet("products")]
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any, VaryByHeader = "Accept")]
public async Task<ActionResult<IEnumerable<ProductDto>>> GetProducts()
{
    return Ok(await _svc.GetProductsAsync());
}

// Minimal API
app.MapGet("/api/products", async (IProductService svc) => await svc.GetProductsAsync())
   .CacheOutput(p => p.Expire(TimeSpan.FromMinutes(1)).Tag("products"));

// Output Caching (.NET 7+)
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(p => p.Expire(TimeSpan.FromSeconds(30)));
    options.AddPolicy("products", p => p.Expire(TimeSpan.FromMinutes(5)).Tag("products"));
    options.AddPolicy("product-detail", p => p.Expire(TimeSpan.FromMinutes(10)).VaryByRouteValue("id"));
});

app.UseOutputCache();

app.MapGet("/api/products", async (IProductService svc) => await svc.GetProductsAsync())
   .CacheOutput("products");

app.MapGet("/api/products/{id}", async (int id, IProductService svc) => await svc.GetProductAsync(id))
   .CacheOutput("product-detail");

// Invalidate on write
app.MapPost("/api/products", [InvalidateCacheOutput("products")] async (CreateProductDto dto, IProductService svc) =>
{
    return await svc.CreateProductAsync(dto);
});
```

#### 4. **Cache Invalidation Strategies**

```csharp
public class CacheInvalidationService
{
    private readonly IDistributedCache _cache;
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<CacheInvalidationService> _logger;
    
    // 1. Direct Invalidation
    public async Task InvalidateProductAsync(int productId)
    {
        await _cache.RemoveAsync($"product:{productId}");
        await _cache.RemoveAsync($"product:detail:{productId}");
    }
    
    // 2. Tag-Based Invalidation (Redis Sets)
    public async Task InvalidateByTagAsync(string tag)
    {
        var db = _redis.GetDatabase();
        var keys = await db.SetMembersAsync($"tag:{tag}");
        
        foreach (var key in keys)
        {
            await _cache.RemoveAsync(key.ToString());
        }
        await db.KeyDeleteAsync($"tag:{tag}");
    }
    
    // 3. Pattern Invalidation (SCAN - careful in production)
    public async Task InvalidatePatternAsync(string pattern)
    {
        var server = _redis.GetServer(_redis.GetEndPoints()[0]);
        var keys = server.Keys(pattern: $"Nanolite:{pattern}*").ToArray();
        
        if (keys.Length > 0)
        {
            var db = _redis.GetDatabase();
            await db.KeyDeleteAsync(keys);
        }
    }
    
    // 4. Pub/Sub for Distributed Invalidation (Multi-instance)
    public async Task PublishInvalidationAsync(string key)
    {
        var sub = _redis.GetSubscriber();
        await sub.PublishAsync("cache:invalidate", key);
    }
    
    // Subscribe in startup
    public void SubscribeToInvalidation()
    {
        var sub = _redis.GetSubscriber();
        sub.Subscribe("cache:invalidate", (channel, key) =>
        {
            _cache.RemoveAsync(key.ToString()).GetAwaiter().GetResult();
            _logger.LogInformation("Invalidated via Pub/Sub: {Key}", key);
        });
    }
}

// Tagging on Write
public async Task<ProductDto> UpdateProductAsync(int id, UpdateProductDto dto)
{
    var product = await _db.Products.FindAsync(id);
    // ... update ...
    await _db.SaveChangesAsync();
    
    // Invalidate
    await _cache.RemoveAsync($"product:{id}");
    await _cache.RemoveAsync($"product:detail:{id}");
    
    // Tag-based
    var db = _redis.GetDatabase();
    await db.SetAddAsync("tag:products", $"product:{id}");
    await db.SetAddAsync("tag:products", $"product:detail:{id}");
    
    return _mapper.Map<ProductDto>(product);
}
```

#### 5. **Hybrid Cache (Memory + Redis)**

```csharp
public class HybridCacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    private readonly IConnectionMultiplexer _redis;
    private readonly TimeSpan _localTtl = TimeSpan.FromMinutes(2);
    private readonly TimeSpan _remoteTtl = TimeSpan.FromMinutes(30);
    
    public async Task<T> GetOrAddAsync<T>(string key, Func<Task<T>> factory)
    {
        // L1: Memory Cache (fastest)
        if (_memoryCache.TryGetValue(key, out T localValue))
            return localValue;
        
        // L2: Redis
        var remoteValue = await _distributedCache.GetAsync(key);
        if (remoteValue != null)
        {
            var deserialized = JsonSerializer.Deserialize<T>(remoteValue);
            _memoryCache.Set(key, deserialized, _localTtl);
            return deserialized;
        }
        
        // Miss: Load from source
        var value = await factory();
        if (value != null)
        {
            var serialized = JsonSerializer.SerializeToUtf8Bytes(value);
            
            // Store in both
            await _distributedCache.SetAsync(key, serialized, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = _remoteTtl
            });
            _memoryCache.Set(key, value, _localTtl);
        }
        return value;
    }
    
    public async Task InvalidateAsync(string key)
    {
        // Remove from both
        _memoryCache.Remove(key);
        await _distributedCache.RemoveAsync(key);
        
        // Notify other instances via Pub/Sub
        var sub = _redis.GetSubscriber();
        await sub.PublishAsync("cache:invalidate", key);
    }
}
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Cache-Aside** | Read-heavy, eventual consistency OK | Cache-Through for everything | Complexity, consistency issues |
| **TTL + Sliding** | Unpredictable access | No expiration | Memory leaks, stale data |
| **Cache Invalidation on Write** | Data accuracy critical | TTL only for critical data | Stale reads |
| **Serialization: JSON** | Cross-language, debuggable | BinaryFormatter | Security, versioning |
| **Key Prefixing** | Multi-tenant, multi-app | Raw keys | Collisions, debugging hard |
| **Circuit Breaker** | Redis unavailable | No fallback | Cascade failures |

---

### INTERVIEW QUESTIONS (Top 10)

#### 1. **Conceptual**: "Explain the difference between IMemoryCache and IDistributedCache"

**Answer:**
```
IMemoryCache:
- In-process, single server
- Stores objects directly (no serialization)
- Fastest (no network)
- Lost on app restart
- No built-in synchronization across instances

IDistributedCache:
- External (Redis, SQL Server, NCache)
- Byte[] only (serialization required)
- Network latency (~1-5ms)
- Survives restarts
- Shared across multiple app instances
- Supports atomic operations

When to use which:
- Single instance, small data, maximum speed → MemoryCache
- Multiple instances, shared data, persistence needed → DistributedCache
- Best of both → Hybrid (L1 Memory + L2 Redis)
```

#### 2. **Code**: "Implement a thread-safe GetOrAdd for MemoryCache"

```csharp
public static class MemoryCacheExtensions
{
    // Uses Lazy<T> for thread-safe factory execution
    public static T GetOrAdd<T>(this IMemoryCache cache, string key, Func<T> factory, MemoryCacheEntryOptions options = null)
    {
        if (cache.TryGetValue(key, out T value))
            return value;
        
        // Lazy ensures factory runs only once even under concurrent access
        var lazy = new Lazy<T>(factory, LazyThreadSafetyMode.ExecutionAndPublication);
        
        // Try to add the Lazy itself (not the value)
        if (cache.TryGetValue(key, out Lazy<T> existingLazy))
        {
            return existingLazy.Value; // Another thread won
        }
        
        cache.Set(key, lazy, options ?? new MemoryCacheEntryOptions());
        return lazy.Value;
    }
    
    // Async version
    public static async Task<T> GetOrAddAsync<T>(this IMemoryCache cache, string key, Func<Task<T>> factory, MemoryCacheEntryOptions options = null)
    {
        if (cache.TryGetValue(key, out T value))
            return value;
        
        var lazy = new Lazy<Task<T>>(factory, LazyThreadSafetyMode.ExecutionAndPublication);
        
        if (cache.TryGetValue(key, out Lazy<Task<T>> existingLazy))
        {
            return await existingLazy.Value;
        }
        
        cache.Set(key, lazy, options ?? new MemoryCacheEntryOptions());
        return await lazy.Value;
    }
}
```

#### 3. **Design**: "How to handle cache stampede (thundering herd)?"

**Answer:**
```
Problem: Cache expires → 1000 requests hit DB simultaneously

Solutions:
1. **Probabilistic Early Expiration**
   - Randomize TTL: baseTTL + random(0, 10%)
   - Not all keys expire at once

2. **Lock-Based (Mutex)**
   - Only one request rebuilds, others wait
   - Use Redis SETNX or SemaphoreSlim

3. **Stale-While-Revalidate**
   - Serve stale data while rebuilding in background
   - Return stale immediately, async refresh

4. **Redis Lua Script (Atomic Check-And-Set)**
   EVAL "if redis.call('exists', KEYS[1]) == 0 then redis.call('set', KEYS[1], ARGV[1], 'EX', ARGV[2]) return 1 else return 0 end" 1 key value ttl

Implementation:
public async Task<T> GetOrAddWithLockAsync<T>(string key, Func<Task<T>> factory, TimeSpan ttl)
{
    // Try fast path
    var cached = await _cache.GetAsync(key);
    if (cached != null) return Deserialize<T>(cached);
    
    // Acquire distributed lock
    var lockKey = $"lock:{key}";
    var lockValue = Guid.NewGuid().ToString();
    var acquired = await _redis.GetDatabase().StringSetAsync(lockKey, lockValue, TimeSpan.FromSeconds(30), When.NotExists);
    
    if (!acquired)
    {
        // Wait and retry (or serve stale)
        await Task.Delay(50);
        return await GetOrAddWithLockAsync(key, factory, ttl); // Retry
    }
    
    try
    {
        // Double-check after lock
        cached = await _cache.GetAsync(key);
        if (cached != null) return Deserialize<T>(cached);
        
        var value = await factory();
        await _cache.SetAsync(key, Serialize(value), new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = ttl });
        return value;
    }
    finally
    {
        // Release lock (only if we own it)
        var script = LuaScript.Prepare("if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end");
        await _redis.GetDatabase().ScriptEvaluateAsync(script, new[] { (RedisKey)lockKey }, new[] { (RedisValue)lockValue });
    }
}
```

#### 4. **Performance**: "Cache serialization - JSON vs MessagePack vs Protobuf"

```csharp
// Benchmark considerations:
// JSON: Human readable, slower, larger
// MessagePack: Binary, faster, smaller, schema-less
// Protobuf: Fastest, smallest, requires .proto schema

public class CacheSerializer
{
    // JSON (Default)
    public byte[] SerializeJson<T>(T obj) => JsonSerializer.SerializeToUtf8Bytes(obj);
    public T DeserializeJson<T>(byte[] data) => JsonSerializer.Deserialize<T>(data);
    
    // MessagePack (NuGet: MessagePack)
    public byte[] SerializeMsgPack<T>(T obj) => MessagePackSerializer.Serialize(obj);
    public T DeserializeMsgPack<T>(byte[] data) => MessagePackSerializer.Deserialize<T>(data);
    
    // Protobuf (NuGet: Google.Protobuf)
    // Requires [ProtoContract] attributes
}

// Recommendation:
// - Internal cache: MessagePack (fast, no schema management)
// - Cross-service: Protobuf (contract-first)
// - Debugging: JSON
```

#### 5. **Code**: "Implement Output Caching with ETags"

```csharp
public class ETagOutputCacheFilter : IAsyncActionFilter
{
    private readonly IDistributedCache _cache;
    
    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var request = context.HttpContext.Request;
        var response = context.HttpContext.Response;
        
        // Generate cache key from route + query
        var cacheKey = $"etag:{request.Path}{request.QueryString}";
        
        // Check If-None-Match header
        var ifNoneMatch = request.Headers.IfNoneMatch.FirstOrDefault();
        
        // Try get cached ETag
        var cachedETag = await _cache.GetStringAsync(cacheKey + ":etag");
        
        if (!string.IsNullOrEmpty(ifNoneMatch) && ifNoneMatch == cachedETag)
        {
            context.Result = new StatusCodeResult(304); // Not Modified
            return;
        }
        
        // Execute action
        var executedContext = await next();
        
        if (executedContext.Result is ObjectResult objectResult && objectResult.Value != null)
        {
            // Generate ETag from content hash
            var content = JsonSerializer.Serialize(objectResult.Value);
            var etag = $"\"{ComputeHash(content)}\"";
            
            // Store ETag and content
            await _cache.SetStringAsync(cacheKey + ":etag", etag, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            });
            
            response.Headers.ETag = etag;
            response.Headers.CacheControl = "public, max-age=600";
        }
    }
    
    private static string ComputeHash(string input)
    {
        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToBase64String(hash)[..16];
    }
}
```

#### 6. **Architecture**: "Design a caching layer for a product catalog with 10M products"

```csharp
// Multi-level caching strategy
public class ProductCatalogCache
{
    private readonly IMemoryCache _l1Cache;      // Hot products (top 1%)
    private readonly IDistributedCache _l2Cache; // All products
    private readonly IConnectionMultiplexer _redis;
    private readonly IProductRepository _repo;
    
    // L1: 5 min TTL, 10K entries max (hot products)
    // L2: 1 hour TTL, full catalog
    // DB: Source of truth
    
    public async Task<ProductDto> GetProductAsync(int id)
    {
        // L1 Check
        if (_l1Cache.TryGetValue($"p:{id}", out ProductDto l1Hit))
            return l1Hit;
        
        // L2 Check
        var l2Data = await _l2Cache.GetAsync($"p:{id}");
        if (l2Data != null)
        {
            var product = Deserialize<ProductDto>(l2Data);
            _l1Cache.Set($"p:{id}", product, TimeSpan.FromMinutes(5));
            return product;
        }
        
        // DB Fallback
        var product = await _repo.GetByIdAsync(id);
        if (product != null)
        {
            var dto = MapToDto(product);
            var serialized = Serialize(dto);
            
            // Populate both levels
            await _l2Cache.SetAsync($"p:{id}", serialized, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            });
            _l1Cache.Set($"p:{id}", dto, TimeSpan.FromMinutes(5));
        }
        return product;
    }
    
    public async Task InvalidateProductAsync(int id)
    {
        _l1Cache.Remove($"p:{id}");
        await _l2Cache.RemoveAsync($"p:{id}");
        
        // Invalidate list caches
        await _l2Cache.RemoveAsync("products:list:all");
        await _l2Cache.RemoveAsync("products:featured");
        
        // Pub/Sub to other instances
        await _redis.GetSubscriber().PublishAsync("catalog:invalidate", $"p:{id}");
    }
    
    // Pre-warm hot products on startup
    public async Task WarmupAsync()
    {
        var hotProducts = await _repo.GetTopSellingAsync(1000);
        foreach (var p in hotProducts)
        {
            var dto = MapToDto(p);
            _l1Cache.Set($"p:{p.Id}", dto, TimeSpan.FromMinutes(10));
            await _l2Cache.SetAsync($"p:{p.Id}", Serialize(dto), 
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(2) });
        }
    }
}
```

#### 7. **Debugging**: "Cache shows stale data - troubleshooting checklist"

```
☐ Check TTL configuration (Absolute vs Sliding)
☐ Verify invalidation on write paths (Create/Update/Delete)
☐ Check for multiple cache instances (dev vs prod Redis)
☐ Verify serialization/deserialization round-trip
☐ Check for cached nulls (cache penetration)
☐ Verify cache key uniqueness (tenant, user, params)
☐ Check distributed cache sync (Pub/Sub working?)
☐ Monitor cache hit ratio (should be >80% for read-heavy)
☐ Check for clock skew between servers
☐ Verify cache warming on deployment
```

#### 8. **Code**: "Implement cache-aside with fallback and metrics"

```csharp
public class InstrumentedCacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<InstrumentedCacheService> _logger;
    private readonly Meter _meter = new("Nanolite.Caching");
    private readonly Counter<long> _hits;
    private readonly Counter<long> _misses;
    private readonly Counter<long> _errors;
    
    public InstrumentedCacheService(IDistributedCache cache, ILogger<InstrumentedCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
        _hits = _meter.CreateCounter<long>("cache.hits");
        _misses = _meter.CreateCounter<long>("cache.misses");
        _errors = _meter.CreateCounter<long>("cache.errors");
    }
    
    public async Task<T> GetOrAddAsync<T>(string key, Func<Task<T>> factory, TimeSpan? ttl = null)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var cached = await _cache.GetAsync(key);
            if (cached != null)
            {
                _hits.Add(1);
                _logger.LogDebug("Cache HIT: {Key} in {Elapsed}ms", key, stopwatch.ElapsedMilliseconds);
                return Deserialize<T>(cached);
            }
            
            _misses.Add(1);
            _logger.LogDebug("Cache MISS: {Key}", key);
            
            var value = await factory();
            if (value != null)
            {
                await _cache.SetAsync(key, Serialize(value), new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = ttl ?? TimeSpan.FromMinutes(10)
                });
            }
            
            return value;
        }
        catch (Exception ex)
        {
            _errors.Add(1);
            _logger.LogError(ex, "Cache error for key: {Key}", key);
            
            // Fallback: execute factory directly (graceful degradation)
            _logger.LogWarning("Cache unavailable, falling back to source");
            return await factory();
        }
    }
}
```

#### 9. **Security**: "Prevent cache poisoning and data leakage"

```csharp
public class SecureCacheService
{
    // 1. Key Isolation - prevent key collision attacks
    public string GetSecureKey(string userId, string resourceType, string resourceId)
    {
        // Use HMAC or namespace prefix
        return $"user:{userId}:{resourceType}:{resourceId}";
    }
    
    // 2. Encrypt sensitive data in cache
    public async Task SetEncryptedAsync<T>(string key, T value, TimeSpan ttl, IDataProtector protector)
    {
        var json = JsonSerializer.Serialize(value);
        var encrypted = protector.Protect(json);
        await _cache.SetAsync(key, Encoding.UTF8.GetBytes(encrypted), new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl
        });
    }
    
    public async Task<T> GetDecryptedAsync<T>(string key, IDataProtector protector)
    {
        var data = await _cache.GetAsync(key);
        if (data == null) return default;
        
        var encrypted = Encoding.UTF8.GetString(data);
        var json = protector.Unprotect(encrypted);
        return JsonSerializer.Deserialize<T>(json);
    }
    
    // 3. Cache penetration protection (null caching)
    public async Task<T> GetOrAddWithNullCacheAsync<T>(string key, Func<Task<T>> factory, TimeSpan ttl)
    {
        var cached = await _cache.GetAsync(key);
        if (cached != null)
        {
            if (cached.Length == 0) // Special marker for null
                return default;
            return Deserialize<T>(cached);
        }
        
        var value = await factory();
        if (value == null)
        {
            // Cache empty result with shorter TTL to prevent repeated DB hits
            await _cache.SetAsync(key, Array.Empty<byte>(), new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1)
            });
            return default;
        }
        
        await _cache.SetAsync(key, Serialize(value), new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl
        });
        return value;
    }
}
```

#### 10. **Testing**: "Unit test cache logic without real Redis"

```csharp
public class CacheServiceTests
{
    private readonly Mock<IDistributedCache> _mockCache;
    private readonly CacheService _service;
    
    public CacheServiceTests()
    {
        _mockCache = new Mock<IDistributedCache>();
        _service = new CacheService(_mockCache.Object);
    }
    
    [Fact]
    public async Task GetAsync_ReturnsCachedValue_WhenExists()
    {
        // Arrange
        var key = "test-key";
        var expected = new ProductDto { Id = 1, Name = "Test" };
        var serialized = JsonSerializer.SerializeToUtf8Bytes(expected);
        
        _mockCache.Setup(x => x.GetAsync(key, It.IsAny<CancellationToken>()))
                  .ReturnsAsync(serialized);
        
        // Act
        var result = await _service.GetAsync<ProductDto>(key);
        
        // Assert
        Assert.Equal(expected.Id, result.Id);
        Assert.Equal(expected.Name, result.Name);
    }
    
    [Fact]
    public async Task GetAsync_CallsFactoryAndCaches_WhenMiss()
    {
        // Arrange
        var key = "test-key";
        var expected = new ProductDto { Id = 1, Name = "Test" };
        
        _mockCache.Setup(x => x.GetAsync(key, It.IsAny<CancellationToken>()))
                  .ReturnsAsync((byte[])null);
        
        byte[] capturedBytes = null;
        _mockCache.Setup(x => x.SetAsync(key, It.IsAny<byte[]>(), It.IsAny<DistributedCacheEntryOptions>(), It.IsAny<CancellationToken>()))
                  .Callback<string, byte[], DistributedCacheEntryOptions, CancellationToken>((k, v, o, c) => capturedBytes = v)
                  .Returns(Task.CompletedTask);
        
        // Act
        var result = await _service.GetOrAddAsync(key, () => Task.FromResult(expected));
        
        // Assert
        Assert.Equal(expected.Id, result.Id);
        Assert.NotNull(capturedBytes);
        
        var cached = JsonSerializer.Deserialize<ProductDto>(capturedBytes);
        Assert.Equal(expected.Name, cached.Name);
    }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Sliding vs Absolute**: Sliding resets on access; Absolute is fixed. Combine both.
- [ ] **Cache Nulls**: Cache penetration (repeated null queries) → Cache empty marker with short TTL
- [ ] **Key Collision**: Use prefixes (`app:module:key`), hash long keys
- [ ] **Serialization**: Version tolerance, handle schema changes
- [ ] **Memory Pressure**: Set SizeLimit on MemoryCache, monitor eviction
- [ ] **Distributed Lock**: Always use TryFinally, handle lock expiry during long ops
- [ ] **Redis Connection**: Pool connections, handle reconnection, don't create per request
- [ ] **Cache Stampede**: Use probabilistic early expiry or locks
- [ ] **Consistency**: Eventual consistency is mandatory - don't assume strong consistency
- [ ] **Monitoring**: Hit ratio, latency, memory, evictions, errors

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Caching Service**
```csharp
public interface ICacheService
{
    Task<T> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? ttl = null);
    Task RemoveAsync(string key);
    Task<T> GetOrAddAsync<T>(string key, Func<Task<T>> factory, TimeSpan? ttl = null);
    Task InvalidateByPrefixAsync(string prefix);
}

public class RedisCacheService : ICacheService
{
    private readonly IDistributedCache _cache;
    private readonly IConnectionMultiplexer _redis;
    private readonly JsonSerializerOptions _options = new() { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };
    
    public async Task<T> GetAsync<T>(string key)
    {
        var data = await _cache.GetAsync(key);
        return data == null ? default : JsonSerializer.Deserialize<T>(data, _options);
    }
    
    public async Task SetAsync<T>(string key, T value, TimeSpan? ttl = null)
    {
        var data = JsonSerializer.SerializeToUtf8Bytes(value, _options);
        await _cache.SetAsync(key, data, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl ?? TimeSpan.FromMinutes(10)
        });
    }
    
    public async Task RemoveAsync(string key) => await _cache.RemoveAsync(key);
    
    public async Task<T> GetOrAddAsync<T>(string key, Func<Task<T>> factory, TimeSpan? ttl = null)
    {
        var cached = await GetAsync<T>(key);
        if (cached != null) return cached;
        
        var value = await factory();
        if (value != null) await SetAsync(key, value, ttl);
        return value;
    }
    
    public async Task InvalidateByPrefixAsync(string prefix)
    {
        var server = _redis.GetServer(_redis.GetEndPoints()[0]);
        var keys = server.Keys(pattern: $"{prefix}*").ToArray();
        if (keys.Length > 0)
        {
            var db = _redis.GetDatabase();
            await db.KeyDeleteAsync(keys);
        }
    }
}
```

#### 2. **Output Cache Invalidation**
```csharp
// Tag-based invalidation
app.MapPost("/api/products", async (CreateProductDto dto, IProductService svc, IOutputCacheStore cache) =>
{
    var product = await svc.CreateAsync(dto);
    await cache.EvictByTagAsync("products", CancellationToken.None);
    return Results.Created($"/api/products/{product.Id}", product);
});

// Custom cache key
app.MapGet("/api/products", async (IProductService svc, HttpContext http) =>
{
    var userId = http.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? "anonymous";
    var cacheKey = $"products:user:{userId}";
    
    // Or use OutputCache with custom key
    http.Response.Headers["Cache-Key"] = cacheKey;
    
    return await svc.GetProductsAsync();
}).CacheOutput(p => p.VaryByHeader("Authorization"));
```

#### 3. **Redis Connection Resilience**
```csharp
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
{
    var config = ConfigurationOptions.Parse(builder.Configuration.GetConnectionString("Redis"));
    config.AbortOnConnectFail = false;
    config.ConnectRetry = 3;
    config.ReconnectRetryPolicy = new ExponentialRetry(5000);
    config.KeepAlive = 180;
    config.DefaultVersion = new Version(7, 0);
    
    var muxer = ConnectionMultiplexer.Connect(config);
    
    // Logging
    muxer.ConnectionFailed += (sender, e) => 
        sp.GetRequiredService<ILogger<Program>>().LogError(e.Exception, "Redis connection failed");
    muxer.ConnectionRestored += (sender, e) => 
        sp.GetRequiredService<ILogger<Program>>().LogInformation("Redis connection restored");
    muxer.ErrorMessage += (sender, e) => 
        sp.GetRequiredService<ILogger<Program>>().LogError(e.Message);
    
    return muxer;
});
```

---

### PRACTICE PROBLEMS

1. **Implement**: Multi-level cache (Memory + Redis) with Pub/Sub invalidation
2. **Design**: Cache strategy for user timeline (write-heavy, read-heavy)
3. **Optimize**: Reduce cache serialization overhead for 100K req/sec
4. **Debug**: Cache hit ratio 40% - identify and fix root causes
5. **Secure**: Implement encrypted cache for PII data with key rotation

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| IMemoryCache vs IDistributedCache | In-process vs external. Objects vs byte[]. Single vs multi-server |
| Cache-Aside Pattern | Check cache → Miss → Load from source → Store in cache → Return |
| Cache Stampede | Thundering herd on expiry. Fix: Lock, Probabilistic TTL, Stale-While-Revalidate |
| Sliding vs Absolute Expiration | Sliding: resets on access. Absolute: fixed. Use both. |
| Redis Serialization | JSON (debug), MessagePack (perf), Protobuf (contract) |
| Cache Invalidation Strategies | Direct (key), Tag-based, Pattern (SCAN), Pub/Sub (distributed) |
| Hybrid Cache | L1: MemoryCache (fast, local). L2: Redis (shared, persistent). Pub/Sub sync |
| Output Caching | HTTP-level. Cache-Control, ETag, Vary. .NET 7+: AddOutputCache() |
| Cache Penetration | Query non-existent keys → DB hit. Fix: Cache null with short TTL |
| Cache Avalanche | Mass expiry → DB overload. Fix: Randomized TTL, Pre-warm |

---

## NEXT TOPIC: `07-REALTIME-SIGNALR.md`

> **Study Tip**: Build a cached product service with: L1 MemoryCache (2min), L2 Redis (30min), tag-based invalidation, metrics (hit/miss), and unit tests with mock IDistributedCache.