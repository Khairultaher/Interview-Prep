# Senior .NET Viva: LINQ and EF Core — Tough Questions & Answers

> A consolidated revision guide for senior .NET developer interviews. Examples use modern C# and EF Core. Provider behavior and SQL translation can vary, so inspect generated SQL and test against your actual database provider.

## Contents

1. [Change tracking](#1-how-does-ef-core-track-entity-changes)
2. [Eager, explicit, and lazy loading](#2-eager-explicit-and-lazy-loading)
3. [`Remove` versus `ExecuteDelete`](#3-dbsetremove-versus-executedelete)
4. [Shadow properties](#4-what-are-shadow-properties)
5. [Multi-tenancy](#5-how-would-you-handle-multi-tenancy)
6. [`AsNoTracking`](#6-how-does-asnotracking-affect-performance)
7. [Bulk insert and update](#7-how-do-you-optimize-bulk-operations)
8. [TPH, TPT, and TPC inheritance](#8-what-are-tph-tpt-and-tpc)
9. [Optimistic concurrency](#9-how-do-you-handle-concurrency-conflicts)
10. [Deferred versus immediate LINQ execution](#10-deferred-versus-immediate-execution)
11. [Senior-level follow-up topics](#11-senior-level-follow-up-topics)

---

## 1. How does EF Core track entity changes?

EF Core's `ChangeTracker` maintains entity state and, for snapshot tracking, original property values. When `SaveChanges()` runs, EF Core detects changes, creates database commands, executes them (normally in a transaction where supported), and updates tracked states after success.

### Entity states

| State | Meaning |
|---|---|
| `Detached` | Not tracked by this context |
| `Unchanged` | Tracked; no detected changes |
| `Added` | Insert on save |
| `Modified` | Update on save |
| `Deleted` | Delete on save |

### Example

```csharp
var student = await db.Students.SingleAsync(s => s.Id == 1);

student.Name = "Amina";

// With snapshot tracking, automatic detection usually happens
// when Entries(), SaveChanges(), or another detection-triggering API runs.
db.ChangeTracker.DetectChanges();

Console.WriteLine(db.Entry(student).State); // Modified

await db.SaveChangesAsync();
Console.WriteLine(db.Entry(student).State); // Unchanged after successful save
```

### How does it know what changed?

- **Snapshot tracking:** EF Core records original values and compares them with current values during change detection.
- **Notification tracking:** Entities that implement suitable change/property notifications can notify EF Core about changes.
- **Change-tracking proxies:** Proxies can intercept property changes, subject to configuration and entity requirements.
- **Relationship fix-up:** EF Core synchronizes foreign keys and navigation properties for tracked entities as relationships change.

`SaveChanges()` normally calls change detection automatically. Disabling `AutoDetectChangesEnabled` can help in carefully measured bulk-like tracked workflows, but then the application must ensure changes are detected when needed.

**Viva distinction:** A POCO property assignment does not necessarily change `EntityEntry.State` to `Modified` immediately under snapshot tracking. Detection may not occur until EF Core scans the entity.

---

## 2. Eager, explicit, and lazy loading

These strategies control when related entities are retrieved.

### Eager loading

Fetch related data as part of the query.

```csharp
var courses = await db.Courses
    .Include(c => c.Students)
    .ThenInclude(s => s.Address)
    .ToListAsync();
```

Use it when the required object graph is known in advance. Multiple collection includes can cause row multiplication; consider `AsSplitQuery()` and compare its extra round trips with the cost of a single large join.

### Explicit loading

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

### Lazy loading

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

---

## 3. `DbSet.Remove()` versus `ExecuteDelete()`

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

---

## 4. What are shadow properties?

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

## 5. How would you handle multi-tenancy?

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

## 6. How does `AsNoTracking()` affect performance?

By default, entity queries are tracked. Tracking adds bookkeeping and supports identity resolution and change persistence. For read-only entity queries, `AsNoTracking()` often reduces memory and CPU overhead.

```csharp
var students = await db.Students
    .AsNoTracking()
    .Where(s => s.IsActive)
    .ToListAsync();
```

For a read-only result that benefits from returning one CLR instance per database identity within the result, consider:

```csharp
var students = await db.Students
    .AsNoTrackingWithIdentityResolution()
    .Include(s => s.Courses)
    .ToListAsync();
```

| Mode | Tracks in context? | Identity resolution? |
|---|---:|---:|
| Tracking query | Yes | Yes, using context identity map |
| `AsNoTracking()` | No | No |
| `AsNoTrackingWithIdentityResolution()` | No | Yes, using a temporary tracker for query materialization |

**Caution:** `AsNoTracking()` does not make every query faster. Projection to DTOs, database indexes, query shape, payload size, and round trips can matter more. A projection containing no entity instances generally has nothing to track.

---

## 7. How do you optimize bulk operations?

### Inserts

```csharp
db.Students.AddRange(students);
await db.SaveChangesAsync();
```

`AddRange()` is convenient, but it does not guarantee a single SQL statement or a provider-native bulk copy. For very large imports, evaluate batching, bounded context lifetimes, provider bulk-copy features, or a suitable bulk library. Benchmark with realistic data and constraints.

### Set-based updates

EF Core 7+:

```csharp
var updated = await db.Students
    .Where(s => s.IsActive && s.GraduationYear < 2020)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(s => s.IsActive, false)
        .SetProperty(s => s.UpdatedAt, DateTime.UtcNow));
```

This translates to a set-based database update and bypasses tracked entity updates, `SaveChanges()`, and normal per-entity change tracking. Do not assume `SaveChanges` overrides or `SaveChanges` interceptors will run for it. Command interceptors may observe the database command.

### Practical optimization checklist

- Avoid loading entities if a set-based statement can express the operation.
- Use bounded batches for very large workloads.
- Avoid one context tracking hundreds of thousands of entities.
- Use transactions when multiple operations must be atomic.
- Check foreign keys, indexes, triggers, constraints, and concurrency requirements.
- Measure generated SQL and execution plans.
- Avoid assuming `AddRange()` is equivalent to a dedicated bulk insert.

---

## 8. What are TPH, TPT, and TPC?

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

### TPH — Table Per Hierarchy

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

### TPT — Table Per Type

Each mapped type has a table; derived tables share the primary key with the base table.

```csharp
modelBuilder.Entity<Person>().ToTable("People");
modelBuilder.Entity<Student>().ToTable("Students");
modelBuilder.Entity<Teacher>().ToTable("Teachers");
```

- Clearer separation of type-specific columns.
- Queries across the hierarchy can require joins.
- Performance should be measured for the actual query workload.

### TPC — Table Per Concrete Type

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

---

## 9. How do you handle concurrency conflicts?

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

---

## 10. Deferred versus immediate execution

LINQ operators such as `Where` and `Select` generally build a query pipeline. Terminal operations such as `ToList`, `Count`, `First`, and `Any` execute it. The exact behavior depends on the LINQ provider and operator.

### Deferred execution

```csharp
IQueryable<Student> query = db.Students
    .Where(s => s.IsActive); // Query expression composed; normally no SQL yet.

query = query.OrderBy(s => s.Name); // Still composing.
var students = await query.ToListAsync(); // SQL executes here.
```

`IQueryable<T>` builds an expression tree that a provider (such as EF Core) translates. By contrast, `IEnumerable<T>` uses delegates and runs LINQ-to-Objects when enumerated.

### Re-enumeration

```csharp
var query = db.Students.Where(s => s.IsActive);

var firstRead = await query.ToListAsync();
var secondRead = await query.ToListAsync(); // Executes again.
```

Materializing with `ToList()` stores the result in memory; it does not keep the query live. `IQueryable` is not itself cached data.

### Important distinctions

- `CountAsync()`, `AnyAsync()`, `FirstOrDefaultAsync()`, and `ToListAsync()` execute database queries when used with EF Core's async provider.
- `AsEnumerable()` switches subsequent operators to LINQ-to-Objects; it does not itself materialize all rows.
- `ToList()` materializes results and may consume significant memory.
- Streaming with `AsAsyncEnumerable()` can reduce buffering, but context lifetime and provider behavior still matter.
- `IQueryable` is provider-specific: a C# expression that works for LINQ-to-Objects may not translate to SQL.

---

## 11. Senior-level follow-up topics

These are excellent follow-up areas after the ten core questions.

### A. N+1 queries

A query loads a set of parent rows, then each parent triggers another query for related data. Detect it through SQL logging, tracing, and query-count tests. Fix it with projection, eager loading, or an explicit batched query.

### B. Projection versus entity materialization

For read APIs, project only required fields into a DTO:

```csharp
var rows = await db.Students
    .Where(s => s.IsActive)
    .Select(s => new StudentSummaryDto(s.Id, s.Name, s.Courses.Count))
    .ToListAsync();
```

This can reduce transferred columns and avoid materializing full entity graphs. Ensure the projection is translatable by the provider.

### C. `IQueryable` versus `IEnumerable`

- `IQueryable<T>` carries an expression tree and delegates execution to a provider.
- `IEnumerable<T>` represents enumeration using .NET code.
- Calling `AsEnumerable()` can move later filtering to the client, potentially pulling too much data.
- EF Core generally throws for untranslatable expressions in most query positions rather than silently evaluating all filtering on the client.

### D. `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`

| Operator | Zero matches | Multiple matches |
|---|---|---|
| `First` | Throws | Returns first |
| `FirstOrDefault` | Default value | Returns first |
| `Single` | Throws | Throws |
| `SingleOrDefault` | Default value | Throws |

Use `Single` when uniqueness is a required invariant, ideally backed by a unique database constraint. Add ordering when the meaning of “first” must be deterministic.

### E. Transactions and `SaveChanges`

A single `SaveChanges()` call is transactional by default when the provider supports transactions. Use an explicit transaction when multiple saves or commands must succeed or fail together. Be aware of execution strategies and retry policies when combining retries with explicit transactions.

### F. Interceptors versus logging

- **Logging:** observes events and commands; best for diagnostics.
- **Interceptors:** can observe and, for supported operations, modify or suppress certain operations.
- **`SaveChanges` override/interceptor:** useful for cross-cutting persistence behavior such as auditing.
- **Command interceptor:** useful for database-command diagnostics or targeted behavior.
- Set-based `ExecuteUpdate`/`ExecuteDelete` bypass the normal `SaveChanges` pipeline.

Avoid placing core business rules only in an interceptor: they can be hidden, difficult to reason about, and bypassed by other data paths.

### G. Repository and Unit of Work

`DbContext` already provides Unit of Work behavior (tracking changes until save) and repository-like access through `DbSet<TEntity>`. A generic repository that simply forwards every EF method may add indirection without value.

A custom repository can still be useful when it expresses domain-specific operations, hides persistence details from the domain, or supports a meaningful boundary. Decide based on architecture and testing needs—not a blanket rule that every project must have one.

### H. Migrations and model snapshots

`dotnet ef migrations add AddStudentEmail` compares the current EF model with the model snapshot from the previous migration and generates migration operations. `dotnet ef database update` applies pending migrations to the target database.

- The snapshot is a design-time model representation used to scaffold future migrations.
- It is not a live backup of the database and does not prove that a production database matches the model.
- Review generated migration code, data-loss risks, indexes, constraints, and deployment order.
- For production, consider reviewed SQL scripts or migration bundles and a controlled deployment process.

### I. Query performance workflow

1. Identify the slow endpoint and measure duration, allocations, and database time.
2. Inspect generated SQL (`ToQueryString()` is useful for query inspection, but does not execute it).
3. Check query count, returned row count, columns, joins, and query plan.
4. Add/adjust indexes based on predicates, joins, ordering, and selectivity.
5. Project only necessary columns and avoid N+1 queries.
6. Evaluate tracking, split queries, compiled queries, and batching where relevant.
7. Re-measure under realistic data and concurrency.

### J. Common trick questions

- **Does `Include()` always mean one SQL query?** No. Split-query configuration can execute multiple queries.
- **Does `AsNoTracking()` update entities automatically?** No; changes to those objects are not persisted merely by calling `SaveChanges()`.
- **Does `ExecuteUpdate()` invoke `SaveChanges()`?** No.
- **Is a `DbContext` thread-safe?** No. Do not run parallel operations on one context; await each operation or use separate contexts.
- **Should a context live for the whole application?** Usually not. In web apps, a scoped context per unit of work/request is common; background work should use appropriately bounded lifetimes.
- **Does `GroupBy()` always translate to SQL `GROUP BY`?** No. Translation depends on query shape and provider; some grouping shapes can be assembled after results are returned.
- **Does a navigation property automatically load its related rows?** No. It needs eager, explicit, lazy loading, or relationship fix-up from already tracked entities.
- **Does a global query filter guarantee tenant security?** No. It is one layer and can be bypassed by deliberate APIs or alternate data paths.

---

## Suggested viva practice method

For each topic, practise answering in this order:

1. Define the concept precisely.
2. Explain what happens at runtime or in SQL.
3. Show a short code example.
4. State performance and correctness trade-offs.
5. Name a production failure mode and how you would diagnose it.

A senior-level answer should explain not only *what* an API does, but also when it is the wrong tool and how you would prove the chosen approach is correct.
