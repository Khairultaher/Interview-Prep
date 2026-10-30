# EF Core, LINQ, and SQL Server

Each concept is explained once. Where several notes covered the same topic, they have been merged into a single section, and related topics in other guides are linked instead of repeated.

## Contents

- [EF Core & LINQ: Senior .NET Engineer / Solution Architect Interview Guide](#ef-core-linq-senior-interview-guide-md)
- [EF Core Viva — More Quick Questions](#ef-core-viva-preparation-md)
- [EF Core Performance: Hands-On Benchmark Lab](#ef-core-performance-interview-and-benchmark-lab-md)
- [SQL Server: Grouping, PIVOT/UNPIVOT, and Window Functions](#sql-server-advanced-grouping-and-window-functions-md)
- [Additional Questpond question sets](#additional-questpond-question-sets)

---

<a id="ef-core-linq-senior-interview-guide-md"></a>

## EF Core & LINQ: Senior .NET Engineer / Solution Architect Interview Guide

> A consolidated study guide focused on internals, trade-offs, production design, and interview questions. It avoids repeating the same introductory topics in multiple sections.
>
> **Version note:** EF Core behavior and available APIs vary by version and database provider. Verify version-specific features against the documentation for the version used in the interview or project.

#### Contents

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

#### 1. Mental model and architecture

##### 1.1 What each component does

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

##### 1.2 Typical write path

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

##### 1.3 `DbContext` lifetime

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

#### 2. Query execution: LINQ to SQL

##### 2.1 `IQueryable` versus `IEnumerable`

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

##### 2.2 Query pipeline (conceptual)

1. C# query operators construct expression trees.
2. EF Core normalizes and preprocesses the expression.
3. The provider translates supported parts into a database query.
4. EF Core creates/uses a query execution plan and database command.
5. Parameters are sent to the provider.
6. The database executes SQL and returns rows.
7. EF materializes entities/DTOs and performs identity resolution when applicable.

EF caches query compilation work for recurring query shapes. Different constants, expression shapes, dynamic predicates, and provider behavior can affect cache reuse. Parameterized query patterns are generally preferable to generating many distinct expression shapes.

##### 2.3 Inspect generated SQL

```csharp
var query = db.Orders
    .Where(o => o.Status == OrderStatus.Open)
    .Select(o => new { o.Id, o.CreatedAt, o.Total });

string sql = query.ToQueryString();
```

`ToQueryString()` is useful for debugging, but the executed command can differ due to provider behavior, parameter values, terminal operators, and runtime execution. Also inspect database execution plans and actual I/O.

##### 2.4 Projection is often better than loading full entities

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

#### 3. Change tracking and `SaveChanges`

##### 3.1 Entity states

| State | Meaning |
|---|---|
| `Detached` | Not tracked by this context. |
| `Added` | EF intends to insert the entity. |
| `Unchanged` | Tracked; no detected modifications. |
| `Modified` | One or more properties are marked modified. |
| `Deleted` | EF intends to delete the entity. |

An entity returned by a normal tracking query is typically `Unchanged` when first materialized. An `AsNoTracking()` query returns entities that are not tracked by the context.

##### 3.2 Snapshot tracking and notifications

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

##### 3.3 How `SaveChanges` works

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

##### 3.4 `AcceptAllChanges` and retry awareness

```csharp
await db.SaveChangesAsync(
    acceptAllChangesOnSuccess: false,
    cancellationToken);

// After the application has safely handled the outcome:
db.ChangeTracker.AcceptAllChanges();
```

This overload is advanced and should not be used casually. It can be useful in carefully designed workflows where accepting tracked state must be delayed. If the database committed but the client lost the response, the outcome may be ambiguous; blindly retrying inserts can duplicate data. Use idempotency keys, unique constraints, or a deliberate retry strategy.

##### 3.5 Disconnected updates and PATCH

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

##### 3.6 Identity resolution

A tracking context normally maintains one tracked instance per entity key. If the same row appears multiple times in a query graph, EF can reuse the same instance.

- `AsTracking()`: tracks entities and uses the context's identity map.
- `AsNoTracking()`: avoids normal context tracking; repeated occurrences may produce separate instances.
- `AsNoTrackingWithIdentityResolution()`: uses temporary identity resolution for the query without retaining those entities in the context's normal change tracker.

##### 3.7 Shadow properties

A shadow property belongs to EF Core's model but is not declared on the CLR entity class. It can represent a foreign key or metadata such as audit information.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Student>()
        .Property<DateTime>("CreatedAt")
        .HasDefaultValueSql("SYSUTCDATETIME()");
}
```

Access a tracked entity's shadow value:

```csharp
var student = await db.Students.SingleAsync(s => s.Id == id);
var createdAt = db.Entry(student).Property<DateTime>("CreatedAt").CurrentValue;
```

Shadow values are maintained by the context while tracked; they are not ordinary CLR properties. For audit timestamps, consider whether a database default, `SaveChanges` override, interceptor, or explicit domain property best fits the requirements.


---

#### 4. Transactions, concurrency, and consistency

##### 4.1 Automatic versus explicit transactions

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

##### 4.2 Savepoints and execution strategies

EF Core may create a savepoint before `SaveChanges` when it runs inside an existing transaction and the provider supports it. A failure may roll back to that savepoint. SQL Server Multiple Active Result Sets (MARS) affects savepoint behavior; check provider documentation.

Connection resiliency execution strategies may retry transient failures. A manually controlled transaction must be coordinated with the execution strategy; do not assume a transaction block can be safely retried without wrapping the entire unit of work appropriately. Retried work must also be designed for idempotency.

##### 4.3 Optimistic concurrency

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

**Resolving a conflict in practice:** inspect what's in the database now, then apply your policy.

Optimistic concurrency allows concurrent reads but detects a conflicting write at update/delete time. Configure a concurrency token so EF Core includes the original token in the write predicate.

For SQL Server, a `rowversion` column is a common option:

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public byte[] RowVersion { get; set; } = [];
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Student>()
        .Property(s => s.RowVersion)
        .IsRowVersion();
}
```

Handle the conflict:

```csharp
try
{
    await db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        var databaseValues = await entry.GetDatabaseValuesAsync();

        if (databaseValues is null)
        {
            // The row was deleted.
        }
        else
        {
            // Compare entry.OriginalValues, entry.CurrentValues,
            // and databaseValues, then apply a conflict policy.
        }
    }

    // Resolve the conflict deliberately; do not blindly retry forever.
}
```

`rowversion` is SQL Server-specific. Other providers may use another concurrency token, such as an application-managed version number. A conflict policy may let the user choose, merge selected fields, reject the change, or retry after refreshing values.


##### 4.4 Isolation levels and lost updates

Know the practical differences among Read Committed, Repeatable Read, Snapshot, and Serializable for the target database. Stronger isolation can reduce certain anomalies but may increase blocking, deadlocks, or version-store pressure. Correctness depends on the database engine and configuration.

For inventory or financial invariants, consider a conditional database update, constraint, or stored procedure rather than relying only on a read-then-write sequence.

##### 4.5 Cross-service consistency

A database transaction cannot normally atomically commit changes across independent service databases and a message broker. Use patterns such as:
- **Transactional outbox:** persist business changes and an outgoing event in one database transaction; publish asynchronously.
- **Inbox/deduplication:** recognize duplicate messages.
- **Saga/compensation:** coordinate a multi-step business process when a global ACID transaction is unavailable.

---

#### 5. Relationships and model mapping

##### 5.1 Relationship essentials

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

##### 5.2 Cascade behavior is not the same as database cascade behavior

EF may mark dependent entities for deletion when tracked relationships are severed, while the database may also enforce `ON DELETE` actions. The exact behavior depends on relationship configuration, entity state, whether dependents are loaded, and provider/schema settings. Know which layer enforces each rule.

##### 5.3 Inheritance mapping: TPH, TPT, and TPC

EF Core supports three common relational inheritance mapping strategies.

```csharp
public abstract class Person
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

public class Student : Person
{
    public int Grade { get; set; }
}

public class Teacher : Person
{
    public string Subject { get; set; } = "";
}
```

###### TPH — Table Per Hierarchy

All hierarchy types share one table, usually with a discriminator column.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>().ToTable("People");
    // TPH is the default inheritance strategy.
}
```

- Usually efficient for polymorphic queries because one table is queried.
- Derived-type columns may be nullable for rows of other types.
- Constraints that depend on the discriminator may require careful database design.

###### TPT — Table Per Type

Each mapped type has a table; derived tables share the primary key with the base table.

```csharp
modelBuilder.Entity<Person>().ToTable("People");
modelBuilder.Entity<Student>().ToTable("Students");
modelBuilder.Entity<Teacher>().ToTable("Teachers");
```

- Clearer separation of type-specific columns.
- Queries across the hierarchy can require joins.
- Performance should be measured for the actual query workload.

###### TPC — Table Per Concrete Type

Each concrete type has a table containing inherited and declared properties. The abstract base type need not have its own table.

```csharp
modelBuilder.Entity<Person>().UseTpcMappingStrategy();
modelBuilder.Entity<Student>().ToTable("Students");
modelBuilder.Entity<Teacher>().ToTable("Teachers");
```

- Avoids joins to a base table for many derived-type queries.
- Repeats inherited columns across concrete tables.
- Key generation and polymorphic queries need consideration.

**Viva tip:** Choose based on query patterns, constraints, and data shape—not just normalization preference. Confirm the capabilities of your EF Core version and provider.

##### 5.4 Conversions, owned data, and JSON

Use value conversions for representation changes such as enum-to-string or a strongly typed identifier. A converter does not automatically make every complex object a queryable JSON document. JSON mapping capabilities vary by EF Core version and provider.

Use a `ValueComparer` when EF needs correct equality, hashing, or snapshot behavior for mutable/custom values. Otherwise, a mutable collection stored in a converted property may not be detected as expected.

##### 5.5 Global query filters and tenant security

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

**Multi-tenancy isolation models**

Common isolation models:

1. **Database per tenant:** strong isolation and independent lifecycle, with greater provisioning and operational cost.
2. **Schema per tenant:** can fit some relational designs, but schema management and provider support add complexity.
3. **Shared database with `TenantId` columns:** operationally efficient, but every access path must enforce tenant isolation.

A global query filter can help with row-based tenancy:

```csharp
public sealed class SchoolContext : DbContext
{
    private readonly string _tenantId;

    public SchoolContext(DbContextOptions<SchoolContext> options, ITenantContext tenant)
        : base(options) => _tenantId = tenant.TenantId;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Student>()
            .HasQueryFilter(s => s.TenantId == _tenantId);
    }
}
```

Important safeguards:

- Resolve tenant identity from trusted authenticated context, not an unvalidated client-supplied value.
- Enforce tenant ownership in writes as well as reads.
- Consider composite keys and unique indexes that include `TenantId`.
- Be very cautious with `IgnoreQueryFilters()`.
- Ensure background jobs, raw SQL, bulk operations, exports, and admin paths enforce the same boundary.
- A query filter is a useful guardrail, not a complete security boundary by itself. Database row-level security can add defense in depth.


---

#### 6. Performance engineering

##### 6.1 Common sources of poor performance

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

##### 6.2 N+1 and loading strategies

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

##### Eager, explicit, and lazy loading in detail

These strategies control when related entities are retrieved.

###### Eager loading

Fetch related data as part of the query.

```csharp
var courses = await db.Courses
    .Include(c => c.Students)
    .ThenInclude(s => s.Address)
    .ToListAsync();
```

Use it when the required object graph is known in advance. Multiple collection includes can cause row multiplication; consider `AsSplitQuery()` and compare its extra round trips with the cost of a single large join.

###### Explicit loading

Load a relationship deliberately after retrieving an entity.

```csharp
var course = await db.Courses.SingleAsync(c => c.Id == courseId);

await db.Entry(course)
    .Collection(c => c.Students)
    .LoadAsync();

// For a reference navigation:
await db.Entry(course)
    .Reference(c => c.Department)
    .LoadAsync();
```

You can filter what is loaded:

```csharp
await db.Entry(course)
    .Collection(c => c.Students)
    .Query()
    .Where(s => s.IsActive)
    .LoadAsync();
```

###### Lazy loading

A navigation access can trigger a database query. EF Core supports proxy-based lazy loading through `UseLazyLoadingProxies()` with overridable navigation properties, or lazy loading through `ILazyLoader`.

```csharp
services.AddDbContext<SchoolContext>(options =>
    options.UseSqlServer(connectionString)
           .UseLazyLoadingProxies());

public class Course
{
    public int Id { get; set; }
    public virtual ICollection<Student> Students { get; set; } = new List<Student>();
}
```

```csharp
var course = await db.Courses.SingleAsync(c => c.Id == courseId);
// May issue another query when the navigation is accessed:
var count = course.Students.Count;
```

**Risks:** hidden I/O, N+1 queries, serialization-triggered queries, and access after the context is disposed. Lazy loading is not automatically more efficient; use it intentionally.


##### 6.3 Offset versus keyset pagination

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

##### 6.4 Bulk operations

`AddRange` is convenient but does not turn ordinary EF inserts into a database-native bulk-copy operation. For large volumes, evaluate:
- Batching tracked inserts.
- Provider-specific bulk-copy APIs or a vetted bulk library.
- `ExecuteUpdate` / `ExecuteDelete` where supported (EF Core 7+).
- Set-based SQL or stored procedures.

`ExecuteUpdate` and `ExecuteDelete` execute directly in the database without loading entities and without synchronizing already tracked instances. If the same rows are tracked, clear/reload or use a fresh context to avoid stale in-memory state.

**`Remove()` vs `ExecuteDelete()`**

| Concern | `Remove()` | `ExecuteDelete()` |
|---|---|---|
| Entity required in memory? | Usually yes, unless attaching a stub | No |
| Change tracker | Marks tracked entity `Deleted` | Bypasses the change tracker |
| When SQL runs | At `SaveChanges()` | Immediately |
| Related behavior | EF's tracked delete behavior and database constraints apply | Database constraints/cascades apply; no tracked graph processing |
| Best fit | Ordinary entity lifecycle and domain logic | Set-based bulk deletion |

```csharp
var student = await db.Students.SingleAsync(s => s.Id == id);
db.Students.Remove(student);
await db.SaveChangesAsync();
```

Set-based alternative (EF Core 7+):

```csharp
var deleted = await db.Students
    .Where(s => s.IsInactive)
    .ExecuteDeleteAsync();
```

`ExecuteDeleteAsync()` executes immediately and does not synchronize already tracked instances. If the context has tracked students matching the predicate, their in-memory values/states can now be stale. Prefer a short-lived context or clear/reload affected tracked data. Check provider translation and database constraints.


##### 6.5 Compiled queries and model optimization

Compiled queries can help in hot paths with stable, frequently executed query shapes, but measure first: database latency often dominates. EF Core also supports compiled models for reducing model initialization cost in suitable applications; this is different from compiled queries. Do not describe compiled models as a universal source-generator feature.

##### 6.6 DbContext pooling versus connection pooling

- **DbContext pooling:** reuses context instances to reduce EF initialization overhead.
- **ADO.NET connection pooling:** reuses physical database connections.

They solve different problems. Connection pooling is generally managed by the provider; context pooling requires care with per-request state.

##### 6.7 Logging and query diagnosis

Use EF logging and interceptors for observability, but avoid logging secrets or sensitive parameter values. For real performance work, inspect:
1. Generated SQL.
2. Actual query plan.
3. Index usage and logical reads.
4. Round trips and row counts.
5. Allocations and materialization costs.
6. Locking, waits, and database telemetry.

Use **Query Store** (SQL Server / Azure SQL) to spot plan regressions over time, and compare estimated vs actual row counts in the plan.

##### 6.8 Quick tuning rules

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

**Keep predicates index-friendly (sargable)**

Avoid wrapping indexed columns in transformations when possible. For example, use a date range rather than comparing `o.CreatedAt.Date` to a date, where that translation would apply a function to the database column.

```csharp
var start = requestedDate.Date;
var end = start.AddDays(1);

var orders = await db.Orders
    .Where(o => o.CreatedAt >= start && o.CreatedAt < end)
    .ToListAsync(cancellationToken);
```

Index selection depends on the complete workload and execution plan; LINQ syntax alone does not guarantee index use.

**Key lookups**

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

**EF Core or Dapper?**

Use EF Core when change tracking, aggregate persistence, relationship
mapping, and maintainability are valuable. Consider Dapper or raw SQL
for specialized read models, reporting, or queries that need precise SQL
control. This is a workload-specific choice, not a universal rule that
EF Core is only for writes.

**What to monitor in a high-traffic EF Core service**

-   Endpoint p50/p95/p99 latency and error rate.
-   Database command duration and query count per request.
-   Query Store regressions, CPU, data IO, log IO, waits, and blocking.
-   Connection pool pressure and timeouts.
-   Rows returned versus rows read, plus memory allocation and GC.
-   Deadlocks, concurrency conflicts, retries, and failed transactions.

------------------------------------------------------------------------

**Rules of thumb**

- `AsNoTracking()` on the root query also applies to entities loaded via `Include()`.
- Read-only entity query → `AsNoTracking()`; repeated references that must share instances → `AsNoTrackingWithIdentityResolution()`.
- Multiple included collections causing huge joins → evaluate `AsSplitQuery()`.
- API response → narrow DTO projection.
- PLINQ is for in-memory work; it does not speed up a database query.
- Any performance claim → verify with SQL, execution plan, and measurements.


---

#### 7. Advanced LINQ

##### 7.1 `Func<T, bool>` versus `Expression<Func<T, bool>>`

```csharp
Func<Customer, bool> localPredicate =
    c => c.IsActive;

Expression<Func<Customer, bool>> queryPredicate =
    c => c.IsActive;
```

A `Func` is executable .NET code. An expression tree is a structured representation that a provider can inspect and potentially translate. Passing a compiled delegate into a query can force work to occur in memory or make translation impossible, depending on how it is used.

##### 7.2 `Any` versus `Count`

Prefer `Any()` when the question is whether at least one row exists:

```csharp
bool hasOpenOrders = await db.Orders
    .AnyAsync(o => o.CustomerId == customerId &&
                   o.Status == OrderStatus.Open,
              cancellationToken);
```

`Count()` asks for the count and may require more work. The database optimizer can optimize both, but use the operator that matches the intent.

##### 7.3 Left join pattern

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

##### 7.4 `GroupBy` translation

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

##### 7.5 Deferred execution and repeated enumeration

```csharp
var query = db.Customers.Where(c => c.IsActive);

// Each enumeration can execute the query again.
var first = await query.ToListAsync(cancellationToken);
var second = await query.ToListAsync(cancellationToken);
```

Materialize once when reuse is intended and the result size is acceptable. Do not materialize early by habit if the query could still be filtered, ordered, or projected in SQL.

##### 7.6 Dynamic predicates

Build expression trees or compose predicates carefully when filters are optional. Avoid string-concatenated SQL from user input. Verify that dynamic expression construction preserves parameterization and query-cache-friendly shapes.

##### 7.7 `First` vs `Single` (and `OrDefault`)

| Operator | Zero matches | Multiple matches |
|---|---|---|
| `First` | Throws | Returns first |
| `FirstOrDefault` | Default value | Returns first |
| `Single` | Throws | Throws |
| `SingleOrDefault` | Default value | Throws |

Use `Single` when uniqueness is a required invariant, ideally backed by a unique database constraint. Add ordering when the meaning of “first” must be deterministic.


---

#### 8. Production architecture and security

##### 8.1 Repository and Unit of Work choices

`DbContext` already provides unit-of-work behavior and `DbSet` offers repository-like operations. A custom repository can still be valuable when it expresses meaningful domain operations or hides persistence details. Avoid a generic repository that merely duplicates every `DbSet` method and blocks useful EF features without adding value.

##### 8.2 CQRS and read/write separation

A practical approach:
- Use EF Core for aggregate-oriented writes and transactional business rules.
- Use projections for simple read endpoints.
- Consider Dapper, views, or specialized read models for complex/high-volume reporting when profiling justifies it.
- Keep read/write separation driven by requirements, not by fashion.

##### 8.3 Database constraints are the final integrity guard

Validate business rules in the application for useful errors, but enforce critical invariants in the database where possible:
- Unique constraints/indexes.
- Foreign keys.
- Check constraints.
- Required columns and appropriate data types.
- Concurrency tokens.

Application validation alone cannot prevent races between concurrent requests.

##### 8.4 Interceptors versus logging and domain events

Interceptors can observe or influence EF operations such as commands, connections, transactions, and `SaveChanges`. Use them for cross-cutting persistence concerns such as diagnostics or carefully designed auditing.

Do not use a command interceptor as a substitute for domain logic. A `SaveChangesInterceptor` can help with audit fields, but it must handle added/modified/deleted entries, recursion risks, async paths, and sensitive data correctly. Domain events should represent business meaning, not merely SQL execution.

##### 8.5 Soft delete and auditing

A global query filter can hide soft-deleted rows, but the write path must consistently mark deletion and audit metadata. Decide whether audit records must be immutable, whether updates to audit rows are prohibited, and whether the database or application is the trusted source.

##### 8.6 Raw SQL and injection

Use parameterized APIs:

```csharp
var customers = await db.Customers
    .FromSqlInterpolated(
        $"SELECT * FROM dbo.Customers WHERE City = {city}")
    .ToListAsync(cancellationToken);
```

For `FromSqlRaw`, pass parameters separately; do not concatenate untrusted values into SQL. Parameters cannot generally stand in for SQL identifiers such as table names or sort directions, so dynamic identifiers require strict allow-listing.

##### 8.7 Async and cancellation

Use async database APIs in request paths to avoid blocking threads while I/O is pending. Pass cancellation tokens through the full call chain. Async does not make one `DbContext` thread-safe, and it does not automatically make a query faster at the database level.

---

#### 9. Testing, migrations, and operations

##### 9.1 Testing options

- **Unit tests:** test pure business logic without a database.
- **SQLite in-memory:** useful relational behavior, but not identical to SQL Server/PostgreSQL.
- **Production-provider integration tests:** best for validating provider-specific SQL, migrations, constraints, and transaction behavior.
- **EF Core InMemory provider:** useful for some tests but not a relational substitute; it does not reproduce all relational constraints and query semantics.

##### 9.2 Migrations in production

Recommended discipline:
1. Review generated migration code and SQL.
2. Test on a production-like database and realistic data volume.
3. Prefer backward-compatible expand/migrate/contract changes for zero-downtime deployments.
4. Avoid blindly applying migrations at application startup in multi-instance production deployments.
5. Plan backups, locks, long-running index operations, and rollback/forward-fix strategy.
6. Use a controlled deployment pipeline with permissions appropriate for schema changes.

**What is the model snapshot?**

`dotnet ef migrations add AddStudentEmail` compares the current EF model with the model snapshot from the previous migration and generates migration operations. `dotnet ef database update` applies pending migrations to the target database.

- The snapshot is a design-time model representation used to scaffold future migrations.
- It is not a live backup of the database and does not prove that a production database matches the model.
- Review generated migration code, data-loss risks, indexes, constraints, and deployment order.
- For production, consider reviewed SQL scripts or migration bundles and a controlled deployment process.


##### 9.3 Temporal tables, soft deletes, and audit logs are different

- **Temporal tables:** database-managed row history, when supported/configured.
- **Soft delete:** application convention that marks records as deleted.
- **Audit log:** records who did what and when, often with business context.

They solve different requirements and can coexist.

##### 9.4 Test the actual query plan

An in-memory test cannot establish that a SQL query is fast. Use realistic data volume, indexes, the target database engine, and actual execution plans. Include cancellation, transient failure, deadlock, concurrency conflict, and duplicate-message scenarios where relevant.

---

#### 10. Interview MCQs with answers

Try to answer each question before opening its answer.

##### Q1. When does a LINQ query against `DbSet<T>` usually execute?

A. When `Where` is called  
B. When `OrderBy` is called  
C. When the query is enumerated or a terminal operator executes  
D. When the `DbContext` is constructed  

**Answer: C.** Query operators usually compose an expression; materialization or enumeration triggers execution.

##### Q2. What is the main difference between `IQueryable<T>` and `IEnumerable<T>`?

A. `IQueryable` always executes on the database  
B. `IQueryable` exposes an expression/provider model; `IEnumerable` represents in-process enumeration  
C. `IEnumerable` cannot be filtered  
D. They are identical  

**Answer: B.** An `IQueryable` provider may translate expressions, but not every expression is translatable.

##### Q3. Does EF Core automatically configure a row-version column on every entity?

A. Yes  
B. Only for SQL Server  
C. No; concurrency tokens must be configured  
D. Only when using `SaveChangesAsync`  

**Answer: C.** Optimistic concurrency depends on a configured concurrency token or equivalent mechanism.

##### Q4. What does `AsNoTrackingWithIdentityResolution()` do?

A. Tracks every entity normally  
B. Avoids normal context tracking while resolving duplicate entity identities within the result  
C. Caches SQL results across requests  
D. Disables identity columns in the database  

**Answer: B.** It uses identity resolution without adding the returned entities to the context's normal tracked set.

##### Q5. What is a likely consequence of calling `Update()` on a disconnected entity graph?

A. Only changed properties are always updated  
B. The graph may be marked modified, potentially causing unintended updates  
C. EF automatically reloads every row first  
D. It bypasses database constraints  

**Answer: B.** Disconnected graphs require careful state management and validation.

##### Q6. What does a single relational `SaveChanges` normally provide when EF controls the transaction?

A. Distributed transaction across every service  
B. Atomicity for the changes in that save operation  
C. Automatic conflict resolution  
D. Automatic retry of every failure  

**Answer: B.** This does not make cross-service operations atomic or decide concurrency policy.

##### Q7. Why can a query with several collection `Include`s be slow?

A. `Include` always forces client evaluation  
B. Joins can multiply rows and duplicate parent data  
C. Tracking is always disabled  
D. SQL indexes cannot be used  

**Answer: B.** Evaluate projection, split queries, and actual execution plans.

##### Q8. Which is generally preferable for checking whether any matching rows exist?

A. `Count() > 0` in every case  
B. `Any()`  
C. `ToList().Count > 0`  
D. `First()` and catch every exception  

**Answer: B.** It expresses existence and can allow the database to stop after finding a match.

##### Q9. Does `ExecuteUpdate` update already tracked entities in memory?

A. Yes, always  
B. No; tracked instances can become stale  
C. Only when lazy loading is enabled  
D. Only after `AcceptAllChanges`  

**Answer: B.** Set-based operations bypass normal tracked-entity synchronization.

##### Q10. What is the purpose of a transactional outbox?

A. Cache every query  
B. Atomically store business changes and a message record in the same database transaction  
C. Replace foreign keys  
D. Make message delivery exactly once automatically  

**Answer: B.** Publishing is usually asynchronous and consumers still need idempotency/deduplication.

##### Q11. Is `DbContext` safe for concurrent operations?

A. Yes, if methods are async  
B. Yes, if the connection string is pooled  
C. No; use separate contexts for concurrent units of work  
D. Only reads can run concurrently on one context  

**Answer: C.** Async does not change the context's thread-safety contract.

##### Q12. What is the main limitation of EF Core's InMemory provider for integration testing?

A. It cannot store objects  
B. It does not reproduce relational database behavior faithfully  
C. It always needs SQL Server  
D. It cannot be used in tests  

**Answer: B.** Use a relational provider for relational behavior and the real provider for provider-specific validation.

##### Q13. What is a common benefit of keyset pagination?

A. Supports arbitrary page jumps more efficiently in every case  
B. Avoids scanning/skipping a large offset when using a suitable ordered key and index  
C. Removes the need for ordering  
D. Guarantees a consistent snapshot across requests  

**Answer: B.** It requires a stable order and is not a universal replacement for offset pagination.

##### Q14. What does a value converter do?

A. Automatically makes every object queryable as JSON  
B. Converts a model value to/from its provider representation  
C. Replaces entity tracking  
D. Creates database indexes automatically  

**Answer: B.** Mutable/custom values may also need a `ValueComparer`.

##### Q15. Why is `IgnoreQueryFilters()` a security concern in a multi-tenant application?

A. It disables SQL parameterization  
B. It can bypass tenant or soft-delete filters  
C. It disables transactions  
D. It prevents indexes from being used  

**Answer: B.** Authorization and tenant isolation must be enforced beyond a convention that code can bypass.

##### Q16. What does `ToQueryString()` provide?

A. A guaranteed byte-for-byte copy of the command executed in every situation  
B. A debugging representation of the query SQL  
C. The database's actual execution plan  
D. All parameter values including secrets  

**Answer: B.** Use command logs and database tooling to inspect actual execution and plans.

##### Q17. Which is a sound reason to use a custom repository over `DbContext` directly?

A. Every project must have a generic repository  
B. It encapsulates meaningful domain persistence operations or a useful boundary  
C. It guarantees better performance  
D. It removes the need for integration tests  

**Answer: B.** Abstraction should provide value rather than mechanically duplicating EF APIs.

##### Q18. What should a robust migration process include?

A. Apply migrations automatically on every production instance startup  
B. Review generated SQL, test on realistic data, and plan deployment/rollback  
C. Assume all schema changes are instant  
D. Skip migration testing if unit tests pass  

**Answer: B.** Schema changes can lock tables, take time, and require coordinated deployment.

##### Q19. What is the difference between context pooling and connection pooling?

A. They are the same feature  
B. Context pooling reuses EF context instances; connection pooling reuses database connections  
C. Context pooling is a database index feature  
D. Connection pooling tracks entity states  

**Answer: B.** They address different initialization and resource costs.

##### Q20. A `GroupBy` query is most likely to translate cleanly when it:

A. Returns arbitrary full grouping objects in every provider/version  
B. Groups and immediately projects keys with aggregates  
C. Calls `ToList()` before grouping in every case  
D. Uses only client-side methods  

**Answer: B.** Translation depends on provider, EF Core version, and expression shape.

---

#### 11. Scenario-based interview questions

##### Scenario A: Duplicate orders after retry

**Problem:** A client submits an order. The database commits, but the API loses the connection before returning success. The client retries and creates a duplicate.

**Strong answer should mention:**
- The commit outcome may be ambiguous.
- A database transaction alone does not make an HTTP retry idempotent.
- Use a client/request idempotency key with a unique constraint.
- Persist the key and result transactionally with the order.
- Make downstream event consumers idempotent; consider the outbox pattern.

##### Scenario B: Inventory overselling

**Problem:** Two requests read stock quantity 1 and both attempt to buy the final item.

**Strong answer should mention:**
- A read-then-write sequence has a race.
- Use a conditional atomic update such as `UPDATE ... WHERE Quantity >= @requested`.
- Check affected rows and enforce relevant constraints.
- Alternatively use a concurrency token with a deliberate retry/conflict policy.
- Keep the transaction scope short and test concurrent requests.

##### Scenario C: API becomes slow as data grows

**Problem:** An endpoint was fast with 500 rows but is slow with 5 million.

**Investigation plan:**
1. Capture actual SQL and parameter types.
2. Check actual execution plan, indexes, logical reads, and row estimates.
3. Check N+1 round trips and over-fetching.
4. Replace unnecessary entity graph materialization with projection.
5. Add bounded pagination; consider keyset pagination.
6. Measure database time separately from EF materialization and serialization.
7. Validate the improvement against production-like data.

##### Scenario D: Tenant data leaks between requests

**Problem:** The application uses a global query filter and context pooling.

**Strong answer should mention:**
- Verify tenant context state cannot leak across reused context instances.
- Do not trust tenant IDs from request payloads.
- Apply tenant authorization to writes as well as reads.
- Audit raw SQL and `IgnoreQueryFilters()` usage.
- Consider database row-level security or database-per-tenant where the threat model requires it.
- Add cross-tenant integration tests.

##### Scenario E: Updating a disconnected graph

**Problem:** A frontend sends an `Order` with lines, and the API calls `Update(order)`.

**Strong answer should mention:**
- `Update` can mark many graph entries modified.
- The request may omit properties or contain unauthorized child IDs.
- Load the aggregate, validate ownership and allowed changes, then apply them.
- Use explicit state setting only when the API contract truly represents a partial update.
- Enforce foreign keys and concurrency rules in the database.

##### Scenario F: A read endpoint uses many `Include`s

**Problem:** The query returns duplicate parent data and consumes excessive memory.

**Strong answer should mention:**
- Inspect generated SQL and row multiplication.
- Project into a DTO if the API needs a read model rather than a mutable entity graph.
- Compare split and single queries.
- Add appropriate indexes and pagination.
- Consider a dedicated read model if the query is fundamentally reporting-oriented.

#### Trick questions

- **Does `Include()` always mean one SQL query?** No. Split-query configuration can execute multiple queries.
- **Does `AsNoTracking()` update entities automatically?** No; changes to those objects are not persisted merely by calling `SaveChanges()`.
- **Does `ExecuteUpdate()` invoke `SaveChanges()`?** No.
- **Is a `DbContext` thread-safe?** No. Do not run parallel operations on one context; await each operation or use separate contexts.
- **Should a context live for the whole application?** Usually not. In web apps, a scoped context per unit of work/request is common; background work should use appropriately bounded lifetimes.
- **Does `GroupBy()` always translate to SQL `GROUP BY`?** No. Translation depends on query shape and provider; some grouping shapes can be assembled after results are returned.
- **Does a navigation property automatically load its related rows?** No. It needs eager, explicit, lazy loading, or relationship fix-up from already tracked entities.
- **Does a global query filter guarantee tenant security?** No. It is one layer and can be bypassed by deliberate APIs or alternate data paths.


---

#### 12. Final revision checklist

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

#### Suggested official references

- EF Core documentation: https://learn.microsoft.com/ef/core/
- Efficient querying: https://learn.microsoft.com/ef/core/performance/efficient-querying
- Change tracking: https://learn.microsoft.com/ef/core/change-tracking/
- Transactions: https://learn.microsoft.com/ef/core/saving/transactions
- Concurrency handling: https://learn.microsoft.com/ef/core/saving/concurrency
- Interceptors: https://learn.microsoft.com/ef/core/logging-events-diagnostics/interceptors
- Testing: https://learn.microsoft.com/ef/core/testing/
- LINQ documentation: https://learn.microsoft.com/dotnet/csharp/linq/

---

<a id="ef-core-viva-preparation-md"></a>

## EF Core Viva — More Quick Questions

Short answers to the EF Core questions **not** already explained in the main guide above (change tracking, loading, concurrency, transactions, `AsNoTracking`, split queries, and bulk updates are covered there).

#### 1. Name two ways to create a pooled context.

``` csharp
services.AddDbContextPool<AppDbContext>(o => o.UseSqlServer(cs));
services.AddPooledDbContextFactory<AppDbContext>(o => o.UseSqlServer(cs));
```

The first suits scoped injection; the factory suits explicit context
creation, such as background jobs.

#### 2. When should you use `IDbContextFactory`?

When the context lifetime does not align with a DI scope: background
services, desktop apps, multiple independent units of work in one
request, or long-lived services. It does not make a context thread-safe;
use separate contexts for concurrent operations.

Do not inject a scoped context directly into a singleton. Use
`IDbContextFactory<TContext>` or `IServiceScopeFactory` to create a
scope per unit of work. Dispose factory-created contexts.

``` csharp
services.AddDbContextFactory<AppDbContext>(o => o.UseSqlServer(cs));

public sealed class Worker(IDbContextFactory<AppDbContext> factory)
{
    public async Task RunAsync(CancellationToken ct)
    {
        await using var db = await factory.CreateDbContextAsync(ct);
        // Work
    }
}
```


#### 3. How do you correct a generated migration?

Review and edit `Up()`/`Down()` before applying it---for example,
replace an unintended drop/add with `RenameColumn`. If unapplied,
`dotnet ef migrations remove` lets you regenerate it. If already
deployed to a shared/production database, normally create a corrective
migration instead of rewriting history. Inspect generated SQL.

#### 4. How should production migrations be executed?

Generate and review a SQL script (`dotnet ef migrations script`), test
it against a production-like database, verify backup/recovery, then
apply it through a controlled deployment pipeline or DBA. `--idempotent`
helps when databases may be at different migration levels. Avoid
uncontrolled application-startup migrations because of permissions and
concurrent-deployment risks.

#### 5. How do you seed initial data?

Use `HasData()` for fixed model-managed/reference data, or custom
initialization code for dynamic data and runtime-dependent logic.

``` csharp
modelBuilder.Entity<OrderStatus>().HasData(
    new OrderStatus { Id = 1, Name = "Pending" },
    new OrderStatus { Id = 2, Name = "Complete" });
```

`HasData` requires explicit keys and changes are managed by migrations.

#### 6. What changed in EF Core 9 seeding?

EF Core 9 introduced `UseSeeding()` and `UseAsyncSeeding()` for general
initialization logic, integrated with `EnsureCreated`/`Migrate`
workflows and migration locking. Keep the logic idempotent. `HasData()`
remains available for fixed, migration-managed data.

``` csharp
optionsBuilder.UseSqlServer(cs).UseSeeding((context, _) =>
{
    if (!context.Set<AppSetting>().Any(x => x.Key == "Theme"))
    {
        context.Add(new AppSetting { Key = "Theme", Value = "Default" });
        context.SaveChanges();
    }
});
```

#### 7. How do you create a database using EF?

For a migrations-based app, run `dotnet ef migrations add InitialCreate`
then `dotnet ef database update`, or apply a reviewed migration script.
`EnsureCreated()` creates a schema without migrations and should not be
mixed with a migrations-managed schema.

#### 8. Can EF Core create migrations for multiple databases?

Yes. Use multiple `DbContext` types and specify `--context` for each.
The same context/migrations can also be applied to multiple database
instances of the same provider. Different providers may require separate
provider-specific migrations.

#### 9. What are EF Core interceptors?

They can observe, modify, or suppress operations such as database
commands, connections, and `SaveChanges`. Use them for cross-cutting
behavior such as command diagnostics or auditing; ordinary logging is
simpler when you only need logs.

#### 10. How do you implement soft deletion?

Add `IsDeleted`/`DeletedAt`, apply a global query filter, and turn
delete operations into updates.

``` csharp
modelBuilder.Entity<Customer>().HasQueryFilter(c => !c.IsDeleted);
```

`IgnoreQueryFilters()` can bypass filters. Bulk operations bypass change
tracking and `SaveChanges` hooks, so enforce soft-delete policy on those
paths too.

#### 11. How do you insert multiple entities?

``` csharp
db.Products.AddRange(products);
await db.SaveChangesAsync();
```

EF tracks the new entities and may batch compatible commands. A single
`SaveChanges` is transactional by default for relational providers.

#### 12. How do you delete an existing entity?

``` csharp
var product = await db.Products.FindAsync(id);
if (product is null) return;
db.Products.Remove(product);
await db.SaveChangesAsync();
```

Foreign-key constraints and configured delete behavior govern related
rows.

#### 13. How do you delete without selecting first?

Attach a key-only stub and mark it deleted, or use a set-based delete in
EF Core 7+.

``` csharp
db.Entry(new Product { Id = id }).State = EntityState.Deleted;
await db.SaveChangesAsync();

// Alternative:
await db.Products.Where(p => p.Id == id).ExecuteDeleteAsync();
```

Bulk deletion bypasses tracking and `SaveChanges` hooks.

#### 14. Explain transaction isolation levels.

-   **Read Uncommitted:** dirty reads may occur.
-   **Read Committed:** prevents dirty reads; repeated reads may see
    changes.
-   **Repeatable Read:** rows read remain stable against updates;
    phantom behavior varies by provider.
-   **Serializable:** strongest standard level; may increase
    blocking/reduce concurrency.
-   **Snapshot:** where supported/configured, reads a consistent
    version; write conflicts can still occur. Behavior depends on the
    database and configuration, e.g. SQL Server's read-committed
    snapshot setting.

#### 15. How do you execute custom SQL?

Use parameterized `ExecuteSql`/`ExecuteSqlInterpolated` for commands,
and `FromSql`/`FromSqlInterpolated` for entity-shaped result rows.

``` csharp
var count = await db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Products SET IsActive = {false} WHERE CategoryId = {categoryId}");
```

Never concatenate untrusted input. SQL parameters generally cannot
represent identifiers such as table or column names.

#### 16. How do you query a database view?

Map a read model, commonly as a keyless entity:

``` csharp
modelBuilder.Entity<SalesSummary>()
    .HasNoKey()
    .ToView("vw_SalesSummary");
```

Then query `db.Set<SalesSummary>()`. `ToView()` maps the query source;
it does not necessarily create the view, which may need a migration or
SQL script.

#### 17. How do you call a stored procedure?

For a command, use parameterized SQL execution:

``` csharp
await db.Database.ExecuteSqlInterpolatedAsync(
    $"EXEC dbo.RebuildSummary @CategoryId={categoryId}");
```

For rows, use `FromSql` on a compatible entity/keyless type where the
provider/version supports it. For multiple result sets or complex
outputs, ADO.NET may be more appropriate.

#### 18. How do you log generated SQL?

``` csharp
optionsBuilder
    .UseSqlServer(cs)
    .LogTo(Console.WriteLine, LogLevel.Information);
```

In ASP.NET Core, the standard logging pipeline can route EF logs to your
chosen sink. `EnableSensitiveDataLogging()` may expose personal or
secret parameter values; use only in controlled development
environments.

#### 19. What are the ways to configure entity mapping?

1.  **Conventions** --- EF infers mappings from names/types.
2.  **Data annotations** --- attributes such as `[Key]`, `[Required]`.
3.  **Fluent API** --- configuration in `OnModelCreating`.
4.  **`IEntityTypeConfiguration<TEntity>`** --- separate mapping
    classes. Fluent API generally takes precedence over annotations,
    which override many conventions.

#### 20. How are nullable reference types handled?

With C# nullable reference types enabled, `string Name` is required by
convention and `string? Description` is optional. This affects model
nullability and migrations. Enabling NRT in an existing project can
cause many columns to change.

#### 21. How do you configure one-to-one?

``` csharp
modelBuilder.Entity<User>()
    .HasOne(u => u.Profile)
    .WithOne(p => p.User)
    .HasForeignKey<UserProfile>(p => p.UserId);
```

`UserProfile` is the dependent. The foreign key should be unique for a
true one-to-one relationship.

#### 22. How do you configure one-to-many?

``` csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .HasForeignKey(p => p.BlogId);
```

The dependent (`Post`) holds the foreign key. EF can often infer the
relationship by convention.

#### 23. How do you configure many-to-many?

EF Core 5+ supports skip navigations:

``` csharp
modelBuilder.Entity<Post>()
    .HasMany(p => p.Tags)
    .WithMany(t => t.Posts);
```

EF creates a join table by convention. Use an explicit join entity if
the relationship has extra fields such as `CreatedAt`.

#### 24. What are owned types for?

They model a value-like object that belongs to an owner and has
ownership semantics, e.g. an `Address` owned by `Customer`. Depending on
mapping, properties can be stored in the owner's table or a separate
table.

#### 25. What are complex types for?

Available from EF Core 8, complex types represent structured values
without their own entity identity/key, such as `Address` or `Money`. Use
them when the value is part of the containing entity rather than an
independently identified entity. Exact support varies by
version/provider.

#### 26. How do you store an enum as a string?

``` csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();
```

The database stores enum names. Renaming members can require data
migration; consider explicit stable mappings if stored values are part
of a long-lived contract.

#### 27. How do you implement audit trails?

Inspect `ChangeTracker` entries in `SaveChanges`/`SaveChangesAsync`
overrides or a `SaveChangesInterceptor`; record entity/key, operation,
changed values, user, and timestamp. Bulk
`ExecuteUpdate`/`ExecuteDelete` bypass these hooks. Database triggers or
temporal tables may be preferable if all writers must be audited.

#### 28. How do you configure an auto-increment primary key?

Numeric keys are commonly generated on insert by convention. You can
specify:

``` csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Id).ValueGeneratedOnAdd();
```

SQL Server normally maps an integer key to `IDENTITY`; exact behavior
depends on provider/type. `ValueGeneratedOnAdd()` describes
value-generation behavior, not one universal database mechanism.

#### 29. Can compiled queries be asynchronous?

Yes, with `EF.CompileAsyncQuery()`:

``` csharp
private static readonly Func<AppDbContext, int, IAsyncEnumerable<Product>>
    ProductsByCategory = EF.CompileAsyncQuery(
        (AppDbContext db, int categoryId) =>
            db.Products.Where(p => p.CategoryId == categoryId));

await foreach (var product in ProductsByCategory(db, categoryId))
{
    // Process
}
```

Reuse the delegate with different contexts/parameters; keep the query
shape stable.

#### 30. How do you search case-insensitively?

It depends on database collation/provider. On SQL Server, a
case-insensitive collation is often best. Where supported:

``` csharp
var users = await db.Users.Where(u =>
    EF.Functions.Collate(u.Name, "SQL_Latin1_General_CP1_CI_AS") == searchName)
    .ToListAsync();
```

Collation names are database-specific. `ToLower()` on a column may hurt
index use; `Like()` does not itself guarantee case-insensitive matching.

#### 31. What are JSON columns?

They store structured JSON in a database column. EF Core's mapping/query
support depends on version and provider. JSON is useful for
flexible/nested data but has trade-offs in validation, indexing,
queryability, and schema evolution. Frequently queried fields may be
better as relational columns.

#### 32. How do you implement cascade deletion?

``` csharp
modelBuilder.Entity<Order>()
    .HasMany(o => o.Items)
    .WithOne(i => i.Order)
    .HasForeignKey(i => i.OrderId)
    .OnDelete(DeleteBehavior.Cascade);
```

Deleting the principal can delete dependents. Actual behavior depends on
relationship requiredness, provider, and database constraints. Consider
`Restrict`/`NoAction` or `SetNull` where appropriate. Cascade delete is
physical deletion, not soft deletion.

------------------------------------------------------------------------

#### High-value distinctions to revise

  -----------------------------------------------------------------------
  Concepts                            Difference
  ----------------------------------- -----------------------------------
  Context pooling vs connection       Reuses EF context objects vs
  pooling                             database connections

  `HasData` vs `UseSeeding`           Fixed migration-managed data vs
                                      initialization logic

  `EnsureCreated` vs migrations       Direct schema creation vs versioned
                                      schema changes

  `Update(entity)` vs marking         May mark an entire graph modified
  properties                          vs limiting writes

  `SaveChanges` vs                    Tracked changes/hooks vs direct
  `ExecuteUpdate/Delete`              set-based SQL

  Owned vs complex types              Ownership semantics vs value
                                      structure without entity identity

  Eager/explicit/lazy loading         Up front / requested manually / on
                                      navigation access

  Concurrency token vs isolation      Detect conflicting writes vs
  level                               control transaction visibility

  Query filter vs authorization       Default filtering vs complete
                                      security enforcement
  -----------------------------------------------------------------------

#### Viva answer formula

1.  Define the concept in one or two sentences.
2.  Say when you would use it.
3.  Give a short code example if relevant.
4.  Mention one trade-off, limitation, or production concern.

---

<a id="ef-core-performance-interview-and-benchmark-lab-md"></a>

## EF Core Performance: Hands-On Benchmark Lab

A consolidated, non-repetitive guide for Senior .NET Developers and
Solution Architects.

#### 1. Hands-on EF Core benchmark project

##### 1.1 Objective

Build a repeatable benchmark comparing: 1. Tracking vs. no-tracking
materialization. 2. Entity graph loading vs. DTO projection. 3. Offset
vs. keyset pagination. 4. Ordinary vs. compiled query execution.

Use SQL Server or Azure SQL for database-level comparisons. Results
depend on data, indexes, network latency, hardware, EF Core/provider
versions, and database configuration; do not assume a particular winner
without measuring.

##### 1.2 Create the project

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

##### 1.3 Minimal model

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

##### 1.4 DbContext and index configuration

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

##### 1.5 Create a benchmark class

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

##### 1.6 Add compiled-query comparison

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

##### 1.7 Add the benchmark entry point

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

##### 1.8 Prepare representative data

Use a dedicated lab database. Seed enough rows to make query plans and
pagination differences observable---for example, hundreds of thousands
of orders with realistic distributions of tenants, statuses, dates,
items, and payments.

Ensure: - Data distribution is not uniform unless production is
uniform. - Indexes and statistics reflect the intended scenario. - The
database is warmed up before steady-state runs. - Each compared query
returns equivalent logical results. - Database metrics are captured
alongside application benchmarks.

##### 1.9 Capture SQL and execution plans

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

##### 1.10 Results worksheet

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

#### 2. Production review checklist

##### Query design

-   [ ] Project only required columns for read endpoints.
-   [ ] Use no-tracking queries where entity updates are not needed.
-   [ ] Avoid accidental N+1 queries.
-   [ ] Evaluate `Include` and split queries for large graphs.
-   [ ] Use deterministic pagination and keyset pagination for
    sequential browsing where appropriate.

##### Database design

-   [ ] Design indexes from real predicates and ordering.
-   [ ] Validate indexes with execution plans and workload metrics.
-   [ ] Review foreign keys, unique constraints, and data types.
-   [ ] Check parameter-sensitive plans and statistics where performance
    varies by parameter.

##### Reliability

-   [ ] Use short-lived `DbContext` instances.
-   [ ] Handle concurrency conflicts deliberately.
-   [ ] Keep related writes atomic.
-   [ ] Make retries safe and avoid retrying non-idempotent work
    blindly.
-   [ ] Use migrations or reviewed deployment scripts with a production
    rollout plan.

##### Observability

-   [ ] Capture slow-query telemetry without exposing sensitive values.
-   [ ] Monitor Query Store, CPU, IO, blocking, and timeouts.
-   [ ] Benchmark representative workloads before and after changes.
-   [ ] Test against the production database provider for critical query
    behavior.

#### 3. Official Microsoft documentation

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

#### Key takeaway

For senior engineers and architects, EF Core performance work is a
measurement loop:

**Observe → inspect generated SQL → inspect the execution plan → change
query or index design → benchmark again.**

Avoid universal rules such as "every query needs an index," "all reads
must use no-tracking," or "EF Core is always slower than Dapper." The
correct design depends on the query, data distribution, provider,
workload, and consistency requirements.

---

<a id="sql-server-advanced-grouping-and-window-functions-md"></a>

## SQL Server: Grouping, PIVOT/UNPIVOT, and Window Functions

A compact, runnable reference with nonduplicated examples. All queries use T-SQL.

#### 1. GROUPING SETS, ROLLUP, and CUBE

These extend `GROUP BY` to return multiple aggregation levels in one query.

```sql
DROP TABLE IF EXISTS dbo.DepartmentSales;
CREATE TABLE dbo.DepartmentSales (
    Department varchar(20),
    SalesYear int,
    Amount int
);
INSERT INTO dbo.DepartmentSales VALUES
('HR', 2023, 100), ('HR', 2024, 200),
('IT', 2023, 300), ('IT', 2024, 400);
```

| Feature | Groupings generated for `(Department, SalesYear)` | Purpose |
|---|---|---|
| `GROUPING SETS` | Only those you specify | Custom summaries |
| `ROLLUP` | `(Department, SalesYear)`, `(Department)`, `()` | Hierarchical subtotals and grand total |
| `CUBE` | `(Department, SalesYear)`, `(Department)`, `(SalesYear)`, `()` | Every combination |

**Sample data** (`dbo.DepartmentSales`):

| Department | SalesYear | Amount |
|---|---|---|
| HR | 2023 | 100 |
| HR | 2024 | 200 |
| IT | 2023 | 300 |
| IT | 2024 | 400 |

In the results below, `NULL` means "all values" for that column, so that row is a subtotal or the grand total. Without `ORDER BY` the row order isn't guaranteed; the rows are shown grouped for readability.

##### `GROUPING SETS`: only the groupings you ask for

```sql
SELECT Department, SalesYear, SUM(Amount) AS Total
FROM dbo.DepartmentSales
GROUP BY GROUPING SETS ((Department, SalesYear), (Department), ());
```

| Department | SalesYear | Total | Comes from |
|---|---|---|---|
| HR | 2023 | 100 | `(Department, SalesYear)` |
| HR | 2024 | 200 | `(Department, SalesYear)` |
| IT | 2023 | 300 | `(Department, SalesYear)` |
| IT | 2024 | 400 | `(Department, SalesYear)` |
| HR | NULL | 300 | `(Department)`: HR subtotal |
| IT | NULL | 700 | `(Department)`: IT subtotal |
| NULL | NULL | 1000 | `()`: grand total |

You pick the groupings. For example, `GROUPING SETS ((Department), (SalesYear))` would return **only** the department totals and the year totals, with no detail rows and no grand total.

##### `ROLLUP`: hierarchical subtotals (left to right)

```sql
SELECT Department, SalesYear, SUM(Amount) AS Total
FROM dbo.DepartmentSales
GROUP BY ROLLUP (Department, SalesYear);
```

| Department | SalesYear | Total | Level |
|---|---|---|---|
| HR | 2023 | 100 | Detail |
| HR | 2024 | 200 | Detail |
| HR | NULL | 300 | HR subtotal |
| IT | 2023 | 300 | Detail |
| IT | 2024 | 400 | Detail |
| IT | NULL | 700 | IT subtotal |
| NULL | NULL | 1000 | Grand total |

These are the same rows as the `GROUPING SETS` query above, because `ROLLUP (A, B)` is shorthand for `GROUPING SETS ((A, B), (A), ())`. Column order matters: `ROLLUP (SalesYear, Department)` would give **year** subtotals instead.

##### `CUBE`: every combination

```sql
SELECT Department, SalesYear, SUM(Amount) AS Total
FROM dbo.DepartmentSales
GROUP BY CUBE (Department, SalesYear);
```

| Department | SalesYear | Total | Level |
|---|---|---|---|
| HR | 2023 | 100 | Detail |
| HR | 2024 | 200 | Detail |
| IT | 2023 | 300 | Detail |
| IT | 2024 | 400 | Detail |
| HR | NULL | 300 | Department subtotal |
| IT | NULL | 700 | Department subtotal |
| NULL | 2023 | 400 | **Year subtotal** (only `CUBE` adds these) |
| NULL | 2024 | 600 | **Year subtotal** |
| NULL | NULL | 1000 | Grand total |

`CUBE (A, B)` = `GROUPING SETS ((A, B), (A), (B), ())`. With *n* columns it produces 2ⁿ groupings, so use it carefully on many columns.

**Tip: label the subtotal rows.** A real `NULL` in the data looks the same as a subtotal `NULL`. Use `GROUPING()` to tell them apart:

```sql
SELECT
    CASE WHEN GROUPING(Department) = 1 THEN 'All departments' ELSE Department END AS Department,
    CASE WHEN GROUPING(SalesYear)  = 1 THEN 'All years' ELSE CAST(SalesYear AS varchar(10)) END AS SalesYear,
    SUM(Amount) AS Total
FROM dbo.DepartmentSales
GROUP BY ROLLUP (Department, SalesYear);
```

`GROUPING(col)` returns `1` when that column was rolled up for the row, and `0` otherwise.

`NULL` in subtotal rows represents an aggregated-away dimension, but source data can also contain real `NULL`s. Use `GROUPING(Department)` or `GROUPING_ID(Department, SalesYear)` to distinguish them. `ORDER BY` is needed for a guaranteed display order.

#### 2. PIVOT and UNPIVOT

- **PIVOT**: rows → columns, with an aggregation.
- **UNPIVOT**: columns → rows. SQL Server `UNPIVOT` excludes `NULL` values.

##### PIVOT: separate source table

```sql
DROP TABLE IF EXISTS dbo.SalesLong;
CREATE TABLE dbo.SalesLong (
    SalesYear int,
    Product varchar(20),
    Amount int
);
INSERT INTO dbo.SalesLong VALUES
(2023, 'Apple', 100), (2023, 'Banana', 150), (2023, 'Mango', 120),
(2024, 'Apple', 110), (2024, 'Banana', 160), (2024, 'Mango', 130);

SELECT SalesYear, [Apple], [Banana], [Mango]
FROM (SELECT SalesYear, Product, Amount FROM dbo.SalesLong) AS src
PIVOT (
    SUM(Amount) FOR Product IN ([Apple], [Banana], [Mango])
) AS p
ORDER BY SalesYear;
```

| SalesYear | Apple | Banana | Mango |
|---|---:|---:|---:|
| 2023 | 100 | 150 | 120 |
| 2024 | 110 | 160 | 130 |

##### UNPIVOT: independent wide table

```sql
DROP TABLE IF EXISTS dbo.SalesWide;
CREATE TABLE dbo.SalesWide (
    SalesYear int,
    Apple int,
    Banana int,
    Mango int
);
INSERT INTO dbo.SalesWide VALUES
(2023, 100, 150, 120),
(2024, 110, 160, 130);

SELECT SalesYear, Product, Amount
FROM dbo.SalesWide
UNPIVOT (
    Amount FOR Product IN ([Apple], [Banana], [Mango])
) AS u
ORDER BY SalesYear, Product;
```

This returns one row per `(SalesYear, Product)` with its amount. `PIVOT` aggregates duplicate source rows, so pivoting and unpivoting are not always lossless inverses.

#### 3. Window-function syntax and supporting clauses

Window functions calculate values across related rows **without collapsing the original rows**.

```sql
function_expression OVER (
    [PARTITION BY column_list]
    [ORDER BY sort_expression]
    [ROWS BETWEEN frame_start AND frame_end]
)
```

This is a conceptual template: the allowed clauses depend on the specific function.

| Clause or keyword | Meaning |
|---|---|
| `OVER (...)` | Defines the window |
| `PARTITION BY` | Restarts calculations for each group; optional |
| `ORDER BY` inside `OVER` | Defines processing order; required by ranking, offset and most analytic functions |
| `ROWS` | Defines a frame by physical row positions |
| `RANGE` | Groups rows with equal ordering values (peers); SQL Server supports only limited RANGE boundaries |
| `UNBOUNDED PRECEDING` | First row of partition |
| `n PRECEDING` | n rows before current row (`ROWS` only for numeric offsets) |
| `CURRENT ROW` | Current row (`RANGE` also includes peers at its boundary) |
| `n FOLLOWING` | n rows after current row (`ROWS` only for numeric offsets) |
| `UNBOUNDED FOLLOWING` | Last row of partition |

**Important:** `LAG`, `LEAD`, and ranking functions do **not** accept a `ROWS BETWEEN` frame in SQL Server. For ordered aggregate windows, explicitly specify `ROWS` when row-by-row behavior is intended; otherwise the default can be `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

#### 4. Window-function catalog (SQL Server)

| Category | Functions | What they do |
|---|---|---|
| Ranking | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` | Number, rank, or bucket ordered rows |
| Offset | `LAG`, `LEAD` | Read previous/next row by offset |
| Value | `FIRST_VALUE`, `LAST_VALUE` | Read first/last value in the applicable window frame |
| Distribution | `PERCENT_RANK`, `CUME_DIST` | Relative rank and cumulative distribution |
| Percentile | `PERCENTILE_CONT`, `PERCENTILE_DISC` | Interpolated/discrete percentile; use `WITHIN GROUP (...) OVER (...)` |
| Aggregates usable as windows | `SUM`, `AVG`, `MIN`, `MAX`, `COUNT`, `COUNT_BIG`, `STDEV`, `STDEVP`, `VAR`, `VARP` | Aggregation without reducing row count |
| Additional aggregate | `STRING_AGG` | **Not** supported with `OVER` in SQL Server |

**Version notes:** `PERCENTILE_CONT` and `PERCENTILE_DISC` are available well before SQL Server 2022. `NTH_VALUE` is **not** a SQL Server T-SQL function (it exists in other databases). SQL Server does **not** support `EXCLUDE CURRENT ROW` window-frame syntax. SQL Server 2022+ (compatibility level 160+) supports a reusable `WINDOW` clause; availability depends on version/compatibility.

#### 5. Runnable window examples: frames, LAG, and LEAD

```sql
DROP TABLE IF EXISTS dbo.MonthlySales;
CREATE TABLE dbo.MonthlySales (
    SalesPerson varchar(20),
    SalesMonth int,
    Amount int
);
INSERT INTO dbo.MonthlySales VALUES
('Alice', 1, 100), ('Alice', 2, 150), ('Alice', 3, 200),
('Bob', 1, 120), ('Bob', 2, 180), ('Bob', 3, 160);
```

##### Three frame types in one query

```sql
SELECT SalesPerson, SalesMonth, Amount,
    SUM(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS RunningTotal,
    SUM(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
    ) AS ForwardTotal,
    SUM(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS FullPartitionTotal
FROM dbo.MonthlySales
ORDER BY SalesPerson, SalesMonth;
```

| SalesPerson | Month | Amount | RunningTotal | ForwardTotal | FullPartitionTotal |
|---|---:|---:|---:|---:|---:|
| Alice | 1 | 100 | 100 | 450 | 450 |
| Alice | 2 | 150 | 250 | 350 | 450 |
| Alice | 3 | 200 | 450 | 200 | 450 |
| Bob | 1 | 120 | 120 | 460 | 460 |
| Bob | 2 | 180 | 300 | 340 | 460 |
| Bob | 3 | 160 | 460 | 160 | 460 |

##### Previous and next rows with LAG / LEAD

```sql
SELECT SalesPerson, SalesMonth, Amount,
    LAG(Amount, 1, 0) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
    ) AS PreviousAmount,
    LEAD(Amount, 1, 0) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
    ) AS NextAmount
FROM dbo.MonthlySales
ORDER BY SalesPerson, SalesMonth;
```

| SalesPerson | Month | Amount | PreviousAmount | NextAmount |
|---|---:|---:|---:|---:|
| Alice | 1 | 100 | 0 | 150 |
| Alice | 2 | 150 | 100 | 200 |
| Alice | 3 | 200 | 150 | 0 |
| Bob | 1 | 120 | 0 | 180 |
| Bob | 2 | 180 | 120 | 160 |
| Bob | 3 | 160 | 180 | 0 |

`1` is the offset; `0` is the fallback when there is no previous/next row. These functions use row position within the partition, not a window frame.

##### Moving average: one row before through one row after

```sql
SELECT SalesPerson, SalesMonth, Amount,
    AVG(CAST(Amount AS decimal(10,2))) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS ThreeRowMovingAverage
FROM dbo.MonthlySales
ORDER BY SalesPerson, SalesMonth;
```

At partition boundaries, the frame contains fewer than three rows.

##### LAST_VALUE gotcha

```sql
SELECT SalesPerson, SalesMonth, Amount,
    LAST_VALUE(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS LastMonthAmount
FROM dbo.MonthlySales;
```

The explicit full frame is important: without it, `LAST_VALUE` commonly returns the current row's value rather than the partition's final value.

#### 6. Interview reminders

- `GROUP BY` reduces rows; `PARTITION BY` inside `OVER` keeps rows.
- `ROLLUP` follows the listed hierarchy; `CUBE` generates all combinations; `GROUPING SETS` lets you choose.
- `PIVOT` converts values to columns; `UNPIVOT` converts columns to values.
- `RANK` leaves gaps after ties; `DENSE_RANK` does not; `ROW_NUMBER` assigns unique sequential numbers.
- Add a unique tie-breaker to window `ORDER BY` when deterministic row positioning matters.
- Window-frame clauses work with applicable aggregate/value functions, **not** with `LAG`/`LEAD`.

---

## Additional Questpond question sets

### ORM

##### Theory questions

###### 1. What is an ORM? (PDF p. 10)
**Answer:** An Object-Relational Mapper maps between application objects and relational database tables, and often translates object queries to SQL. It can reduce routine data-access code, but generated queries and tracking behavior still need review and measurement.

###### 2. Have you used an ORM in your project? Which one? (PDF p. 10)
**Answer:** Give your actual experience, the ORM and version, and a specific use case. For example: “I used EF Core for transactional CRUD and migrations; I used targeted SQL for reporting queries after measuring that it was more suitable.”

###### 3. What is Dapper? (PDF p. 10)
**Answer:** Dapper is a lightweight .NET micro-ORM that maps rows from SQL queries to objects. Developers write and control SQL directly; it provides less abstraction/tracking than EF Core and can be useful when precise SQL control is needed.

###### 4. What is EF Core? (PDF p. 10)
**Answer:** Entity Framework Core is Microsoft's cross-platform ORM for .NET. It maps entity models to database schemas, translates LINQ queries, supports change tracking, migrations, and multiple database providers.

###### 5. EF Core vs. EF 6 (Framework) (PDF p. 10)
**Answer:** EF Core is the modern, actively developed, cross-platform ORM with a different architecture and feature set. EF 6 (often called EF6) is the older ORM, historically tied to .NET Framework though later versions support modern .NET; they are not drop-in equivalents, so check provider and feature compatibility before migrating.

###### 6. What are navigation properties in EF Core? (PDF p. 10)
**Answer:** Navigation properties represent relationships between entities, such as `Order.Customer` or `Customer.Orders`. EF Core uses relationship metadata and foreign keys to query and track related entities; related data is loaded explicitly, eagerly, or (when configured) lazily.

###### 7. How did you implement EF Core with a relational database? (PDF p. 10)
**Answer:** Configure a `DbContext` with a provider and connection configuration, define entities/relationships and constraints, register the context with dependency injection, and use migrations or an approved schema process. Keep contexts short-lived (usually scoped per request) and parameterize external input.

###### 8. Which approach did you follow in your project? (PDF p. 10)
**Answer:** State the actual approach—code-first, database-first, or an existing-schema mapping—and explain why it fit ownership, migration control, and team workflow. For a shared/legacy schema, coordinate migrations with DBAs; for an application-owned schema, migrations can provide an auditable evolution path.

###### 9. Code first vs. database first (PDF p. 10)
**Answer:** Code-first treats the model and migrations as the main schema definition; database-first starts from an existing schema and generates/matches model classes. Choose based on schema ownership and governance; both require reviewed schema changes and testing.

###### 10. Commands used for code first (PDF p. 10)
**Answer:** Common EF Core CLI commands include `dotnet ef migrations add MigrationName`, `dotnet ef database update`, and `dotnet ef migrations list`; `dotnet ef migrations script` generates a deployable SQL script. Install/use the EF CLI tool and provider appropriate to the project.

###### 11. What are Fluent APIs? (PDF p. 11)
**Answer:** Fluent APIs configure model metadata through chained method calls. In EF Core, `OnModelCreating` and `IEntityTypeConfiguration<T>` can define keys, relationships, indexes, constraints, and conversions when attributes are insufficient or configuration should remain outside entity classes.

###### 12. How do you call a stored procedure from EF Core? (PDF p. 11)
**Answer:** Use parameterized APIs: `FromSql`/`FromSqlInterpolated` for a query returning mapped entities, or `Database.ExecuteSql`/`ExecuteSqlInterpolated` for commands, using the APIs supported by the EF Core version. Avoid concatenating untrusted values into SQL and account for provider-specific procedure syntax.

###### 13. What is lazy loading? (PDF p. 11)
**Answer:** Lazy loading fetches related data when a navigation property is accessed, usually through proxies or injected loaders. It can simplify access but may trigger hidden queries and N+1 performance problems; use deliberately and inspect generated SQL.

###### 14. Eager loading in EF Core (PDF p. 11)
**Answer:** Eager loading retrieves related data as part of the initial query, using `Include`/`ThenInclude` or projections. It reduces later round trips but can produce large joins or duplicate rows; project only the required data and consider split queries when appropriate.

###### 15. Tracking and no-tracking in EF Core (PDF p. 11)
**Answer:** Tracking queries keep entities in the change tracker so modifications can be persisted with `SaveChanges`. `AsNoTracking()` avoids tracking overhead for read-only results; use tracking when the loaded entities will be edited through that context.

###### 16. Can we handle transactions in EF Core? (PDF p. 11)
**Answer:** Yes. A single `SaveChanges` is transactional when supported by the provider. For multiple operations, use `Database.BeginTransactionAsync()` and commit/rollback; consider execution strategies/retries and avoid holding transactions open longer than necessary.

###### 17. What is an explicit transaction? (PDF p. 11)
**Answer:** An explicit transaction is one the application starts and controls across one or more database operations, committing only when all succeed and rolling back on failure. It is needed when several operations must be atomic beyond one `SaveChanges` call.

---

### SQL

##### Theory questions

###### 1. What is normalization? State its forms. (PDF p. 12)
**Answer:** Normalization organizes relational data to reduce redundancy and update anomalies. Common forms: 1NF (atomic values/no repeating groups), 2NF (1NF and no partial dependency on part of a composite key), 3NF (2NF and no inappropriate transitive dependency on a key), and BCNF (every determinant is a candidate key).

###### 2. What is Boyce–Codd normal form? (PDF p. 12)
**Answer:** BCNF requires every nontrivial functional dependency `X → Y` to have `X` as a superkey. It is stricter than 3NF and can remove certain anomalies left by 3NF.

###### 3. Indexes in SQL (PDF p. 12)
**Answer:** An index is an auxiliary structure that speeds up data lookup/order at the cost of storage and extra work on writes. Choose keys/includes and filtered indexes from observed query patterns, inspect actual execution plans, and avoid excessive or redundant indexes.

###### 4. Clustered vs. nonclustered index (PDF p. 12)
**Answer:** In SQL Server, a clustered index defines the row-storage ordering at the leaf level, so a table has at most one. A nonclustered index is a separate lookup structure with key and row locator; a table can have multiple, each adding storage and write cost.

###### 5. What is an execution plan? (PDF p. 12)
**Answer:** An execution plan describes the operators and strategy the database engine chose to execute a query, including scans/seeks, joins, and estimates/actuals. Use actual plans and runtime statistics to investigate bottlenecks, while treating estimates as estimates rather than proof.

###### 6. Primary key vs. unique constraint (PDF p. 12)
**Answer:** Both enforce uniqueness. A table has one primary key, which identifies its rows and cannot contain `NULL`; it may be composite. A table can have multiple unique constraints/indexes, whose treatment of `NULL` is DBMS-specific (SQL Server permits one `NULL` per ordinary unique key unless a filtered index is used).

###### 7. `UNION` vs. `UNION ALL` (PDF p. 12)
**Answer:** Both combine compatible result sets. `UNION` removes duplicate rows, usually requiring extra work; `UNION ALL` retains duplicates and is generally cheaper when de-duplication is unnecessary.

###### 8. `ISNULL` vs. `COALESCE` (PDF p. 12)
**Answer:** Both can replace null values. SQL Server `ISNULL` is a two-argument function whose result type is based on its first argument; `COALESCE` accepts multiple expressions and follows `CASE` type-precedence rules, and may evaluate expressions more than once. Choose with type/side-effect behavior in mind.

###### 9. `TRUNCATE` vs. `DELETE` (PDF p. 12)
**Answer:** `DELETE` removes selected rows and supports a `WHERE` clause; it logs row deletions and fires applicable delete triggers. `TRUNCATE TABLE` removes all rows by deallocating pages, is generally faster, resets identity in SQL Server, and has restrictions such as foreign-key references; transaction and logging details are engine-specific.

###### 10. Table vs. view (PDF p. 12)
**Answer:** A table stores data. A view is a saved query that presents data from one or more tables and generally stores no result rows, although indexed/materialized view features can store indexed results. Views can simplify access but are not automatically a security boundary.

###### 11. Stored procedure vs. function (PDF p. 12)
**Answer:** A stored procedure is invoked as a command and can perform procedural work and return result sets/output parameters. A function returns a scalar or table value and is generally subject to restrictions on side effects and invocation context. Exact rules vary by database engine.

###### 12. What is a CTE? (PDF p. 12)
**Answer:** A common table expression is a named query expression scoped to a single statement, introduced with `WITH`; it can improve readability and may be recursive. A CTE is not inherently materialized or indexed.

###### 13. Temporary table vs. CTE (PDF p. 12)
**Answer:** A CTE is scoped to one statement and is useful for structuring a query; it normally has no separately managed indexes/statistics. A temporary table persists for its session/scope, can be reused across statements and indexed, but adds creation/storage overhead. Choose based on reuse, cardinality, and plan behavior.

###### 14. Temporary table vs. table variable (PDF p. 12)
**Answer:** Both hold temporary data. SQL Server temp tables generally support richer indexing and statistics and often suit larger/intermediate datasets; table variables have different scope/optimization behavior and may suit small/simple sets. Modern SQL Server behavior varies by version, so benchmark the actual query.

###### 15. Subqueries vs. CTE (PDF p. 12)
**Answer:** Both express nested query logic; many optimizers transform them into equivalent plans. A CTE can make multi-step or recursive logic easier to read, while a subquery is concise for local expressions. Neither is automatically faster solely because of its syntax.

###### 16. SQL injection and how will you prevent it? (PDF p. 12)
**Answer:** SQL injection occurs when untrusted input is interpreted as executable SQL. Use parameterized queries/prepared statements or safe ORM APIs, allowlist dynamic identifiers/sort options, apply least-privilege DB access, and avoid concatenating user input into SQL.

###### 17. Triggers in SQL; what is their purpose? (PDF p. 12)
**Answer:** A trigger runs automatically in response to database events such as insert/update/delete. It can enforce certain database-level rules or record auditing, but hidden side effects and multi-row behavior make triggers harder to maintain; use explicit application logic or constraints where suitable.

###### 18. What is `ROW_NUMBER`? (PDF p. 12)
**Answer:** `ROW_NUMBER()` is a window function assigning a unique sequential number within each partition according to an `ORDER BY`. Use a deterministic tie-breaker if stable row numbering matters.

###### 19. `RANK` vs. `DENSE_RANK` (PDF p. 12)
**Answer:** Both assign equal ranks to ties within a partition. `RANK` leaves gaps after tied positions (1, 1, 3); `DENSE_RANK` does not (1, 1, 2).

###### 20. What is a partition? (PDF p. 12)
**Answer:** In a window function, `PARTITION BY` divides rows into independent groups over which the function is evaluated. Database table partitioning is a separate physical-storage feature; clarify which meaning is intended.

###### 21. Ties (PDF p. 12)
**Answer:** Ties occur when rows have equal values under the ranking/order expression. `RANK`/`DENSE_RANK` assign equal ranks; `ROW_NUMBER` still assigns distinct numbers. Add a tie-breaker only when a deterministic ordering is required.

###### 22. What are magic tables? (PDF p. 12)
**Answer:** In SQL Server trigger terminology, `inserted` and `deleted` are logical transition tables containing affected rows—new values and old values respectively, depending on operation. Triggers must handle multiple rows rather than assuming a single row.

###### 23. What is a deadlock? (PDF p. 12)
**Answer:** A deadlock occurs when concurrent transactions wait in a cycle for resources held by each other; the engine selects a victim and rolls it back. Reduce risk with consistent access order, short transactions, suitable indexes/isolation, and retry handling for deadlock victims.

###### 24. What is `WITH (NOLOCK)`? (PDF p. 12)
**Answer:** In SQL Server, `NOLOCK` is equivalent to `READ UNCOMMITTED` for a table reference. It can return dirty, missing, or duplicated rows and does not eliminate all blocking; use row-versioning isolation (such as RCSI) or a different design when appropriate rather than treating it as a harmless performance switch.

###### 25. What is an isolation level? (PDF p. 12)
**Answer:** An isolation level controls how concurrent transactions may observe each other's changes and which anomalies are possible. SQL Server options include Read Uncommitted, Read Committed, Repeatable Read, Snapshot, and Serializable; stronger isolation can reduce anomalies but increase contention or version-store use.

###### 26. What is a transaction and how do you implement it? (PDF p. 12)
**Answer:** A transaction groups operations into a unit with atomicity and consistency expectations, isolated according to its configured level and committed durably on success. Use `BEGIN TRANSACTION`, `COMMIT`, and `ROLLBACK` (or client transaction APIs), handle errors, and keep the transaction scope short.

###### 27. `MERGE` statement (PDF p. 13)
**Answer:** SQL Server `MERGE` matches source and target rows and can apply insert/update/delete actions in one statement. It can be convenient but has concurrency and implementation caveats; test carefully, use correct locking/constraints, and consider separate statements when safer or clearer.

###### 28. How do you send unlimited parameters through stored procedures? (PDF p. 13)
**Answer:** A procedure cannot practically accept an unlimited list of scalar parameters. For variable-length input, pass a table-valued parameter (SQL Server), a structured JSON/XML payload, or stage the data and pass a batch identifier; enforce size limits and validate the payload.

###### 29. How do you optimize queries returning 100 million records? (PDF p. 13)
**Answer:** First question whether all 100 million rows must be returned; paginate or stream only required columns and filter at the source. Review actual plans, indexes, statistics, partitioning and parallelism, use bulk/export or analytical pipelines for full extracts, and measure network, client memory, and end-to-end throughput. Do not load the entire result into application memory.

---

### SQL scenarios

##### Employee names with a duplicated mobile number *(PDF pp.18–19)*

Given the `EmpName` and `MobileNo` rows, return names whose number occurs more than once:

```sql
SELECT e.EmpName
FROM Employee AS e
WHERE e.MobileNo IN (
    SELECT MobileNo
    FROM Employee
    GROUP BY MobileNo
    HAVING COUNT(*) > 1
)
ORDER BY e.EmpName;
```

For the supplied data this returns `A, B, D, E, F, H`. The PDF's stated output also includes `C`, but `C`'s number (`8999999999`) occurs only once; that listed output conflicts with the table. `G` is also unique.

##### Count consumers by orders/refills between April 1 and June 30, 2023, including zero *(PDF pp.19–20)*

Count orders for each consumer in the inclusive date interval, then group consumers by that count. `LEFT JOIN` keeps consumers with no matching orders. The half-open date range also works when `DeliverDate` is a timestamp.

```sql
WITH PerConsumer AS
(
    SELECT c.Id AS ConsumerId, COUNT(o.OrderId) AS RefillCount
    FROM tblConsumer AS c
    LEFT JOIN tblCustomer AS o
      ON o.ConsumerId = c.Id
     AND o.DeliverDate >= '2023-04-01'
     AND o.DeliverDate <  '2023-07-01'
    GROUP BY c.Id
)
SELECT RefillCount, COUNT(*) AS ConsumerCount
FROM PerConsumer
GROUP BY RefillCount
ORDER BY RefillCount;
```

Expected output:

| RefillCount | ConsumerCount |
|---:|---:|
| 0 | 2 |
| 2 | 2 |
| 3 | 1 |
| 5 | 1 |

The range includes both April 1 and June 30. There is no row for a refill count of 4 because no consumer has four orders in that interval.

##### Find duplicate records in a table *(PDF p.23)*

The source does not specify a table or which columns define a duplicate. Group by the business-key columns and retain groups occurring more than once; for example:

```sql
SELECT ColumnA, ColumnB, COUNT(*) AS DuplicateCount
FROM dbo.YourTable
GROUP BY ColumnA, ColumnB
HAVING COUNT(*) > 1;
```

Replace `ColumnA, ColumnB` with the columns that define duplicate identity. To retrieve every full row in duplicate groups, join this grouped result back to the table on those key columns.

##### Find the Nth highest salary *(PDF p.22)*

Assumption: “Nth highest” means the Nth **distinct** salary. `DENSE_RANK` gives ties the same rank:

```sql
WITH Ranked AS
(
    SELECT Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS SalaryRank
    FROM Employee
)
SELECT DISTINCT Salary
FROM Ranked
WHERE SalaryRank = @N;
```

If the intended meaning is the Nth row including duplicate salaries, use `ROW_NUMBER()` with a deterministic tie-breaker instead.
