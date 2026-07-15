# ENTITY FRAMEWORK CORE
## Exam-Style Study Notes

---

## TOPIC: Entity Framework Core Deep Dive

### WHY? (Problem Statement)

**Before EF Core:**
- **ADO.NET**: Raw SQL, manual mapping, SQL injection risk, verbose
- **EF6**: Windows-only, heavy, no async, tied to .NET Framework
- **Dapper**: Micro-ORM, manual SQL, no change tracking, no LINQ

**What EF Core Solves:**
| Problem | EF Core Solution |
|---------|------------------|
| Cross-platform | .NET Core / .NET 5+ (Windows, Linux, macOS) |
| Performance | Compiled queries, batching, pooling, interceptors |
| Flexibility | Code-First, Database-First, Schema migrations |
| Testability | InMemory provider, SQLite in-memory for unit tests |
| Modern .NET | Async/await, DI, source generators, AOT support |
| Cloud-native | Azure SQL, Cosmos DB, PostgreSQL, MySQL providers |

**Real-World Analogy:**
- ADO.NET = Writing raw SQL letters by hand
- EF6 = Old translation service (slow, Windows-only)
- EF Core = Modern AI translator (fast, cross-platform, learns your patterns)

---

### HOW? (Internal Mechanism)

#### 1. **DbContext Lifecycle & Change Tracking**

```
┌─────────────────────────────────────────────────────────────────┐
│                        DbContext Scope                          │
│  (Typically Scoped per HTTP Request)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐    ┌──────────────┐    ┌────────────────────┐   │
│  │  Query   │───►│ Change Tracker │───►│  SaveChanges()   │   │
│  │  (LINQ)  │    │  (Identity Map)│    │  (Unit of Work)  │   │
│  └──────────┘    └──────────────┘    └────────────────────┘   │
│       │                │                        │              │
│       ▼                ▼                        ▼              │
│  Translation      Entity State            SQL Generation      │
│  (IQueryable)     (Added/Modified/         (Batching,          │
│                   Deleted/Unchanged)        Parameters)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Change Tracker Internals:**
```csharp
// When you query:
var user = await db.Users.FindAsync(1);
// 1. Check Identity Map (Dictionary<EntityKey, EntityEntry>)
// 2. If not found: Execute SQL, materialize, attach to tracker
// 3. Return tracked entity (snapshot stored for change detection)

// When you modify:
user.Name = "New Name";
// 1. Property setter detected via proxy or snapshot comparison
// 2. EntityEntry.State = Modified
// 3. Original values preserved in tracker

// When SaveChanges():
// 1. DetectChanges() - compare current vs snapshot
// 2. Build DML (INSERT/UPDATE/DELETE) per entity
// 3. Execute in transaction (default)
// 4. Update snapshots, reset state to Unchanged
```

#### 2. **Query Pipeline (LINQ → SQL)**

```csharp
var query = db.Users
    .Where(u => u.IsActive)
    .OrderBy(u => u.CreatedAt)
    .Take(10)
    .Select(u => new { u.Id, u.Name });

// Pipeline:
IQueryable<User> 
    │
    ├─► Expression Tree (Expression<Func<User, bool>>)
    │       │
    │       ▼
    ├─► Query Compilation (cached per query shape)
    │       │
    │       ▼
    ├─► Relational Command Tree (provider-specific)
    │       │
    │       ▼
    ├─► SQL Generation (dialect-specific: T-SQL, PostgreSQL, etc.)
    │       │
    │       ▼
    └─► Execution (DbDataReader → Materialization → Tracking)
```

**Key Performance Points:**
- **Query Plan Caching**: Parameterized queries reuse plans
- **Compiled Queries**: `EF.CompileQuery` for hot paths
- **Client vs Server Evaluation**: EF Core 3.0+ throws on client eval (mostly)

#### 3. **Migration System**

```
dotnet ef migrations add InitialCreate
        │
        ▼
┌───────────────────┐
│  Snapshot Model   │  ← Current model state (C#)
│  (ModelSnapshot)  │
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  Migration Class  │  ← Up() / Down() methods
│  (Timestamp.cs)   │
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  __EFMigrationsHistory │  ← Applied migrations tracking table
└───────────────────┘
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Configuration Patterns**

**Fluent API (Recommended):**
```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Entity configuration
        modelBuilder.Entity<User>(entity =>
        {
            entity.ToTable("Users", "dbo");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Id).ValueGeneratedOnAdd();
            entity.Property(e => e.Email).IsRequired().HasMaxLength(256);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(100);
            entity.HasIndex(e => e.Email).IsUnique();
            
            // Owned type (value object)
            entity.OwnsOne(e => e.Address, addr =>
            {
                addr.Property(a => a.Street).HasMaxLength(200);
                addr.Property(a => a.City).HasMaxLength(100);
            });
            
            // Shadow property
            entity.Property<DateTime>("CreatedAt");
        });

        // Relationships
        modelBuilder.Entity<Order>()
            .HasOne(o => o.User)
            .WithMany(u => u.Orders)
            .HasForeignKey(o => o.UserId)
            .OnDelete(DeleteBehavior.Cascade);

        // Global query filter (soft delete)
        modelBuilder.Entity<User>()
            .HasQueryFilter(u => !u.IsDeleted);
    }
}
```

**Entity Type Configuration (Separate Classes - Clean Architecture):**
```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.ToTable("Users");
        builder.HasKey(u => u.Id);
        builder.Property(u => u.Email).IsRequired().HasMaxLength(256);
        builder.HasIndex(u => u.Email).IsUnique();
        builder.HasQueryFilter(u => !u.IsDeleted);
    }
}

// In OnModelCreating:
modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
```

#### 2. **Relationships Deep Dive**

```csharp
// One-to-One (Principal → Dependent)
modelBuilder.Entity<Profile>()
    .HasOne(p => p.User)
    .WithOne(u => u.Profile)
    .HasForeignKey<Profile>(p => p.UserId);  // FK on dependent

// One-to-Many (Standard)
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer)
    .WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId);

// Many-to-Many (EF Core 5+ implicit join entity)
modelBuilder.Entity<Post>()
    .HasMany(p => p.Tags)
    .WithMany(t => t.Posts)
    .UsingEntity<PostTag>(  // Explicit join entity for payload
        j => j.HasOne(pt => pt.Tag).WithMany().HasForeignKey(pt => pt.TagId),
        j => j.HasOne(pt => pt.Post).WithMany().HasForeignKey(pt => pt.PostId),
        j => j.HasKey(pt => new { pt.PostId, pt.TagId }));

// Self-Referencing (Hierarchy)
modelBuilder.Entity<Category>()
    .HasOne(c => c.Parent)
    .WithMany(c => c.Children)
    .HasForeignKey(c => c.ParentId)
    .OnDelete(DeleteBehavior.Restrict);
```

#### 3. **Advanced Querying**

```csharp
// Eager Loading (Include/ThenInclude)
var orders = await db.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items).ThenInclude(i => i.Product)
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();

// Explicit Loading (when you don't want to include)
var customer = await db.Customers.FindAsync(id);
await db.Entry(customer).Collection(c => c.Orders).LoadAsync();
await db.Entry(customer).Reference(c => c.Profile).LoadAsync();

// Select (Projection) - Avoids tracking, better performance
var dto = await db.Users
    .Where(u => u.IsActive)
    .Select(u => new UserDto(u.Id, u.Name, u.Email, u.Orders.Count))
    .ToListAsync();

// Split Query (EF Core 5+) - Avoids Cartesian explosion
var data = await db.Customers
    .Include(c => c.Orders)
    .Include(c => c.Addresses)
    .AsSplitQuery()  // Generates 3 separate queries
    .ToListAsync();

// Raw SQL (when LINQ isn't enough)
var users = await db.Users
    .FromSqlRaw("SELECT * FROM Users WHERE IsActive = 1")
    .ToListAsync();

// Parameterized (safe)
var users = await db.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {name}")
    .ToListAsync();

// Stored Procedure
var result = await db.Database
    .SqlQueryRaw<OrderSummary>("EXEC GetOrderSummary @UserId = {0}", userId)
    .ToListAsync();
```

#### 4. **Performance Patterns**

```csharp
// 1. AsNoTracking - Read-only queries
var users = await db.Users.AsNoTracking().ToListAsync();

// 2. Identity Resolution (Default) vs NoTrackingWithIdentityResolution
var users = await db.Users
    .AsNoTrackingWithIdentityResolution()
    .ToListAsync();  // Same instance for same entity

// 3. Batch Updates/Deletes (EF Core 7+)
await db.Users
    .Where(u => u.LastLogin < DateTime.UtcNow.AddYears(-1))
    .ExecuteUpdateAsync(u => u.SetProperty(x => x.IsActive, false));

await db.Users
    .Where(u => u.IsDeleted)
    .ExecuteDeleteAsync();

// 4. Compiled Queries (Hot paths)
private static readonly Func<AppDbContext, int, Task<User?>> _getUserById =
    EF.CompileAsyncQuery((AppDbContext db, int id) => 
        db.Users.FirstOrDefault(u => u.Id == id));

// Usage
var user = await _getUserById(db, 123);

// 5. Connection Resiliency (Transient failures)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null);
        sqlOptions.CommandTimeout(30);
    }));

// 6. DbContext Pooling (High throughput)
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString), poolSize: 128);
```

#### 5. **Concurrency Handling**

```csharp
// Optimistic Concurrency (RowVersion / Timestamp)
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    [Timestamp]  // byte[] - SQL Server rowversion
    public byte[] RowVersion { get; set; }
}

// Usage - Automatic conflict detection
try
{
    product.Price = 99.99m;
    await db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Reload and retry, or show conflict to user
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync();
    if (databaseValues == null)
        throw new Exception("Entity deleted by another user");
    
    // Merge or reject
    entry.OriginalValues.SetValues(databaseValues);
    await db.SaveChangesAsync();
}

// Pessimistic Concurrency (Locking - rare)
using var transaction = await db.Database.BeginTransactionAsync();
var product = await db.Products
    .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}", id)
    .FirstAsync();
// Modify...
await db.SaveChangesAsync();
await transaction.CommitAsync();
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Repository + Unit of Work** | Complex domains, testability | DbContext directly in controllers | Tight coupling, hard to test |
| **Specification Pattern** | Complex dynamic queries | Giant `Where` clauses in services | Unmaintainable, not reusable |
| **CQRS with EF Core** | Read/write scaling | Single model for everything | Read model != Write model |
| **Async All The Way** | All DB operations | `.Result` / `.Wait()` on async | Deadlocks, thread pool starvation |
| **AsNoTracking for Reads** | List/Detail views | Tracking everything | Memory pressure, change detection overhead |
| **Bulk Operations** | Large datasets | Loop + SaveChanges | N+1 round trips, slow |
| **Global Query Filters** | Soft delete, multi-tenant | Manual `.Where(x => !x.IsDeleted)` | Forgetting filter = data leaks |

---

### INTERVIEW QUESTIONS (Top 10)

#### 1. **Conceptual**: "Explain EF Core Change Tracking"

**Answer:**
```
Change Tracker = Identity Map + State Manager per DbContext

Flow:
1. Query → Entity materialized → Attached to tracker (Unchanged) + Snapshot stored
2. Property change → Detected via proxy (virtual) or snapshot comparison → State = Modified
3. Add/Remove → State = Added/Deleted
4. SaveChanges() → DetectChanges() → Generate DML → Execute in transaction → Update snapshots

States: Detached → Added/Unchanged/Modified/Deleted

Key: DbContext is NOT thread-safe. Scoped per request.
```

#### 2. **Code**: "Fix the N+1 problem in this code"

```csharp
// BAD - N+1 queries
var orders = await db.Orders.ToListAsync();
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name);  // Triggers query per order!
}

// FIX 1: Eager Loading
var orders = await db.Orders.Include(o => o.Customer).ToListAsync();

// FIX 2: Projection (Best for read-only)
var orders = await db.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToListAsync();

// FIX 3: Explicit Loading (when conditional)
var orders = await db.Orders.ToListAsync();
foreach (var order in orders.Where(o => o.CustomerId > 0))
{
    await db.Entry(order).Reference(o => o.Customer).LoadAsync();
}
```

#### 3. **Design**: "How to implement Soft Delete globally?"

```csharp
// 1. Interface
public interface ISoftDelete
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}

// 2. Base Entity
public abstract class BaseEntity : ISoftDelete
{
    public int Id { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}

// 3. Global Query Filter in OnModelCreating
foreach (var entityType in modelBuilder.Model.GetEntityTypes())
{
    if (typeof(ISoftDelete).IsAssignableFrom(entityType.ClrType))
    {
        modelBuilder.Entity(entityType.ClrType)
            .HasQueryFilter(EF.Property<bool>(entityType.ClrType, "IsDeleted") == false);
    }
}

// 4. Override SaveChanges for auto-soft-delete
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    foreach (var entry in ChangeTracker.Entries<ISoftDelete>()
        .Where(e => e.State == EntityState.Deleted))
    {
        entry.State = EntityState.Modified;
        entry.Entity.IsDeleted = true;
        entry.Entity.DeletedAt = DateTime.UtcNow;
    }
    return await base.SaveChangesAsync(ct);
}

// 5. IncludeDeleted when needed
var all = await db.Users.IgnoreQueryFilters().ToListAsync();
```

#### 4. **Performance**: "Query returns 100K rows - how to optimize?"

**Answer:**
```
1. **Pagination**: .Skip().Take() - never load all
2. **Projection**: Select only needed columns (DTO)
3. **AsNoTracking**: Disable change tracking
4. **AsSplitQuery**: Avoid Cartesian explosion on Includes
5. **Indexing**: Ensure WHERE/ORDER BY columns indexed
6. **Batch/Streaming**: Use IAsyncEnumerable for large results
7. **Raw SQL**: For complex reporting queries
8. **Caching**: Redis for frequent reads
```

#### 5. **Code**: "Write a bulk upsert (insert or update) for 10K products"

```csharp
// EF Core 7+ ExecuteUpdate/ExecuteDelete don't support upsert
// Use raw SQL with MERGE (SQL Server) or ON CONFLICT (PostgreSQL)

public async Task BulkUpsertProductsAsync(List<Product> products)
{
    var sql = @"
        MERGE INTO Products AS target
        USING (VALUES 
            @p0, @p1, @p2, ...  -- Parameters for each product
        ) AS source (Id, Name, Price, UpdatedAt)
        ON target.Id = source.Id
        WHEN MATCHED THEN UPDATE SET Name = source.Name, Price = source.Price, UpdatedAt = source.UpdatedAt
        WHEN NOT MATCHED THEN INSERT (Id, Name, Price, UpdatedAt) VALUES (source.Id, source.Name, source.Price, source.UpdatedAt);";

    // Better: Use a library like EFCore.BulkExtensions
    await db.BulkInsertOrUpdateAsync(products, options => 
    {
        options.BatchSize = 1000;
        options.SqlBulkCopyOptions = SqlBulkCopyOptions.CheckConstraints;
    });
}
```

#### 6. **Architecture**: "DbContext in Clean Architecture"

```csharp
// Domain Layer - No EF Core references
public class Order : BaseEntity
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    public void AddItem(ProductId productId, int quantity, Money price) { ... }
}

// Infrastructure Layer
public class OrderEntityConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id).HasConversion(id => id.Value, value => new OrderId(value));
        builder.OwnsMany(o => o.Items, itemBuilder => { ... });
    }
}

public class ApplicationDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }
}
```

#### 7. **Debugging**: "SaveChanges throws DbUpdateException - how to diagnose?"

```csharp
try
{
    await db.SaveChangesAsync();
}
catch (DbUpdateException ex)
{
    // 1. Get inner exception (SQL error)
    var sqlException = ex.InnerException as SqlException;
    Console.WriteLine($"SQL Error: {sqlException?.Number} - {sqlException?.Message}");
    
    // 2. Check entries causing issue
    foreach (var entry in ex.Entries)
    {
        Console.WriteLine($"Entity: {entry.Entity.GetType().Name}, State: {entry.State}");
        foreach (var prop in entry.Properties)
        {
            Console.WriteLine($"  {prop.Metadata.Name}: {prop.CurrentValue} (Original: {prop.OriginalValue})");
        }
    }
    
    // 3. Common error codes (SQL Server)
    // 2601/2627: Unique constraint violation
    // 547: Foreign key violation
    // 2628: String truncation
    // 1205: Deadlock
}
```

#### 8. **Concurrency**: "Implement optimistic locking with RowVersion"

```csharp
// Entity
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    [Timestamp]  // Maps to rowversion in SQL Server
    public byte[] Version { get; set; }
}

// Service
public async Task<Result> UpdatePriceAsync(int id, decimal newPrice)
{
    var product = await db.Products.FindAsync(id);
    if (product == null) return Result.NotFound();
    
    product.Price = newPrice;
    
    try
    {
        await db.SaveChangesAsync();
        return Result.Success();
    }
    catch (DbUpdateConcurrencyException)
    {
        return Result.Conflict("Product was modified by another user. Please refresh.");
    }
}

// Alternative: Manual version check
public async Task<Result> UpdatePriceManualAsync(int id, decimal newPrice, byte[] expectedVersion)
{
    var product = await db.Products.FindAsync(id);
    if (product == null) return Result.NotFound();
    
    if (!product.Version.SequenceEqual(expectedVersion))
        return Result.Conflict("Stale data");
    
    product.Price = newPrice;
    await db.SaveChangesAsync();
    return Result.Success();
}
```

#### 9. **Migration**: "Production migration strategy for zero-downtime"

**Answer:**
```
1. **Backward-Compatible Migrations Only**
   - Add columns (nullable or with default)
   - Never drop columns/tables in same deployment
   - Rename = Add new + Copy data + Switch reads + Drop old (multi-deploy)

2. **Migration Steps**
   Deploy v1 (Code + Migration: Add column nullable)
   → Deploy v2 (Code uses new column)
   → Deploy v3 (Migration: Make column non-null, drop old)

3. **Tools**
   - EF Core Migrations for schema
   - Flyway/Liquibase for complex data migrations
   - Blue-Green deployment for instant rollback

4. **Data Migration Pattern**
   migrationBuilder.Sql(@"
       UPDATE NewTable SET NewCol = OldCol FROM OldTable 
       WHERE NewTable.Id = OldTable.Id");
```

#### 10. **Testing**: "Unit test EF Core logic without database"

```csharp
// Option 1: InMemory Provider (Fast, but not relational)
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase("TestDb")
    .Options;

// Option 2: SQLite In-Memory (Closer to real SQL)
var connection = new SqliteConnection("DataSource=:memory:");
connection.Open();
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseSqlite(connection)
    .Options;
using (var ctx = new AppDbContext(options)) ctx.Database.EnsureCreated();

// Option 3: Testcontainers (Real DB, Integration Tests)
[Fact]
public async Task CreateOrder_ShouldPersist()
{
    await using var container = new MsSqlBuilder().Build();
    await container.StartAsync();
    
    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseSqlServer(container.GetConnectionString())
        .Options;
    
    using var db = new AppDbContext(options);
    await db.Database.EnsureCreatedAsync();
    
    // Test...
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Tracking Issues**: Same entity instance in multiple DbContexts = exception
- [ ] **Include vs Select**: Include loads entire entity; Select projects
- [ ] **Lazy Loading**: Requires `virtual` + proxies package + `UseLazyLoadingProxies()` - avoid in web APIs
- [ ] **Cartesian Explosion**: Multiple `Include` collections = huge result set → use `AsSplitQuery()`
- [ ] **Client Evaluation**: EF Core 3+ throws; use `AsEnumerable()` explicitly if needed
- [ ] **Value Converters**: For enums, value objects, encryption
- [ ] **Owned Types**: Value objects stored in same table (Address in User table)
- [ ] **Shadow Properties**: Properties not in entity class (CreatedAt, UpdatedAt)
- [ ] **Temporal Tables**: SQL Server 2016+ built-in history (`modelBuilder.Entity<User>().ToTable(tb => tb.IsTemporal())`)
- [ ] **Json Columns**: `OwnsOne` with `ToJson()` for PostgreSQL/SQL Server JSON
- [ ] **Bulk Operations**: Use `EFCore.BulkExtensions` for >1000 rows
- [ ] **Transaction Scope**: `DbContext.Database.BeginTransaction()` vs `TransactionScope` (distributed)

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Base DbContext with Auditing**
```csharp
public abstract class AuditableDbContext : DbContext
{
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var entries = ChangeTracker.Entries<IAuditable>()
            .Where(e => e.State is EntityState.Added or EntityState.Modified);
        
        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = DateTime.UtcNow;
                entry.Entity.CreatedBy = GetCurrentUserId();
            }
            entry.Entity.UpdatedAt = DateTime.UtcNow;
            entry.Entity.UpdatedBy = GetCurrentUserId();
        }
        return await base.SaveChangesAsync(ct);
    }
}
```

#### 2. **Specification Pattern**
```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    Expression<Func<T, object>> OrderBy { get; }
    Expression<Func<T, object>> OrderByDescending { get; }
    int Take { get; }
    int Skip { get; }
    bool IsPagingEnabled { get; }
}

public class BaseSpecification<T> : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria { get; private set; } = _ => true;
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    // ... implementations
    
    protected void AddInclude(Expression<Func<T, object>> include) => Includes.Add(include);
}

// Usage
var spec = new BaseSpecification<User>()
    .Where(u => u.IsActive)
    .Include(u => u.Orders)
    .OrderBy(u => u.CreatedAt)
    .Paginate(page, pageSize);

var users = await db.Users.WithSpecification(spec).ToListAsync();
```

#### 3. **Interceptors (EF Core 7+)**
```csharp
public class AuditingInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, 
        InterceptionResult<int> result)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChanges(eventData, result);
    }
    
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, 
        InterceptionResult<int> result, 
        CancellationToken ct)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChangesAsync(eventData, result, ct);
    }
    
    private void UpdateAuditFields(DbContext? context)
    {
        if (context == null) return;
        foreach (var entry in context.ChangeTracker.Entries<IAuditable>())
        {
            // ... audit logic
        }
    }
}

// Registration
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString)
           .AddInterceptors(sp.GetRequiredService<AuditingInterceptor>());
});
```

---

### PRACTICE PROBLEMS

1. **Design**: Model a multi-tenant SaaS with shared database, tenant isolation via query filters
2. **Performance**: Optimize a query loading 50K orders with customer, items, products
3. **Migration**: Add non-nullable column to 10M row table with zero downtime
4. **Concurrency**: Implement seat reservation system (concert tickets) with optimistic locking
5. **Testing**: Write unit tests for a specification-based query builder

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| EF Core Change Tracker purpose | Identity Map + State Manager per DbContext scope |
| DbContext lifetime in ASP.NET Core | Scoped (per request) - NOT singleton, NOT transient |
| AsNoTracking vs AsNoTrackingWithIdentityResolution | NoTracking: new instance each query. WithIdentity: same instance for same key |
| Global Query Filter use case | Soft delete, Multi-tenant - automatic WHERE clause |
| N+1 problem fix | Include/ThenInclude (eager), Select (projection), Explicit Load |
| Split Query when | Multiple collection includes causing Cartesian explosion |
| Optimistic concurrency in EF Core | [Timestamp] byte[] RowVersion or [ConcurrencyCheck] property |
| Bulk insert 10K+ rows | EFCore.BulkExtensions or raw SQL MERGE/COPY |
| DbContext thread safety | NOT thread-safe - one per request (Scoped) |
| Value Converter example | Enum ↔ string, ValueObject ↔ JSON, Encrypted ↔ Plain |

---

## NEXT TOPIC: `04-DI-MIDDLEWARE.md`

> **Study Tip**: Create a small console app demonstrating: Change tracking states, N+1 fix, Global query filter, Optimistic concurrency. Run and verify SQL with `LogTo(Console.WriteLine)`.