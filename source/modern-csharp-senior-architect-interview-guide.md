# Modern C# Interview Guide: Senior Developer vs. Architect

A practical guide to modern C# features, production examples, and refactoring legacy code. It focuses on features that improve correctness, readability, maintainability, and performance in day-to-day .NET development.

> **Reference baseline:** Microsoft Learn's official C# documentation. Match the language version to the project's target SDK and `LangVersion`; newer syntax may not be available in older projects.

## Contents

1. [What interviewers expect](#1-what-interviewers-expect)
2. [Feature-to-production mapping](#2-feature-to-production-mapping)
3. [Bad code to modern C# refactoring](#3-bad-code-to-modern-c-refactoring)
4. [Interview discussion prompts](#4-interview-discussion-prompts)
5. [Quick checklist](#5-quick-checklist)
6. [Official Microsoft Learn references](#6-official-microsoft-learn-references)

---

## 1. What interviewers expect

The difference is not simply the number of language features a candidate can name. Senior developers should apply features correctly; architects should explain trade-offs, operational consequences, and when *not* to use them.

### Senior .NET developer

| Topic / feature | What to demonstrate | Common concern |
|---|---|---|
| `async` / `await` | Keep I/O-bound call chains asynchronous; propagate cancellation; handle failures | Blocking with `.Result` or `.Wait()` |
| LINQ and collections | Choose the right operator and collection; understand deferred execution and repeated enumeration | Materializing large sequences unnecessarily |
| Nullable reference types | Model optional values accurately and handle nulls explicitly | Suppressing warnings without understanding them |
| Pattern matching and switch expressions | Express type/state checks clearly and handle unexpected cases | Large, fragile conditional chains |
| Records, `init`, and immutability | Use value-based equality where appropriate; keep data contracts clear | Treating every entity as a record |
| Generics and interfaces | Build type-safe reusable abstractions without overengineering | Abstractions with no practical benefit |
| Dependency injection | Select correct service lifetimes and keep dependencies explicit | Captive dependencies, service locator patterns |
| EF Core | Project only required data, inspect generated SQL, understand tracking and query translation | N+1 queries and accidental client-side work |
| Exceptions and logging | Use exceptions for exceptional failures; log useful context without leaking secrets | Swallowing exceptions or logging sensitive data |
| Testing | Unit-test business rules and integration-test important boundaries | Mocking every dependency and never testing integration |

### Staff / architect level

| Topic / feature | What to demonstrate | Trade-off to discuss |
|---|---|---|
| Value vs. reference types | Understand copying, identity, allocation, and lifetime | A `struct` is not automatically faster or stack-allocated |
| `Span<T>` / `Memory<T>` | Use allocation-conscious APIs in measured hot paths | Complexity, lifetime restrictions, and maintainability |
| `IAsyncEnumerable<T>` | Stream large or incremental results asynchronously | Backpressure, cancellation, error timing, resource lifetime |
| `ValueTask<T>` | Use only when profiling and API semantics justify it | More complex consumption rules; not a default replacement for `Task<T>` |
| GC and allocation profiling | Diagnose allocation rate, retention, LOH, and latency | Pooling can increase complexity and retain memory |
| `ConfigureAwait(false)` | Understand synchronization-context capture, especially in reusable libraries | It is not a universal performance switch; ASP.NET Core typically has no custom request context |
| Records and immutable models | Reduce accidental mutation and clarify value semantics | EF Core entities and identity-rich domain objects may need reference identity |
| Trimming, Native AOT, and source generation | Design for startup, deployment size, and restricted reflection | Compatibility and build complexity |
| API and domain contracts | Use validation, explicit errors, versioning, and stable boundaries | Avoid leaking persistence models into public APIs |
| Distributed systems | Design idempotency, retries, timeouts, observability, and consistency | Retries can duplicate side effects; eventual consistency affects UX |
| Architecture | Explain boundaries, coupling, deployment, ownership, and failure modes | Avoid applying Clean Architecture, CQRS, or microservices mechanically |

### Interview rule of thumb

- **Senior:** “I can implement this safely, diagnose it, and maintain it.”
- **Architect:** “I can justify this choice, compare alternatives, and explain its operational and organizational consequences.”

---

## 2. Feature-to-production mapping

### 2.1 `async` / `await`, cancellation, and timeouts

**Use for:** HTTP calls, database access, file I/O, and other asynchronous I/O.

```csharp
public async Task<UserDto?> GetUserAsync(
    int id,
    CancellationToken cancellationToken)
{
    var user = await _dbContext.Users
        .AsNoTracking()
        .Where(x => x.Id == id)
        .Select(x => new UserDto(x.Id, x.Name, x.Email))
        .SingleOrDefaultAsync(cancellationToken);

    return user;
}
```

**Why it helps**
- Avoids blocking a request thread while asynchronous I/O is pending.
- Propagates cancellation from the caller.
- Projects only the data the endpoint needs.

**Architect note:** `async` does not automatically make CPU-bound work faster or run it on another thread. Apply appropriate timeouts at remote-service boundaries and make retry policies safe for the operation.

### 2.2 Nullable reference types and explicit null handling

Enable nullable reference types for projects where feasible, then model optional data explicitly.

```csharp
public sealed record UserDto(
    int Id,
    string Name,
    string? MiddleName,
    string Email);
```

```csharp
if (user?.Profile?.Email is { Length: > 0 } email)
{
    await _mailer.SendAsync(email, cancellationToken);
}
```

**Why it helps:** Makes optional values visible in APIs and catches many potential null dereferences at compile time. Nullable annotations are compile-time analysis, not automatic runtime validation of incoming data.

### 2.3 Records, `init`, and immutability

**Use for:** DTOs, commands, events, and data that naturally has value semantics.

```csharp
public sealed record CreateOrderCommand(
    int CustomerId,
    IReadOnlyList<CreateOrderItem> Items);

public sealed record CreateOrderItem(
    int ProductId,
    int Quantity);
```

Records provide value-based equality and convenient non-destructive mutation with `with`. A record is not deeply immutable if it contains mutable objects or collections.

**Architect note:** Do not choose records automatically for EF Core entities. Entities often have identity and lifecycle semantics better represented by classes.

### 2.4 Pattern matching and switch expressions

**Use for:** State transitions, mapping known variants, and simplifying branching.

```csharp
public static string GetStatusLabel(OrderStatus status) =>
    status switch
    {
        OrderStatus.Pending   => "Pending",
        OrderStatus.Paid      => "Paid",
        OrderStatus.Cancelled => "Cancelled",
        _ => throw new ArgumentOutOfRangeException(nameof(status), status, null)
    };
```

**Why it helps:** Makes the mapping explicit and provides a clear place to handle unknown values. For domain transitions, validate whether the transition is allowed rather than only mapping labels.

### 2.5 `required` and `init`

**Use for:** Configuration and data objects where callers must supply key members.

```csharp
public sealed class EmailOptions
{
    public required string SenderAddress { get; init; }
    public required string SubjectPrefix { get; init; }
}
```

`required` helps enforce initialization at compile time for normal C# callers. It does not replace runtime validation of configuration, JSON, database data, or other external input.

### 2.6 LINQ, projection, and deferred execution

```csharp
var activeUsers = await _dbContext.Users
    .AsNoTracking()
    .Where(user => user.IsActive)
    .OrderBy(user => user.Name)
    .Select(user => new UserListItem(user.Id, user.Name))
    .ToListAsync(cancellationToken);
```

**Why it helps:** Expresses the query's intent and lets EF Core translate supported operations into SQL.

**Watch for**
- Re-enumerating a deferred query.
- Calling `ToList()` before filtering when the database could do the filtering.
- Using methods that cannot be translated by the query provider.
- Returning too many rows without pagination.
- Using `Count()` when only existence (`Any()`) is needed.

`IEnumerable<T>` describes enumeration; `IQueryable<T>` describes a query that a provider may translate. Neither interface alone guarantees where or how efficiently execution occurs.

### 2.7 `Span<T>`, `ReadOnlySpan<T>`, and `Memory<T>`

**Use for:** Parsing and measured performance-sensitive code where copying or allocations matter.

```csharp
public static bool TryReadCode(ReadOnlySpan<char> input, out int code)
{
    return int.TryParse(input, out code);
}
```

`Span<T>` and `ReadOnlySpan<T>` are byref-like types with lifetime restrictions; they cannot be stored in ordinary heap objects or used freely across `await`. `Memory<T>` supports scenarios where a memory region must be represented across asynchronous operations.

**Architect note:** Profile before optimizing. A clear, allocation-light implementation is preferable to premature low-level complexity.

### 2.8 `IAsyncEnumerable<T>` for streaming

**Use for:** Large result sets, incremental processing, or streams where buffering everything would be wasteful.

```csharp
public async IAsyncEnumerable<int> GetIdsAsync(
    [System.Runtime.CompilerServices.EnumeratorCancellation]
    CancellationToken cancellationToken = default)
{
    for (var id = 1; id <= 100; id++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await Task.Yield();
        yield return id;
    }
}
```

Consumers can use `await foreach`. For database streaming, verify provider behavior and keep the data context/resource alive for the duration of enumeration.

### 2.9 Collection expressions and target-typed construction

Modern syntax can make simple initialization more readable.

```csharp
int[] ids = [10, 20, 30];
List<string> roles = ["Reader", "Editor"];
```

Collection expressions require a compatible target type and language version. Use them when they improve clarity, not merely because the syntax is newer.

### 2.10 `nameof`, interpolation, and raw string literals

```csharp
ArgumentNullException.ThrowIfNull(request);

_logger.LogInformation(
    "Processing order {OrderId} for customer {CustomerId}",
    request.OrderId,
    request.CustomerId);
```

- `nameof` provides refactor-friendly identifiers for APIs and validation.
- Structured logging keeps values queryable; prefer it to interpolated log messages.
- Raw string literals are useful for multiline JSON, SQL, and test data when supported by the project's language version.

Never put passwords, access tokens, secrets, or unnecessary personal data in logs.

### 2.11 Dependency injection and service lifetimes

```csharp
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<IOrderRepository, OrderRepository>();
```

- **Transient:** new instance each time it is requested.
- **Scoped:** one instance per DI scope (commonly one ASP.NET Core request).
- **Singleton:** one instance for the service provider's lifetime.

Do not capture a scoped service inside a singleton. Singleton services must be safe for concurrent use where they are shared concurrently. A hosted service that needs scoped dependencies should create a scope when doing scoped work.

### 2.12 EF Core: projection, tracking, and query shape

```csharp
var orders = await _dbContext.Orders
    .AsNoTracking()
    .Where(order => order.CustomerId == customerId)
    .Select(order => new OrderSummary(
        order.Id,
        order.CreatedAt,
        order.Total))
    .ToListAsync(cancellationToken);
```

**Production checklist**
- Inspect generated SQL and query plans for slow queries.
- Add indexes based on actual query patterns and execution plans.
- Avoid loading entire graphs when a projection is sufficient.
- Detect N+1 queries and accidental repeated database calls.
- Use transactions when a group of changes must be atomic.
- Pass cancellation tokens to supported asynchronous operations.

### 2.13 Exceptions, validation, and error contracts

```csharp
public static void ValidateQuantity(int quantity)
{
    if (quantity <= 0)
    {
        throw new ArgumentOutOfRangeException(
            nameof(quantity), "Quantity must be greater than zero.");
    }
}
```

Use exceptions for exceptional failures, not as the routine control flow for expected outcomes. Validate untrusted input at system boundaries, enforce domain invariants in the domain model, and return consistent API error responses (for example, Problem Details in ASP.NET Core).

### 2.14 Cancellation and resilience at service boundaries

```csharp
public async Task<ProductDto?> GetProductAsync(
    int id,
    CancellationToken cancellationToken)
{
    using var response = await _httpClient.GetAsync(
        $"products/{id}", cancellationToken);

    if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
        return null;

    response.EnsureSuccessStatusCode();

    return await response.Content.ReadFromJsonAsync<ProductDto>(
        cancellationToken: cancellationToken);
}
```

**Architect note:** Configure `HttpClient` through `IHttpClientFactory` or an appropriate managed client pattern. Define timeout, retry, circuit-breaker, and fallback behavior based on the dependency and operation. Do not blindly retry non-idempotent requests.

---

## 3. Bad code to modern C# refactoring

These examples show useful refactoring patterns. Modern syntax is not automatically better: the goal is clearer intent, correctness, and measured performance.

### Case 1: Nested null checks

**Before**

```csharp
if (user != null)
{
    if (user.Profile != null)
    {
        if (user.Profile.Email != null)
        {
            Send(user.Profile.Email);
        }
    }
}
```

**After**

```csharp
if (user?.Profile?.Email is { } email)
{
    Send(email);
}
```

Use this when the nested checks are simply guarding access. Keep explicit branches when they communicate different business outcomes.

### Case 2: Blocking asynchronous work

**Before**

```csharp
var order = orderService.GetAsync(orderId).Result;
```

**After**

```csharp
var order = await orderService.GetAsync(orderId, cancellationToken);
```

Make the calling method asynchronous too, and propagate cancellation where appropriate. Avoid `.Result` and `.Wait()` in request-processing paths.

### Case 3: String-based conditional chains

**Before**

```csharp
if (status == "Pending")
    return "Waiting";
else if (status == "Paid")
    return "Complete";
else
    return "Unknown";
```

**After**

```csharp
return status switch
{
    "Pending" => "Waiting",
    "Paid" => "Complete",
    _ => "Unknown"
};
```

For important domain states, prefer an enum or a strongly typed model over scattered magic strings.

### Case 4: Repeated loops for simple transformations

**Before**

```csharp
var names = new List<string>();

foreach (var user in users)
{
    if (user.IsActive)
        names.Add(user.Name);
}
```

**After**

```csharp
var names = users
    .Where(user => user.IsActive)
    .Select(user => user.Name)
    .ToList();
```

LINQ is a good fit for transformations. A `foreach` loop may be clearer when the logic is stateful, complex, or involves side effects. Avoid `Where(...).ToList().ForEach(...)` just to create a fluent-looking chain.

### Case 5: Mutable data transfer object

**Before**

```csharp
public class UserDto
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}
```

**After, when value semantics fit**

```csharp
public sealed record UserDto(int Id, string Name);
```

Choose a record when value-based equality and concise data modeling are useful. Keep a class when mutability, inheritance, ORM behavior, or identity semantics are needed.

### Case 6: Repeated string concatenation

**Before**

```csharp
var message = "Order " + orderId + " belongs to " + customerName;
```

**After**

```csharp
var message = $"Order {orderId} belongs to {customerName}";
```

For logs, use structured logging placeholders instead of constructing a message string eagerly.

### Case 7: Magic values

**Before**

```csharp
if (order.Status == 2)
{
    // ...
}
```

**After**

```csharp
if (order.Status == OrderStatus.Paid)
{
    // ...
}
```

Use a meaningful enum or named constant where it represents a stable concept. Ensure enum values and serialization/database mappings remain compatible with existing contracts.

### Case 8: Returning a collection when only existence matters

**Before**

```csharp
var matchingUsers = users
    .Where(user => user.Email == email)
    .ToList();

if (matchingUsers.Count > 0)
{
    // ...
}
```

**After**

```csharp
if (users.Any(user => user.Email == email))
{
    // ...
}
```

For an EF Core query, this can translate to an existence query. If the data is already materialized, it still avoids building an unnecessary list.

### Case 9: Unnecessary full-entity materialization

**Before**

```csharp
var users = await dbContext.Users
    .ToListAsync(cancellationToken);

var names = users
    .Where(user => user.IsActive)
    .Select(user => user.Name)
    .ToList();
```

**After**

```csharp
var names = await dbContext.Users
    .Where(user => user.IsActive)
    .Select(user => user.Name)
    .ToListAsync(cancellationToken);
```

The second version lets the database filter and return only the required column. Confirm the generated SQL and query plan for performance-sensitive queries.

### Case 10: Catching and hiding every exception

**Before**

```csharp
try
{
    await service.ProcessAsync(cancellationToken);
}
catch
{
    return;
}
```

**After, if the failure should propagate**

```csharp
await service.ProcessAsync(cancellationToken);
```

If the application can recover, catch a specific exception, log meaningful context, and return or translate a deliberate outcome. Do not log and rethrow indiscriminately at every layer, which can create duplicate logs.

### Case 11: Service locator / manual dependency construction

**Before**

```csharp
public sealed class OrderController
{
    public async Task<IActionResult> Get(int id)
    {
        var service = new OrderService(new OrderRepository());
        // ...
    }
}
```

**After**

```csharp
public sealed class OrderController(IOrderService orderService) : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<IActionResult> Get(
        int id,
        CancellationToken cancellationToken)
    {
        var order = await orderService.GetAsync(id, cancellationToken);
        return order is null ? NotFound() : Ok(order);
    }
}
```

Register dependencies with the container and keep construction separate from business behavior. The primary-constructor syntax requires C# 12; use a conventional constructor in older projects.

### Case 12: Premature low-level optimization

**Before (unnecessary complexity)**

```csharp
// Custom pooling or Span-based parsing added before profiling,
// making a simple business operation harder to understand.
```

**Better approach**
1. Measure with a representative workload.
2. Identify the actual bottleneck.
3. Optimize the narrow hot path.
4. Add tests and benchmark the change.
5. Keep the simpler implementation if the benefit is negligible.

`Span<T>`, pooling, caching, and custom concurrency primitives are tools—not default coding standards.

---

## 4. Interview discussion prompts

Use these to practice explaining the *why*, not just the syntax.

1. When would you use a record instead of a class? When would you avoid a record?
2. How do nullable reference types help, and what problems do they not solve?
3. Explain deferred LINQ execution and how it can cause repeated database queries.
4. How do `IEnumerable<T>` and `IQueryable<T>` differ in a typical EF Core application?
5. Why is `async` useful for I/O-bound operations? Why does it not automatically improve CPU-bound work?
6. When should you use `Task`, `ValueTask`, or `IAsyncEnumerable<T>`?
7. What are the lifetime risks when injecting scoped services into singletons?
8. How would you diagnose a high-allocation or high-GC application?
9. When would `Span<T>` be justified, and what restrictions does it introduce?
10. How do you make retries safe when a request may already have produced a side effect?
11. How would you prevent an EF Core endpoint from returning too much data?
12. What would make you choose a modular monolith over microservices?
13. How would you introduce a breaking API change without disrupting existing clients?
14. How do you design logs, metrics, traces, and health checks for production diagnosis?
15. How do you balance modern language features with the team's target framework, maintainability, and deployment constraints?

A strong answer typically includes the use case, alternatives, trade-offs, failure modes, and how you would verify the result.

---

## 5. Quick checklist

### Daily coding
- [ ] Use asynchronous APIs for I/O and propagate cancellation.
- [ ] Enable nullable reference types and resolve warnings intentionally.
- [ ] Prefer clear names, `nameof`, and meaningful types over magic values.
- [ ] Use records only when value semantics fit.
- [ ] Use pattern matching and switch expressions when they improve clarity.
- [ ] Use LINQ for clear transformations; avoid unnecessary materialization.
- [ ] Use structured logging and avoid sensitive information.
- [ ] Validate untrusted input and preserve domain invariants.
- [ ] Test business rules and important integration boundaries.

### Senior / architect
- [ ] Explain DI lifetime and concurrency implications.
- [ ] Diagnose EF Core SQL and query-plan issues.
- [ ] Understand allocations, GC, and resource disposal.
- [ ] Use streaming and low-allocation APIs when justified by measurements.
- [ ] Design cancellation, timeout, retry, and idempotency behavior.
- [ ] Make API contracts, security, observability, and deployment constraints explicit.
- [ ] Choose architectural patterns based on requirements rather than fashion.

---

## 6. Official Microsoft Learn references

Use these as the source of truth for language details and version-specific behavior:

- [C# documentation](https://learn.microsoft.com/en-us/dotnet/csharp/)
- [What's new in C#](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/)
- [C# language reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/)
- [C# programming guide](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/)
- [Asynchronous programming with async and await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
- [LINQ in C#](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)
- [Nullable reference types](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)
- [Records](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records)
- [ASP.NET Core fundamentals](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/)
- [Dependency injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [Entity Framework Core documentation](https://learn.microsoft.com/en-us/ef/core/)

**Maintenance note:** C# language features and runtime behavior evolve. Check the official “What's new in C#” documentation and the project's target framework before adopting version-specific syntax.
