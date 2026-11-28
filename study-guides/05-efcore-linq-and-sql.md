# EF Core, LINQ, and SQL Server

Each concept is explained once. Where several notes covered the same topic, they have been merged into a single section, and related topics in other guides are linked instead of repeated.

## Contents

- [EF Core & LINQ: Senior .NET Engineer / Solution Architect Interview Guide](#ef-core-linq-senior-interview-guide-md)
- [LINQ Complex Queries: Joins and Grouping (Viva Practice)](#linq-joins-and-grouping-practice)
- [EF Core Viva — More Quick Questions](#ef-core-viva-preparation-md)
- [EF Core Performance: Hands-On Benchmark Lab](#ef-core-performance-interview-and-benchmark-lab-md)
- [SQL Server: Grouping, PIVOT/UNPIVOT, and Window Functions](#sql-server-advanced-grouping-and-window-functions-md)
- [Database Indexing and Transaction Isolation: An Engine-Agnostic Guide](#database-indexing-and-isolation)
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
| `DbConnection` / `DbTransaction` | Underlying ADO.NET (the low-level .NET database access API that EF Core is built on) connection and transaction abstractions. |

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

`IQueryable<T>` does not guarantee server execution for every expression: translation depends on the provider and the expression. In modern EF Core, an untranslatable expression in most query locations throws rather than silently evaluating the entire filter on the client. The exception is the final `Select` projection, which EF may run in memory after the rows return:

```csharp
// Throws InvalidOperationException: EF cannot turn a custom C# method into SQL.
var bad = await db.Orders.Where(o => IsVip(o.CustomerId)).ToListAsync();

// Fine: the final Select runs in memory, after the rows come back.
var ok = await db.Orders.Select(o => new { o.Id, Label = FormatLabel(o) }).ToListAsync();
```

Fix for the first form: express the rule with plain properties (`o.Customer.IsVip`) so it becomes SQL.

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

// Same query, query expression syntax
var queryQ =
    from o in db.Orders
    where o.Status == OrderStatus.Open
    select new { o.Id, o.CreatedAt, o.Total };
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

// Same projection, query expression syntax (the terminal ToListAsync still needs method syntax)
var summariesQ = await (
    from o in db.Orders
    where o.CustomerId == customerId
    orderby o.CreatedAt descending
    select new OrderSummaryDto(
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

The returned integer is the number of state entries written by the save operation (for example, 3 if you added 1 order and modified 2 customers), not necessarily the number of physical SQL statements or rows in every provider-specific case.

##### 3.4 `AcceptAllChanges` and retry awareness

```csharp
await db.SaveChangesAsync(
    acceptAllChangesOnSuccess: false,
    cancellationToken);

// After the application has safely handled the outcome:
db.ChangeTracker.AcceptAllChanges();
```

Normally, after a successful save EF marks every entity `Unchanged` ("accepts" the changes). With `acceptAllChangesOnSuccess: false` the SQL runs but entities keep their `Added`/`Modified` states until you call `AcceptAllChanges()`. That lets you, for example, save inside a retried or distributed transaction and only reset tracking once the whole outer operation has succeeded. If you forget to call it, the next `SaveChanges` will try to write the same changes again.

This overload is advanced and should not be used casually. Retries have a related hazard: if the database committed but the client lost the response, the outcome is ambiguous, and blindly retrying an insert can duplicate data (two identical orders). Fix it with an idempotency key (a client-generated unique id per request) backed by a unique constraint, or a deliberate retry strategy.

##### 3.5 Disconnected updates and PATCH

A "disconnected" entity is one that was not loaded by the current context, for example an object deserialized from an HTTP request body (PATCH is the HTTP verb for a partial update). The context does not know its original values, so it cannot tell which properties the client really changed.

Assumes `request` is a DTO (data transfer object) with `Id`, `Name`, and `Phone`, and `requestEntity` is a `Customer` built from the request body.

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

`Update` marks every property of every entity in the graph as modified. If the client omitted `Phone`, it arrives as `null` and `UPDATE ... SET Phone = NULL` overwrites the real value; a forged child `Id` can also modify rows the caller does not own. For a deliberate partial update, attach a stub (an entity holding only the key) and mark only the intended property:

```csharp
var stub = new Customer { Id = request.Id, Name = request.Name }; // not loaded from the DB
db.Attach(stub);                                      // state: Unchanged
db.Entry(stub).Property(c => c.Name).IsModified = true; // only Name is written
await db.SaveChangesAsync(cancellationToken);          // UPDATE Customers SET Name = @p WHERE Id = @id
```

Validate authorization and allowed fields before applying updates.

##### 3.6 Identity resolution

A tracking context normally maintains one tracked instance per entity key. If the same row appears multiple times in a query graph, EF can reuse the same instance.

- `AsTracking()`: tracks entities and uses the context's identity map.
- `AsNoTracking()`: avoids normal context tracking; repeated occurrences may produce separate instances.
- `AsNoTrackingWithIdentityResolution()`: uses temporary identity resolution for the query without retaining those entities in the context's normal change tracker.

##### 3.7 Shadow properties

A shadow property belongs to EF Core's model but is not declared on the CLR (Common Language Runtime, i.e. the .NET C#) entity class. It can represent a foreign key or metadata such as audit information.

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

A savepoint is a named marker inside a transaction that lets the database undo only the work after the marker. EF Core may create one before `SaveChanges` when it runs inside your explicit transaction and the provider supports it, so a failed save undoes just that save while your earlier work in the transaction survives. SQL Server Multiple Active Result Sets (MARS, several open readers on one connection) affects savepoint behavior; check provider documentation.

An execution strategy (connection resiliency, e.g. `EnableRetryOnFailure()`) automatically retries when a transient error occurs, such as a dropped connection. The catch: a retry re-runs only a delegate, so a manual `BeginTransaction` placed outside it is not retried as a unit and throws `InvalidOperationException`. Wrap the whole unit of work:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync();
    // ... SaveChanges calls ...
    await tx.CommitAsync();
});
```

Retried work must also be idempotent (safe to run twice), because the first attempt may have partly succeeded.

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

When updating, EF includes the original concurrency token in the update predicate (`UPDATE Items SET Quantity = @q WHERE Id = @id AND RowVersion = @originalVersion`). If someone else changed the row since you read it, the version no longer matches, zero rows are updated, and EF throws `DbUpdateConcurrencyException` instead of silently overwriting their change (a "lost update").

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

Know the practical differences among Read Committed, Repeatable Read, Snapshot, and Serializable for the target database (full explanation: [Database Indexing and Transaction Isolation](#database-indexing-and-isolation)). Stronger isolation can reduce certain anomalies but may increase blocking (one transaction waits for another's locks), deadlocks (two transactions wait on each other), or version-store pressure (tempdb space used to keep old row versions for Snapshot). Correctness depends on the database engine and configuration.

A lost update is the classic read-then-write race:

```text
Stock = 5.  Request A reads 5.  Request B reads 5.
A writes 5 - 1 = 4.   B writes 5 - 1 = 4.     -> Stock is 4, should be 3.
```

For inventory or financial invariants, do the check and change in one atomic statement rather than a read-then-write sequence:

```csharp
// Atomic: the database decrements only if enough stock remains.
var rows = await db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Inventory SET Quantity = Quantity - {qty} WHERE Id = {id} AND Quantity >= {qty}");
if (rows == 0) { /* out of stock or lost the race: reject */ }
```

A constraint (`CHECK (Quantity >= 0)`), stored procedure, or concurrency token achieves the same goal.

##### 4.5 Cross-service consistency

A database transaction cannot normally atomically commit changes across independent service databases and a message broker. Use patterns such as:
- **Transactional outbox:** persist business changes and an outgoing event in one database transaction; publish asynchronously.
- **Inbox/deduplication:** recognize duplicate messages.
- **Saga/compensation:** coordinate a multi-step business process when a global ACID (Atomic, Consistent, Isolated, Durable) transaction is unavailable; if step 3 fails, run "undo" steps (for example, refund the payment taken in step 2).

Why it matters: if you save the order and then publish `OrderPlaced` to a broker, a crash between the two leaves an order with no event (or an event with no order). The outbox writes the event row in the same database transaction as the order; a background publisher sends it later.

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

There are two independent layers. EF's layer acts on entities it is tracking; the database's layer is the foreign key's `ON DELETE CASCADE` rule. They can disagree:

```csharp
// Order has Lines with a required FK. Delete the order WITHOUT loading its lines:
db.Orders.Remove(order);
await db.SaveChangesAsync();
// EF only knows about tracked lines. If the FK is ON DELETE CASCADE the database
// deletes the lines; if it is NO ACTION/RESTRICT the DELETE fails with an FK violation.
```

Behavior depends on relationship configuration, entity state, whether dependents are loaded, and provider/schema settings. Know which layer enforces each rule, and set `OnDelete(DeleteBehavior...)` explicitly for important relationships.

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

Resulting shape (one table, `Discriminator` says which class each row is):

| Id | Name | Discriminator | Grade | Subject |
|---|---|---|---|---|
| 1 | Ana | Student | 10 | NULL |
| 2 | Raj | Teacher | NULL | Math |

TPT would instead give `People(Id, Name)`, `Students(Id, Grade)`, `Teachers(Id, Subject)`; TPC gives `Students(Id, Name, Grade)` and `Teachers(Id, Name, Subject)` with no `People` table.

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
- Key generation needs care: separate tables must not hand out the same `Id` to a Student and a Teacher (use a shared sequence or GUID keys). Polymorphic queries (`db.People.ToList()`) become a `UNION ALL` across the concrete tables.

**Viva tip:** Choose based on query patterns, constraints, and data shape—not just normalization preference. Confirm the capabilities of your EF Core version and provider.

##### 5.4 Conversions, owned data, and JSON

Use value conversions for representation changes such as enum-to-string or a strongly typed identifier. A converter does not automatically make every complex object a queryable JSON document. JSON mapping capabilities vary by EF Core version and provider.

Use a `ValueComparer` when EF needs correct equality, hashing, or snapshot behavior for mutable/custom values. Without one, EF cannot tell that the value changed:

```csharp
// Tags is List<string> stored as one comma-separated column via a converter.
product.Tags.Add("sale");            // mutates the list in place
await db.SaveChangesAsync();         // nothing saved: EF compared the same list reference to itself
```

Fix: register a `ValueComparer<List<string>>` that compares by content (`SequenceEqual`) and copies the list for the snapshot.

##### 5.5 Global query filters and tenant security

```csharp
modelBuilder.Entity<Document>()
    .HasQueryFilter(d => d.TenantId == _tenantContext.TenantId
                      && !d.IsDeleted);
```

Assumes `_tenantContext` is an injected service on the `DbContext` that exposes the current tenant's id (see the fuller example below). A global query filter appends that predicate to every query on `Document` automatically (`WHERE TenantId = @t AND IsDeleted = 0`).

Filters are useful for tenant scoping and soft deletion, but they are not a complete security boundary:
- `IgnoreQueryFilters()` can bypass them (one careless call exposes every tenant's rows).
- Raw SQL and other access paths require review: `ExecuteSql`, Dapper queries, stored procedures, and reporting tools never apply the EF filter, and a hand-written `FromSql` query only gets it if you remember to add the tenant predicate yourself.
- Tenant state must be correct for each context, especially with pooling (a pooled context reused with the previous request's tenant value leaks that tenant's data).
- Writes must validate tenant ownership too: filters only affect reads, so a client could still insert or update a row with another tenant's `TenantId`.
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

- N+1 queries from lazy loading or repeated queries in loops: 1 query for the list plus 1 more per row, so 1,000 orders means 1,001 round trips (example below).
- Loading entire entity graphs when a DTO projection is sufficient (wide rows, unused columns, extra memory).
- Missing or poorly ordered indexes: the database scans the whole table instead of seeking to matching rows.
- Large `Skip`/`Take` offsets: `Skip(100000)` still makes the database read and discard 100,000 rows (see 6.3).
- Unbounded result sets and unnecessary materialization: a `ToList()` with no `Take` on a growing table eventually exhausts memory.
- Tracking large read-only graphs: change tracking keeps a snapshot of each entity; use `AsNoTracking()`.
- Client-side transformations after materialization: filtering/sorting in C# after the rows were already fetched.
- Too many round trips or oversized joins.
- Poorly selective predicates (a filter that still matches most rows, e.g. `IsActive = 1`) and non-sargable SQL (a filter that wraps the column in a function so the index cannot be used; see 6.8).
- Excessive parallelism against one context (not thread-safe) or the database (too many concurrent queries exhaust the connection pool).

##### 6.2 N+1 and loading strategies

```csharp
// Risk (N+1): if lazy loading is enabled, accessing Lines can issue one query per order.
var orders = await db.Orders.ToListAsync(cancellationToken);   // query 1
foreach (var o in orders)
    Console.WriteLine(o.Lines.Count);                          // queries 2..N+1, one per order

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

`Include` is not always best. Multiple collection includes can multiply joined rows (cartesian explosion): a blog with 100 posts and 50 contributors returns 100 x 50 = 5,000 joined rows for that single blog, with the blog's columns repeated on each. Compare single-query and split-query behavior; split queries run one SQL query per collection, which removes the multiplication but adds round trips and may observe inconsistent data across commands (a row changes between the two queries) without an appropriate transaction/isolation.

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

**Risks:**
- **Hidden I/O:** `course.Students.Count` looks like a property read but is a database call, so it can sit inside a loop unnoticed (N+1, see above).
- **Serialization-triggered queries:** returning `course` from an API makes the JSON serializer walk every navigation, firing a query for each (and possibly looping through circular references).
- **Access after the context is disposed:** once the `DbContext` is gone the proxy cannot query, so `course.Students` throws `ObjectDisposedException`.

Lazy loading is not automatically more efficient; use it intentionally.


##### 6.3 Offset versus keyset pagination

Offset pagination (`Skip`/`Take`) is convenient for numbered pages but
may become more expensive at large offsets: page 5,000 of 20 rows means the database reads and throws away 100,000 rows first, and rows inserted meanwhile can shift items between pages. Keyset pagination ("seek" pagination) uses the
last-seen sort key and is often better for sequential browsing. Assumes `offset`, `pageSize`, `lastCreatedAt`, and `lastId` are local variables and `OrderListItem` is a small DTO record.

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

"Sargable" means the database can use an index seek for the predicate. Wrapping the indexed column in a function forces it to compute the function for every row (a full scan). For example, `o.CreatedAt.Date == requestedDate` translates to `WHERE CAST(CreatedAt AS date) = @d`, which defeats an index on `CreatedAt`. Use a date range instead:

```csharp
// Bad: function applied to the column, so the index on CreatedAt cannot seek.
var slow = db.Orders.Where(o => o.CreatedAt.Date == requestedDate);

// Good: compare the bare column to a range.
var start = requestedDate.Date;
var end = start.AddDays(1);

var orders = await db.Orders
    .Where(o => o.CreatedAt >= start && o.CreatedAt < end)
    .ToListAsync(cancellationToken);
```

Index selection depends on the complete workload and execution plan; LINQ syntax alone does not guarantee index use.

**Key lookups**

A nonclustered index may find qualifying rows, then SQL Server performs
key lookups (one extra read in the main table per row) to retrieve columns absent from that index. A high number of
lookups can be expensive: 10,000 matching rows means 10,000 extra reads. A covering index adds those columns via `INCLUDE`
so the index alone answers the query. Do this only when the query workload justifies its extra storage and slower writes.
Below, a query filtering by `CustomerId` and selecting `Status` and `CreatedAt` is fully covered:

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
- PLINQ (Parallel LINQ, `.AsParallel()`) is for in-memory work; it does not speed up a database query, and running parallel queries on one `DbContext` throws.
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

A `Func` is executable .NET code. An expression tree is a structured representation that a provider can inspect and potentially translate. Passing a compiled delegate into a query can force work to occur in memory or make translation impossible, depending on how it is used:

```csharp
Func<Customer, bool> pred = c => c.IsActive;

var bad  = db.Customers.AsEnumerable().Where(pred).ToList(); // downloads ALL customers, filters in C#
var good = db.Customers.Where(c => c.IsActive).ToList();     // WHERE IsActive = 1 runs in SQL
```

When a helper method takes a predicate for use in a query, declare the parameter as `Expression<Func<T, bool>>`, not `Func<T, bool>`; with `Func`, the call binds to `Enumerable.Where` and silently loads the whole table.

##### 7.2 `Any` versus `Count`

Prefer `Any()` when the question is whether at least one row exists:

```csharp
bool hasOpenOrders = await db.Orders
    .AnyAsync(o => o.CustomerId == customerId &&
                   o.Status == OrderStatus.Open,
              cancellationToken);

// Query expression syntax: there is no keyword for Any, so mix in a method call
bool hasOpenOrdersQ = await (
    from o in db.Orders
    where o.CustomerId == customerId && o.Status == OrderStatus.Open
    select o).AnyAsync(cancellationToken);
```

`Count()` asks for the count and may require more work (scanning every matching row to total them, when you only needed to know "is there at least one?"). `Any()` translates to `WHERE EXISTS (...)`, which can stop at the first match. The database optimizer can optimize both, but use the operator that matches the intent.

##### 7.3 Left join pattern

A left join keeps every customer, even those with no orders (their order columns are null). A traditional LINQ left join uses `GroupJoin` (the `into orders` part), `SelectMany` (the second `from`), and `DefaultIfEmpty` (supplies one null row for an empty group):

```csharp
// Customers: (1, Ana), (2, Raj).  Orders: (10, CustomerId 1).
// Result: (1, Ana, 10) and (2, Raj, null)   <- Raj is kept despite having no orders.
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

The same left join in method (lambda) syntax:

```csharp
var resultM = db.Customers
    .GroupJoin(db.Orders, c => c.Id, o => o.CustomerId, (c, orders) => new { c, orders })
    .SelectMany(
        x => x.orders.DefaultIfEmpty(),
        (x, order) => new { x.c.Id, x.c.Name, OrderId = order == null ? (int?)null : order.Id });

// .NET 10 / EF Core 10: one operator instead of GroupJoin + SelectMany + DefaultIfEmpty
var resultL = db.Customers.LeftJoin(
    db.Orders,
    c => c.Id,
    o => o.CustomerId,
    (c, order) => new { c.Id, c.Name, OrderId = order == null ? (int?)null : order.Id });
```

Translation depends on expression shape and provider. `LeftJoin`/`RightJoin` arrived in .NET 10 / EF Core 10, and a native `FullJoin` is reported for EF Core 11; see [section 22](#linq-version-features) for what each version adds.

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

// Same query, query expression syntax
var totalsQ = await (
    from line in db.OrderLines
    group line by line.ProductId into g
    select new { ProductId = g.Key, Quantity = g.Sum(line => line.Quantity) })
    .ToListAsync(cancellationToken);
```

Returning full `IGrouping<TKey,TElement>` objects (each group with all its member rows) is not equivalent to SQL `GROUP BY`, which can only return one row per group. EF may have to download all the rows and group them in memory, or throw, depending on version and query shape:

```csharp
// Risky: asks for every row of every group, which SQL GROUP BY cannot return.
var groups = await db.OrderLines.GroupBy(l => l.ProductId).ToListAsync();
// Safe: reduce each group to aggregates (as above), or fetch rows then group in memory deliberately.
```

For worked examples of every join type, multi-table joins, and grouping across joined tables, see [LINQ Complex Queries: Joins and Grouping](#linq-joins-and-grouping-practice).

##### 7.5 Deferred execution and repeated enumeration

```csharp
var query = db.Customers.Where(c => c.IsActive);

// Each enumeration can execute the query again.
var first = await query.ToListAsync(cancellationToken);
var second = await query.ToListAsync(cancellationToken);

// Query expression syntax: building the query does not run it either
var queryQ = from c in db.Customers where c.IsActive select c;
var snapshot = await queryQ.ToListAsync(cancellationToken);   // runs once; reuse `snapshot`
```

Materialize once when reuse is intended and the result size is acceptable. Do not materialize early by habit if the query could still be filtered, ordered, or projected in SQL.

##### 7.6 Dynamic predicates

Build expression trees or compose predicates carefully when filters are optional. The simplest safe pattern is to chain `Where` only when the filter was supplied:

```csharp
// Assumes name (string?) and minTotal (decimal?) are optional search inputs.
IQueryable<Order> q = db.Orders;
if (!string.IsNullOrWhiteSpace(name)) q = q.Where(o => o.CustomerName.Contains(name));
if (minTotal is not null)             q = q.Where(o => o.Total >= minTotal);
var result = await q.ToListAsync();

// Query expression syntax: a where clause can hold the condition directly
var resultQ = await (
    from o in db.Orders
    where string.IsNullOrWhiteSpace(name) || o.CustomerName.Contains(name)   // optional-filter trick: SQL gets (@p IS NULL OR ...)
    where minTotal == null || o.Total >= minTotal
    select o).ToListAsync();
```

The inline `||` form keeps one query shape (one cached plan) but the SQL carries both branches; chaining `Where` conditionally produces the leanest SQL per request but several shapes.

Avoid string-concatenated SQL from user input (SQL injection: input like `'; DROP TABLE Orders;--` becomes part of the command). Verify that dynamic expression construction preserves parameterization and query-cache-friendly shapes (values should be parameters, not baked-in constants).

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

Application validation alone cannot prevent races between concurrent requests. Example: two requests both run `if (!db.Users.Any(u => u.Email == e))` and both see "free", then both insert, giving duplicate emails. A unique index makes the second insert fail with a `DbUpdateException` you can turn into a friendly "email already used" error.

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

`FromSqlInterpolated` looks like string interpolation but turns each `{city}` into a SQL parameter (`@p0`), so input such as `x' OR 1=1 --` is treated as plain data. For `FromSqlRaw`, pass parameters separately; do not concatenate untrusted values into SQL:

```csharp
// Bad: input becomes part of the SQL text (injection).
db.Customers.FromSqlRaw("SELECT * FROM dbo.Customers WHERE City = '" + city + "'");
// Good: parameter placeholder + separate argument.
db.Customers.FromSqlRaw("SELECT * FROM dbo.Customers WHERE City = {0}", city);
```

Note: in EF Core, `FromSql` must return all columns of the entity type. Parameters cannot generally stand in for SQL identifiers such as table names or sort directions, so dynamic identifiers require strict allow-listing.

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
3. Prefer backward-compatible expand/migrate/contract changes for zero-downtime deployments. Example for renaming `Name` to `FullName`: (expand) add `FullName` while old code still uses `Name`; (migrate) deploy code that writes both and backfill old rows; (contract) once no code reads `Name`, drop it. A direct rename would break the old app version still running during the rollout.
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

<a id="linq-joins-and-grouping-practice"></a>

## LINQ Complex Queries: Joins and Grouping (Viva Practice)

Hands-on examples for the questions most often asked in a viva: every join type, joining many tables, and grouping across joined tables. Each query is shown in both **query syntax** (query expression: `from … join … group … select`) and **method syntax** (lambda: `.Join(…)`, `.GroupBy(…)`, `.Select(…)`), so you can compare the two forms and write either one when asked. The compiler turns query syntax into the same method calls. Sections 7.3 and 7.4 above give the short theory; this section is the practice set.

> Every `q` below is an `IQueryable` against EF Core unless noted; execute it with `ToListAsync()`. The same queries also run in LINQ to Objects (on `List<T>`), except where a note says otherwise. Always confirm translation with `ToQueryString()`.

#### 1. Sample model

```csharp
public enum OrderStatus { Pending, Shipped, Delivered, Cancelled }

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string City { get; set; } = "";
    public List<Order> Orders { get; set; } = [];
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
    public int? EmployeeId { get; set; }             // sales rep, optional
    public Employee? Employee { get; set; }
    public DateTime OrderDate { get; set; }
    public OrderStatus Status { get; set; }
    public List<OrderLine> Lines { get; set; } = [];
}

public class OrderLine
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public Order Order { get; set; } = null!;
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal ListPrice { get; set; }
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    public List<OrderLine> OrderLines { get; set; } = [];
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Product> Products { get; set; } = [];
}

public class Department
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Employee> Employees { get; set; } = [];
}

public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Salary { get; set; }
    public int? DepartmentId { get; set; }           // nullable on purpose
    public Department? Department { get; set; }
    public int? ManagerId { get; set; }              // self-reference
    public Employee? Manager { get; set; }
}

public class ShipmentLine                            // no FK navigation: joined by composite key
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int ShippedQty { get; set; }
}

public class Promotion                               // joined by date range, not by key
{
    public int Id { get; set; }
    public string Code { get; set; } = "";
    public DateTime StartDate { get; set; }
    public DateTime EndDate { get; set; }
}
```

Small data set used to explain join results:

| Departments | | Employees | | | |
|---|---|---|---|---|---|
| **Id** | **Name** | **Name** | **DepartmentId** | **ManagerId** | **Salary** |
| 1 | IT | Rahim (Id 1) | 1 | — | 900 |
| 2 | HR | Karim (Id 2) | 1 | 1 | 600 |
| 3 | Finance | Sara (Id 3) | 2 | 1 | 700 |
| | | Nila (Id 4) | — | 2 | 650 |

What each join returns for `Employees` (left) and `Departments` (right):

| Join type | Rows returned | Count |
|---|---|---|
| Inner | Rahim–IT, Karim–IT, Sara–HR | 3 |
| Left outer | inner rows + Nila–NULL | 4 |
| Right outer | inner rows + NULL–Finance | 4 |
| Full outer | inner rows + Nila–NULL + NULL–Finance | 5 |
| Cross | every employee × every department | 4 × 3 = 12 |
| Semi (departments that have employees) | IT, HR (IT appears once, not twice) | 2 |
| Anti (departments without employees) | Finance | 1 |
| Anti (employees without a department) | Nila | 1 |

---

#### 2. Inner join

```csharp
// Query syntax
var q =
    from e in db.Employees
    join d in db.Departments on e.DepartmentId equals (int?)d.Id
    select new { Employee = e.Name, Department = d.Name };

// Method syntax
var q2 = db.Employees.Join(
    db.Departments,
    e => e.DepartmentId,          // outer key
    d => (int?)d.Id,              // inner key
    (e, d) => new { Employee = e.Name, Department = d.Name });

// Navigation (preferred when the relationship is mapped)
var q3 = db.Employees
    .Where(e => e.Department != null)
    .Select(e => new { Employee = e.Name, Department = e.Department!.Name });
```

```sql
SELECT e.Name, d.Name
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.Id;
```

Viva points:
- `join` uses the keyword `equals`, not `==`. The **outer** sequence's key goes on the left and the **inner** key on the right; swapping them is a compile error.
- Both keys must have the same type. `int?` vs `int` fails type inference, so cast one side: `(int?)d.Id`.
- `join` supports **only equality**. For `<`, `>`, or `BETWEEN`, see [non-equi join](#linq-non-equi-join).

---

#### 3. Joining many tables (inner)

Customer → Order → OrderLine → Product → Category:

```csharp
var q =
    from c in db.Customers
    join o in db.Orders on c.Id equals o.CustomerId
    join l in db.OrderLines on o.Id equals l.OrderId
    join p in db.Products on l.ProductId equals p.Id
    join cat in db.Categories on p.CategoryId equals cat.Id
    where o.Status == OrderStatus.Delivered
    select new
    {
        Customer = c.Name,
        OrderId = o.Id,
        Product = p.Name,
        Category = cat.Name,
        LineTotal = l.Quantity * l.UnitPrice
    };
```

Method syntax needs an intermediate anonymous object after every `Join` (this is what the compiler generates for query syntax, called *transparent identifiers*):

```csharp
var q2 = db.Customers
    .Join(db.Orders,     c => c.Id,          o => o.CustomerId, (c, o) => new { c, o })
    .Join(db.OrderLines, x => x.o.Id,        l => l.OrderId,    (x, l) => new { x.c, x.o, l })
    .Join(db.Products,   x => x.l.ProductId, p => p.Id,         (x, p) => new { x.c, x.o, x.l, p })
    .Join(db.Categories, x => x.p.CategoryId, cat => cat.Id,    (x, cat) => new { x.c, x.o, x.l, x.p, cat })
    .Where(x => x.o.Status == OrderStatus.Delivered)
    .Select(x => new
    {
        Customer = x.c.Name,
        OrderId = x.o.Id,
        Product = x.p.Name,
        Category = x.cat.Name,
        LineTotal = x.l.Quantity * x.l.UnitPrice
    });
```

The same result with navigations; EF generates the joins:

```csharp
var q3 = db.OrderLines
    .Where(l => l.Order.Status == OrderStatus.Delivered)
    .Select(l => new
    {
        Customer = l.Order.Customer.Name,
        OrderId = l.OrderId,
        Product = l.Product.Name,
        Category = l.Product.Category.Name,
        LineTotal = l.Quantity * l.UnitPrice
    });
```

**Start from the table at the grain you need.** Here one result row = one order line, so start from `OrderLines`. Joining one-to-many multiplies rows (fan-out): a customer with 3 orders × 4 lines appears 12 times.

---

#### 4. Left outer join

##### 4.1 Classic pattern: `join … into` + `DefaultIfEmpty()`

```csharp
var q =
    from e in db.Employees
    join d in db.Departments on e.DepartmentId equals (int?)d.Id into deptGroup
    from d in deptGroup.DefaultIfEmpty()
    select new
    {
        Employee = e.Name,
        Department = d != null ? d.Name : "(none)"
    };
// Rahim–IT, Karim–IT, Sara–HR, Nila–(none)
```

Method syntax:

```csharp
var q2 = db.Employees
    .GroupJoin(db.Departments,
        e => e.DepartmentId,
        d => (int?)d.Id,
        (e, depts) => new { e, depts })
    .SelectMany(
        x => x.depts.DefaultIfEmpty(),
        (x, d) => new
        {
            Employee = x.e.Name,
            Department = d != null ? d.Name : "(none)"
        });
```

How it works: `GroupJoin` pairs each employee with a (possibly empty) collection of matching departments, `DefaultIfEmpty()` turns an empty collection into one `null`, and `SelectMany` flattens it back to rows.

##### 4.2 Shorter form: `from … in …Where(…).DefaultIfEmpty()`

```csharp
var q =
    from e in db.Employees
    from d in db.Departments
        .Where(d => d.Id == e.DepartmentId)
        .DefaultIfEmpty()
    select new { Employee = e.Name, Department = d != null ? d.Name : "(none)" };

// Method syntax
var q2 = db.Employees.SelectMany(
    e => db.Departments
        .Where(d => d.Id == e.DepartmentId)
        .DefaultIfEmpty(),
    (e, d) => new { Employee = e.Name, Department = d != null ? d.Name : "(none)" });
```

EF translates this to `LEFT JOIN`. This form also allows extra conditions and non-equality conditions inside the `Where`.

##### 4.3 .NET 10 / EF Core 10: `LeftJoin` operator

```csharp
var q = db.Employees.LeftJoin(
    db.Departments,
    e => e.DepartmentId,
    d => (int?)d.Id,
    (e, d) => new { Employee = e.Name, Department = d != null ? d.Name : "(none)" });
```

There is no new query-syntax keyword; in query syntax you still write `join … into … DefaultIfEmpty()`.

##### 4.4 Trap: filtering the right side in `where` turns it into an inner join

Requirement: *all departments, with their employees earning more than 600; keep departments that have none.*

```csharp
// WRONG: the where runs after the join; rows with e == null are removed,
// so Finance disappears. (In LINQ to Objects, e.Salary on null also throws.)
var wrong =
    from d in db.Departments
    join e in db.Employees on (int?)d.Id equals e.DepartmentId into eg
    from e in eg.DefaultIfEmpty()
    where e.Salary > 600
    select new { Department = d.Name, Employee = e.Name };

// RIGHT: filter the right side before it is joined (SQL: condition in ON).
var right =
    from d in db.Departments
    join e in db.Employees.Where(x => x.Salary > 600)
        on (int?)d.Id equals e.DepartmentId into eg
    from e in eg.DefaultIfEmpty()
    select new { Department = d.Name, Employee = e != null ? e.Name : null };
// IT–Rahim, HR–Sara, Finance–NULL

// RIGHT, method syntax
var right2 = db.Departments
    .GroupJoin(db.Employees.Where(x => x.Salary > 600),
        d => (int?)d.Id,
        e => e.DepartmentId,
        (d, emps) => new { d, emps })
    .SelectMany(
        x => x.emps.DefaultIfEmpty(),
        (x, e) => new { Department = x.d.Name, Employee = e != null ? e.Name : null });
```

```sql
-- right
SELECT d.Name, e.Name
FROM Departments d
LEFT JOIN Employees e ON e.DepartmentId = d.Id AND e.Salary > 600;
```

##### 4.5 Left join across several tables

Every order with its customer (required), sales rep (optional), and the rep's department (optional):

```csharp
var q =
    from o in db.Orders
    join c in db.Customers on o.CustomerId equals c.Id                    // inner
    join e in db.Employees on o.EmployeeId equals (int?)e.Id into reps    // left
    from e in reps.DefaultIfEmpty()
    join d in db.Departments on e.DepartmentId equals (int?)d.Id into ds  // left, chained
    from d in ds.DefaultIfEmpty()
    select new
    {
        OrderId = o.Id,
        Customer = c.Name,
        SalesRep = e != null ? e.Name : null,
        Department = d != null ? d.Name : null
    };

// Method syntax: Join for the inner join, GroupJoin + SelectMany for each left join
var q1 = db.Orders
    .Join(db.Customers, o => o.CustomerId, c => c.Id, (o, c) => new { o, c })
    .GroupJoin(db.Employees, x => x.o.EmployeeId, e => (int?)e.Id, (x, reps) => new { x.o, x.c, reps })
    .SelectMany(x => x.reps.DefaultIfEmpty(), (x, e) => new { x.o, x.c, e })
    .GroupJoin(db.Departments, x => x.e.DepartmentId, d => (int?)d.Id, (x, ds) => new { x.o, x.c, x.e, ds })
    .SelectMany(x => x.ds.DefaultIfEmpty(), (x, d) => new
    {
        OrderId = x.o.Id,
        Customer = x.c.Name,
        SalesRep = x.e != null ? x.e.Name : null,
        Department = d != null ? d.Name : null
    });
```

`e.DepartmentId` when `e` is null works in EF (SQL null semantics) but throws in LINQ to Objects. The navigation version is shorter, and EF makes optional navigations `LEFT JOIN`s automatically:

```csharp
var q2 = db.Orders.Select(o => new
{
    OrderId = o.Id,
    Customer = o.Customer.Name,              // required -> INNER JOIN
    SalesRep = o.Employee!.Name,             // optional -> LEFT JOIN, null if missing
    Department = o.Employee!.Department!.Name
});
```

Null-conditional `?.` is **not allowed in expression trees** (compiler error CS8072), so use `x != null ? x.Prop : null`, or `!` with EF's null propagation.

---

#### 5. Right outer join

LINQ query syntax has no right join. Swap the sides and use a left join:

```csharp
// "Employees RIGHT JOIN Departments" == "Departments LEFT JOIN Employees"
var q =
    from d in db.Departments
    join e in db.Employees on (int?)d.Id equals e.DepartmentId into eg
    from e in eg.DefaultIfEmpty()
    select new { Employee = e != null ? e.Name : null, Department = d.Name };
// Rahim–IT, Karim–IT, Sara–HR, NULL–Finance

// Method syntax
var q1 = db.Departments
    .GroupJoin(db.Employees, d => (int?)d.Id, e => e.DepartmentId, (d, emps) => new { d, emps })
    .SelectMany(
        x => x.emps.DefaultIfEmpty(),
        (x, e) => new { Employee = e != null ? e.Name : null, Department = x.d.Name });
```

.NET 10 / EF Core 10 adds `RightJoin`, which keeps every **inner** row:

```csharp
var q2 = db.Employees.RightJoin(
    db.Departments,
    e => e.DepartmentId,
    d => (int?)d.Id,
    (e, d) => new { Employee = e != null ? e.Name : null, Department = d.Name });
```

---

#### 6. Full outer join

There is no full-join operator. Combine a **left join** (all employees, with their department if any) with the **right-only rows** (a right anti join: departments that no employee matches, here Finance):

```csharp
var leftPart =
    from e in db.Employees
    join d in db.Departments on e.DepartmentId equals (int?)d.Id into dg
    from d in dg.DefaultIfEmpty()
    select new { Employee = e.Name, Department = d != null ? d.Name : null };

var rightOnly =
    from d in db.Departments
    where !db.Employees.Any(e => e.DepartmentId == d.Id)
    select new { Employee = (string?)null, Department = d.Name };

var fullOuter = leftPart.Concat(rightOnly);   // SQL: UNION ALL
// Rahim–IT, Karim–IT, Sara–HR, Nila–NULL, NULL–Finance

// Method syntax
var leftPart2 = db.Employees
    .GroupJoin(db.Departments, e => e.DepartmentId, d => (int?)d.Id, (e, ds) => new { e, ds })
    .SelectMany(
        x => x.ds.DefaultIfEmpty(),
        (x, d) => new { Employee = x.e.Name, Department = d != null ? d.Name : null });

var rightOnly2 = db.Departments
    .Where(d => !db.Employees.Any(e => e.DepartmentId == d.Id))
    .Select(d => new { Employee = (string?)null, Department = d.Name });

var fullOuter2 = leftPart2.Concat(rightOnly2);
```

**EF Core 11 (release candidate at the time of writing; verify on the official docs):** a native `FullJoin` operator is reported, translating to one SQL `FULL JOIN`. It follows the `LeftJoin`/`RightJoin` shape, and again has no query-syntax keyword:

```csharp
var full = db.Employees.FullJoin(
    db.Departments,
    e => e.DepartmentId,
    d => (int?)d.Id,
    (e, d) => new
    {
        Employee   = e != null ? e.Name : null,      // both sides can be null in a full join
        Department = d != null ? d.Name : null
    });
```

- Use `Concat` (`UNION ALL`) rather than `Union` (`UNION`): the two parts never overlap, and `Union` would also remove genuinely duplicate rows and add a distinct/sort step.
- Both projections must have the same shape: the same property names, order, and types.
- If the provider can't translate the set operation, run both halves and `Concat` in memory.

---

#### 7. Cross join

```csharp
// Query syntax: two from clauses with no condition
var q =
    from e in db.Employees
    from d in db.Departments
    select new { Employee = e.Name, Department = d.Name };   // 4 × 3 = 12 rows

// Method syntax
var q2 = db.Employees.SelectMany(
    e => db.Departments,
    (e, d) => new { Employee = e.Name, Department = d.Name });
```

Real use: build a complete grid, then left join actual data onto it so missing combinations show as zero (for example every category × every month in a sales report).

---

#### 8. Self join

```csharp
// Employee with manager (left self join keeps the top boss)
var q =
    from e in db.Employees
    join m in db.Employees on e.ManagerId equals (int?)m.Id into mg
    from m in mg.DefaultIfEmpty()
    select new { Employee = e.Name, Manager = m != null ? m.Name : "(top)" };
// Rahim–(top), Karim–Rahim, Sara–Rahim, Nila–Karim

// Classic viva question: employees earning more than their manager
var q2 =
    from e in db.Employees
    join m in db.Employees on e.ManagerId equals (int?)m.Id
    where e.Salary > m.Salary
    select new { Employee = e.Name, e.Salary, Manager = m.Name, ManagerSalary = m.Salary };
// Nila (650) > Karim (600)

// Managers with their number of direct reports (self join + group)
var q3 =
    from e in db.Employees
    join m in db.Employees on e.ManagerId equals (int?)m.Id
    group e by new { m.Id, m.Name } into g
    select new { Manager = g.Key.Name, DirectReports = g.Count() };
// Rahim 2, Karim 1
```

The same three queries in method syntax:

```csharp
var m1 = db.Employees
    .GroupJoin(db.Employees, e => e.ManagerId, m => (int?)m.Id, (e, ms) => new { e, ms })
    .SelectMany(
        x => x.ms.DefaultIfEmpty(),
        (x, m) => new { Employee = x.e.Name, Manager = m != null ? m.Name : "(top)" });

var m2 = db.Employees
    .Join(db.Employees, e => e.ManagerId, m => (int?)m.Id, (e, m) => new { e, m })
    .Where(x => x.e.Salary > x.m.Salary)
    .Select(x => new { Employee = x.e.Name, x.e.Salary, Manager = x.m.Name, ManagerSalary = x.m.Salary });

var m3 = db.Employees
    .Join(db.Employees, e => e.ManagerId, m => (int?)m.Id, (e, m) => new { e, m })
    .GroupBy(x => new { x.m.Id, x.m.Name })
    .Select(g => new { Manager = g.Key.Name, DirectReports = g.Count() });

// Navigation version of the first query
var m4 = db.Employees.Select(e => new
{
    Employee = e.Name,
    Manager = e.Manager != null ? e.Manager.Name : "(top)"
});
```

---

#### 9. Semi join and anti join (`EXISTS` / `NOT EXISTS`)

```csharp
// Semi join: departments that have at least one employee      -> IT, HR
var withStaff  = db.Departments.Where(d => d.Employees.Any());                                // method
var withStaffQ = from d in db.Departments where d.Employees.Any() select d;                   // query

// Without a navigation: correlate on the key
var withStaff2  = db.Departments.Where(d => db.Employees.Any(e => e.DepartmentId == d.Id));
var withStaff2Q = from d in db.Departments
                  where db.Employees.Any(e => e.DepartmentId == d.Id)
                  select d;

// Anti join: departments with no employees                     -> Finance
var empty  = db.Departments.Where(d => !d.Employees.Any());
var emptyQ = from d in db.Departments where !d.Employees.Any() select d;

// Anti join written as left join + null check (query syntax)
var empty2 =
    from d in db.Departments
    join e in db.Employees on (int?)d.Id equals e.DepartmentId into eg
    from e in eg.DefaultIfEmpty()
    where e == null
    select d.Name;

// Same, method syntax
var empty3 = db.Departments
    .GroupJoin(db.Employees, d => (int?)d.Id, e => e.DepartmentId, (d, emps) => new { d, emps })
    .SelectMany(x => x.emps.DefaultIfEmpty(), (x, e) => new { x.d, e })
    .Where(x => x.e == null)
    .Select(x => x.d.Name);

// Multi-level semi join: customers who bought anything in "Electronics"
var buyers = db.Customers.Where(c =>
    c.Orders.Any(o => o.Lines.Any(l => l.Product.Category.Name == "Electronics")));

var buyersQ =
    from c in db.Customers
    where c.Orders.Any(o => o.Lines.Any(l => l.Product.Category.Name == "Electronics"))
    select c;

// IN list
int[] ids = [1, 3];
var some  = db.Departments.Where(d => ids.Contains(d.Id));             // WHERE Id IN (...)
var someQ = from d in db.Departments where ids.Contains(d.Id) select d;
```

Why `Any()` instead of a join plus `Distinct()`: a join returns IT twice (two employees), so you'd need `Distinct()`. `Any()` expresses "exists", translates to `EXISTS`, and never duplicates the outer row.

---

#### 10. Composite-key join

The anonymous types on both sides must have the **same property names and types, in the same order**; otherwise they are two different types and the compiler reports that it cannot infer the join key type. Assumes `ShipmentLines` rows record what was shipped for each (`OrderId`, `ProductId`) pair:

```csharp
var q =
    from l in db.OrderLines
    join s in db.ShipmentLines
        on new { l.OrderId, l.ProductId } equals new { s.OrderId, s.ProductId }
    select new { l.OrderId, l.ProductId, l.Quantity, s.ShippedQty };

// When the column names differ, name the members explicitly:
//   on new { K1 = a.OrderNo, K2 = a.Sku } equals new { K1 = b.OrderId, K2 = b.ProductCode }
```

Method syntax:

```csharp
var q2 = db.OrderLines.Join(
    db.ShipmentLines,
    l => new { l.OrderId, l.ProductId },
    s => new { s.OrderId, s.ProductId },
    (l, s) => new { l.OrderId, l.ProductId, l.Quantity, s.ShippedQty });
```

**Group, then join (a common complex query):** one order line can have several shipments, so total the shipments first, then left join to find the pending quantity:

```csharp
var shipped =
    from s in db.ShipmentLines
    group s by new { s.OrderId, s.ProductId } into g
    select new { g.Key.OrderId, g.Key.ProductId, Qty = g.Sum(x => x.ShippedQty) };

var pending =
    from l in db.OrderLines
    join s in shipped
        on new { l.OrderId, l.ProductId } equals new { s.OrderId, s.ProductId } into sg
    from s in sg.DefaultIfEmpty()
    let shippedQty = s != null ? s.Qty : 0
    where l.Quantity > shippedQty
    select new
    {
        l.OrderId,
        l.ProductId,
        Ordered = l.Quantity,
        Shipped = shippedQty,
        Pending = l.Quantity - shippedQty
    };

// Method syntax (`let` becomes an extra Select that carries the value forward)
var shipped2 = db.ShipmentLines
    .GroupBy(s => new { s.OrderId, s.ProductId })
    .Select(g => new { g.Key.OrderId, g.Key.ProductId, Qty = g.Sum(x => x.ShippedQty) });

var pending2 = db.OrderLines
    .GroupJoin(shipped2,
        l => new { l.OrderId, l.ProductId },
        s => new { s.OrderId, s.ProductId },
        (l, sg) => new { l, sg })
    .SelectMany(x => x.sg.DefaultIfEmpty(), (x, s) => new { x.l, ShippedQty = s != null ? s.Qty : 0 })
    .Where(x => x.l.Quantity > x.ShippedQty)
    .Select(x => new
    {
        x.l.OrderId,
        x.l.ProductId,
        Ordered = x.l.Quantity,
        Shipped = x.ShippedQty,
        Pending = x.l.Quantity - x.ShippedQty
    });
```

Joining the raw `ShipmentLines` instead would multiply each order line by its number of shipments.

---

<a id="linq-non-equi-join"></a>

#### 11. Non-equi join (range / inequality)

`join … equals` can't express ranges. Use a second `from` with a `Where`:

```csharp
// Each order with the promotion active on its date (left join, range condition)
var q =
    from o in db.Orders
    from p in db.Promotions
        .Where(p => o.OrderDate >= p.StartDate && o.OrderDate <= p.EndDate)
        .DefaultIfEmpty()
    select new { o.Id, o.OrderDate, Promo = p != null ? p.Code : null };

// Method syntax
var q2 = db.Orders.SelectMany(
    o => db.Promotions
        .Where(p => o.OrderDate >= p.StartDate && o.OrderDate <= p.EndDate)
        .DefaultIfEmpty(),
    (o, p) => new { o.Id, o.OrderDate, Promo = p != null ? p.Code : null });
```

```sql
SELECT o.Id, o.OrderDate, p.Code
FROM Orders o
LEFT JOIN Promotions p ON o.OrderDate >= p.StartDate AND o.OrderDate <= p.EndDate;
```

Drop `.DefaultIfEmpty()` for an inner join. If promotions overlap, an order appears once per matching promotion. Example: order 7 dated 2024-03-05 with promotions `SPRING` (Mar 1-Mar 31) and `FLASH` (Mar 5-Mar 6) returns two rows, `(7, SPRING)` and `(7, FLASH)`; an order dated in no promotion's range returns one row with `Promo = null` (left join) or none (inner join).

---

#### 12. Group join: a hierarchical result (not flattened)

Assumes `customers` and `orders` are in-memory `List<Customer>` / `List<Order>` (not `DbSet`s). A group join returns one result per customer with that customer's orders nested inside (customers with no orders get `OrderCount = 0` and an empty list), instead of one flat row per customer/order pair:

```csharp
// LINQ to Objects
var q =
    from c in customers
    join o in orders on c.Id equals o.CustomerId into custOrders
    select new
    {
        c.Name,
        OrderCount = custOrders.Count(),
        Orders = custOrders.Select(o => new { o.Id, o.OrderDate }).ToList()
    };

// LINQ to Objects, method syntax
var q1 = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, custOrders) => new
    {
        c.Name,
        OrderCount = custOrders.Count(),
        Orders = custOrders.Select(o => new { o.Id, o.OrderDate }).ToList()
    });

// EF Core: use the navigation instead
var q2 = db.Customers.Select(c => new
{
    c.Name,
    OrderCount = c.Orders.Count(),
    Orders = c.Orders
        .OrderByDescending(o => o.OrderDate)
        .Select(o => new { o.Id, o.OrderDate })
        .ToList()
});
```

In EF Core, a `GroupJoin` that isn't followed by `SelectMany`/`DefaultIfEmpty` (a left join) is not reliably translated, so use the navigation or a correlated subquery.

**`GroupJoin` vs `GroupBy`:** `GroupJoin` correlates **two** sequences, and the outer item is kept even with zero matches. `GroupBy` partitions **one** sequence by a key; a key with no rows can't exist.

---

#### 13. `GroupBy` basics

##### 13.1 Single key with aggregates

```csharp
var q =
    from e in db.Employees
    group e by e.DepartmentId into g
    select new
    {
        DepartmentId = g.Key,                    // null is its own group (Nila)
        Headcount = g.Count(),
        TotalSalary = g.Sum(x => x.Salary),
        AvgSalary = g.Average(x => x.Salary),
        MinSalary = g.Min(x => x.Salary),
        MaxSalary = g.Max(x => x.Salary)
    };

// Method syntax
var q2 = db.Employees
    .GroupBy(e => e.DepartmentId)
    .Select(g => new { DepartmentId = g.Key, Headcount = g.Count(), TotalSalary = g.Sum(x => x.Salary) });
```

`group X by K`: **X** is what goes into each group (the element), **K** is the key. `group e.Salary by e.DepartmentId` makes each group a list of salaries, so `g.Sum()` needs no lambda.

##### 13.2 Multiple keys (anonymous type)

```csharp
var q =
    from o in db.Orders
    group o by new { o.Customer.City, o.OrderDate.Year } into g
    orderby g.Key.City, g.Key.Year
    select new { g.Key.City, g.Key.Year, Orders = g.Count() };

// Method syntax
var q2 = db.Orders
    .GroupBy(o => new { o.Customer.City, o.OrderDate.Year })
    .OrderBy(g => g.Key.City)
    .ThenBy(g => g.Key.Year)
    .Select(g => new { g.Key.City, g.Key.Year, Orders = g.Count() });
```

##### 13.3 `WHERE` vs `HAVING`

```csharp
var q = db.Orders
    .Where(o => o.Status != OrderStatus.Cancelled)        // before GroupBy -> WHERE
    .GroupBy(o => o.CustomerId)
    .Where(g => g.Count() >= 5)                           // after GroupBy  -> HAVING
    .Select(g => new { CustomerId = g.Key, Orders = g.Count() })
    .OrderByDescending(x => x.Orders);

// Query syntax: the where after "into g" is the HAVING
var q2 =
    from o in db.Orders
    where o.Status != OrderStatus.Cancelled
    group o by o.CustomerId into g
    where g.Count() >= 5
    orderby g.Count() descending
    select new { CustomerId = g.Key, Orders = g.Count() };
```

```sql
SELECT CustomerId, COUNT(*) AS Orders
FROM Orders
WHERE Status <> 3          -- 3 = Cancelled; filters ROWS before grouping
GROUP BY CustomerId
HAVING COUNT(*) >= 5       -- filters GROUPS after counting (aggregates are not allowed in WHERE)
ORDER BY Orders DESC;
```

Meaning: `WHERE` removes individual orders first (cancelled ones never get counted). `HAVING` then drops customers whose remaining count is under 5. A row-level filter belongs before `GroupBy` (cheaper: fewer rows to group); only conditions on aggregates like `Count()` go after it.

##### 13.4 Computed keys

```csharp
// Year + month
var monthly =
    from o in db.Orders
    group o by new { o.OrderDate.Year, o.OrderDate.Month } into g
    orderby g.Key.Year, g.Key.Month
    select new { g.Key.Year, g.Key.Month, Orders = g.Count() };

// Bucket / band (CASE WHEN in SQL)
var bands =
    from p in db.Products
    group p by p.ListPrice < 100 ? "Budget"
             : p.ListPrice < 500 ? "Mid"
             : "Premium" into g
    select new { Band = g.Key, Products = g.Count(), AvgPrice = g.Average(x => x.ListPrice) };

// Method syntax
var monthly2 = db.Orders
    .GroupBy(o => new { o.OrderDate.Year, o.OrderDate.Month })
    .OrderBy(g => g.Key.Year).ThenBy(g => g.Key.Month)
    .Select(g => new { g.Key.Year, g.Key.Month, Orders = g.Count() });

var bands2 = db.Products
    .GroupBy(p => p.ListPrice < 100 ? "Budget"
                : p.ListPrice < 500 ? "Mid"
                : "Premium")
    .Select(g => new { Band = g.Key, Products = g.Count(), AvgPrice = g.Average(x => x.ListPrice) });
```

##### 13.5 Find duplicates

```csharp
var duplicates = db.Customers
    .GroupBy(c => new { c.Name, c.City })
    .Where(g => g.Count() > 1)
    .Select(g => new { g.Key.Name, g.Key.City, Copies = g.Count() });

// Query syntax
var duplicatesQ =
    from c in db.Customers
    group c by new { c.Name, c.City } into g
    where g.Count() > 1
    select new { g.Key.Name, g.Key.City, Copies = g.Count() };
```

##### 13.6 The three `GroupBy` overload shapes (key / element / result selector)

```csharp
// (a) key selector only: each group holds whole Employee rows
var a  = db.Employees.GroupBy(e => e.DepartmentId)
                     .Select(g => new { g.Key, Total = g.Sum(e => e.Salary) });
var aQ = from e in db.Employees
         group e by e.DepartmentId into g
         select new { g.Key, Total = g.Sum(e => e.Salary) };

// (b) key + element selector: each group holds only salaries, so Sum()/Max() take no lambda
var b  = db.Employees.GroupBy(e => e.DepartmentId, e => e.Salary)
                     .Select(g => new { g.Key, Total = g.Sum(), Top = g.Max() });
var bQ = from e in db.Employees
         group e.Salary by e.DepartmentId into g           // "group ELEMENT by KEY"
         select new { g.Key, Total = g.Sum(), Top = g.Max() };

// (c) key + result selector: the Select is folded into the GroupBy call
//     (query syntax has no equivalent keyword; it is always group ... into g select ...)
var c  = db.Employees.GroupBy(
             e => e.DepartmentId,
             (key, rows) => new { DepartmentId = key, Headcount = rows.Count(), Payroll = rows.Sum(r => r.Salary) });
var cQ = from e in db.Employees
         group e by e.DepartmentId into g
         select new { DepartmentId = g.Key, Headcount = g.Count(), Payroll = g.Sum(r => r.Salary) };
```

Viva point: query syntax can end a group with `group … by …` (no `into`) and return the raw `IGrouping`s, but any further filtering, ordering, or aggregation needs `into g`, which starts a *query continuation*.

##### 13.7 Different kinds of grouping keys

```csharp
// By date part (time stripped) -> orders per day
var perDay  = db.Orders.GroupBy(o => o.OrderDate.Date)
                       .Select(g => new { Day = g.Key, Orders = g.Count() });
var perDayQ = from o in db.Orders
              group o by o.OrderDate.Date into g
              select new { Day = g.Key, Orders = g.Count() };

// By enum -> orders per status
var perStatus  = db.Orders.GroupBy(o => o.Status)
                          .Select(g => new { Status = g.Key, Orders = g.Count() });
var perStatusQ = from o in db.Orders
                 group o by o.Status into g
                 select new { Status = g.Key, Orders = g.Count() };

// By a boolean expression -> a two-way split (partition)
var split  = db.Employees.GroupBy(e => e.Salary >= 700)
                         .Select(g => new { HighEarner = g.Key, Count = g.Count() });
var splitQ = from e in db.Employees
             group e by e.Salary >= 700 into g
             select new { HighEarner = g.Key, Count = g.Count() };

// By a string function -> customers per first letter of the name
var byInitial  = db.Customers.GroupBy(c => c.Name.Substring(0, 1))
                             .Select(g => new { Initial = g.Key, Customers = g.Count() });
var byInitialQ = from c in db.Customers
                 group c by c.Name.Substring(0, 1) into g
                 select new { Initial = g.Key, Customers = g.Count() };

// By a constant -> ONE group = whole-table aggregates in a single row (no GROUP BY in SQL)
var overall  = db.OrderLines.GroupBy(_ => 1)
                            .Select(g => new { Lines = g.Count(), Units = g.Sum(l => l.Quantity), Revenue = g.Sum(l => l.Quantity * l.UnitPrice) });
var overallQ = from l in db.OrderLines
               group l by 1 into g
               select new { Lines = g.Count(), Units = g.Sum(l => l.Quantity), Revenue = g.Sum(l => l.Quantity * l.UnitPrice) };
```

Notes: `Date`, `Year`, `Month`, and `Substring` translate to `CONVERT(date, …)`, `DATEPART`, and `SUBSTRING`. A `null` key forms its own group (SQL `GROUP BY` does the same). The constant key is the way to get several aggregates in one round trip: `GroupBy(_ => 1)` is cheaper than five separate `SumAsync`/`CountAsync` calls. When the table might be empty it returns **no row** (not a row of zeros), so use `FirstOrDefaultAsync()` and default the result.

##### 13.8 Every common aggregate in one group

```csharp
var stats =
    from e in db.Employees
    where e.DepartmentId != null
    group e by e.DepartmentId into g
    select new
    {
        DepartmentId = g.Key,
        Headcount    = g.Count(),                          // COUNT(*)
        BigCount     = g.LongCount(),                      // COUNT_BIG(*)
        HighEarners  = g.Count(x => x.Salary >= 700),      // COUNT(CASE WHEN ...)
        TotalSalary  = g.Sum(x => x.Salary),
        AvgSalary    = g.Average(x => x.Salary),
        LowSalary    = g.Min(x => x.Salary),
        TopSalary    = g.Max(x => x.Salary),
        Managers     = g.Select(x => x.ManagerId).Distinct().Count(),   // COUNT(DISTINCT ...)
        HasUnmanaged = g.Any(x => x.ManagerId == null),    // EXISTS / CASE WHEN
        AllPaid      = g.All(x => x.Salary > 0)            // NOT EXISTS (... <= 0)
    };

var statsM = db.Employees
    .Where(e => e.DepartmentId != null)
    .GroupBy(e => e.DepartmentId)
    .Select(g => new
    {
        DepartmentId = g.Key,
        Headcount    = g.Count(),
        BigCount     = g.LongCount(),
        HighEarners  = g.Count(x => x.Salary >= 700),
        TotalSalary  = g.Sum(x => x.Salary),
        AvgSalary    = g.Average(x => x.Salary),
        LowSalary    = g.Min(x => x.Salary),
        TopSalary    = g.Max(x => x.Salary),
        Managers     = g.Select(x => x.ManagerId).Distinct().Count(),
        HasUnmanaged = g.Any(x => x.ManagerId == null),
        AllPaid      = g.All(x => x.Salary > 0)
    });
```

`Average` of an `int` column returns `double`, and of `decimal` returns `decimal`. `Sum`/`Average` on an empty or all-null group yield `NULL` in SQL; cast to a nullable type and use `?? 0` if the value is mapped to a non-nullable property.

##### 13.9 `HAVING` on different aggregates

```csharp
// Departments whose payroll exceeds 1,000 AND whose average salary is under 800
var heavy =
    from e in db.Employees
    group e by e.DepartmentId into g
    where g.Sum(x => x.Salary) > 1_000 && g.Average(x => x.Salary) < 800
    select new { DepartmentId = g.Key, Payroll = g.Sum(x => x.Salary) };

var heavyM = db.Employees
    .GroupBy(e => e.DepartmentId)
    .Where(g => g.Sum(x => x.Salary) > 1_000 && g.Average(x => x.Salary) < 800)
    .Select(g => new { DepartmentId = g.Key, Payroll = g.Sum(x => x.Salary) });

// Customers who have at least one Cancelled order (HAVING with Any)
var troubled =
    from o in db.Orders
    group o by o.CustomerId into g
    where g.Any(x => x.Status == OrderStatus.Cancelled)
    select g.Key;

var troubledM = db.Orders
    .GroupBy(o => o.CustomerId)
    .Where(g => g.Any(x => x.Status == OrderStatus.Cancelled))
    .Select(g => g.Key);

// Groups with exactly one distinct value (e.g. customers who always ship to one city) -> Min == Max
var single =
    from o in db.Orders
    group o by o.CustomerId into g
    where g.Min(x => x.Customer.City) == g.Max(x => x.Customer.City)
    select g.Key;
```

```sql
SELECT DepartmentId, SUM(Salary)
FROM Employees
GROUP BY DepartmentId
HAVING SUM(Salary) > 1000 AND AVG(Salary) < 800;
```

##### 13.10 Ordering by an aggregate, then paging the groups

```csharp
// Page 2 (rows 11-20) of customers ranked by revenue
var page =
    (from l in db.OrderLines
     group l by l.Order.CustomerId into g
     orderby g.Sum(x => x.Quantity * x.UnitPrice) descending, g.Key      // g.Key = tie-breaker so paging is stable
     select new { CustomerId = g.Key, Revenue = g.Sum(x => x.Quantity * x.UnitPrice) })
    .Skip(10).Take(10);

var pageM = db.OrderLines
    .GroupBy(l => l.Order.CustomerId)
    .Select(g => new { CustomerId = g.Key, Revenue = g.Sum(x => x.Quantity * x.UnitPrice) })
    .OrderByDescending(x => x.Revenue).ThenBy(x => x.CustomerId)
    .Skip(10).Take(10);                                                  // OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY
```

`Skip`/`Take` without a deterministic `OrderBy` (including a tie-breaker) can repeat or drop groups between pages.

##### 13.11 Highest / lowest row per group (the whole row, not just the aggregate)

`g.Max(x => x.Salary)` returns only the number. To return the employee who earns it, order inside the group and take the first:

```csharp
// Highest-paid employee in each department
var topPaid =
    from e in db.Employees
    group e by e.DepartmentId into g
    select g.OrderByDescending(x => x.Salary).ThenBy(x => x.Id).First();

var topPaidM = db.Employees
    .GroupBy(e => e.DepartmentId)
    .Select(g => g.OrderByDescending(x => x.Salary).ThenBy(x => x.Id).First());

// Navigation alternative that avoids GroupBy entirely (and keeps departments with no staff)
var topPaid2 = db.Departments.Select(d => new
{
    d.Name,
    TopEarner = d.Employees.OrderByDescending(e => e.Salary).Select(e => e.Name).FirstOrDefault()
});
```

`ThenBy(x => x.Id)` makes "the first" deterministic when two people share the top salary. EF Core 11 is reported to translate `g.MaxBy(…)` / `g.MinBy(…)` per group as well; see [section 22](#linq-version-features).

##### 13.12 Group in memory after fetching: `GroupBy`, `ToLookup`, `ToDictionary`

When you need each group's *members* (not just aggregates), fetch the rows with one query and group on the client. `AsEnumerable()` marks the switch from SQL to LINQ to Objects:

```csharp
var rows = await db.Orders
    .Where(o => o.OrderDate.Year == 2025)
    .Select(o => new { o.CustomerId, o.Id, o.OrderDate, o.Status })
    .ToListAsync();                                     // SQL ends here

// GroupBy: deferred, each group keeps its members
var byCustomer =
    from o in rows
    group o by o.CustomerId into g
    select new { CustomerId = g.Key, Orders = g.OrderBy(x => x.OrderDate).ToList() };

var byCustomerM = rows
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Orders = g.OrderBy(x => x.OrderDate).ToList() });

// ToLookup: built immediately, O(1) lookup, a missing key returns an EMPTY sequence (not an exception)
ILookup<int, int> orderIdsByCustomer = rows.ToLookup(o => o.CustomerId, o => o.Id);
int[] mine = orderIdsByCustomer[42].ToArray();

// ToDictionary on grouped data: build the groups first (duplicate keys would throw)
Dictionary<int, int> countByCustomer = rows
    .GroupBy(o => o.CustomerId)
    .ToDictionary(g => g.Key, g => g.Count());

// string.Join per group has no SQL translation, so it must run in memory
var statusList =
    from o in rows
    group o.Status by o.CustomerId into g
    select new { CustomerId = g.Key, Statuses = string.Join(", ", g.Distinct()) };

var statusListM = rows
    .GroupBy(o => o.CustomerId, o => o.Status)
    .Select(g => new { CustomerId = g.Key, Statuses = string.Join(", ", g.Distinct()) });
```

Deferred vs immediate: `GroupBy` re-runs each time you enumerate it; `ToLookup`/`ToDictionary` snapshot once. In EF Core, a `GroupBy` whose groups are returned whole is evaluated on the client after fetching every row (see [section 19](#linq-group-in-sql-shape-in-memory)).

---

#### 14. Join then group (multiple tables)

##### 14.1 Revenue per category (3 tables)

```csharp
var q =
    from l in db.OrderLines
    join p in db.Products on l.ProductId equals p.Id
    join c in db.Categories on p.CategoryId equals c.Id
    group l by new { c.Id, c.Name } into g
    orderby g.Sum(x => x.Quantity * x.UnitPrice) descending
    select new
    {
        CategoryId = g.Key.Id,
        Category = g.Key.Name,
        Units = g.Sum(x => x.Quantity),
        Revenue = g.Sum(x => x.Quantity * x.UnitPrice)
    };

// Method syntax with explicit joins
var q2 = db.OrderLines
    .Join(db.Products,   l => l.ProductId,    p => p.Id, (l, p) => new { l, p })
    .Join(db.Categories, x => x.p.CategoryId, c => c.Id, (x, c) => new { x.l, c })
    .GroupBy(x => new { x.c.Id, x.c.Name })
    .Select(g => new
    {
        CategoryId = g.Key.Id,
        Category = g.Key.Name,
        Units = g.Sum(x => x.l.Quantity),
        Revenue = g.Sum(x => x.l.Quantity * x.l.UnitPrice)
    });

// Navigation version
var q3 = db.OrderLines
    .GroupBy(l => new { l.Product.Category.Id, l.Product.Category.Name })
    .Select(g => new { g.Key.Name, Revenue = g.Sum(l => l.Quantity * l.UnitPrice) });
```

Group by `Id` **and** `Name`: two categories can share a name, and in SQL every non-aggregated column in the `SELECT` must be in the `GROUP BY`.

##### 14.2 Five tables, three keys, `HAVING`, distinct counts

Delivered revenue by **customer city × category × year**, only groups above 1,000:

```csharp
var q =
    from c in db.Customers
    join o in db.Orders on c.Id equals o.CustomerId
    join l in db.OrderLines on o.Id equals l.OrderId
    join p in db.Products on l.ProductId equals p.Id
    join cat in db.Categories on p.CategoryId equals cat.Id
    where o.Status == OrderStatus.Delivered
    group new { l.Quantity, l.UnitPrice, OrderId = o.Id, CustomerId = c.Id }
        by new { c.City, Category = cat.Name, o.OrderDate.Year } into g
    let revenue = g.Sum(x => x.Quantity * x.UnitPrice)
    where revenue > 1_000                                   // HAVING
    orderby g.Key.Year, revenue descending
    select new
    {
        g.Key.City,
        g.Key.Category,
        g.Key.Year,
        Revenue = revenue,
        Units = g.Sum(x => x.Quantity),
        Orders = g.Select(x => x.OrderId).Distinct().Count(),       // COUNT(DISTINCT)
        Customers = g.Select(x => x.CustomerId).Distinct().Count()
    };
```

Method syntax with navigations:

```csharp
var q2 = db.OrderLines
    .Where(l => l.Order.Status == OrderStatus.Delivered)
    .GroupBy(l => new
    {
        l.Order.Customer.City,
        Category = l.Product.Category.Name,
        l.Order.OrderDate.Year
    })
    .Select(g => new
    {
        g.Key.City,
        g.Key.Category,
        g.Key.Year,
        Revenue = g.Sum(l => l.Quantity * l.UnitPrice),
        Orders = g.Select(l => l.OrderId).Distinct().Count()
    })
    .Where(x => x.Revenue > 1_000)
    .OrderBy(x => x.Year)
    .ThenByDescending(x => x.Revenue);
```

**Grain trap:** after joining `Orders` to `OrderLines`, `g.Count()` counts **lines**, not orders. Use `Distinct().Count()` on the order ID, or group at the order level. Similarly, `g.Sum(x => x.Order.Freight)` after this join adds the same order's freight once per line.

---

#### 15. Left join + group: include zero counts

Requirement: *every customer with their number of orders, including customers with none.*

```csharp
// TRAP: g.Count() counts the null row from DefaultIfEmpty, so a customer
// with no orders gets 1, not 0 (SQL: COUNT(*) instead of COUNT(o.Id)).
var wrong =
    from c in db.Customers
    join o in db.Orders on c.Id equals o.CustomerId into og
    from o in og.DefaultIfEmpty()
    group o by new { c.Id, c.Name } into g
    select new { g.Key.Name, Orders = g.Count() };

// Fix 1: count only non-null rows
var fixed1 =
    from c in db.Customers
    join o in db.Orders on c.Id equals o.CustomerId into og
    from o in og.DefaultIfEmpty()
    group o by new { c.Id, c.Name } into g
    select new { g.Key.Name, Orders = g.Count(o => o != null) };

// Fix 1, method syntax
var fixed1m = db.Customers
    .GroupJoin(db.Orders, c => c.Id, o => o.CustomerId, (c, og) => new { c, og })
    .SelectMany(x => x.og.DefaultIfEmpty(), (x, o) => new { x.c, o })
    .GroupBy(x => new { x.c.Id, x.c.Name }, x => x.o)     // second lambda = element selector
    .Select(g => new { g.Key.Name, Orders = g.Count(o => o != null) });

// Fix 2 (simplest): correlated subquery via navigation
var fixed2 = db.Customers.Select(c => new
{
    c.Name,
    Orders = c.Orders.Count(),
    Revenue = c.Orders.SelectMany(o => o.Lines).Sum(l => (decimal?)(l.Quantity * l.UnitPrice)) ?? 0
});

// Fix 2, query syntax
var fixed2q =
    from c in db.Customers
    select new
    {
        c.Name,
        Orders = c.Orders.Count(),
        Revenue = (from o in c.Orders
                   from l in o.Lines
                   select (decimal?)(l.Quantity * l.UnitPrice)).Sum() ?? 0
    };
```

`Sum` over an empty set is `NULL` in SQL, so cast to `decimal?` and coalesce with `?? 0`; otherwise EF can throw when it materializes null into a non-nullable `decimal`.

---

#### 16. Conditional aggregation (pivot-style columns)

```csharp
// Order count per status as columns
var q =
    from o in db.Orders
    group o by new { o.CustomerId, o.Customer.Name } into g
    select new
    {
        Customer = g.Key.Name,
        Total = g.Count(),
        Pending = g.Count(o => o.Status == OrderStatus.Pending),
        Shipped = g.Count(o => o.Status == OrderStatus.Shipped),
        Delivered = g.Count(o => o.Status == OrderStatus.Delivered),
        CancelRatePct = 100.0 * g.Count(o => o.Status == OrderStatus.Cancelled) / g.Count()
    };

// Revenue per category with one column per quarter for a given year
var byQuarter =
    from l in db.OrderLines
    where l.Order.OrderDate.Year == year
    group new { Month = l.Order.OrderDate.Month, Amount = l.Quantity * l.UnitPrice }
        by l.Product.Category.Name into g
    select new
    {
        Category = g.Key,
        Q1 = g.Sum(x => x.Month <= 3 ? x.Amount : 0),
        Q2 = g.Sum(x => x.Month >= 4 && x.Month <= 6 ? x.Amount : 0),
        Q3 = g.Sum(x => x.Month >= 7 && x.Month <= 9 ? x.Amount : 0),
        Q4 = g.Sum(x => x.Month >= 10 ? x.Amount : 0)
    };

// Method syntax (GroupBy overload with key selector + element selector)
var q2 = db.Orders
    .GroupBy(o => new { o.CustomerId, o.Customer.Name })
    .Select(g => new
    {
        Customer = g.Key.Name,
        Total = g.Count(),
        Pending = g.Count(o => o.Status == OrderStatus.Pending),
        Delivered = g.Count(o => o.Status == OrderStatus.Delivered)
    });

var byQuarter2 = db.OrderLines
    .Where(l => l.Order.OrderDate.Year == year)
    .GroupBy(
        l => l.Product.Category.Name,
        l => new { Month = l.Order.OrderDate.Month, Amount = l.Quantity * l.UnitPrice })
    .Select(g => new
    {
        Category = g.Key,
        Q1 = g.Sum(x => x.Month <= 3 ? x.Amount : 0),
        Q2 = g.Sum(x => x.Month >= 4 && x.Month <= 6 ? x.Amount : 0),
        Q3 = g.Sum(x => x.Month >= 7 && x.Month <= 9 ? x.Amount : 0),
        Q4 = g.Sum(x => x.Month >= 10 ? x.Amount : 0)
    });
```

These translate to `SUM(CASE WHEN … THEN … ELSE 0 END)`, the portable equivalent of SQL `PIVOT` (see [PIVOT and UNPIVOT](#sql-server-advanced-grouping-and-window-functions-md)). If an older provider rejects `Count(predicate)`, use `Sum(x => cond ? 1 : 0)`.

---

#### 17. Top N and first row per group

```csharp
// Top 3 products by revenue in each category
var top3 =
    from c in db.Categories
    select new
    {
        Category = c.Name,
        TopProducts = c.Products
            .Select(p => new
            {
                p.Name,
                Revenue = p.OrderLines.Sum(l => (decimal?)(l.Quantity * l.UnitPrice)) ?? 0
            })
            .OrderByDescending(x => x.Revenue)
            .Take(3)
            .ToList()
    };

// Same, method syntax
var top3m = db.Categories.Select(c => new
{
    Category = c.Name,
    TopProducts = c.Products
        .Select(p => new
        {
            p.Name,
            Revenue = p.OrderLines.Sum(l => (decimal?)(l.Quantity * l.UnitPrice)) ?? 0
        })
        .OrderByDescending(x => x.Revenue)
        .Take(3)
        .ToList()
});

// Latest order of each customer (EF Core 6+ translates with ROW_NUMBER())
var latest = db.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => g.OrderByDescending(o => o.OrderDate).First());   // exactly one row per customer, even on date ties

var latestQ =
    from o in db.Orders
    group o by o.CustomerId into g
    select g.OrderByDescending(o => o.OrderDate).First();

// Portable alternative: group to find the max, then join back
var lastDates =
    from o in db.Orders
    group o by o.CustomerId into g
    select new { CustomerId = g.Key, LastDate = g.Max(o => o.OrderDate) };

var latest2 =
    from o in db.Orders
    join x in lastDates
        on new { o.CustomerId, Date = o.OrderDate } equals new { x.CustomerId, Date = x.LastDate }
    select o;          // ties on the same date return more than one row

// Join-back, method syntax
var lastDates2 = db.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, LastDate = g.Max(o => o.OrderDate) });

var latest3 = db.Orders.Join(
    lastDates2,
    o => new { o.CustomerId, Date = o.OrderDate },
    x => new { x.CustomerId, Date = x.LastDate },
    (o, x) => o);
```

---

#### 18. Share of total (subquery inside the group)

```csharp
var q =
    from l in db.OrderLines
    group l by l.Product.Category.Name into g
    let revenue = g.Sum(x => x.Quantity * x.UnitPrice)
    select new
    {
        Category = g.Key,
        Revenue = revenue,
        SharePct = revenue * 100 / db.OrderLines.Sum(x => x.Quantity * x.UnitPrice)
    };

// Method syntax
var q2 = db.OrderLines
    .GroupBy(l => l.Product.Category.Name)
    .Select(g => new { Category = g.Key, Revenue = g.Sum(x => x.Quantity * x.UnitPrice) })
    .Select(x => new
    {
        x.Category,
        x.Revenue,
        SharePct = x.Revenue * 100 / db.OrderLines.Sum(l => l.Quantity * l.UnitPrice)
    });
```

The grand total runs as a scalar subquery (a `SELECT SUM(...)` embedded in the outer query, evaluated once). Alternatively, fetch the total in its own query first. Guard against dividing by zero when the table can be empty:

```csharp
// Bad: if OrderLines is empty the Sum is NULL/0 and the division fails or returns null
// Good: fetch the total first and skip the division when it is 0
var total = await db.OrderLines.SumAsync(l => (decimal?)(l.Quantity * l.UnitPrice)) ?? 0;
var shares = total == 0 ? [] : await db.OrderLines
    .GroupBy(l => l.Product.Category.Name)
    .Select(g => new { Category = g.Key, SharePct = g.Sum(x => x.Quantity * x.UnitPrice) * 100 / total })
    .ToListAsync();      // `total` is sent as a parameter, not a subquery
```

---

<a id="linq-group-in-sql-shape-in-memory"></a>

#### 19. Group in SQL, shape in memory

Nested groups (category → year → revenue) and window-style calculations (running total, rank) are not translated from ordinary LINQ. Aggregate **flat** in the database, then reshape the small result in memory:

```csharp
var flat = await db.OrderLines
    .GroupBy(l => new { Category = l.Product.Category.Name, l.Order.OrderDate.Year })
    .Select(g => new
    {
        g.Key.Category,
        g.Key.Year,
        Revenue = g.Sum(l => l.Quantity * l.UnitPrice)
    })
    .ToListAsync();                                    // SQL GROUP BY ends here

// Nested grouping
var nested = flat
    .GroupBy(x => x.Category)
    .Select(cg => new
    {
        Category = cg.Key,
        Total = cg.Sum(x => x.Revenue),
        Years = cg.OrderBy(x => x.Year).Select(x => new { x.Year, x.Revenue }).ToList()
    });

// Nested grouping, query expression syntax
var nestedQ =
    from x in flat
    group x by x.Category into cg
    select new
    {
        Category = cg.Key,
        Total = cg.Sum(x => x.Revenue),
        Years = (from y in cg orderby y.Year select new { y.Year, y.Revenue }).ToList()
    };

// Running total per category, plus rank by revenue
var withRunning = flat
    .GroupBy(x => x.Category)
    .SelectMany(cg =>
    {
        decimal running = 0;
        return cg.OrderBy(x => x.Year).Select(x => new
        {
            x.Category,
            x.Year,
            x.Revenue,
            RunningTotal = running += x.Revenue
        });
    });

var ranked = flat
    .OrderByDescending(x => x.Revenue)
    .Select((x, index) => new { Rank = index + 1, x.Category, x.Year, x.Revenue });

// Query syntax has no keyword for the index overload of Select, so rank needs method syntax,
// or a mix: number the rows after the ordered query expression
var rankedQ = (from x in flat orderby x.Revenue descending select x)
    .Select((x, index) => new { Rank = index + 1, x.Category, x.Year, x.Revenue });
```

For large data, use SQL window functions (`SUM() OVER`, `ROW_NUMBER()`) through a view or raw SQL instead; see [window functions](#sql-server-advanced-grouping-and-window-functions-md).

Related in-memory notes:
- `GroupBy` on `IEnumerable` is **deferred**; `ToLookup` builds the groups **immediately** and supports fast key lookup (`lookup[key]`).
- Returning raw groups from EF (`db.Orders.GroupBy(o => o.CustomerId).ToListAsync()`) is supported in newer versions only as the final operator. It fetches every row and groups during materialization, not with SQL `GROUP BY`.

---

#### 20. Cheat sheet

| Need | Query syntax | Method syntax | SQL |
|---|---|---|---|
| Inner join | `join b in B on a.K equals b.K` | `A.Join(B, a => a.K, b => b.K, (a, b) => …)` | `INNER JOIN` |
| Left join | `join … into g from b in g.DefaultIfEmpty()` | `GroupJoin` + `SelectMany(DefaultIfEmpty)`; .NET 10 `LeftJoin` | `LEFT JOIN` |
| Right join | swap sides of a left join | .NET 10 `RightJoin` | `RIGHT JOIN` |
| Full join | left join `Concat` right-only rows | same; EF Core 11 `FullJoin` (reported) | `FULL OUTER JOIN` |
| Cross join | `from a in A from b in B` | `A.SelectMany(a => B, …)` | `CROSS JOIN` |
| Non-equi join | `from b in B.Where(b => range…)` | `SelectMany` + `Where` | `JOIN … ON a.X BETWEEN …` |
| Semi / anti join | `where B.Any(…)` / `!B.Any(…)` | `Where(a => B.Any(…))` | `EXISTS` / `NOT EXISTS` |
| Composite key | `on new { a.X, a.Y } equals new { b.X, b.Y }` | anonymous key selectors | `ON a.X = b.X AND a.Y = b.Y` |
| Group | `group x by key into g` | `GroupBy(key)` | `GROUP BY` |
| Having | `where` after `into g` | `.Where(g => …)` after `GroupBy` | `HAVING` |
| Group, element selector | `group e.Salary by e.DeptId into g` | `GroupBy(e => e.DeptId, e => e.Salary)` | `GROUP BY DeptId` |
| Group, result selector | (always `group … into g select …`) | `GroupBy(key, (k, rows) => …)` | `GROUP BY` |
| Group by constant | `group x by 1 into g` | `GroupBy(_ => 1)` | aggregate with no `GROUP BY` |
| Row with max per group | `select g.OrderByDescending(…).First()` | same; EF Core 11 `g.MaxBy(…)` | `ROW_NUMBER()` / `TOP (1)` subquery |
| Distinct count | `g.Select(x => x.Id).Distinct().Count()` | same | `COUNT(DISTINCT Id)` |
| Conditional sum | `g.Sum(x => cond ? v : 0)` | same | `SUM(CASE WHEN …)` |
| Count per key (in memory) | `from x in xs group x by k into g select (g.Key, g.Count())` | .NET 9 `xs.CountBy(k)` | `COUNT(*) … GROUP BY` |
| Aggregate per key (in memory) | `group … into g select (g.Key, g.Sum(…))` | .NET 9 `xs.AggregateBy(k, seed, func)` | `SUM … GROUP BY` |

#### 21. Viva quick answers

- **Query syntax or method syntax?** They compile to the same method calls and generate the same SQL. Query syntax reads better for multi-table joins, `let`, and left joins, because the compiler carries every range variable for you. Method syntax is required for operators that have no keyword (`Count`, `Sum`, `Any`, `Take`, `Distinct`, `Concat`, `LeftJoin`), and the two styles are often mixed: `(from … select …).CountAsync()`.
- **Why `equals` and not `==` in `join`?** `join` is an equi-join with two separate key selectors: the left expression can see only the outer variable and the right only the inner one. `equals` makes that rule explicit.
- **Can `join` use `>` or `<`?** No. Use `from … in …Where(…)` (`SelectMany`) for ranges and inequalities.
- **What does `into` mean?** After `join`, it creates a group join. After `group` or `select`, it is a *query continuation*: the earlier range variables go out of scope, and you continue from the new variable.
- **How do you write a left join in LINQ?** `join … into g` + `from x in g.DefaultIfEmpty()`, or `LeftJoin` from .NET 10. Guard the right-side variable against null.
- **How do you write right and full joins?** Right: swap the sides of a left join (or `RightJoin` from .NET 10). Full: left join `Concat` the right-only rows.
- **Do null keys match?** In a SQL join, `NULL = NULL` is not true, and LINQ to Objects `Join` also skips null keys. `GroupBy`, however, puts all null keys into **one** group, in both SQL and LINQ.
- **`Count()` after a left join returns 1 for "no rows". Why?** `DefaultIfEmpty` yields one null element. Use `Count(x => x != null)` or a navigation `c.Orders.Count()`.
- **When are explicit joins needed, and when are navigations better?** Prefer navigations when the relationship is mapped: they are shorter, and EF picks the right inner/left joins. Use explicit `join` for unmapped relationships, composite or ad-hoc keys, joins to grouped subqueries, and when the interviewer asks for one.
- **Why do joins produce duplicates?** One-to-many fan-out. Use `Any()` for existence, aggregate before joining, or `Distinct()` as a last resort.
- **`Concat` vs `Union`?** `Concat` = `UNION ALL` (keeps duplicates, cheaper); `Union` = `UNION` (removes duplicates).
- **Why can't you use `?.` in an EF query?** Expression trees can't contain null-propagating operators (CS8072). Use a ternary null check.
- **Where should filtering go?** Filter rows **before** grouping (`WHERE`); filter on aggregates **after** grouping (`HAVING`). For a left join, put right-side filters **inside** the joined source, not in a later `where`.

---

<a id="linq-version-features"></a>

#### 22. What recent versions added (with query and lambda forms)

| Version | LINQ / EF Core addition | Query syntax keyword? |
|---|---|---|
| .NET 9 (C# 13) | `CountBy`, `AggregateBy`, `Index()` (LINQ to Objects) | No |
| .NET 10 / EF Core 10 | `LeftJoin`, `RightJoin`; `ExecuteUpdateAsync` with an ordinary lambda; named query filters | No |
| EF Core 11 (RC at time of writing) | `FullJoin`; per-group `MaxBy`/`MinBy` (reported, verify) | No |

None of these added a query-syntax keyword. Query expressions still cover only `from/where/select/group/into/orderby/join/let`; everything newer is method syntax, and you mix the two with parentheses: `(from … select …).CountBy(…)`.

##### 22.1 .NET 9: `CountBy`, `AggregateBy`, `Index` (LINQ to Objects)

These replace a `GroupBy` + `Select` when you only need one aggregate per key, without building the groups:

```csharp
record Sale(string Region, string Product, decimal Amount);
List<Sale> sales = [ new("EU", "A", 10), new("EU", "B", 20), new("US", "A", 5) ];

// CountBy: (key, count) pairs
var counts  = sales.CountBy(s => s.Region);                          // EU=2, US=1
var countsQ = from s in sales                                        // the query-syntax equivalent
              group s by s.Region into g
              select KeyValuePair.Create(g.Key, g.Count());

// AggregateBy: (key, running aggregate), with a seed and an accumulator
var totals  = sales.AggregateBy(s => s.Region, 0m, (sum, s) => sum + s.Amount);   // EU=30, US=5
var totalsQ = from s in sales
              group s by s.Region into g
              select KeyValuePair.Create(g.Key, g.Sum(s => s.Amount));

// AggregateBy with a seed per key (e.g. keep a list per key)
var names = sales.AggregateBy(
    s => s.Region,
    _ => new List<string>(),
    (list, s) => { list.Add(s.Product); return list; });

// Index: element + position, instead of Select((x, i) => (i, x))
foreach (var (i, sale) in sales.Index())
    Console.WriteLine($"{i}: {sale.Product}");
var indexed  = sales.Select((s, i) => (i, s));                       // pre-.NET 9 form
```

Use them on in-memory data. For `IQueryable` against SQL Server, I could not confirm EF Core translates `CountBy`/`AggregateBy`, so use `GroupBy(...).Select(g => new { g.Key, Count = g.Count() })` in database queries and check `ToQueryString()` before relying on them.

##### 22.2 .NET 10 / EF Core 10: `LeftJoin` and `RightJoin`

```csharp
// Lambda form (new operators)
var left  = db.Customers.LeftJoin(db.Orders, c => c.Id, o => o.CustomerId,
                (c, o) => new { c.Name, OrderId = (int?)o!.Id });
var right = db.Orders.RightJoin(db.Customers, o => o.CustomerId, c => c.Id,
                (o, c) => new { c.Name, OrderId = o != null ? (int?)o.Id : null });

// Query form: unchanged, the classic pattern still works and produces the same LEFT JOIN
var leftQ =
    from c in db.Customers
    join o in db.Orders on c.Id equals o.CustomerId into og
    from o in og.DefaultIfEmpty()
    select new { c.Name, OrderId = o != null ? (int?)o.Id : null };

// Left join + group (the "include zero counts" case) using LeftJoin
var perCustomer = db.Customers
    .LeftJoin(db.Orders, c => c.Id, o => o.CustomerId, (c, o) => new { c, o })
    .GroupBy(x => new { x.c.Id, x.c.Name })
    .Select(g => new { g.Key.Name, Orders = g.Count(x => x.o != null) });   // count only real orders
```

##### 22.3 EF Core 10: `ExecuteUpdateAsync` with an ordinary lambda, and named query filters

```csharp
// Before EF Core 10, a conditional update needed an expression-tree trick.
// Now the setters are plain statements, so ordinary `if` works:
bool rename = newName is not null;
await db.Products
    .Where(p => p.CategoryId == categoryId)
    .ExecuteUpdateAsync(s =>
    {
        s.SetProperty(p => p.ListPrice, p => p.ListPrice * 1.1m);
        if (rename) s.SetProperty(p => p.Name, newName!);
    });
// ExecuteUpdate/ExecuteDelete have no query-syntax form; end a query expression with a method call:
await (from p in db.Products where p.CategoryId == categoryId select p)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.ListPrice, p => p.ListPrice * 1.1m));

// Named query filters: several filters per entity, disabled individually
modelBuilder.Entity<Product>()
    .HasQueryFilter("SoftDelete", p => !p.IsDeleted)
    .HasQueryFilter("Tenant",     p => p.TenantId == tenantId);

var all      = db.Products.IgnoreQueryFilters(["SoftDelete"]);   // lambda form: keep the tenant filter only
var allQ     = from p in db.Products.IgnoreQueryFilters(["SoftDelete"]) select p;   // query form
```

##### 22.4 EF Core 11 (release candidate at the time of writing): `FullJoin`, per-group `MaxBy`/`MinBy`

Reported by a third-party article on the 11.0 RC (it requires .NET 11); confirm against the official docs and `ToQueryString()` before quoting it in an interview:

```csharp
// Best (longest) play per month: one SQL statement, a correlated TOP (1) subquery per group
var topPerMonth = db.Plays
    .GroupBy(p => p.PlayedAt.Month)
    .Select(g => new { Month = g.Key, TopTrack = g.MaxBy(p => p.Seconds)!.Track.Title });

// Same intent in query syntax, which works on EF Core 10 and earlier too
var topPerMonthQ =
    from p in db.Plays
    group p by p.PlayedAt.Month into g
    select new
    {
        Month = g.Key,
        TopTrack = g.OrderByDescending(p => p.Seconds).First().Track.Title
    };

// MaxBy / MinBy without GroupBy: the whole row with the largest or smallest key
var busiest = await db.Artists.MaxByAsync(a => a.Tracks.Count());
```

##### 22.5 C# 14 and expression trees (why upgrades sometimes break queries)

C# 14 added first-class `Span<T>` conversions, so `array.Contains(x)` inside an expression tree can bind to `MemoryExtensions.Contains` (a span method) instead of `Enumerable.Contains`. EF Core 10 handles this, but older EF Core versions on a newer compiler can fail to translate it. Workarounds: use a `List<T>`, call `.AsEnumerable()` on the array, or upgrade EF Core. Null-conditional assignment (`a?.b = c`) is also not allowed in an expression tree, the same restriction as `?.` (see the viva answers in section 21).

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

Terms: a **dirty read** sees another transaction's uncommitted change
(which may later roll back). A **non-repeatable read** reads the same row
twice and gets different values because someone committed an update in
between. A **phantom** is a re-run of the same `WHERE` query returning
extra (or fewer) rows because someone inserted/deleted matching rows.

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

Why `ToLower()` hurts: wrapping the column in a function means the
database must compute `LOWER(Name)` for every row, so it cannot seek the
index on `Name` and scans the whole table instead.

``` csharp
// Bad: LOWER(Name) = 'ali' -> index scan (every row evaluated)
db.Users.Where(u => u.Name.ToLower() == "ali");
// Good: on a case-insensitive column/collation (the SQL Server default),
// plain equality is already case-insensitive and can use the index
db.Users.Where(u => u.Name == "ali");
```

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

Table terms explained:

- **Context pooling vs connection pooling:** `AddDbContextPool` reuses and
  resets the C# `DbContext` objects (saves allocation, so do not keep
  per-request state in fields of a pooled context). Connection pooling
  (ADO.NET, on by default) reuses the underlying TCP/database connections.
  They are independent and both can be on.
- **`Update(entity)` vs marking properties:** `db.Update(order)` marks
  every column of the entity (and any reachable new/untracked graph) as
  modified, so the `UPDATE` sets all columns and can overwrite another
  user's change. Marking one property sends only that column:
  `db.Entry(order).Property(o => o.Status).IsModified = true;`
- **Query filter vs authorization:** a global filter like
  `HasQueryFilter(c => c.TenantId == tenantId)` is a convenience that
  `IgnoreQueryFilters()`, raw SQL, or bulk operations can bypass, so it
  is not a security boundary; still check permissions in code (or use
  database row-level security).
- **Concurrency token vs isolation level:** a token (`[Timestamp]`/
  `rowversion`) detects that someone changed *this row* since you read it,
  and `SaveChanges` throws `DbUpdateConcurrencyException`. An isolation
  level controls what your transaction can *see* and how long locks last.

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

What each pair isolates:

| Pair | What it shows |
|---|---|
| `TrackingRead` vs `NoTrackingRead` | Tracking stores a snapshot of every returned entity for change detection; `AsNoTracking` skips that, so it uses less CPU and memory on read-only queries. |
| `EntityGraph` vs `Projection` | The graph loads full `Order`, `OrderItem` and `Payment` rows (every column) just to count them; the projection fetches only the id and two counts. |
| `OffsetPagination` vs `KeysetPagination` | `Skip(50_000)` makes SQL Server walk past and discard 50,000 rows before returning 20; `Id > 50000` seeks straight to the start via the key index, so cost does not grow with page depth. |

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

Ensure:

- Data distribution is not uniform unless production is uniform (real data is skewed: one tenant may own 80% of rows, and that changes the plan).
- Indexes and statistics (the row-distribution summaries the optimizer uses to pick a plan) reflect the intended scenario.
- The database is warmed up before steady-state runs (first runs include cold cache and plan compilation).
- Each compared query returns equivalent logical results (otherwise you are comparing different work).
- Database metrics are captured alongside application benchmarks.

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
metrics. Record:

- Duration and CPU.
- Logical reads (8 KB pages read from memory/disk) and rows read.
- Actual vs. estimated row counts (a big gap means stale statistics and often a bad plan).
- Index seek (jump to matching rows) vs scan (read everything), and key lookups (extra trips to the clustered index for columns the index lacks).
- Sorts, spills (sort/hash overflowing memory into tempdb), blocking, and memory grants.

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

##### Checklist terms explained

- **N+1 queries:** one query loads N parents, then lazy loading or a loop fires one more query per parent (N extra round trips).
  ```csharp
  // Bad: 1 query for orders + 1 per order for Items
  foreach (var o in await db.Orders.ToListAsync()) Console.WriteLine(o.Items.Count);
  // Good: one query
  var counts = await db.Orders.Select(o => new { o.Id, N = o.Items.Count }).ToListAsync();
  ```
- **Split queries:** a single `Include` of two collections joins them and returns Items x Payments rows per order (cartesian explosion); `AsSplitQuery()` sends one query per collection instead.
- **Deterministic pagination:** the `OrderBy` must end in a unique column (e.g. `.OrderBy(o => o.CreatedAt).ThenBy(o => o.Id)`); otherwise rows with equal dates can appear on two pages or on none.
- **Parameter-sensitive plans (parameter sniffing):** SQL Server caches the plan built for the first parameter value; if `TenantId = 1` has 5 rows and `TenantId = 2` has 5 million, the cached plan suits one and is slow for the other.
- **Safe retries:** `EnableRetryOnFailure` re-runs a command after a transient error; a non-idempotent operation (e.g. charging a card, then losing the connection before the reply) can be executed twice unless guarded by an idempotency key or unique constraint.
- **Short-lived `DbContext`:** a context lives for one unit of work (one request); a long-lived one accumulates tracked entities (memory growth, stale data) and is not thread-safe.

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

Why it matters: `RANGE` treats rows with equal `ORDER BY` values as peers and includes all of them, so ties change the running total (and `RANGE` is also slower, spilling to disk). Assume Alice has two rows in month 2, amounts 150 and 50:

```sql
SUM(Amount) OVER (PARTITION BY SalesPerson ORDER BY SalesMonth)                    -- default RANGE
SUM(Amount) OVER (PARTITION BY SalesPerson ORDER BY SalesMonth ROWS UNBOUNDED PRECEDING) -- ROWS
```

| Month | Amount | Default (RANGE) | `ROWS` |
|---:|---:|---:|---:|
| 1 | 100 | 100 | 100 |
| 2 | 150 | 300 | 250 |
| 2 | 50 | 300 | 300 |

With `RANGE` both month-2 rows show 300 (peers are included together); with `ROWS` you get a true row-by-row running total. Always write `ROWS` for a running total.

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

The explicit full frame is important. The default frame ends at the current row, so without it `LAST_VALUE` just returns the current row's own value:

| SalesPerson | Month | Amount | Default frame | Full frame (above) |
|---|---:|---:|---:|---:|
| Alice | 1 | 100 | 100 | 200 |
| Alice | 2 | 150 | 150 | 200 |
| Alice | 3 | 200 | 200 | 200 |

#### 6. Interview reminders

- `GROUP BY` reduces rows; `PARTITION BY` inside `OVER` keeps rows.
- `ROLLUP` follows the listed hierarchy; `CUBE` generates all combinations; `GROUPING SETS` lets you choose.
- `PIVOT` converts values to columns; `UNPIVOT` converts columns to values.
- `RANK` leaves gaps after ties; `DENSE_RANK` does not; `ROW_NUMBER` assigns unique sequential numbers.
- Add a unique tie-breaker to window `ORDER BY` when deterministic row positioning matters.
- Window-frame clauses work with applicable aggregate/value functions, **not** with `LAG`/`LEAD`.

---

<a id="database-indexing-and-isolation"></a>

## Database Indexing and Transaction Isolation: An Engine-Agnostic Guide

*Source: "Indexing, Partitioning, Sharding & Transaction Isolation — A Database-Agnostic System Design Guide" (Parts 1 and 4). Partitioning, sharding, quorums, and distributed transactions are in [08 — Partitioning, Sharding, and Distributed Data](08-system-design-and-distributed-systems.md#partitioning-sharding-distributed-data). SQL Server specifics (clustered vs nonclustered, execution plans) are in the Questpond SQL section at the end of this guide; EF Core isolation and concurrency are in section 4.4 of the EF Core guide above.*

### The mental model

Every scaling topic answers: *"there is too much data, load, or concurrency for X to handle alone — what do we split and what do we trade away?"*

```text
X = one query        → INDEXING      (do less work per query)
X = one table/store  → PARTITIONING  (do less work per logical dataset)
X = one node         → SHARDING      (do less work per node)
X = one transaction  → ISOLATION     (do less coordination between concurrent operations)
```

**Escalation order:** right index → query/schema fix → cache → read replica → partition → shard. Sharding is where you *arrive*, not where you start.

The **RUM conjecture** explains why no index structure wins universally: a design can optimize at most two of **R**ead, **U**pdate (write), and **M**emory (space) amplification; improving one costs another.

### What an index is

A secondary data structure mapping a search key to a row location, maintained automatically. Every index trades **faster matching reads** for **slower writes, extra storage, and extra I/O**. The right question is not "should this be indexed?" but "is this access pattern hot enough, and the read/write ratio favorable enough, to pay on every write?"

### Index structure families

| Family | Optimized for | How it works | Used by |
|---|---|---|---|
| **B-Tree** | Reads, equality, ranges, `ORDER BY … LIMIT` | Shallow high-fanout tree, sorted linked leaves; writes modify pages in place (random I/O, page splits) | SQL Server, PostgreSQL, MySQL/InnoDB, Oracle, MongoDB defaults |
| **LSM-Tree** | Write throughput | Writes go to an in-memory memtable, flush to immutable sorted files (SSTables), background compaction merges them; Bloom filters skip files on reads | Cassandra, ScyllaDB, HBase, RocksDB |
| **Hash** | Pure equality, O(1) | Key hash → bucket; no ordering, no ranges | Redis/Memcached, partition-key lookup in DynamoDB/Cassandra |
| **Inverted** | Full-text, "contains" | token → list of documents/rows | Elasticsearch/Lucene, PostgreSQL GIN |
| **Bitmap** | Low-cardinality analytics | One bit per row per value; filters become bitwise AND/OR | Warehouses (Oracle bitmap) |
| **R-Tree / GiST** | Spatial, nearest-neighbor | Bounding-box hierarchy | PostGIS, spatial indexes |
| **Block-range / zone map** | Huge naturally ordered data | Min/max per block, tiny footprint | PostgreSQL BRIN, Snowflake, Redshift |

B-Trees favor reads and ranges; LSM-Trees favor appends and pay with read amplification and **compaction write amplification** (the same data is rewritten as levels merge).

### Composite index design method

For a B-Tree-style index in any engine:

1. List the columns used with **equality** (`=`).
2. Pick the **one** column used for a range (`<`, `>`, `BETWEEN`) or `ORDER BY`.
3. Index key = `(equality columns…, then the single range/sort column)`.
4. Columns only needed in the result go in as covering/`INCLUDE` columns.

```sql
-- WHERE tenant_id = ? AND status = ? AND created_at > ?
-- ORDER BY created_at DESC  LIMIT 50   SELECT id, total
CREATE INDEX IX_Orders_Tenant_Status_Created
  ON Orders (TenantId, Status, CreatedAt DESC)
  INCLUDE (Total);                     -- SQL Server / PostgreSQL syntax
```

Why order matters, using sample index entries sorted by `(TenantId, Status, CreatedAt)`:

| TenantId | Status | CreatedAt |
|---:|---:|---|
| 1 | 0 | Jan 05 |
| 1 | 1 | Jan 02 |
| 1 | 1 | Jan 09 |
| 2 | 1 | Jan 01 |

`WHERE TenantId = 1 AND Status = 1 AND CreatedAt > Jan 03` seeks to the (1, 1) slice, which is already in date order, and reads from Jan 09 onward. `WHERE Status = 1` alone cannot seek: Status-1 rows are scattered between tenants.

Equality columns narrow the search to a contiguous slice; the trailing column is already sorted within that slice, so the engine seeks to the start and reads forward without an in-memory sort. Only **one** range/sort predicate can be absorbed; a second range degrades to a post-filter.

### Column order is load-bearing for key columns

**Leftmost-prefix rule.** An index on `(a, b, c)` is sorted by `a`, then `b` within `a`, then `c` within `(a, b)`. It can *seek* on `a`, `(a, b)`, or `(a, b, c)` but not on `b`, `c`, or `(b, c)` alone.

```text
Q1: WHERE a = ? AND b = ?     → served by (a, b, c)
Q2: WHERE b = ? AND c = ?     → NOT served: no leading a. Needs an index starting with b.
```

Among equality-only columns any order satisfies a full-equality query, but choose deliberately:

1. **Prefix reusability** — put the column most often queried alone first (`(tenant_id, status)` also serves `WHERE tenant_id = ?`; `(status, tenant_id)` does not).
2. **Selectivity** — a highly selective leading column narrows fastest and improves estimates (a performance concern, not correctness).
3. **Non-negotiable:** the range/sort column goes **last**. Columns after a range predicate lose their sorted grouping.

**Sort direction is part of order.** `ORDER BY created_at DESC, id ASC` needs an index declared `(created_at DESC, id ASC)` or the engine still sorts.

**INCLUDE columns are payload.** They are never used to seek, filter, or sort, so their order is irrelevant; choose them carefully (each adds storage and write cost) but do not agonize over sequence.

### Covering, partial, and filtered indexes

- **Covering index:** contains every column the query needs, so the base row is never fetched (SQL Server `INCLUDE`, PostgreSQL index-only scans). Avoids key lookups.
- **Partial / filtered index:** indexes only rows matching a predicate (`WHERE status = 'active'`) — a compact index over a hot subset without physically splitting the table.

### The hidden cost: write amplification

```text
2,000 writes/sec × 11 indexes ≈ up to 22,000 index-entry writes/sec
```

More random page writes and lock contention in B-Tree engines; more entries to flush and compact in LSM engines; more replicated index traffic in distributed stores. Indexes accumulate one at a time until write latency degrades for reasons no single index explains.

**Checklist before adding an index:**

1. Is the access pattern hot enough to matter? Measure.
2. Does an existing index serve it via a shared prefix?
3. Composite, partial/filtered, or covering — which shape matches the query?
4. What is the write rate, and what does one more index cost per write?
5. Can it be built online/concurrently without blocking production?
6. After release, check usage stats and drop unused indexes.

### Transaction isolation: anomalies

| Phenomenon | Definition | Example |
|---|---|---|
| **Dirty read** | Reading another transaction's uncommitted change | Balance update later rolled back |
| **Non-repeatable read** | Re-reading the same row gives a different value | Price changed by a commit between two reads |
| **Phantom read** | Re-running a query returns a different *set* of rows | `COUNT` goes 10 → 12 within one transaction |
| **Lost update** | Two read-modify-write transactions; one overwrites the other | Two "read balance, subtract 100, write" operations race |
| **Write skew** | Two transactions read overlapping data, write *different* rows, and break a cross-row invariant | Two on-call staff each see the other covering and both go off-call |

### ANSI isolation levels

| Level | Dirty read | Non-repeatable | Phantom | Write skew |
|---|:---:|:---:|:---:|:---:|
| Read Uncommitted | possible | possible | possible | possible |
| Read Committed | prevented | possible | possible | possible |
| Repeatable Read | prevented | prevented | **possible** (the standard allows it; PostgreSQL and InnoDB prevent it in practice, SQL Server does not) | possible |
| Serializable | prevented | prevented | prevented | prevented |

That table is the **specification**; engines deliver different things under the same name:

| Engine | Default | Divergence |
|---|---|---|
| SQL Server | Read Committed | Adds non-standard **Snapshot** isolation (MVCC); `READ_COMMITTED_SNAPSHOT` makes Read Committed use row versioning |
| PostgreSQL | Read Committed | Repeatable Read is snapshot isolation (no phantoms, but write skew possible); Serializable uses SSI (snapshot + dependency-cycle detection) |
| MySQL/InnoDB | Repeatable Read | Gap locks also prevent phantoms for locking reads |
| Oracle | Read Committed | No Read Uncommitted; Serializable is multiversion-based |
| Many NoSQL/distributed stores | varies | Use consistency models instead of ANSI levels |

**Never assume "Repeatable Read" or "Serializable" means the same guarantee across products; check the engine's documentation.**

### How isolation is implemented

- **Pessimistic locking (2PL):** acquire locks before reading/writing and block conflicts. Simple to reason about; lower concurrency; deadlocks possible.
- **MVCC:** keep multiple row versions so readers see a snapshot without blocking writers (and vice versa). High read concurrency; old versions must be garbage-collected (PostgreSQL `VACUUM`, SQL Server version store/tempdb).
- **Optimistic concurrency control (OCC):** read without locks; at commit check that nothing changed (version/`rowversion` column) and retry on conflict. Great when conflicts are rare, wasteful when frequent. This is EF Core's `[ConcurrencyCheck]`/`rowversion` model.

### Beyond ANSI: distributed consistency models

Once data is replicated or sharded, isolation is not the whole story; you also need a *consistency model* for what a reader sees relative to writes elsewhere.

| Model | Guarantee | Cost | Examples |
|---|---|---|---|
| **Linearizability / strict serializability** | Reads always reflect the latest committed write, globally | Highest coordination (consensus per op) | Spanner, CockroachDB, etcd |
| **Causal consistency** | If B depended on A, every replica shows A before B | Much cheaper; avoids "effect before cause" | Some multi-region session stores |
| **Eventual consistency** | Replicas converge if writes stop; no ordering in between | Cheapest, most available | DNS, Cassandra at low consistency, DynamoDB eventual reads |
| **Tunable** | Chosen per operation via quorum settings (R/W/N) | Flexible | Cassandra, DynamoDB, MongoDB read/write concerns |

CAP says that under a network partition you choose availability or consistency; PACELC adds that *even without a partition* you trade latency for consistency on every operation.

### What strong isolation costs

Published PostgreSQL SSI results: roughly 5–20% throughput overhead versus plain snapshot isolation, and meaningfully better than classic two-phase locking on read-heavy workloads. The real cost is **aborts**: the strongest levels can abort a transaction that looks fine in isolation because of a concurrent conflict.

> **Any code path run at the strongest level must be written to retry the *entire* operation from the start** on a serialization failure (SQLSTATE `40001`, SQL Server error 1205 for deadlock victims, or an OCC version mismatch). In EF Core, wrap the whole unit of work in an execution strategy, not just the failing statement.

### Choosing a level

```text
Does correctness depend on an invariant spanning MULTIPLE records or operations?
 ├─ YES → Serializable, or explicit locking on exactly the rows the invariant depends on
 └─ NO (single-record read-modify-write)
       ├─ Prefer ONE atomic conditional write over read-then-write:
       │     UPDATE Accounts SET Balance = Balance - 100
       │     WHERE Id = @id AND Balance >= 100
       │   (or compare-and-swap / conditional put in a KV store)
       └─ Otherwise Read Committed is fine for most CRUD and reporting
```

| System | Typical choice | Why |
|---|---|---|
| Ledger / double-entry | Serializable, or explicit locks in a fixed order | Debits = credits across records |
| Seat / room / inventory booking | Serializable, or an exclusion/unique constraint | Write skew is exactly "two people book the same slot" |
| Job queue | Default isolation + atomic claim (conditional update, `SKIP LOCKED`/`READPAST`) | Need is "one worker per job," not a cross-row invariant |
| Analytics / reporting | Read Committed or eventual reads | Slight staleness acceptable |
| General CRUD API | Read Committed (or snapshot) | Matches framework assumptions |
| Multi-region app | Causal/eventual for most reads; strong only where needed ("did my write land") | Global linearizability is rarely needed |

Always **lock multiple rows in the same order everywhere** (for example, ascending key) to avoid deadlocks.

### Isolation is harder once you shard

A single node gives you its strongest isolation "for free" (one log, one snapshot source, one lock manager). Across shards each node has its own snapshot, log, and clock, so no one node sees a conflict cycle spanning shards. You need 2PC for atomicity plus a global ordering mechanism (synchronized/hybrid clocks, as distributed-SQL platforms do) — **or choose the shard key so cross-shard transactions are rare to never needed.** The cheapest strong guarantee is a good shard key (see [08](08-system-design-and-distributed-systems.md#partitioning-sharding-distributed-data)).

### Interview Q&A

**Q: Index `(a, b, c)` — which queries can it serve?**
Seeks on `a`, `(a, b)`, `(a, b, c)`. Filtering only on `b` or `c` cannot seek. Equality columns first, one range/sort column last.

**Q: Why not index every column?**
Every index multiplies write work and storage; unused indexes cost writes for no read benefit. Measure access patterns, share prefixes, and drop unused indexes.

**Q: B-Tree vs LSM-Tree?**
B-Tree: read-heavy and range workloads, in-place page updates. LSM: write-heavy/append workloads, sequential writes, paid for with read amplification and compaction.

**Q: What is write skew and which levels prevent it?**
Two transactions read overlapping data and write different rows, breaking a cross-row invariant. Only Serializable (or explicit locking/constraints) prevents it; snapshot/Repeatable Read alone does not.

**Q: How do you avoid a lost update?**
A single atomic conditional `UPDATE`, optimistic concurrency with a version column, or locking reads — not read-then-write in application code.

**Q: What must application code do at Serializable?**
Retry the whole transaction on serialization failure.

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
**Answer:** Normalization organizes relational data to reduce redundancy and update anomalies (e.g. a customer's city stored on every order row, so changing it means editing many rows and risking inconsistency). Common forms: 1NF (atomic values/no repeating groups), 2NF (1NF and no partial dependency on part of a composite key), 3NF (2NF and no inappropriate transitive dependency on a key), and BCNF (every determinant is a candidate key).

Quick examples: a `Phones = '111, 222'` column breaks 1NF (split into rows). In `OrderLine(OrderId, ProductId, ProductName, Qty)` with key `(OrderId, ProductId)`, `ProductName` depends only on `ProductId`, a partial dependency that breaks 2NF (move it to `Product`). In `Employee(Id, DeptId, DeptName)`, `DeptName` depends on `DeptId`, not directly on `Id`, a transitive dependency that breaks 3NF (move it to `Department`).

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
**Answer:** Both assign equal ranks to ties within a partition. `RANK` leaves gaps after tied positions (1, 1, 3); `DENSE_RANK` does not (1, 1, 2). For salaries 90, 90, 80: `RANK` = 1, 1, 3; `DENSE_RANK` = 1, 1, 2; `ROW_NUMBER` = 1, 2, 3.

###### 20. What is a partition? (PDF p. 12)
**Answer:** In a window function, `PARTITION BY` divides rows into independent groups over which the function is evaluated. Database table partitioning is a separate physical-storage feature; clarify which meaning is intended.

###### 21. Ties (PDF p. 12)
**Answer:** Ties occur when rows have equal values under the ranking/order expression. `RANK`/`DENSE_RANK` assign equal ranks; `ROW_NUMBER` still assigns distinct numbers. Add a tie-breaker only when a deterministic ordering is required.

###### 22. What are magic tables? (PDF p. 12)
**Answer:** In SQL Server trigger terminology, `inserted` and `deleted` are logical transition tables containing affected rows—new values and old values respectively, depending on operation. Triggers must handle multiple rows rather than assuming a single row. For `UPDATE`, `deleted` holds the old values and `inserted` the new; for `INSERT` only `inserted` has rows; for `DELETE` only `deleted`. Bad: `SELECT @id = Id FROM inserted` (reads one arbitrary row of a multi-row statement). Good: `INSERT INTO Audit SELECT Id, GETDATE() FROM inserted;` (handles every row).

###### 23. What is a deadlock? (PDF p. 12)
**Answer:** A deadlock occurs when concurrent transactions wait in a cycle for resources held by each other; the engine selects a victim and rolls it back. Reduce risk with consistent access order, short transactions, suitable indexes/isolation, and retry handling for deadlock victims. Example: T1 updates row A then wants B; T2 updates row B then wants A; each waits for the other forever, so SQL Server kills one (error 1205). Fix: make both update A then B.

###### 24. What is `WITH (NOLOCK)`? (PDF p. 12)
**Answer:** In SQL Server, `NOLOCK` is equivalent to `READ UNCOMMITTED` for a table reference. It can return dirty, missing, or duplicated rows and does not eliminate all blocking; use row-versioning isolation (such as RCSI, Read Committed Snapshot Isolation, where readers see the last committed version instead of waiting) or a different design when appropriate rather than treating it as a harmless performance switch. Example: a report using `SELECT SUM(Balance) FROM Accounts WITH (NOLOCK)` while a transfer is mid-flight can count the money as debited from one account and not yet credited to the other (a dirty read), and can miss or double-count rows during page splits.

###### 25. What is an isolation level? (PDF p. 12)
**Answer:** An isolation level controls how concurrent transactions may observe each other's changes and which anomalies are possible (see the [engine-agnostic anomaly and isolation tables](#database-indexing-and-isolation)). SQL Server options include Read Uncommitted, Read Committed, Repeatable Read, Snapshot, and Serializable; stronger isolation can reduce anomalies but increase contention or version-store use.

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

Assumes a table `Employee(EmpName, MobileNo)` (the source PDF's data is not reproduced here). Tiny illustration: rows `(A, 111)`, `(B, 111)`, `(C, 222)` return `A, B`, because the inner query finds numbers appearing more than once (`111`) and the outer query lists everyone holding them.

Return names whose number occurs more than once:

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

Count orders for each consumer in the inclusive date interval, then group consumers by that count. `LEFT JOIN` keeps consumers with no matching orders. The half-open date range (`>= Apr 1` and `< Jul 1`) also works when `DeliverDate` is a timestamp, because `<= '2023-06-30'` would drop orders made after midnight on June 30. Assumes `tblConsumer(Id)` and an orders table (confusingly named `tblCustomer` in the source) with `OrderId`, `ConsumerId`, `DeliverDate`. The date test sits in the `ON` clause, not `WHERE`: a `WHERE` on `o.DeliverDate` would discard the NULL rows of consumers with no orders and silently remove the zero-count group. `COUNT(o.OrderId)` (not `COUNT(*)`) counts 0 for those NULL rows.

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

Example: salaries 90, 90, 80, 70. With `@N = 2`, `DENSE_RANK` returns 80 (distinct values: 90, 80, 70); `ROW_NUMBER` would return 90 (the 2nd row). The CTE is needed because a window function cannot appear directly in `WHERE`.
