# EF Core Performance: Senior Interview Questions & Hands-On Benchmark Lab

A consolidated, non-repetitive guide for Senior .NET Developers and
Solution Architects.

## 1. Performance interview questions

### 1.1 Why can `AsNoTracking()` improve read-query performance?

By default, EF Core tracks entity instances returned by entity queries.
Tracking supports change detection and identity resolution, but adds CPU
and memory overhead. For read-only queries, use `AsNoTracking()` when
change tracking is not needed.

``` csharp
var users = await db.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserListItem(u.Id, u.Name))
    .ToListAsync(cancellationToken);
```

Use `AsNoTrackingWithIdentityResolution()` when a no-tracking result
contains repeated references to the same entity and identity resolution
is useful. Measure before making a query-wide default change.

### 1.2 What commonly causes slow EF Core queries?

Typical causes include: - Missing or poorly ordered indexes. - Returning
complete entities when only a few columns are needed. - Loading large
object graphs or multiple sibling collections. - N+1 queries from lazy
loading or repeated database calls. - Deep offset pagination. -
Excessive tracking or materialization. - Non-sargable predicates, stale
statistics, parameter-sensitive plans, or database resource limits.

**Senior approach:** inspect generated SQL, actual execution plans,
logical reads, row counts, Query Store, and application timings before
choosing a fix.

### 1.3 Why can multiple `Include` calls be expensive?

Including multiple sibling collections can multiply rows returned by a
single joined query. This is often called *cartesian explosion*. Use
projection when the endpoint needs a summary rather than a full entity
graph. For graph loading, evaluate `AsSplitQuery()` against the
generated SQL and consistency requirements.

``` csharp
var summaries = await db.Orders
    .AsNoTracking()
    .Select(o => new OrderSummary(
        o.Id,
        o.Customer.Name,
        o.Items.Sum(i => (decimal?)i.Price) ?? 0m,
        o.Payments.Count))
    .ToListAsync(cancellationToken);
```

Split queries reduce join duplication in some cases but may issue
multiple SQL statements; consider round trips and transaction/isolation
semantics.

### 1.4 How do you detect and prevent N+1 queries?

**Symptoms:** query count grows with the number of returned rows;
latency rises under load.

**Prevention:** - Avoid implicit lazy loading in API paths. - Project
required related data into DTOs. - Use `Include` intentionally for
entity graphs. - Inspect command logs and use database monitoring. - Add
integration tests or telemetry that detect unexpected query counts.

### 1.5 Offset pagination vs. keyset pagination?

Offset pagination (`Skip`/`Take`) is convenient for numbered pages but
may become more expensive at large offsets. Keyset pagination uses the
last-seen sort key and is often better for sequential browsing.

``` csharp
// Offset pagination
var page = await db.Orders
    .OrderByDescending(o => o.CreatedAt)
    .ThenByDescending(o => o.Id)
    .Skip(offset)
    .Take(pageSize)
    .Select(o => new OrderListItem(o.Id, o.CreatedAt))
    .ToListAsync(cancellationToken);
```

``` csharp
// Keyset pagination: lastCreatedAt and lastId come from the previous page.
// The unique Id tie-breaker makes ordering deterministic.
var page = await db.Orders
    .Where(o => o.CreatedAt < lastCreatedAt ||
        (o.CreatedAt == lastCreatedAt && o.Id < lastId))
    .OrderByDescending(o => o.CreatedAt)
    .ThenByDescending(o => o.Id)
    .Take(pageSize)
    .Select(o => new OrderListItem(o.Id, o.CreatedAt))
    .ToListAsync(cancellationToken);
```

Support the sort with a suitable composite index, for example
`(CreatedAt DESC, Id DESC)` where appropriate for the provider and
query. Keyset pagination is not a drop-in replacement when users need
arbitrary page numbers.

### 1.6 What is a key lookup in a SQL Server execution plan?

A nonclustered index may find qualifying rows, then SQL Server performs
key lookups to retrieve columns absent from that index. A high number of
lookups can be expensive. Consider a covering index with `INCLUDE`
columns only when the query workload justifies its storage and write
costs.

``` csharp
modelBuilder.Entity<Order>()
    .HasIndex(o => o.CustomerId)
    .IncludeProperties(o => new { o.Status, o.CreatedAt });
```

### 1.7 When should you use compiled queries?

Compiled queries can reduce EF Core query-processing overhead for
stable, frequently executed query shapes. They do not fix poor SQL,
missing indexes, network latency, or database bottlenecks. Benchmark
first; ordinary EF Core query caching already helps many workloads.

### 1.8 Why are large inserts often slow with normal EF Core writes?

Tracking many entities increases memory and change-detection costs, and
ordinary persistence may require many database commands. For large
imports, consider provider-specific bulk tools such as SQL Server
`SqlBulkCopy`, staging tables, or carefully designed set-based
operations. Preserve required validation, constraints, transaction
boundaries, and error handling.

### 1.9 What does `SaveChanges` guarantee?

A single `SaveChanges` call is transactional by default when the
provider supports transactions: if its set of changes fails, the
transaction is rolled back. Multiple `SaveChanges` calls are separate
units unless enclosed in an explicit transaction. An explicit
transaction is useful when a use case spans multiple saves or other
database operations; avoid adding one without a requirement.

### 1.10 How should optimistic concurrency be handled?

Configure a concurrency token, such as SQL Server `rowversion`, on
entities where overwriting another user's changes is unacceptable. Catch
`DbUpdateConcurrencyException` and apply a defined policy: return a
conflict, reload and merge, or retry only when safe.

``` csharp
public sealed class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public byte[] RowVersion { get; set; } = [];
}

// EF configuration
modelBuilder.Entity<Product>()
    .Property(p => p.RowVersion)
    .IsRowVersion();
```

### 1.11 How do you diagnose a slow EF Core query in production?

1.  Identify the endpoint and capture latency, query count, and database
    timings.
2.  Inspect the generated SQL (`ToQueryString()` is useful for
    inspection, but does not execute the query).
3.  Use Azure SQL Query Store and an actual execution plan where
    available.
4.  Compare estimated and actual row counts; inspect logical reads,
    scans, lookups, sorts, spills, and join choices.
5.  Check indexes, parameter sensitivity, statistics, blocking, and
    database resource saturation.
6.  Change one thing at a time and compare results under representative
    data and load.

### 1.12 EF Core or Dapper?

Use EF Core when change tracking, aggregate persistence, relationship
mapping, and maintainability are valuable. Consider Dapper or raw SQL
for specialized read models, reporting, or queries that need precise SQL
control. This is a workload-specific choice, not a universal rule that
EF Core is only for writes.

### 1.13 What is the right `DbContext` lifetime?

`DbContext` is not thread-safe and should represent a short unit of
work. In ASP.NET Core, `AddDbContext` registers it as scoped by default.
Do not run parallel operations on the same context. For background
services, create a scope or use `IDbContextFactory<TContext>` as
appropriate.

### 1.14 What are the risks of global query filters?

Filters can centralize soft-delete or tenant predicates, but they are
not a complete security boundary by themselves. Carefully control
`IgnoreQueryFilters()`, test tenant isolation, ensure tenant context
cannot be spoofed, and consider database-level security where required.

### 1.15 What would you monitor for a high-traffic EF Core service?

-   Endpoint p50/p95/p99 latency and error rate.
-   Database command duration and query count per request.
-   Query Store regressions, CPU, data IO, log IO, waits, and blocking.
-   Connection pool pressure and timeouts.
-   Rows returned versus rows read, plus memory allocation and GC.
-   Deadlocks, concurrency conflicts, retries, and failed transactions.

------------------------------------------------------------------------

## 2. Hands-on EF Core benchmark project

### 2.1 Objective

Build a repeatable benchmark comparing: 1. Tracking vs. no-tracking
materialization. 2. Entity graph loading vs. DTO projection. 3. Offset
vs. keyset pagination. 4. Ordinary vs. compiled query execution.

Use SQL Server or Azure SQL for database-level comparisons. Results
depend on data, indexes, network latency, hardware, EF Core/provider
versions, and database configuration; do not assume a particular winner
without measuring.

### 2.2 Create the project

``` bash
dotnet new console -n EfCorePerformanceLab
cd EfCorePerformanceLab

dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package BenchmarkDotNet
```

Use a .NET and EF Core version supported by your environment. Keep EF
Core provider packages on compatible versions.

Set a connection string through configuration or an environment variable
rather than hard-coding credentials. For example, use
`ConnectionStrings__Lab` in the environment and read it with
`IConfiguration`.

### 2.3 Minimal model

``` csharp
public sealed class Order
{
    public long Id { get; set; }
    public int TenantId { get; set; }
    public string CustomerEmail { get; set; } = "";
    public int Status { get; set; }
    public decimal Total { get; set; }
    public DateTime CreatedAt { get; set; }

    public List<OrderItem> Items { get; set; } = [];
    public List<Payment> Payments { get; set; } = [];
}

public sealed class OrderItem
{
    public long Id { get; set; }
    public long OrderId { get; set; }
    public decimal Price { get; set; }
    public Order Order { get; set; } = null!;
}

public sealed class Payment
{
    public long Id { get; set; }
    public long OrderId { get; set; }
    public decimal Amount { get; set; }
    public Order Order { get; set; } = null!;
}

public sealed record OrderListItem(long Id, DateTime CreatedAt);
```

### 2.4 DbContext and index configuration

``` csharp
using Microsoft.EntityFrameworkCore;

public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Payment> Payments => Set<Payment>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>(b =>
        {
            b.HasKey(x => x.Id);
            b.Property(x => x.CustomerEmail).HasMaxLength(256);
            b.HasIndex(x => new { x.TenantId, x.Status });
            b.HasIndex(x => new { x.CreatedAt, x.Id });
        });

        modelBuilder.Entity<OrderItem>()
            .HasOne(x => x.Order)
            .WithMany(x => x.Items)
            .HasForeignKey(x => x.OrderId);

        modelBuilder.Entity<Payment>()
            .HasOne(x => x.Order)
            .WithMany(x => x.Payments)
            .HasForeignKey(x => x.OrderId);
    }
}
```

The index definitions are starting hypotheses for this lab, not
universal recommendations. Validate them against actual query
predicates, sort order, selectivity, and write volume.

### 2.5 Create a benchmark class

This example creates a fresh context for each benchmark operation,
avoiding contamination from entities tracked by previous operations. For
a more rigorous test, separate cold-start costs from steady-state query
costs and warm up the database before measurement.

``` csharp
using BenchmarkDotNet.Attributes;
using Microsoft.EntityFrameworkCore;

[MemoryDiagnoser]
public class EfCoreBenchmarks
{
    private DbContextOptions<AppDbContext> _options = null!;
    private const int Offset = 50_000;
    private const int PageSize = 20;
    private const int TenantId = 1;

    [GlobalSetup]
    public void Setup()
    {
        var connectionString =
            Environment.GetEnvironmentVariable("ConnectionStrings__Lab")
            ?? throw new InvalidOperationException(
                "Set ConnectionStrings__Lab before running benchmarks.");

        _options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(connectionString)
            .Options;
    }

    private AppDbContext CreateContext() => new(_options);

    [Benchmark]
    public async Task TrackingRead()
    {
        await using var db = CreateContext();

        _ = await db.Orders
            .Where(o => o.TenantId == TenantId)
            .OrderBy(o => o.Id)
            .Take(500)
            .ToListAsync();
    }

    [Benchmark]
    public async Task NoTrackingRead()
    {
        await using var db = CreateContext();

        _ = await db.Orders
            .AsNoTracking()
            .Where(o => o.TenantId == TenantId)
            .OrderBy(o => o.Id)
            .Take(500)
            .ToListAsync();
    }

    [Benchmark]
    public async Task EntityGraph()
    {
        await using var db = CreateContext();

        _ = await db.Orders
            .AsNoTracking()
            .Where(o => o.TenantId == TenantId)
            .OrderBy(o => o.Id)
            .Take(100)
            .Include(o => o.Items)
            .Include(o => o.Payments)
            .AsSplitQuery()
            .ToListAsync();
    }

    [Benchmark]
    public async Task Projection()
    {
        await using var db = CreateContext();

        _ = await db.Orders
            .AsNoTracking()
            .Where(o => o.TenantId == TenantId)
            .OrderBy(o => o.Id)
            .Take(100)
            .Select(o => new
            {
                o.Id,
                ItemCount = o.Items.Count,
                PaymentCount = o.Payments.Count
            })
            .ToListAsync();
    }

    [Benchmark]
    public async Task OffsetPagination()
    {
        await using var db = CreateContext();

        _ = await db.Orders
            .AsNoTracking()
            .OrderBy(o => o.Id)
            .Skip(Offset)
            .Take(PageSize)
            .Select(o => new OrderListItem(o.Id, o.CreatedAt))
            .ToListAsync();
    }

    [Benchmark]
    public async Task KeysetPagination()
    {
        await using var db = CreateContext();

        _ = await db.Orders
            .AsNoTracking()
            .Where(o => o.Id > Offset)
            .OrderBy(o => o.Id)
            .Take(PageSize)
            .Select(o => new OrderListItem(o.Id, o.CreatedAt))
            .ToListAsync();
    }
}
```

**Important:** The pagination methods above are a controlled
illustration using `Id` as the ordering key. They return equivalent
page-shaped results only if the test data and cursor are designed
accordingly. For production keyset pagination, use the last key from the
preceding page, a deterministic unique ordering, and an index matching
that ordering.

### 2.6 Add compiled-query comparison

Add this method and field to the benchmark class:

``` csharp
private static readonly
    Func<AppDbContext, int, IAsyncEnumerable<Order>> OrdersByTenant =
        EF.CompileAsyncQuery(
            (AppDbContext db, int tenantId) =>
                db.Orders
                    .AsNoTracking()
                    .Where(o => o.TenantId == tenantId)
                    .OrderBy(o => o.Id)
                    .Take(100));

[Benchmark]
public async Task OrdinaryQuery()
{
    await using var db = CreateContext();

    _ = await db.Orders
        .AsNoTracking()
        .Where(o => o.TenantId == TenantId)
        .OrderBy(o => o.Id)
        .Take(100)
        .ToListAsync();
}

[Benchmark]
public async Task CompiledQuery()
{
    await using var db = CreateContext();

    await foreach (var order in OrdersByTenant(db, TenantId))
    {
        // Consume results so the query executes fully.
        _ = order.Id;
    }
}
```

Compiled queries reduce some EF Core query-processing overhead; they do
not bypass database execution, network latency, or SQL Server
execution-plan costs.

### 2.7 Add the benchmark entry point

``` csharp
using BenchmarkDotNet.Running;

BenchmarkRunner.Run<EfCoreBenchmarks>();
```

Run in Release mode:

``` bash
dotnet run -c Release
```

BenchmarkDotNet should be run on a controlled machine with
representative data. Avoid running destructive seed operations or heavy
competing workloads during measurements.

### 2.8 Prepare representative data

Use a dedicated lab database. Seed enough rows to make query plans and
pagination differences observable---for example, hundreds of thousands
of orders with realistic distributions of tenants, statuses, dates,
items, and payments.

Ensure: - Data distribution is not uniform unless production is
uniform. - Indexes and statistics reflect the intended scenario. - The
database is warmed up before steady-state runs. - Each compared query
returns equivalent logical results. - Database metrics are captured
alongside application benchmarks.

### 2.9 Capture SQL and execution plans

Use `ToQueryString()` to inspect SQL generated by EF Core:

``` csharp
var query = db.Orders
    .AsNoTracking()
    .Where(o => o.TenantId == TenantId)
    .OrderBy(o => o.Id)
    .Take(100);

Console.WriteLine(query.ToQueryString());
```

For SQL Server/Azure SQL, compare actual execution plans and Query Store
metrics. Record: - Duration and CPU. - Logical reads and rows read. -
Actual vs. estimated row counts. - Index seek/scan and key lookup
behavior. - Sorts, spills, blocking, and memory grants.

Do not assume an index seek is always better than a scan; a scan can be
appropriate when a large proportion of a table is needed.

### 2.10 Results worksheet

Record your actual results instead of copying assumed numbers:

  --------------------------------------------------------------------------------
  Scenario            Mean time       Allocated  SQL duration / Notes
                                         memory           reads 
  ------------- --------------- --------------- --------------- ------------------
  Tracking read         Measure         Measure         Measure Baseline

  No-tracking           Measure         Measure         Measure Compare
  read                                                          materialization
                                                                overhead

  Entity graph          Measure         Measure         Measure Check row
                                                                multiplication

  DTO                   Measure         Measure         Measure Check selected
  projection                                                    columns and SQL

  Offset                Measure         Measure         Measure Test shallow and
  pagination                                                    deep pages

  Keyset                Measure         Measure         Measure Verify cursor and
  pagination                                                    supporting index

  Ordinary              Measure         Measure         Measure Baseline
  query                                                         query-processing
                                                                cost

  Compiled              Measure         Measure         Measure Compare
  query                                                         steady-state
                                                                overhead
  --------------------------------------------------------------------------------

------------------------------------------------------------------------

## 3. Production review checklist

### Query design

-   [ ] Project only required columns for read endpoints.
-   [ ] Use no-tracking queries where entity updates are not needed.
-   [ ] Avoid accidental N+1 queries.
-   [ ] Evaluate `Include` and split queries for large graphs.
-   [ ] Use deterministic pagination and keyset pagination for
    sequential browsing where appropriate.

### Database design

-   [ ] Design indexes from real predicates and ordering.
-   [ ] Validate indexes with execution plans and workload metrics.
-   [ ] Review foreign keys, unique constraints, and data types.
-   [ ] Check parameter-sensitive plans and statistics where performance
    varies by parameter.

### Reliability

-   [ ] Use short-lived `DbContext` instances.
-   [ ] Handle concurrency conflicts deliberately.
-   [ ] Keep related writes atomic.
-   [ ] Make retries safe and avoid retrying non-idempotent work
    blindly.
-   [ ] Use migrations or reviewed deployment scripts with a production
    rollout plan.

### Observability

-   [ ] Capture slow-query telemetry without exposing sensitive values.
-   [ ] Monitor Query Store, CPU, IO, blocking, and timeouts.
-   [ ] Benchmark representative workloads before and after changes.
-   [ ] Test against the production database provider for critical query
    behavior.

## 4. Official Microsoft documentation

-   [EF Core performance efficient
    querying](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying)
-   [EF Core advanced performance
    topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)
-   [EF Core tracking
    vs. no-tracking](https://learn.microsoft.com/en-us/ef/core/querying/tracking)
-   [EF Core
    pagination](https://learn.microsoft.com/en-us/ef/core/querying/pagination)
-   [EF Core
    transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions)
-   [EF Core concurrency
    handling](https://learn.microsoft.com/en-us/ef/core/saving/concurrency)
-   [EF Core
    testing](https://learn.microsoft.com/en-us/ef/core/testing/)

------------------------------------------------------------------------

## Key takeaway

For senior engineers and architects, EF Core performance work is a
measurement loop:

**Observe → inspect generated SQL → inspect the execution plan → change
query or index design → benchmark again.**

Avoid universal rules such as "every query needs an index," "all reads
must use no-tracking," or "EF Core is always slower than Dapper." The
correct design depends on the query, data distribution, provider,
workload, and consistency requirements.
