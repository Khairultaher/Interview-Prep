# EF Core & LINQ: Senior .NET Engineer / Solution Architect Interview Guide

> A consolidated study guide focused on internals, trade-offs, production design, and interview questions. It avoids repeating the same introductory topics in multiple sections.
>
> **Version note:** EF Core behavior and available APIs vary by version and database provider. Verify version-specific features against the documentation for the version used in the interview or project.

## Contents

1. Mental model and architecture
2. Query execution: LINQ to SQL
3. Change tracking and `SaveChanges`
4. Transactions, concurrency, and consistency
5. Relationships and model mapping
6. Performance engineering
7. Advanced LINQ
8. Production architecture and security
9. Testing, migrations, and operations
10. Interview MCQs with answers
11. Scenario-based interview questions
12. Final revision checklist

---

## 1. Mental model and architecture

### 1.1 What each component does

| Component | Responsibility |
|---|---|
| `DbContext` | Short-lived unit of work; coordinates querying, tracking, saving, and database services. It is **not thread-safe**. |
| `DbSet<TEntity>` | Entry point for querying and adding/removing entity instances of a mapped type. |
| `IQueryable<TEntity>` | Represents a query expression that a provider may translate; building it usually does not execute SQL. |
| EF Core query provider | Processes expression trees, translates supported operations, and coordinates query execution/materialization. |
| Model metadata (`IModel`) | Describes entities, properties, keys, relationships, conversions, and mapping rules. |
| `ChangeTracker` / state manager | Maintains tracked entity entries, identity resolution, original/current values, relationship state, and entity states. |
| `SaveChanges` pipeline | Detects changes, prepares modification commands, executes them, and accepts changes after success. |
| Database provider | Supplies provider-specific SQL translation, parameter handling, batching, and database behavior. |
| `DbConnection` / `DbTransaction` | Underlying ADO.NET connection and transaction abstractions. |

`DbContext` is commonly described as implementing Unit of Work, while `DbSet` resembles a repository-like collection. These are useful mental models, not a requirement to wrap every EF operation in a generic repository.

### 1.2 Typical write path

```text
Application code changes entity objects
        |
        v
DbContext / ChangeTracker maintains entity entries
        |
        v
SaveChanges()
        |
        +--> Detect changes (when enabled/needed)
        +--> Build modification commands
        +--> Order commands for dependencies
        +--> Batch commands where supported
        +--> Execute commands through provider/ADO.NET
        +--> Commit transaction when EF owns it
        +--> Accept changes after success
```

The exact internal classes and sequence are implementation details and can change between EF Core releases.

### 1.3 `DbContext` lifetime

For a typical ASP.NET Core request, a scoped context is a sensible default:

```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

Avoid:
- Registering a context as a singleton.
- Running parallel operations on the same context.
- Keeping a context alive for an entire application or long-running job.
- Treating a context as a global cache.

For background work, create a context per unit of work using `IDbContextFactory<TContext>` or an appropriate scope:

```csharp
public sealed class RebuildWorker(
    IDbContextFactory<AppDbContext> factory)
{
    public async Task RunAsync(CancellationToken cancellationToken)
    {
        await using var db =
            await factory.CreateDbContextAsync(cancellationToken);

        var pending = await db.Jobs
            .Where(j => j.Status == JobStatus.Pending)
            .Select(j => j.Id)
            .ToListAsync(cancellationToken);
    }
}
```

`AddDbContextPool` can reduce context initialization overhead, but pooled contexts are reused. Do not keep request-specific mutable state in a context unless it is reliably reset; tenant state and custom context fields need special care.

---

## 2. Query execution: LINQ to SQL

### 2.1 `IQueryable` versus `IEnumerable`

```csharp
IQueryable<Order> query = db.Orders
    .Where(o => o.Status == OrderStatus.Open);

var rows = await query
    .OrderBy(o => o.CreatedAt)
    .Take(50)
    .ToListAsync(cancellationToken);
```

The operators build a query expression until a terminal/materializing operation executes it. Examples include `ToListAsync`, `SingleAsync`, `FirstAsync`, `CountAsync`, and enumeration.

Compare:

```csharp
// Filter can be translated to SQL.
var query = db.Orders.Where(o => o.Total > 100);

// Materializes rows; later filtering is in memory.
var orders = await db.Orders.ToListAsync(cancellationToken);
var filtered = orders.Where(o => o.Total > 100);
```

`IQueryable<T>` does not guarantee server execution for every expression: translation depends on the provider and the expression. In modern EF Core, an untranslatable expression in most query locations throws rather than silently evaluating the entire filter on the client.

### 2.2 Query pipeline (conceptual)

1. C# query operators construct expression trees.
2. EF Core normalizes and preprocesses the expression.
3. The provider translates supported parts into a database query.
4. EF Core creates/uses a query execution plan and database command.
5. Parameters are sent to the provider.
6. The database executes SQL and returns rows.
7. EF materializes entities/DTOs and performs identity resolution when applicable.

EF caches query compilation work for recurring query shapes. Different constants, expression shapes, dynamic predicates, and provider behavior can affect cache reuse. Parameterized query patterns are generally preferable to generating many distinct expression shapes.

### 2.3 Inspect generated SQL

```csharp
var query = db.Orders
    .Where(o => o.Status == OrderStatus.Open)
    .Select(o => new { o.Id, o.CreatedAt, o.Total });

string sql = query.ToQueryString();
```

`ToQueryString()` is useful for debugging, but the executed command can differ due to provider behavior, parameter values, terminal operators, and runtime execution. Also inspect database execution plans and actual I/O.

### 2.4 Projection is often better than loading full entities

```csharp
var summaries = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new OrderSummaryDto(
        o.Id,
        o.CreatedAt,
        o.Lines.Sum(line => line.Quantity * line.UnitPrice)))
    .ToListAsync(cancellationToken);
```

Benefits:
- Selects only needed columns.
- Avoids materializing unnecessary entity graphs.
- Often reduces tracking and memory costs.
- Makes the API's data shape explicit.

Projection is not automatically optimal: check generated SQL and query plans, especially for correlated aggregates, large joins, and complex projections.

---

## 3. Change tracking and `SaveChanges`

### 3.1 Entity states

| State | Meaning |
|---|---|
| `Detached` | Not tracked by this context. |
| `Added` | EF intends to insert the entity. |
| `Unchanged` | Tracked; no detected modifications. |
| `Modified` | One or more properties are marked modified. |
| `Deleted` | EF intends to delete the entity. |

An entity returned by a normal tracking query is typically `Unchanged` when first materialized. An `AsNoTracking()` query returns entities that are not tracked by the context.

### 3.2 Snapshot tracking and notifications

With snapshot tracking, EF records original values and compares them with current values during change detection. Notification-based strategies can use entity notifications when configured and implemented correctly.

```csharp
var customer = await db.Customers
    .SingleAsync(c => c.Id == id, cancellationToken);

customer.Name = "Updated name";

var entry = db.Entry(customer);
Console.WriteLine(entry.State); // May still be Unchanged before detection.
db.ChangeTracker.DetectChanges();
Console.WriteLine(entry.State); // Modified if the change is detected.
```

Do not assume every property assignment instantly changes the entity state under snapshot tracking.

### 3.3 How `SaveChanges` works

At a high level, EF Core:
1. Detects changes when automatic detection is enabled and applicable.
2. Identifies entries that need insert/update/delete operations.
3. Builds provider-specific modification commands and orders dependent operations.
4. Batches commands when supported and beneficial.
5. Executes the commands, normally using a transaction when needed and supported.
6. Accepts the changes after successful execution, unless configured not to.

```csharp
var affected = await db.SaveChangesAsync(cancellationToken);
```

The returned integer is the number of state entries written by the save operation, not necessarily the number of physical SQL statements or rows in every provider-specific case.

### 3.4 `AcceptAllChanges` and retry awareness

```csharp
await db.SaveChangesAsync(
    acceptAllChangesOnSuccess: false,
    cancellationToken);

// After the application has safely handled the outcome:
db.ChangeTracker.AcceptAllChanges();
```

This overload is advanced and should not be used casually. It can be useful in carefully designed workflows where accepting tracked state must be delayed. If the database committed but the client lost the response, the outcome may be ambiguous; blindly retrying inserts can duplicate data. Use idempotency keys, unique constraints, or a deliberate retry strategy.

### 3.5 Disconnected updates and PATCH

A context does not automatically know the original values of an object deserialized from an HTTP request.

Prefer loading the existing entity and applying allowed changes:

```csharp
var customer = await db.Customers
    .SingleOrDefaultAsync(c => c.Id == request.Id, cancellationToken);

if (customer is null)
    return;

customer.Name = request.Name;
customer.Phone = request.Phone;

await db.SaveChangesAsync(cancellationToken);
```

Avoid blindly doing this with client-supplied graphs:

```csharp
db.Update(requestEntity);
```

`Update` can mark an entire graph as modified and may cause unintended updates. For a deliberate partial update, attach the entity and mark only the intended property:

```csharp
db.Attach(customer);
db.Entry(customer).Property(c => c.Name).IsModified = true;
await db.SaveChangesAsync(cancellationToken);
```

Validate authorization and allowed fields before applying updates.

### 3.6 Identity resolution

A tracking context normally maintains one tracked instance per entity key. If the same row appears multiple times in a query graph, EF can reuse the same instance.

- `AsTracking()`: tracks entities and uses the context's identity map.
- `AsNoTracking()`: avoids normal context tracking; repeated occurrences may produce separate instances.
- `AsNoTrackingWithIdentityResolution()`: uses temporary identity resolution for the query without retaining those entities in the context's normal change tracker.

---

## 4. Transactions, concurrency, and consistency

### 4.1 Automatic versus explicit transactions

For relational providers, a single `SaveChanges` operation is transactional by default when the provider supports transactions and EF controls the transaction. If an explicit or ambient transaction already exists, behavior differs.

```csharp
await using var tx =
    await db.Database.BeginTransactionAsync(cancellationToken);

try
{
    db.Orders.Add(order);
    await db.SaveChangesAsync(cancellationToken);

    db.AuditRecords.Add(audit);
    await db.SaveChangesAsync(cancellationToken);

    await tx.CommitAsync(cancellationToken);
}
catch
{
    await tx.RollbackAsync(cancellationToken);
    throw;
}
```

An explicit transaction is useful when multiple saves or database operations must commit together. Do not wrap every single `SaveChanges` in an explicit transaction without a reason.

### 4.2 Savepoints and execution strategies

EF Core may create a savepoint before `SaveChanges` when it runs inside an existing transaction and the provider supports it. A failure may roll back to that savepoint. SQL Server Multiple Active Result Sets (MARS) affects savepoint behavior; check provider documentation.

Connection resiliency execution strategies may retry transient failures. A manually controlled transaction must be coordinated with the execution strategy; do not assume a transaction block can be safely retried without wrapping the entire unit of work appropriately. Retried work must also be designed for idempotency.

### 4.3 Optimistic concurrency

EF Core does not automatically add a row-version column to every table. Concurrency checking requires a configured concurrency token.

```csharp
public sealed class InventoryItem
{
    public int Id { get; set; }
    public int Quantity { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = [];
}
```

When updating, EF includes the original concurrency token in the update predicate. If zero rows match, EF can throw `DbUpdateConcurrencyException`.

```csharp
try
{
    item.Quantity -= quantity;
    await db.SaveChangesAsync(cancellationToken);
}
catch (DbUpdateConcurrencyException)
{
    // Reload, merge, reject, or ask the caller to retry.
    throw;
}
```

Choose a conflict policy: retry with fresh data, merge fields, reject the request, or return a conflict response. A row-version token detects conflicting writes; it does not itself decide the business policy.

### 4.4 Isolation levels and lost updates

Know the practical differences among Read Committed, Repeatable Read, Snapshot, and Serializable for the target database. Stronger isolation can reduce certain anomalies but may increase blocking, deadlocks, or version-store pressure. Correctness depends on the database engine and configuration.

For inventory or financial invariants, consider a conditional database update, constraint, or stored procedure rather than relying only on a read-then-write sequence.

### 4.5 Cross-service consistency

A database transaction cannot normally atomically commit changes across independent service databases and a message broker. Use patterns such as:
- **Transactional outbox:** persist business changes and an outgoing event in one database transaction; publish asynchronously.
- **Inbox/deduplication:** recognize duplicate messages.
- **Saga/compensation:** coordinate a multi-step business process when a global ACID transaction is unavailable.

---

## 5. Relationships and model mapping

### 5.1 Relationship essentials

Understand:
- One-to-one, one-to-many, many-to-many.
- Foreign keys versus navigation properties.
- Required versus optional relationships.
- Shadow foreign keys.
- Cascade delete and orphan deletion.
- Relationship fix-up while entities are tracked.
- Owned/complex types and value objects, depending on EF Core version.

Prefer explicit Fluent API configuration for important domain rules:

```csharp
modelBuilder.Entity<Order>(entity =>
{
    entity.HasKey(o => o.Id);

    entity.HasMany(o => o.Lines)
        .WithOne(line => line.Order)
        .HasForeignKey(line => line.OrderId)
        .IsRequired();

    entity.Property(o => o.Number)
        .HasMaxLength(40)
        .IsRequired();

    entity.HasIndex(o => new { o.CustomerId, o.CreatedAt });
});
```

### 5.2 Cascade behavior is not the same as database cascade behavior

EF may mark dependent entities for deletion when tracked relationships are severed, while the database may also enforce `ON DELETE` actions. The exact behavior depends on relationship configuration, entity state, whether dependents are loaded, and provider/schema settings. Know which layer enforces each rule.

### 5.3 Inheritance mapping

- **TPH (Table-per-Hierarchy):** one table with a discriminator; often fewer joins, potentially nullable columns.
- **TPT (Table-per-Type):** separate tables for base and derived types; can require more joins.
- **TPC (Table-per-Concrete-Type):** each concrete type stores inherited properties; can duplicate columns and complicate polymorphic queries.

Choose based on query patterns, schema requirements, and measured plans—not solely on object-oriented elegance.

### 5.4 Conversions, owned data, and JSON

Use value conversions for representation changes such as enum-to-string or a strongly typed identifier. A converter does not automatically make every complex object a queryable JSON document. JSON mapping capabilities vary by EF Core version and provider.

Use a `ValueComparer` when EF needs correct equality, hashing, or snapshot behavior for mutable/custom values. Otherwise, a mutable collection stored in a converted property may not be detected as expected.

### 5.5 Global query filters and tenant security

```csharp
modelBuilder.Entity<Document>()
    .HasQueryFilter(d => d.TenantId == _tenantContext.TenantId
                      && !d.IsDeleted);
```

Filters are useful for tenant scoping and soft deletion, but they are not a complete security boundary:
- `IgnoreQueryFilters()` can bypass them.
- Raw SQL and other access paths require review.
- Tenant state must be correct for each context, especially with pooling.
- Writes must validate tenant ownership too.
- Database permissions, row-level security, or database-per-tenant may be appropriate for stronger isolation.

Never trust a client-supplied `TenantId` without authorization checks.

---

## 6. Performance engineering

### 6.1 Common sources of poor performance

- N+1 queries from lazy loading or repeated queries in loops.
- Loading entire entity graphs when a DTO projection is sufficient.
- Missing or poorly ordered indexes.
- Large `Skip`/`Take` offsets.
- Unbounded result sets and unnecessary materialization.
- Tracking large read-only graphs.
- Client-side transformations after materialization.
- Too many round trips or oversized joins.
- Poorly selective predicates and non-sargable SQL.
- Excessive parallelism against one context or the database.

### 6.2 N+1 and loading strategies

```csharp
// Risk: if lazy loading is enabled, accessing Lines can issue one query per order.
var orders = await db.Orders.ToListAsync(cancellationToken);

// Better when a graph is needed:
var ordersWithLines = await db.Orders
    .Include(o => o.Lines)
    .ToListAsync(cancellationToken);

// Often better for API output:
var summaries = await db.Orders
    .Select(o => new OrderSummaryDto(
        o.Id, o.Number, o.Lines.Count))
    .ToListAsync(cancellationToken);
```

`Include` is not always best. Multiple collection includes can multiply joined rows. Compare single-query and split-query behavior; split queries reduce join duplication but add round trips and may observe inconsistent data across commands without appropriate transaction/isolation.

### 6.3 Offset versus keyset pagination

Offset pagination:

```csharp
var page = await db.Events
    .OrderBy(e => e.Id)
    .Skip(offset)
    .Take(pageSize)
    .ToListAsync(cancellationToken);
```

Large offsets may be expensive. Keyset pagination uses the last-seen key:

```csharp
var page = await db.Events
    .Where(e => e.Id > lastSeenId)
    .OrderBy(e => e.Id)
    .Take(pageSize)
    .ToListAsync(cancellationToken);
```

For non-unique sort columns, use a stable composite order (for example, `CreatedAt` plus `Id`) and a matching predicate/index. Keyset pagination is less suitable when the user must jump directly to an arbitrary page number.

### 6.4 Bulk operations

`AddRange` is convenient but does not turn ordinary EF inserts into a database-native bulk-copy operation. For large volumes, evaluate:
- Batching tracked inserts.
- Provider-specific bulk-copy APIs or a vetted bulk library.
- `ExecuteUpdate` / `ExecuteDelete` where supported (EF Core 7+).
- Set-based SQL or stored procedures.

`ExecuteUpdate` and `ExecuteDelete` execute directly in the database without loading entities and without synchronizing already tracked instances. If the same rows are tracked, clear/reload or use a fresh context to avoid stale in-memory state.

### 6.5 Compiled queries and model optimization

Compiled queries can help in hot paths with stable, frequently executed query shapes, but measure first: database latency often dominates. EF Core also supports compiled models for reducing model initialization cost in suitable applications; this is different from compiled queries. Do not describe compiled models as a universal source-generator feature.

### 6.6 DbContext pooling versus connection pooling

- **DbContext pooling:** reuses context instances to reduce EF initialization overhead.
- **ADO.NET connection pooling:** reuses physical database connections.

They solve different problems. Connection pooling is generally managed by the provider; context pooling requires care with per-request state.

### 6.7 Logging and query diagnosis

Use EF logging and interceptors for observability, but avoid logging secrets or sensitive parameter values. For real performance work, inspect:
1. Generated SQL.
2. Actual query plan.
3. Index usage and logical reads.
4. Round trips and row counts.
5. Allocations and materialization costs.
6. Locking, waits, and database telemetry.

---

## 7. Advanced LINQ

### 7.1 `Func<T, bool>` versus `Expression<Func<T, bool>>`

```csharp
Func<Customer, bool> localPredicate =
    c => c.IsActive;

Expression<Func<Customer, bool>> queryPredicate =
    c => c.IsActive;
```

A `Func` is executable .NET code. An expression tree is a structured representation that a provider can inspect and potentially translate. Passing a compiled delegate into a query can force work to occur in memory or make translation impossible, depending on how it is used.

### 7.2 `Any` versus `Count`

Prefer `Any()` when the question is whether at least one row exists:

```csharp
bool hasOpenOrders = await db.Orders
    .AnyAsync(o => o.CustomerId == customerId &&
                   o.Status == OrderStatus.Open,
              cancellationToken);
```

`Count()` asks for the count and may require more work. The database optimizer can optimize both, but use the operator that matches the intent.

### 7.3 Left join pattern

A traditional LINQ left join uses `GroupJoin`, `SelectMany`, and `DefaultIfEmpty`:

```csharp
var result =
    from customer in db.Customers
    join order in db.Orders
        on customer.Id equals order.CustomerId into orders
    from order in orders.DefaultIfEmpty()
    select new
    {
        customer.Id,
        customer.Name,
        OrderId = order == null ? (int?)null : order.Id
    };
```

Translation depends on expression shape and provider. Newer .NET/EF Core versions may provide additional query syntax/operator support; check the target version.

### 7.4 `GroupBy` translation

A `GroupBy` that immediately reduces each group to aggregates is often translatable:

```csharp
var totals = await db.OrderLines
    .GroupBy(line => line.ProductId)
    .Select(group => new
    {
        ProductId = group.Key,
        Quantity = group.Sum(line => line.Quantity)
    })
    .ToListAsync(cancellationToken);
```

Returning full `IGrouping<TKey,TElement>` objects is not equivalent to SQL `GROUP BY` and may require client-side grouping or may not translate, depending on version and query shape.

### 7.5 Deferred execution and repeated enumeration

```csharp
var query = db.Customers.Where(c => c.IsActive);

// Each enumeration can execute the query again.
var first = await query.ToListAsync(cancellationToken);
var second = await query.ToListAsync(cancellationToken);
```

Materialize once when reuse is intended and the result size is acceptable. Do not materialize early by habit if the query could still be filtered, ordered, or projected in SQL.

### 7.6 Dynamic predicates

Build expression trees or compose predicates carefully when filters are optional. Avoid string-concatenated SQL from user input. Verify that dynamic expression construction preserves parameterization and query-cache-friendly shapes.

---

## 8. Production architecture and security

### 8.1 Repository and Unit of Work choices

`DbContext` already provides unit-of-work behavior and `DbSet` offers repository-like operations. A custom repository can still be valuable when it expresses meaningful domain operations or hides persistence details. Avoid a generic repository that merely duplicates every `DbSet` method and blocks useful EF features without adding value.

### 8.2 CQRS and read/write separation

A practical approach:
- Use EF Core for aggregate-oriented writes and transactional business rules.
- Use projections for simple read endpoints.
- Consider Dapper, views, or specialized read models for complex/high-volume reporting when profiling justifies it.
- Keep read/write separation driven by requirements, not by fashion.

### 8.3 Database constraints are the final integrity guard

Validate business rules in the application for useful errors, but enforce critical invariants in the database where possible:
- Unique constraints/indexes.
- Foreign keys.
- Check constraints.
- Required columns and appropriate data types.
- Concurrency tokens.

Application validation alone cannot prevent races between concurrent requests.

### 8.4 Interceptors versus logging and domain events

Interceptors can observe or influence EF operations such as commands, connections, transactions, and `SaveChanges`. Use them for cross-cutting persistence concerns such as diagnostics or carefully designed auditing.

Do not use a command interceptor as a substitute for domain logic. A `SaveChangesInterceptor` can help with audit fields, but it must handle added/modified/deleted entries, recursion risks, async paths, and sensitive data correctly. Domain events should represent business meaning, not merely SQL execution.

### 8.5 Soft delete and auditing

A global query filter can hide soft-deleted rows, but the write path must consistently mark deletion and audit metadata. Decide whether audit records must be immutable, whether updates to audit rows are prohibited, and whether the database or application is the trusted source.

### 8.6 Raw SQL and injection

Use parameterized APIs:

```csharp
var customers = await db.Customers
    .FromSqlInterpolated(
        $"SELECT * FROM dbo.Customers WHERE City = {city}")
    .ToListAsync(cancellationToken);
```

For `FromSqlRaw`, pass parameters separately; do not concatenate untrusted values into SQL. Parameters cannot generally stand in for SQL identifiers such as table names or sort directions, so dynamic identifiers require strict allow-listing.

### 8.7 Async and cancellation

Use async database APIs in request paths to avoid blocking threads while I/O is pending. Pass cancellation tokens through the full call chain. Async does not make one `DbContext` thread-safe, and it does not automatically make a query faster at the database level.

---

## 9. Testing, migrations, and operations

### 9.1 Testing options

- **Unit tests:** test pure business logic without a database.
- **SQLite in-memory:** useful relational behavior, but not identical to SQL Server/PostgreSQL.
- **Production-provider integration tests:** best for validating provider-specific SQL, migrations, constraints, and transaction behavior.
- **EF Core InMemory provider:** useful for some tests but not a relational substitute; it does not reproduce all relational constraints and query semantics.

### 9.2 Migrations in production

Recommended discipline:
1. Review generated migration code and SQL.
2. Test on a production-like database and realistic data volume.
3. Prefer backward-compatible expand/migrate/contract changes for zero-downtime deployments.
4. Avoid blindly applying migrations at application startup in multi-instance production deployments.
5. Plan backups, locks, long-running index operations, and rollback/forward-fix strategy.
6. Use a controlled deployment pipeline with permissions appropriate for schema changes.

### 9.3 Temporal tables, soft deletes, and audit logs are different

- **Temporal tables:** database-managed row history, when supported/configured.
- **Soft delete:** application convention that marks records as deleted.
- **Audit log:** records who did what and when, often with business context.

They solve different requirements and can coexist.

### 9.4 Test the actual query plan

An in-memory test cannot establish that a SQL query is fast. Use realistic data volume, indexes, the target database engine, and actual execution plans. Include cancellation, transient failure, deadlock, concurrency conflict, and duplicate-message scenarios where relevant.

---

## 10. Interview MCQs with answers

Try to answer each question before opening its answer.

### Q1. When does a LINQ query against `DbSet<T>` usually execute?

A. When `Where` is called  
B. When `OrderBy` is called  
C. When the query is enumerated or a terminal operator executes  
D. When the `DbContext` is constructed  

**Answer: C.** Query operators usually compose an expression; materialization or enumeration triggers execution.

### Q2. What is the main difference between `IQueryable<T>` and `IEnumerable<T>`?

A. `IQueryable` always executes on the database  
B. `IQueryable` exposes an expression/provider model; `IEnumerable` represents in-process enumeration  
C. `IEnumerable` cannot be filtered  
D. They are identical  

**Answer: B.** An `IQueryable` provider may translate expressions, but not every expression is translatable.

### Q3. Does EF Core automatically configure a row-version column on every entity?

A. Yes  
B. Only for SQL Server  
C. No; concurrency tokens must be configured  
D. Only when using `SaveChangesAsync`  

**Answer: C.** Optimistic concurrency depends on a configured concurrency token or equivalent mechanism.

### Q4. What does `AsNoTrackingWithIdentityResolution()` do?

A. Tracks every entity normally  
B. Avoids normal context tracking while resolving duplicate entity identities within the result  
C. Caches SQL results across requests  
D. Disables identity columns in the database  

**Answer: B.** It uses identity resolution without adding the returned entities to the context's normal tracked set.

### Q5. What is a likely consequence of calling `Update()` on a disconnected entity graph?

A. Only changed properties are always updated  
B. The graph may be marked modified, potentially causing unintended updates  
C. EF automatically reloads every row first  
D. It bypasses database constraints  

**Answer: B.** Disconnected graphs require careful state management and validation.

### Q6. What does a single relational `SaveChanges` normally provide when EF controls the transaction?

A. Distributed transaction across every service  
B. Atomicity for the changes in that save operation  
C. Automatic conflict resolution  
D. Automatic retry of every failure  

**Answer: B.** This does not make cross-service operations atomic or decide concurrency policy.

### Q7. Why can a query with several collection `Include`s be slow?

A. `Include` always forces client evaluation  
B. Joins can multiply rows and duplicate parent data  
C. Tracking is always disabled  
D. SQL indexes cannot be used  

**Answer: B.** Evaluate projection, split queries, and actual execution plans.

### Q8. Which is generally preferable for checking whether any matching rows exist?

A. `Count() > 0` in every case  
B. `Any()`  
C. `ToList().Count > 0`  
D. `First()` and catch every exception  

**Answer: B.** It expresses existence and can allow the database to stop after finding a match.

### Q9. Does `ExecuteUpdate` update already tracked entities in memory?

A. Yes, always  
B. No; tracked instances can become stale  
C. Only when lazy loading is enabled  
D. Only after `AcceptAllChanges`  

**Answer: B.** Set-based operations bypass normal tracked-entity synchronization.

### Q10. What is the purpose of a transactional outbox?

A. Cache every query  
B. Atomically store business changes and a message record in the same database transaction  
C. Replace foreign keys  
D. Make message delivery exactly once automatically  

**Answer: B.** Publishing is usually asynchronous and consumers still need idempotency/deduplication.

### Q11. Is `DbContext` safe for concurrent operations?

A. Yes, if methods are async  
B. Yes, if the connection string is pooled  
C. No; use separate contexts for concurrent units of work  
D. Only reads can run concurrently on one context  

**Answer: C.** Async does not change the context's thread-safety contract.

### Q12. What is the main limitation of EF Core's InMemory provider for integration testing?

A. It cannot store objects  
B. It does not reproduce relational database behavior faithfully  
C. It always needs SQL Server  
D. It cannot be used in tests  

**Answer: B.** Use a relational provider for relational behavior and the real provider for provider-specific validation.

### Q13. What is a common benefit of keyset pagination?

A. Supports arbitrary page jumps more efficiently in every case  
B. Avoids scanning/skipping a large offset when using a suitable ordered key and index  
C. Removes the need for ordering  
D. Guarantees a consistent snapshot across requests  

**Answer: B.** It requires a stable order and is not a universal replacement for offset pagination.

### Q14. What does a value converter do?

A. Automatically makes every object queryable as JSON  
B. Converts a model value to/from its provider representation  
C. Replaces entity tracking  
D. Creates database indexes automatically  

**Answer: B.** Mutable/custom values may also need a `ValueComparer`.

### Q15. Why is `IgnoreQueryFilters()` a security concern in a multi-tenant application?

A. It disables SQL parameterization  
B. It can bypass tenant or soft-delete filters  
C. It disables transactions  
D. It prevents indexes from being used  

**Answer: B.** Authorization and tenant isolation must be enforced beyond a convention that code can bypass.

### Q16. What does `ToQueryString()` provide?

A. A guaranteed byte-for-byte copy of the command executed in every situation  
B. A debugging representation of the query SQL  
C. The database's actual execution plan  
D. All parameter values including secrets  

**Answer: B.** Use command logs and database tooling to inspect actual execution and plans.

### Q17. Which is a sound reason to use a custom repository over `DbContext` directly?

A. Every project must have a generic repository  
B. It encapsulates meaningful domain persistence operations or a useful boundary  
C. It guarantees better performance  
D. It removes the need for integration tests  

**Answer: B.** Abstraction should provide value rather than mechanically duplicating EF APIs.

### Q18. What should a robust migration process include?

A. Apply migrations automatically on every production instance startup  
B. Review generated SQL, test on realistic data, and plan deployment/rollback  
C. Assume all schema changes are instant  
D. Skip migration testing if unit tests pass  

**Answer: B.** Schema changes can lock tables, take time, and require coordinated deployment.

### Q19. What is the difference between context pooling and connection pooling?

A. They are the same feature  
B. Context pooling reuses EF context instances; connection pooling reuses database connections  
C. Context pooling is a database index feature  
D. Connection pooling tracks entity states  

**Answer: B.** They address different initialization and resource costs.

### Q20. A `GroupBy` query is most likely to translate cleanly when it:

A. Returns arbitrary full grouping objects in every provider/version  
B. Groups and immediately projects keys with aggregates  
C. Calls `ToList()` before grouping in every case  
D. Uses only client-side methods  

**Answer: B.** Translation depends on provider, EF Core version, and expression shape.

---

## 11. Scenario-based interview questions

### Scenario A: Duplicate orders after retry

**Problem:** A client submits an order. The database commits, but the API loses the connection before returning success. The client retries and creates a duplicate.

**Strong answer should mention:**
- The commit outcome may be ambiguous.
- A database transaction alone does not make an HTTP retry idempotent.
- Use a client/request idempotency key with a unique constraint.
- Persist the key and result transactionally with the order.
- Make downstream event consumers idempotent; consider the outbox pattern.

### Scenario B: Inventory overselling

**Problem:** Two requests read stock quantity 1 and both attempt to buy the final item.

**Strong answer should mention:**
- A read-then-write sequence has a race.
- Use a conditional atomic update such as `UPDATE ... WHERE Quantity >= @requested`.
- Check affected rows and enforce relevant constraints.
- Alternatively use a concurrency token with a deliberate retry/conflict policy.
- Keep the transaction scope short and test concurrent requests.

### Scenario C: API becomes slow as data grows

**Problem:** An endpoint was fast with 500 rows but is slow with 5 million.

**Investigation plan:**
1. Capture actual SQL and parameter types.
2. Check actual execution plan, indexes, logical reads, and row estimates.
3. Check N+1 round trips and over-fetching.
4. Replace unnecessary entity graph materialization with projection.
5. Add bounded pagination; consider keyset pagination.
6. Measure database time separately from EF materialization and serialization.
7. Validate the improvement against production-like data.

### Scenario D: Tenant data leaks between requests

**Problem:** The application uses a global query filter and context pooling.

**Strong answer should mention:**
- Verify tenant context state cannot leak across reused context instances.
- Do not trust tenant IDs from request payloads.
- Apply tenant authorization to writes as well as reads.
- Audit raw SQL and `IgnoreQueryFilters()` usage.
- Consider database row-level security or database-per-tenant where the threat model requires it.
- Add cross-tenant integration tests.

### Scenario E: Updating a disconnected graph

**Problem:** A frontend sends an `Order` with lines, and the API calls `Update(order)`.

**Strong answer should mention:**
- `Update` can mark many graph entries modified.
- The request may omit properties or contain unauthorized child IDs.
- Load the aggregate, validate ownership and allowed changes, then apply them.
- Use explicit state setting only when the API contract truly represents a partial update.
- Enforce foreign keys and concurrency rules in the database.

### Scenario F: A read endpoint uses many `Include`s

**Problem:** The query returns duplicate parent data and consumes excessive memory.

**Strong answer should mention:**
- Inspect generated SQL and row multiplication.
- Project into a DTO if the API needs a read model rather than a mutable entity graph.
- Compare split and single queries.
- Add appropriate indexes and pagination.
- Consider a dedicated read model if the query is fundamentally reporting-oriented.

---

## 12. Final revision checklist

Before a senior/architect interview, make sure you can explain and demonstrate:

- [ ] `DbContext`, `DbSet`, model metadata, state manager, and provider responsibilities.
- [ ] Expression trees, query translation, materialization, deferred execution, and query caching.
- [ ] Entity states, snapshot detection, identity resolution, and disconnected updates.
- [ ] The `SaveChanges` pipeline, transaction boundaries, savepoints, and retry implications.
- [ ] Optimistic concurrency, isolation levels, lost updates, and conflict resolution.
- [ ] Relationships, cascade behavior, inheritance mapping, conversions, and shadow properties.
- [ ] N+1 detection, projection, indexes, execution plans, pagination, and bulk operations.
- [ ] Interceptors versus logging, domain events, and database constraints.
- [ ] Tenant isolation, global filters, pooling, and authorization of writes.
- [ ] Migrations, relational testing, idempotency, outbox/inbox, and deployment safety.
- [ ] When EF Core is appropriate and when Dapper, raw SQL, or specialized read models are justified.

## Suggested official references

- EF Core documentation: https://learn.microsoft.com/ef/core/
- Efficient querying: https://learn.microsoft.com/ef/core/performance/efficient-querying
- Change tracking: https://learn.microsoft.com/ef/core/change-tracking/
- Transactions: https://learn.microsoft.com/ef/core/saving/transactions
- Concurrency handling: https://learn.microsoft.com/ef/core/saving/concurrency
- Interceptors: https://learn.microsoft.com/ef/core/logging-events-diagnostics/interceptors
- Testing: https://learn.microsoft.com/ef/core/testing/
- LINQ documentation: https://learn.microsoft.com/dotnet/csharp/linq/
