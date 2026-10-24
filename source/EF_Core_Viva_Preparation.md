# EF Core Viva Preparation --- 50 Questions

Concise answers for viva/interview revision. Examples use modern EF
Core; provider- and version-specific behavior is noted where important.

## 1. How does EF Core translate LINQ to SQL?

EF builds an expression tree from `IQueryable`, translates supported
expressions into provider-specific SQL, executes it, then materializes
results. `ToListAsync()` executes the query; `AsEnumerable()` switches
subsequent operations to client-side LINQ. Unsupported expressions
generally fail translation except in the top-level projection.

## 2. What is the default `DbContext` DI lifetime?

`AddDbContext<TContext>()` registers it as **Scoped** by
default---usually one instance per HTTP request. A context is a
short-lived unit of work and is not thread-safe.

## 3. How can a Singleton use a Scoped `DbContext`?

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

## 4. How does EF context pooling work?

Pooling reuses `DbContext` objects after a unit of work, reducing
allocation/setup overhead. It is different from ADO.NET connection
pooling, which reuses database connections. Be careful with custom
mutable state such as tenant IDs; EF cannot automatically reset
arbitrary fields you add.

## 5. Name two ways to create a pooled context.

``` csharp
services.AddDbContextPool<AppDbContext>(o => o.UseSqlServer(cs));
services.AddPooledDbContextFactory<AppDbContext>(o => o.UseSqlServer(cs));
```

The first suits scoped injection; the factory suits explicit context
creation, such as background jobs.

## 6. When should you use `IDbContextFactory`?

When the context lifetime does not align with a DI scope: background
services, desktop apps, multiple independent units of work in one
request, or long-lived services. It does not make a context thread-safe;
use separate contexts for concurrent operations.

## 7. How do you correct a generated migration?

Review and edit `Up()`/`Down()` before applying it---for example,
replace an unintended drop/add with `RenameColumn`. If unapplied,
`dotnet ef migrations remove` lets you regenerate it. If already
deployed to a shared/production database, normally create a corrective
migration instead of rewriting history. Inspect generated SQL.

## 8. How should production migrations be executed?

Generate and review a SQL script (`dotnet ef migrations script`), test
it against a production-like database, verify backup/recovery, then
apply it through a controlled deployment pipeline or DBA. `--idempotent`
helps when databases may be at different migration levels. Avoid
uncontrolled application-startup migrations because of permissions and
concurrent-deployment risks.

## 9. How do you seed initial data?

Use `HasData()` for fixed model-managed/reference data, or custom
initialization code for dynamic data and runtime-dependent logic.

``` csharp
modelBuilder.Entity<OrderStatus>().HasData(
    new OrderStatus { Id = 1, Name = "Pending" },
    new OrderStatus { Id = 2, Name = "Complete" });
```

`HasData` requires explicit keys and changes are managed by migrations.

## 10. What changed in EF Core 9 seeding?

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

## 11. How do you create a database using EF?

For a migrations-based app, run `dotnet ef migrations add InitialCreate`
then `dotnet ef database update`, or apply a reviewed migration script.
`EnsureCreated()` creates a schema without migrations and should not be
mixed with a migrations-managed schema.

## 12. Can EF Core create migrations for multiple databases?

Yes. Use multiple `DbContext` types and specify `--context` for each.
The same context/migrations can also be applied to multiple database
instances of the same provider. Different providers may require separate
provider-specific migrations.

## 13. What are EF Core interceptors?

They can observe, modify, or suppress operations such as database
commands, connections, and `SaveChanges`. Use them for cross-cutting
behavior such as command diagnostics or auditing; ordinary logging is
simpler when you only need logs.

## 14. How do you implement soft deletion?

Add `IsDeleted`/`DeletedAt`, apply a global query filter, and turn
delete operations into updates.

``` csharp
modelBuilder.Entity<Customer>().HasQueryFilter(c => !c.IsDeleted);
```

`IgnoreQueryFilters()` can bypass filters. Bulk operations bypass change
tracking and `SaveChanges` hooks, so enforce soft-delete policy on those
paths too.

## 15. How do you insert multiple entities?

``` csharp
db.Products.AddRange(products);
await db.SaveChangesAsync();
```

EF tracks the new entities and may batch compatible commands. A single
`SaveChanges` is transactional by default for relational providers.

## 16. How do you update an existing entity?

Query it, change the required properties, then save. This is usually
safest because only intended changes are persisted.

``` csharp
var product = await db.Products.FindAsync(id);
if (product is null) return;
product.Name = request.Name;
await db.SaveChangesAsync();
```

## 17. How do you delete an existing entity?

``` csharp
var product = await db.Products.FindAsync(id);
if (product is null) return;
db.Products.Remove(product);
await db.SaveChangesAsync();
```

Foreign-key constraints and configured delete behavior govern related
rows.

## 18. How do you update without selecting first?

Attach a stub and mark only the intended property modified:

``` csharp
var product = new Product { Id = id, Name = request.Name };
db.Attach(product);
db.Entry(product).Property(p => p.Name).IsModified = true;
await db.SaveChangesAsync();
```

`db.Update(entity)` is shorter but can mark all mapped properties and
related entities modified, overwriting values that were not supplied. EF
Core 7+ also offers set-based `ExecuteUpdateAsync()`.

## 19. How do you delete without selecting first?

Attach a key-only stub and mark it deleted, or use a set-based delete in
EF Core 7+.

``` csharp
db.Entry(new Product { Id = id }).State = EntityState.Deleted;
await db.SaveChangesAsync();

// Alternative:
await db.Products.Where(p => p.Id == id).ExecuteDeleteAsync();
```

Bulk deletion bypasses tracking and `SaveChanges` hooks.

## 20. What are the pitfalls of batch update/delete?

`ExecuteUpdate`/`ExecuteDelete` issue set-based SQL without loading
entities or updating the change tracker. Tracked objects may become
stale; `SaveChanges` overrides/interceptors and normal tracked
concurrency checks do not run; multiple commands need an explicit
transaction if they must be atomic. Older third-party batch libraries
may differ.

## 21. How do you create a manual transaction?

``` csharp
await using var tx = await db.Database.BeginTransactionAsync();
try
{
    db.Orders.Add(order);
    await db.SaveChangesAsync();
    db.AuditRecords.Add(audit);
    await db.SaveChangesAsync();
    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```

A single `SaveChanges` is already transactional for relational
providers. Follow EF guidance when combining manual transactions with
retrying execution strategies.

## 22. Explain transaction isolation levels.

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

## 23. When does `DbUpdateConcurrencyException` occur?

When EF expected an update/delete to affect a row but zero rows were
affected, often because another transaction changed or deleted it after
it was read. It commonly arises with concurrency tokens. It is not the
usual exception for unique-key violations.

## 24. How do you implement optimistic concurrency?

Configure a concurrency token. In SQL Server, `rowversion` is common:

``` csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    [Timestamp] public byte[] Version { get; set; } = [];
}
// Or: modelBuilder.Entity<Product>().Property(p => p.Version).IsRowVersion();
```

EF includes the original token in the update predicate. Catch
`DbUpdateConcurrencyException` and reload, merge, retry, or return a
conflict. `rowversion` is SQL Server-specific.

## 25. How do you execute custom SQL?

Use parameterized `ExecuteSql`/`ExecuteSqlInterpolated` for commands,
and `FromSql`/`FromSqlInterpolated` for entity-shaped result rows.

``` csharp
var count = await db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Products SET IsActive = {false} WHERE CategoryId = {categoryId}");
```

Never concatenate untrusted input. SQL parameters generally cannot
represent identifiers such as table or column names.

## 26. How do you query a database view?

Map a read model, commonly as a keyless entity:

``` csharp
modelBuilder.Entity<SalesSummary>()
    .HasNoKey()
    .ToView("vw_SalesSummary");
```

Then query `db.Set<SalesSummary>()`. `ToView()` maps the query source;
it does not necessarily create the view, which may need a migration or
SQL script.

## 27. How do you call a stored procedure?

For a command, use parameterized SQL execution:

``` csharp
await db.Database.ExecuteSqlInterpolatedAsync(
    $"EXEC dbo.RebuildSummary @CategoryId={categoryId}");
```

For rows, use `FromSql` on a compatible entity/keyless type where the
provider/version supports it. For multiple result sets or complex
outputs, ADO.NET may be more appropriate.

## 28. How do you log generated SQL?

``` csharp
optionsBuilder
    .UseSqlServer(cs)
    .LogTo(Console.WriteLine, LogLevel.Information);
```

In ASP.NET Core, the standard logging pipeline can route EF logs to your
chosen sink. `EnableSensitiveDataLogging()` may expose personal or
secret parameter values; use only in controlled development
environments.

## 29. What are the ways to configure entity mapping?

1.  **Conventions** --- EF infers mappings from names/types.
2.  **Data annotations** --- attributes such as `[Key]`, `[Required]`.
3.  **Fluent API** --- configuration in `OnModelCreating`.
4.  **`IEntityTypeConfiguration<TEntity>`** --- separate mapping
    classes. Fluent API generally takes precedence over annotations,
    which override many conventions.

## 30. How are nullable reference types handled?

With C# nullable reference types enabled, `string Name` is required by
convention and `string? Description` is optional. This affects model
nullability and migrations. Enabling NRT in an existing project can
cause many columns to change.

## 31. How do you configure one-to-one?

``` csharp
modelBuilder.Entity<User>()
    .HasOne(u => u.Profile)
    .WithOne(p => p.User)
    .HasForeignKey<UserProfile>(p => p.UserId);
```

`UserProfile` is the dependent. The foreign key should be unique for a
true one-to-one relationship.

## 32. How do you configure one-to-many?

``` csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .HasForeignKey(p => p.BlogId);
```

The dependent (`Post`) holds the foreign key. EF can often infer the
relationship by convention.

## 33. How do you configure many-to-many?

EF Core 5+ supports skip navigations:

``` csharp
modelBuilder.Entity<Post>()
    .HasMany(p => p.Tags)
    .WithMany(t => t.Posts);
```

EF creates a join table by convention. Use an explicit join entity if
the relationship has extra fields such as `CreatedAt`.

## 34. What are owned types for?

They model a value-like object that belongs to an owner and has
ownership semantics, e.g. an `Address` owned by `Customer`. Depending on
mapping, properties can be stored in the owner's table or a separate
table.

## 35. What are complex types for?

Available from EF Core 8, complex types represent structured values
without their own entity identity/key, such as `Address` or `Money`. Use
them when the value is part of the containing entity rather than an
independently identified entity. Exact support varies by
version/provider.

## 36. How do you store an enum as a string?

``` csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();
```

The database stores enum names. Renaming members can require data
migration; consider explicit stable mappings if stored values are part
of a long-lived contract.

## 37. How do you implement audit trails?

Inspect `ChangeTracker` entries in `SaveChanges`/`SaveChangesAsync`
overrides or a `SaveChangesInterceptor`; record entity/key, operation,
changed values, user, and timestamp. Bulk
`ExecuteUpdate`/`ExecuteDelete` bypass these hooks. Database triggers or
temporal tables may be preferable if all writers must be audited.

## 38. How do you implement multitenancy?

Common strategies: database per tenant, schema per tenant
(provider-specific), or shared database with a `TenantId` column and
global filter.

``` csharp
public string TenantId { get; }

protected override void OnModelCreating(ModelBuilder modelBuilder) =>
    modelBuilder.Entity<Invoice>()
        .HasQueryFilter(i => i.TenantId == TenantId);
```

A filter alone is not a complete security boundary: validate tenant
identity, constrain writes, and consider database row-level security.
With pooling, manage tenant state carefully.

## 39. When should you use `AsNoTracking()`?

For read-only queries when you do not intend to modify/save the returned
entities. It reduces change-tracker overhead.

``` csharp
var products = await db.Products.AsNoTracking()
    .Where(p => p.IsActive).ToListAsync();
```

`AsNoTrackingWithIdentityResolution()` can deduplicate object instances
without normal context tracking.

## 40. Explain eager loading.

Eager loading fetches related data as part of the initial query with
`Include`/`ThenInclude`.

``` csharp
var blogs = await db.Blogs.Include(b => b.Posts)
    .ThenInclude(p => p.Author).ToListAsync();
```

Multiple collection includes can multiply joined rows; inspect SQL and
consider `AsSplitQuery()`.

## 41. Explain explicit and lazy loading.

**Explicit:** request a navigation deliberately,
e.g. `await db.Entry(blog).Collection(b => b.Posts).LoadAsync()`.
**Lazy:** the navigation loads on access when lazy loading is
configured. Lazy loading can cause N+1 queries and hidden database
calls; eager or explicit loading is often easier to control.

## 42. Do you need Repository and Unit of Work patterns?

`DbContext` already provides Unit of Work behavior; `DbSet` offers
repository-like operations. An extra repository is optional and useful
when it expresses domain-specific queries, hides persistence details, or
creates a meaningful test boundary. Avoid layers that merely duplicate
every `DbSet` method.

## 43. How do you configure an auto-increment primary key?

Numeric keys are commonly generated on insert by convention. You can
specify:

``` csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Id).ValueGeneratedOnAdd();
```

SQL Server normally maps an integer key to `IDENTITY`; exact behavior
depends on provider/type. `ValueGeneratedOnAdd()` describes
value-generation behavior, not one universal database mechanism.

## 44. What are global query filters?

Predicates automatically applied to queries for an entity, commonly for
soft deletion or tenant filtering.

``` csharp
modelBuilder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted);
```

`IgnoreQueryFilters()` disables them for a query, so bypasses should be
intentional and authorized.

## 45. What is `AsSplitQuery()`?

It loads included related data using multiple SQL queries instead of one
large join, potentially reducing cartesian explosion and duplicated
data. The trade-offs are more round trips and possible inconsistencies
if data changes between queries; use a suitable transaction/isolation
level when consistency matters.

## 46. What are compiled queries for?

`EF.CompileQuery()` and `EF.CompileAsyncQuery()` precompile stable LINQ
query shapes, reducing repeated EF query-compilation overhead. They do
not reduce database execution time by themselves; use them for hot paths
after profiling.

## 47. Can compiled queries be asynchronous?

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

## 48. How do you search case-insensitively?

It depends on database collation/provider. On SQL Server, a
case-insensitive collation is often best. Where supported:

``` csharp
var users = await db.Users.Where(u =>
    EF.Functions.Collate(u.Name, "SQL_Latin1_General_CP1_CI_AS") == searchName)
    .ToListAsync();
```

Collation names are database-specific. `ToLower()` on a column may hurt
index use; `Like()` does not itself guarantee case-insensitive matching.

## 49. What are JSON columns?

They store structured JSON in a database column. EF Core's mapping/query
support depends on version and provider. JSON is useful for
flexible/nested data but has trade-offs in validation, indexing,
queryability, and schema evolution. Frequently queried fields may be
better as relational columns.

## 50. How do you implement cascade deletion?

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

## High-value distinctions to revise

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

## Viva answer formula

1.  Define the concept in one or two sentences.
2.  Say when you would use it.
3.  Give a short code example if relevant.
4.  Mention one trade-off, limitation, or production concern.
