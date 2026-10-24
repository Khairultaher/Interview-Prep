# EF Core LINQ Query Optimization: Senior .NET Developer Guide

## 1. Choose tracking behavior deliberately

### Read-only query

Use `AsNoTracking()` when the returned entities will not be updated through the current `DbContext`.

```csharp
var students = await db.Students
    .AsNoTracking()
    .Where(s => s.IsActive)
    .Select(s => new StudentDto(s.Id, s.Name))
    .ToListAsync(cancellationToken);
```

### When identity resolution matters

`AsNoTrackingWithIdentityResolution()` performs no normal context tracking, but uses a temporary identity map while materializing results. If the same entity occurs more than once in the result graph, EF Core can reuse one object instance.

```csharp
var students = await db.Students
    .AsNoTrackingWithIdentityResolution()
    .Include(s => s.Class)
    .Include(s => s.Addresses)
    .ToListAsync(cancellationToken);
```

Use it when entity graphs contain repeated references and object identity matters. It has extra overhead compared with plain `AsNoTracking()`. For DTO projections, it is often unnecessary.

**Important:** `AsNoTracking()` or `AsNoTrackingWithIdentityResolution()` on the root query applies to entities materialized by that query, including entities loaded through `Include()`. You do not need to add it to every table separately.

## 2. Understand `AsSplitQuery()`

`AsSplitQuery()` changes how EF Core retrieves included related collections. Instead of one large SQL query with joins for the included collections, EF Core issues multiple SQL queries.

```csharp
var blogs = await db.Blogs
    .AsNoTracking()
    .AsSplitQuery()
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .ToListAsync(cancellationToken);
```

This can reduce duplicated parent data and Cartesian explosion when multiple collections are included. However, it adds database round trips and can observe inconsistent data if rows change between queries. If a consistent snapshot is required, consider an appropriate transaction/isolation level.

`AsSplitQuery()` is not a general optimization for every explicit LINQ `Join`. It primarily affects queries with collection navigation `Include`s. Check the generated SQL and benchmark before choosing it.

## 3. Key distinction

| Feature | `AsNoTrackingWithIdentityResolution()` | `AsSplitQuery()` |
|---|---|---|
| Main concern | Object materialization and identity | SQL query shape |
| What changes | Reuses instances for repeated entity keys without normal context tracking | Splits included collection loading into multiple SQL queries |
| Typical use | Read-only entity graphs with repeated entity references | Multiple included collections causing large joined results |
| Can combine them? | Yes | Yes |

Example combination:

```csharp
var blogs = await db.Blogs
    .AsNoTrackingWithIdentityResolution()
    .AsSplitQuery()
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .ToListAsync(cancellationToken);
```

Only use both when both behaviors solve a demonstrated problem.

## 4. Practical LINQ / EF Core optimization techniques

### Project only the columns you need

```csharp
var results = await db.Orders
    .Where(o => o.Status == OrderStatus.Open)
    .Select(o => new OrderSummaryDto(
        o.Id,
        o.CreatedAt,
        o.Customer.Name,
        o.Total))
    .ToListAsync(cancellationToken);
```

DTO projections usually reduce transferred data and avoid materializing full entities. If the projection contains only scalar values and DTOs, entity change tracking is generally not involved.

### Filter and paginate in the database

```csharp
var page = await db.Orders
    .Where(o => o.Status == OrderStatus.Open)
    .OrderBy(o => o.Id)
    .Skip(pageNumber * pageSize)
    .Take(pageSize)
    .Select(o => new OrderSummaryDto(o.Id, o.CreatedAt, o.Total))
    .ToListAsync(cancellationToken);
```

Validate page inputs and use a deterministic ordering. For deep pagination, consider keyset/seek pagination rather than large `Skip` values.

### Avoid materializing before filtering

```csharp
// Inefficient: loads rows before filtering in memory.
var all = await db.Students.ToListAsync(cancellationToken);
var adults = all.Where(s => s.Age >= 18).ToList();

// Better: filter in SQL.
var adults = await db.Students
    .Where(s => s.Age >= 18)
    .ToListAsync(cancellationToken);
```

### Use `Any()` when you only need existence

```csharp
var hasOverdueOrders = await db.Orders
    .AnyAsync(o => o.CustomerId == customerId && o.IsOverdue, cancellationToken);
```

This expresses an existence check instead of loading matching rows.

### Avoid N+1 queries

Prefer a suitable projection or eager loading for data you know you need. Be cautious with lazy loading inside loops, as it can trigger one extra query per entity.

### Keep predicates index-friendly

Avoid wrapping indexed columns in transformations when possible. For example, use a date range rather than comparing `o.CreatedAt.Date` to a date, where that translation would apply a function to the database column.

```csharp
var start = requestedDate.Date;
var end = start.AddDays(1);

var orders = await db.Orders
    .Where(o => o.CreatedAt >= start && o.CreatedAt < end)
    .ToListAsync(cancellationToken);
```

Index selection depends on the complete workload and execution plan; LINQ syntax alone does not guarantee index use.

### Use set-based updates for bulk changes

EF Core 7+ supports `ExecuteUpdateAsync()` and `ExecuteDeleteAsync()` for supported relational providers.

```csharp
await db.Students
    .Where(s => !s.IsActive)
    .ExecuteUpdateAsync(
        setters => setters.SetProperty(s => s.IsArchived, true),
        cancellationToken);
```

These operations execute directly in the database and do not load entities or use the change tracker. They also bypass normal tracked-entity updates and `SaveChanges` behavior, so account for concurrency, audit logic, and domain events explicitly.

### Do not parallelize operations on one `DbContext`

EF Core does not support multiple concurrent operations on the same `DbContext`. Await each operation before starting another, or use separate contexts when genuine parallel database work is appropriate. PLINQ is for in-memory processing and does not optimize a database query by itself.

## 5. How to investigate a slow query

1. **Measure first:** capture elapsed time, row counts, allocations, and database duration.
2. **Inspect generated SQL:** use `ToQueryString()` during debugging and enable suitable EF Core logging.
3. **Inspect the database execution plan:** check scans, seeks, joins, cardinality estimates, sorts, and missing or unsuitable indexes.
4. **Reduce data:** project fewer columns, filter early, and paginate.
5. **Check round trips:** look for N+1 queries and decide whether single or split queries fit the workload.
6. **Benchmark realistic data:** compare latency and resource usage under representative load.
7. **Re-test correctness:** especially ordering, null behavior, consistency, and concurrency.

## 6. Senior .NET developer / architect viva prompts

Be ready to explain and demonstrate:

- Deferred execution versus immediate execution (`ToList`, `First`, `Count`).
- `IQueryable<T>` versus `IEnumerable<T>` and where query execution happens.
- Client evaluation risks and SQL translation limitations.
- Tracking, no-tracking, identity resolution, and change detection.
- `Include` versus projection; single versus split queries.
- N+1 queries, Cartesian explosion, pagination, and keyset pagination.
- Index design and execution-plan analysis.
- Async I/O, cancellation tokens, connection pooling, and `DbContext` lifetime.
- Transactions, isolation levels, optimistic concurrency, and set-based updates.
- How to measure performance rather than assuming a LINQ rewrite is faster.

**Interview answer pattern:** explain the problem, show the generated SQL or execution plan, describe the trade-off, and state how you would measure and verify the improvement.

## Quick rule of thumb

- **Read-only entity query:** `AsNoTracking()`.
- **Read-only graph where repeated entity references should share instances:** consider `AsNoTrackingWithIdentityResolution()`.
- **Multiple included collections causing large joined results:** evaluate `AsSplitQuery()`.
- **Read model/API response:** prefer a narrow DTO projection.
- **Any performance claim:** verify with SQL, execution plans, and measurements.
