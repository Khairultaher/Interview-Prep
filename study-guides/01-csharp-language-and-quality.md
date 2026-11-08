# C# Language and Code Quality

Each concept is explained once. Where several notes covered the same topic, they have been merged into a single section, and related topics in other guides are linked instead of repeated.

## Contents

- [Modern C# Interview Guide: Senior Developer vs. Architect](#modern-csharp-senior-architect-interview-guide-md)
- [C# Data Types and Collections Cheat Sheet](#csharp-data-types-and-collections-cheatsheet-md)
- [Senior C#/.NET Developer: Code Review, Refactoring & Interview Guide](#senior-csharp-code-review-refactoring-interview-guide-md)
- [Shallow Clone vs. Deep Clone in C#](#shallow-vs-deep-clone-md)
- [Reflection: Get a Property by Name Using `Type`](#get-property-using-type-csharp-md)
- [Additional Questpond question sets](#additional-questpond-question-sets)

---

<a id="modern-csharp-senior-architect-interview-guide-md"></a>

## Modern C# Interview Guide: Senior Developer vs. Architect

A practical guide to modern C# features, production examples, and refactoring legacy code. It focuses on features that improve correctness, readability, maintainability, and performance in day-to-day .NET development.

> **Reference baseline:** Microsoft Learn's official C# documentation. Match the language version to the project's target SDK and `LangVersion`; newer syntax may not be available in older projects.

#### Contents

1. [What interviewers expect](#1-what-interviewers-expect)
2. [Feature-to-production mapping](#2-feature-to-production-mapping)
3. [Bad code to modern C# refactoring](#3-bad-code-to-modern-c-refactoring)
4. [Interview discussion prompts](#4-interview-discussion-prompts)
5. [Quick checklist](#5-quick-checklist)
6. [Official Microsoft Learn references](#6-official-microsoft-learn-references)

---

#### 1. What interviewers expect

The difference is not simply the number of language features a candidate can name. Senior developers should apply features correctly; architects should explain trade-offs, operational consequences, and when *not* to use them.

##### Senior .NET developer

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

##### Staff / architect level

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

##### Interview rule of thumb

- **Senior:** “I can implement this safely, diagnose it, and maintain it.”
- **Architect:** “I can justify this choice, compare alternatives, and explain its operational and organizational consequences.”

---

#### 2. Feature-to-production mapping

##### 2.1 `async` / `await`, cancellation, and timeouts

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

##### 2.2 Nullable reference types and explicit null handling

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

##### 2.3 Records, `init`, and immutability

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

##### 2.4 Pattern matching and switch expressions

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

##### 2.5 `required` and `init`

**Use for:** Configuration and data objects where callers must supply key members.

```csharp
public sealed class EmailOptions
{
    public required string SenderAddress { get; init; }
    public required string SubjectPrefix { get; init; }
}
```

`required` helps enforce initialization at compile time for normal C# callers. It does not replace runtime validation of configuration, JSON, database data, or other external input.

##### 2.6 LINQ, projection, and deferred execution

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

##### 2.7 `Span<T>`, `ReadOnlySpan<T>`, and `Memory<T>`

**Use for:** Parsing and measured performance-sensitive code where copying or allocations matter.

```csharp
public static bool TryReadCode(ReadOnlySpan<char> input, out int code)
{
    return int.TryParse(input, out code);
}
```

`Span<T>` and `ReadOnlySpan<T>` are byref-like types with lifetime restrictions; they cannot be stored in ordinary heap objects or used freely across `await`. `Memory<T>` supports scenarios where a memory region must be represented across asynchronous operations.

**Architect note:** Profile before optimizing. A clear, allocation-light implementation is preferable to premature low-level complexity.

##### 2.8 `IAsyncEnumerable<T>` for streaming

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

##### 2.9 Collection expressions and target-typed construction

Modern syntax can make simple initialization more readable.

```csharp
int[] ids = [10, 20, 30];
List<string> roles = ["Reader", "Editor"];
```

Collection expressions require a compatible target type and language version. Use them when they improve clarity, not merely because the syntax is newer.

##### 2.10 `nameof`, interpolation, and raw string literals

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

##### 2.11 Dependency injection and service lifetimes

Transient / Scoped / Singleton, captive dependencies, and background-worker scopes are explained once in [.NET and ASP.NET Core → Dependency Injection](03-dotnet-and-aspnetcore.md#dependency-injection). Key rule: never capture a scoped service inside a singleton.

##### 2.12 Exceptions, validation, and error contracts

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

##### 2.13 Cancellation and resilience at service boundaries

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

#### 3. Bad code to modern C# refactoring

These examples show useful refactoring patterns. Modern syntax is not automatically better: the goal is clearer intent, correctness, and measured performance.

##### Case 1: Nested null checks

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

##### Case 2: Blocking asynchronous work

**Before**

```csharp
var order = orderService.GetAsync(orderId).Result;
```

**After**

```csharp
var order = await orderService.GetAsync(orderId, cancellationToken);
```

Make the calling method asynchronous too, and propagate cancellation where appropriate. Avoid `.Result` and `.Wait()` in request-processing paths.

##### Case 3: String-based conditional chains

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

##### Case 4: Repeated loops for simple transformations

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

##### Case 5: Mutable data transfer object

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

##### Case 6: Repeated string concatenation

**Before**

```csharp
var message = "Order " + orderId + " belongs to " + customerName;
```

**After**

```csharp
var message = $"Order {orderId} belongs to {customerName}";
```

For logs, use structured logging placeholders instead of constructing a message string eagerly.

##### Case 7: Magic values

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

##### Case 8: Returning a collection when only existence matters

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

##### Case 9: Unnecessary full-entity materialization

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

##### Case 10: Catching and hiding every exception

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

##### Case 11: Service locator / manual dependency construction

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

##### Case 12: Premature low-level optimization

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

#### 4. Interview discussion prompts

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

#### 5. Quick checklist

##### Daily coding
- [ ] Use asynchronous APIs for I/O and propagate cancellation.
- [ ] Enable nullable reference types and resolve warnings intentionally.
- [ ] Prefer clear names, `nameof`, and meaningful types over magic values.
- [ ] Use records only when value semantics fit.
- [ ] Use pattern matching and switch expressions when they improve clarity.
- [ ] Use LINQ for clear transformations; avoid unnecessary materialization.
- [ ] Use structured logging and avoid sensitive information.
- [ ] Validate untrusted input and preserve domain invariants.
- [ ] Test business rules and important integration boundaries.

##### Senior / architect
- [ ] Explain DI lifetime and concurrency implications.
- [ ] Diagnose EF Core SQL and query-plan issues.
- [ ] Understand allocations, GC, and resource disposal.
- [ ] Use streaming and low-allocation APIs when justified by measurements.
- [ ] Design cancellation, timeout, retry, and idempotency behavior.
- [ ] Make API contracts, security, observability, and deployment constraints explicit.
- [ ] Choose architectural patterns based on requirements rather than fashion.

---

#### 6. Official Microsoft Learn references

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

---

<a id="csharp-data-types-and-collections-cheatsheet-md"></a>

## C# Data Types and Collections Cheat Sheet

A deduplicated reference for C#/.NET interviews and everyday
development. Examples target modern C# and .NET 6+ unless noted.

#### 1. Built-in types

##### Integral types

  ---------------------------------------------------------------------------
  C# type          .NET type                           Size Range
  ---------------- ------------------ --------------------- -----------------
  `sbyte`          `System.SByte`                     8-bit -128 to 127

  `byte`           `System.Byte`                      8-bit 0 to 255

  `short`          `System.Int16`                    16-bit -32,768 to 32,767

  `ushort`         `System.UInt16`                   16-bit 0 to 65,535

  `int`            `System.Int32`                    32-bit -2,147,483,648 to
                                                            2,147,483,647

  `uint`           `System.UInt32`                   32-bit 0 to
                                                            4,294,967,295

  `long`           `System.Int64`                    64-bit -9.22e18 to
                                                            9.22e18
                                                            (approximately)

  `ulong`          `System.UInt64`                   64-bit 0 to 1.84e19
                                                            (approximately)

  `nint`           `System.IntPtr`           platform-sized Signed native
                                                            integer

  `nuint`          `System.UIntPtr`          platform-sized Unsigned native
                                                            integer
  ---------------------------------------------------------------------------

``` csharp
byte pixels = 255;
int count = 42;
long largeCount = 9_000_000_000L;
uint unsignedCount = 3_000_000_000U;
nint nativeValue = 10;
```

##### Floating-point and decimal types

  --------------------------------------------------------------------------
  Type                            Size          Approximate Typical use
                                                  precision 
  --------------- -------------------- -------------------- ----------------
  `float`                       32-bit     6--9 significant Graphics,
                                                     digits measurements
                                                            where
                                                            approximation is
                                                            acceptable

  `double`                      64-bit   15--17 significant General
                                                     digits scientific and
                                                            floating-point
                                                            calculations

  `decimal`            128-bit storage   28--29 significant Financial
                                                     digits calculations
                                                            requiring
                                                            decimal
                                                            arithmetic
  --------------------------------------------------------------------------

``` csharp
float ratio = 0.75f;        // f suffix
double distance = 123.456;
decimal price = 19.99m;     // m suffix
```

Floating-point types can have rounding error. `decimal` represents
decimal fractions more naturally, but it is not unlimited-precision
arithmetic.

##### Other built-in types

  ------------------------------------------------------------------------
  Type                    Example                  Notes
  ----------------------- ------------------------ -----------------------
  `bool`                  `bool isReady = true;`   `true` or `false`

  `char`                  `char initial = 'A';`    One UTF-16 code unit;
                                                   not necessarily a full
                                                   Unicode character

  `string`                `string name = "Sam";`   Immutable sequence of
                                                   UTF-16 code units

  `object`                `object value = 42;`     Base type of all C#
                                                   types; value types are
                                                   boxed when converted to
                                                   `object`
  ------------------------------------------------------------------------

``` csharp
bool valid = true;
char letter = 'A';
string message = "Hello";
object boxed = 42;

Console.WriteLine(boxed.GetType()); // System.Int32
```

##### Common framework value types

These are .NET types, not C# language keywords.

  -----------------------------------------------------------------------------------------
  Type                    Example                                   Useful members
  ----------------------- ----------------------------------------- -----------------------
  `DateTime`              `DateTime.UtcNow`                         `AddDays`, `AddHours`,
                                                                    `Date`, `Year`,
                                                                    `Month`, `Day`,
                                                                    `ToString`

  `DateTimeOffset`        `DateTimeOffset.UtcNow`                   `AddDays`,
                                                                    `ToUniversalTime`,
                                                                    `Offset`, `ToString`

  `TimeSpan`              `TimeSpan.FromMinutes(90)`                `TotalSeconds`,
                                                                    `TotalMinutes`, `Add`,
                                                                    `Subtract`

  `Guid`                  `Guid.NewGuid()`                          `Parse`, `TryParse`,
                                                                    `ToString`

  `decimal` helpers       `Math.Round(12.345, 2)` is for `double`   Use `decimal.Round`,
                                                                    `decimal.Floor`,
                                                                    `decimal.Ceiling` for
                                                                    `decimal`

  `DateOnly`              `DateOnly.FromDateTime(DateTime.Today)`   `AddDays`, `DayNumber`,
                                                                    `ToString`

  `TimeOnly`              `TimeOnly.FromDateTime(DateTime.Now)`     `AddHours`,
                                                                    `AddMinutes`,
                                                                    `ToString`
  -----------------------------------------------------------------------------------------

##### Parsing and common numeric members

``` csharp
int n = int.Parse("123");
bool parsed = int.TryParse("123", out int result);

int max = int.MaxValue;
double root = Math.Sqrt(81);
double rounded = Math.Round(3.14159, 2);
decimal money = decimal.Round(19.995m, 2, MidpointRounding.AwayFromZero);

char.IsLetter('A');
char.IsDigit('7');
char.ToUpperInvariant('a');
```

`Parse` throws for invalid input; `TryParse` returns `false`. `Math`
methods generally operate on `double`; use the `decimal` methods when
working with decimal values.

#### 2. Type categories and declarations

##### Value types and reference types

-   **Value types:** built-in numeric types, `bool`, `char`, `enum`, and
    user-defined `struct`/`record struct`. Assignment copies the value.
-   **Reference types:** `class`, `string`, arrays, delegates,
    interfaces, and most collection classes. Assignment copies the
    reference.
-   `string` is a reference type but is immutable.
-   `object` and `dynamic` are different: `object` uses normal
    compile-time member checking; `dynamic` defers member binding to
    runtime.

``` csharp
int? optionalNumber = null;           // Nullable<int>
DateTime? optionalDate = null;
dynamic item = 10;
item = "now a string";

enum Status { Pending, Complete }
Status status = Status.Pending;
string statusText = status.ToString();
Status parsedStatus = Enum.Parse<Status>("Complete");
bool validStatus = Enum.TryParse("Complete", out Status parsed);
```

##### User-defined and related types

  -------------------------------------------------------------------------------------------------
  Construct                           Example / purpose
  ----------------------------------- -------------------------------------------------------------
  `class`                             `class Customer { public string Name { get; set; } = ""; }`
                                      --- reference type

  `struct`                            `struct Point { public int X; public int Y; }` --- value type

  `record`                            `record Person(string Name, int Age);` --- data-oriented
                                      reference type by default

  `record struct`                     `record struct Coordinate(int X, int Y);` --- data-oriented
                                      value type

  `interface`                         `interface IRun { void Run(); }` --- contract

  `enum`                              `enum Color { Red, Green, Blue }` --- named constants

  `delegate`                          `Func<int, int> square = x => x * x;` --- callable reference

  Anonymous type                      `var x = new { Id = 1, Name = "A" };` --- compiler-generated
                                      type

  Tuple                               `(int Id, string Name) row = (1, "A");` --- grouped values

  Pointer                             `int* p` --- unsafe code only; requires an unsafe context
  -------------------------------------------------------------------------------------------------

Common delegate types:

``` csharp
Func<int, int> square = x => x * x;        // Returns a value
Action<string> print = Console.WriteLine;  // Returns void
Predicate<int> isEven = x => x % 2 == 0;   // Returns bool
```

#### 3. Arrays and collection interfaces

##### Array and array shapes

Arrays have a fixed length after creation.

``` csharp
int[] numbers = { 3, 1, 2 };
int[,] matrix = new int[2, 3];       // Multidimensional array
int[][] jagged = new int[2][];       // Array of arrays
jagged[0] = new[] { 1, 2 };

Array.Sort(numbers);
Array.Reverse(numbers);
int index = Array.IndexOf(numbers, 2);
int[] copy = numbers.ToArray();      // LINQ extension method
int length = numbers.Length;
```

Useful `Array` members: `Length`, `Rank`, `GetLength(dimension)`,
`GetValue`, `SetValue`, `Copy`, `CopyTo`, `Clear`, `Sort`, `Reverse`,
`IndexOf`, `Find`, `FindIndex`, `Exists`, `TrueForAll`, `Resize`
(changes the array reference passed by `ref`).

##### Important collection interfaces

  -------------------------------------------------------------------------
  Interface                             Meaning
  ------------------------------------- -----------------------------------
  `IEnumerable<T>`                      Can be iterated

  `ICollection<T>`                      Basic mutable collection operations

  `IList<T>`                            Indexed collection contract

  `IReadOnlyCollection<T>`              Read-only access to count and
                                        enumeration

  `IReadOnlyList<T>`                    Read-only indexed access

  `IDictionary<TKey, TValue>`           Key/value mapping contract

  `IReadOnlyDictionary<TKey, TValue>`   Read-only key/value access

  `ISet<T>`                             Set operations contract

  `IQueryable<T>`                       Query expression intended for a
                                        provider, such as EF Core
  -------------------------------------------------------------------------

"Read-only" interfaces restrict operations through that interface; they
do not guarantee the underlying collection cannot change elsewhere.

#### 4. Standard collections and their built-in members

##### `List<T>`

Resizable, indexed sequence; allows duplicates.

``` csharp
var list = new List<int> { 3, 1, 2 };
list.Add(4);
list.AddRange(new[] { 5, 6 });
list.Insert(0, 9);
bool hasTwo = list.Contains(2);
int position = list.IndexOf(2);
list.Remove(3);
list.RemoveAt(0);
list.Sort();
list.Reverse();
int[] array = list.ToArray();
int count = list.Count;
list.Clear();
```

Key members: `Add`, `AddRange`, `Insert`, `InsertRange`, `Remove`,
`RemoveAt`, `RemoveAll`, `RemoveRange`, `Contains`, `IndexOf`,
`LastIndexOf`, `Find`, `FindAll`, `FindIndex`, `Exists`, `TrueForAll`,
`Sort`, `Reverse`, `BinarySearch`, `ConvertAll`, `CopyTo`, `GetRange`,
`ForEach`, `ToArray`, `Count`, `Capacity`, `Clear`.

##### `ArrayList` (legacy, non-generic)

Stores `object` values, so it can mix types but may require
boxing/unboxing and casts. Prefer `List<T>` in new code.

``` csharp
var items = new System.Collections.ArrayList { 1, "text" };
items.Add(3);
items.Insert(0, "first");
items.Remove(1);
items.RemoveAt(0);
bool found = items.Contains("text");
int count = items.Count;
```

Key members: `Add`, `AddRange`, `Insert`, `InsertRange`, `Remove`,
`RemoveAt`, `RemoveRange`, `Contains`, `IndexOf`, `LastIndexOf`, `Sort`,
`Reverse`, `BinarySearch`, `CopyTo`, `ToArray`, `Count`, `Capacity`,
`Clear`.

##### `Dictionary<TKey, TValue>`

Hash-based key/value collection; keys must be unique.

``` csharp
var ages = new Dictionary<string, int>();
ages.Add("Ava", 30);
ages["Ben"] = 25;                    // Adds or updates
bool exists = ages.ContainsKey("Ava");
bool gotValue = ages.TryGetValue("Ava", out int age);
ages.Remove("Ben");
var keys = ages.Keys;
var values = ages.Values;
int count = ages.Count;
```

Key members: `Add`, indexer `[]`, `TryAdd`, `TryGetValue`,
`ContainsKey`, `ContainsValue`, `Remove`, `Clear`, `EnsureCapacity`,
`TrimExcess`, `GetValueOrDefault` (extension method on supported target
frameworks), `Keys`, `Values`, `Count`.

##### `SortedDictionary<TKey, TValue>` and `SortedList<TKey, TValue>`

Both keep keys sorted by comparer. `SortedDictionary` is tree-based;
`SortedList` stores sorted key/value arrays and can be efficient for
relatively small collections with infrequent inserts/removals.

``` csharp
var sorted = new SortedDictionary<int, string>
{
    [3] = "C",
    [1] = "A"
};
sorted.Add(2, "B");
bool contains = sorted.ContainsKey(2);
sorted.Remove(1);

var sortedList = new SortedList<int, string> { [2] = "B", [1] = "A" };
int keyAtZero = sortedList.Keys[0];
```

Common members: `Add`, indexer `[]`, `ContainsKey`, `ContainsValue`,
`TryGetValue`, `Remove`, `Clear`, `Keys`, `Values`, `Count`.
`SortedList` also has indexed key/value access and
`IndexOfKey`/`IndexOfValue`.

##### `HashSet<T>` and `SortedSet<T>`

Sets store unique values. `HashSet<T>` is hash-based and unordered;
`SortedSet<T>` maintains sort order.

``` csharp
var set = new HashSet<int> { 1, 2, 2 };
set.Add(3);
set.Contains(2);
set.UnionWith(new[] { 3, 4 });
set.IntersectWith(new[] { 2, 4 });
set.ExceptWith(new[] { 1 });
set.SymmetricExceptWith(new[] { 4, 5 });
bool subset = set.IsSubsetOf(new[] { 2, 4, 5 });

var sortedSet = new SortedSet<int> { 5, 1, 3 };
int min = sortedSet.Min;
int max = sortedSet.Max;
var range = sortedSet.GetViewBetween(2, 5);
```

Key members: `Add`, `Remove`, `RemoveWhere`, `Contains`, `Clear`,
`CopyTo`, `SetEquals`, `IsSubsetOf`, `IsSupersetOf`, `IsProperSubsetOf`,
`IsProperSupersetOf`, `Overlaps`, `UnionWith`, `IntersectWith`,
`ExceptWith`, `SymmetricExceptWith`, `Count`. `SortedSet<T>`
additionally offers `Min`, `Max`, `GetViewBetween`, `Reverse`.

##### `Stack<T>` (LIFO)

Last in, first out.

``` csharp
var stack = new Stack<int>();
stack.Push(10);
stack.Push(20);
int top = stack.Peek();       // 20, does not remove
int removed = stack.Pop();    // 20, removes it
bool hasTen = stack.Contains(10);
stack.TryPop(out int value);
stack.TryPeek(out int next);
```

Key members: `Push`, `PushRange`, `Pop`, `TryPop`, `Peek`, `TryPeek`,
`Contains`, `Clear`, `ToArray`, `CopyTo`, `Count`.

##### `Queue<T>` (FIFO)

First in, first out.

``` csharp
var queue = new Queue<string>();
queue.Enqueue("first");
queue.Enqueue("second");
string next = queue.Peek();       // "first", does not remove
string removed = queue.Dequeue(); // "first", removes it
queue.TryDequeue(out string? item);
queue.TryPeek(out string? front);
```

Key members: `Enqueue`, `EnqueueRange` (modern .NET), `Dequeue`,
`TryDequeue`, `Peek`, `TryPeek`, `Contains`, `Clear`, `ToArray`,
`CopyTo`, `Count`.

##### `PriorityQueue<TElement, TPriority>` and heaps

Built-in min-priority queue (.NET 6+); there is no separate `MinHeap<T>`/`MaxHeap<T>`. Usage, max-heap comparers, Top-K, and pitfalls are explained once in [Data Structures and Algorithms → Priority queues and binary heaps](02-data-structures-and-algorithms.md#4-priority-queues-and-binary-heaps).

##### `LinkedList<T>`

Doubly linked list. Adding/removing at a known node is O(1); finding a
value is O(n). It does not support index-based access.

``` csharp
var linked = new LinkedList<string>();
linked.AddFirst("B");
linked.AddFirst("A");
linked.AddLast("D");
LinkedListNode<string>? node = linked.Find("B");
if (node is not null)
{
    linked.AddAfter(node, "C");
    linked.Remove(node);
}
```

Key members: `AddFirst`, `AddLast`, `AddBefore`, `AddAfter`, `Find`,
`FindLast`, `Remove`, `RemoveFirst`, `RemoveLast`, `Clear`, `Contains`,
`CopyTo`, `First`, `Last`, `Count`.

#### 5. Additional collection families

##### Immutable collections (`System.Collections.Immutable`)

Useful when a collection should not change after creation; updates
return a new collection. The package/reference may need to be added
depending on the target framework.

-   `ImmutableArray<T>`
-   `ImmutableList<T>`
-   `ImmutableDictionary<TKey, TValue>`
-   `ImmutableHashSet<T>`
-   `ImmutableSortedDictionary<TKey, TValue>`
-   `ImmutableSortedSet<T>`
-   `ImmutableQueue<T>`
-   `ImmutableStack<T>`

Typical members include `Add`, `AddRange`, `SetItem`, `Remove`,
`RemoveRange`, `Contains`, `ToBuilder`, plus the relevant lookup or
set-operation methods.

``` csharp
using System.Collections.Immutable;

ImmutableList<int> original = ImmutableList.Create(1, 2);
ImmutableList<int> updated = original.Add(3); // original remains unchanged
```

##### Read-only wrappers

-   `ReadOnlyCollection<T>` --- read-only wrapper around an existing
    `IList<T>`.
-   `ReadOnlyDictionary<TKey, TValue>` --- read-only wrapper around a
    dictionary.
-   `IReadOnlyList<T>`, `IReadOnlyCollection<T>`,
    `IReadOnlyDictionary<TKey, TValue>` --- read-only access interfaces.

A wrapper does not make the underlying collection immutable; changes
made through another reference can still be visible.

##### Thread-safe and producer/consumer collections

Namespace: `System.Collections.Concurrent`.

  -----------------------------------------------------------------------------------------
  Type                                   Purpose                 Common members
  -------------------------------------- ----------------------- --------------------------
  `ConcurrentDictionary<TKey, TValue>`   Concurrent key/value    `TryAdd`, `TryGetValue`,
                                         access                  `TryRemove`, `TryUpdate`,
                                                                 `GetOrAdd`, `AddOrUpdate`,
                                                                 `ContainsKey`

  `ConcurrentQueue<T>`                   Thread-safe FIFO        `Enqueue`, `TryDequeue`,
                                                                 `TryPeek`, `ToArray`,
                                                                 `Count`

  `ConcurrentStack<T>`                   Thread-safe LIFO        `Push`, `TryPop`,
                                                                 `TryPeek`, `ToArray`,
                                                                 `Count`

  `ConcurrentBag<T>`                     Thread-safe unordered   `Add`, `TryTake`,
                                         bag                     `TryPeek`, `ToArray`,
                                                                 `Count`

  `BlockingCollection<T>`                Bounded/blocking        `Add`, `TryAdd`, `Take`,
                                         producer-consumer       `TryTake`,
                                         wrapper                 `CompleteAdding`,
                                                                 `GetConsumingEnumerable`
  -----------------------------------------------------------------------------------------

Thread-safe collection operations do not automatically make a multi-step
business operation atomic.

##### Specialized collections

Namespace: `System.Collections.Specialized` unless otherwise noted.

-   `BitArray` --- compact bit storage; members include `Get`, `Set`,
    `SetAll`, `And`, `Or`, `Xor`, `Not`, `Length`.
-   `OrderedDictionary` --- key/value entries accessible by key or
    index; members include `Add`, `Insert`, `Remove`, `RemoveAt`,
    `Contains`, `IndexOf`, `GetKey`, `GetByIndex`, `SetAt`.
-   `NameValueCollection` --- string keys with potentially multiple
    string values; members include `Add`, `Set`, `Get`, `GetValues`,
    `GetKey`, `Remove`, `AllKeys`.
-   `StringCollection` --- legacy string collection; members include
    `Add`, `AddRange`, `Contains`, `IndexOf`, `Insert`, `Remove`,
    `RemoveAt`.
-   `HybridDictionary` --- legacy dictionary optimized to switch
    representation for small/large collections.
-   `StringDictionary` --- legacy string-key/string-value dictionary.

For new general-purpose code, prefer generic collections such as
`List<T>` and `Dictionary<TKey, TValue>`.

##### UI and data collections

-   `ObservableCollection<T>` (`System.Collections.ObjectModel`) ---
    raises change notifications for UI binding. Members include `Add`,
    `Remove`, `Move`, `Clear`; event: `CollectionChanged`.
-   `Collection<T>` --- customizable wrapper collection.
-   `ReadOnlyObservableCollection<T>` --- read-only observable wrapper.
-   `DataTable` / `DataSet` (`System.Data`) --- ADO.NET in-memory
    relational data, not general-purpose collection replacements.

##### Memory and views (not ordinary collections)

-   `Span<T>` / `ReadOnlySpan<T>` --- stack-only views over contiguous
    memory; members include `Length`, `Slice`, `CopyTo`, `ToArray`,
    `IndexOf`.
-   `Memory<T>` / `ReadOnlyMemory<T>` --- heap-storable memory views
    suitable for async workflows; members include `Length`, `Slice`,
    `Span`, `ToArray`.
-   `ArraySegment<T>` --- view over part of an array; members include
    `Array`, `Offset`, `Count`, `AsSpan`, `ToArray`.
-   `Memory<T>` and `Span<T>` are not resizable collections; they view
    existing memory.

#### 6. LINQ methods commonly used with collections

LINQ methods are extension methods, not members unique to `List<T>` or
`Dictionary<TKey,TValue>`. Add `using System.Linq;` when needed (often
implicit in modern .NET templates).

``` csharp
var numbers = new[] { 5, 2, 8, 2, 1 };

var filtered = numbers.Where(x => x > 2);
var projected = numbers.Select(x => x * 10);
var ordered = numbers.OrderBy(x => x);
var distinct = numbers.Distinct();
var first = numbers.First();
var maybeFirst = numbers.FirstOrDefault();
bool anyLarge = numbers.Any(x => x > 7);
bool allPositive = numbers.All(x => x > 0);
int total = numbers.Sum();
int count = numbers.Count();
var groups = numbers.GroupBy(x => x);
var lookup = numbers.ToLookup(x => x % 2);
var list = numbers.ToList();
var set = numbers.ToHashSet();
var map = numbers.Distinct().ToDictionary(x => x, x => x * x);
```

Common LINQ operators: `Where`, `Select`, `SelectMany`, `OrderBy`,
`OrderByDescending`, `ThenBy`, `ThenByDescending`, `GroupBy`, `Join`,
`GroupJoin`, `Distinct`, `DistinctBy`, `Union`, `Intersect`, `Except`,
`Concat`, `Append`, `Prepend`, `Take`, `TakeLast`, `Skip`, `SkipLast`,
`Chunk`, `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `Last`,
`LastOrDefault`, `Any`, `All`, `Contains`, `Count`, `LongCount`, `Sum`,
`Min`, `Max`, `Average`, `Aggregate`, `SequenceEqual`, `ToArray`,
`ToList`, `ToDictionary`, `ToHashSet`, `ToLookup`.

`IQueryable<T>` builds expression trees for a query provider (for
example, EF Core); `IEnumerable<T>` generally executes .NET iteration.
With EF Core, be mindful of which operators run in the database and when
materialization occurs.

#### 7. Quick selection guide

  Need                                            Good starting choice
  ----------------------------------------------- --------------------------------------------------
  Fixed-size indexed data                         `T[]`
  Resizable indexed sequence                      `List<T>`
  Unique values, fast membership checks           `HashSet<T>`
  Unique values in sorted order                   `SortedSet<T>`
  Key/value lookup                                `Dictionary<TKey, TValue>`
  Key/value lookup in key order                   `SortedDictionary<TKey, TValue>`
  FIFO processing                                 `Queue<T>`
  LIFO processing                                 `Stack<T>`
  Retrieve item by priority                       `PriorityQueue<TElement, TPriority>`
  Frequent insertion/removal around known nodes   `LinkedList<T>`
  Concurrent access                               Appropriate `System.Collections.Concurrent` type
  Immutable data                                  `System.Collections.Immutable` type
  High-performance memory slicing                 `Span<T>` / `Memory<T>`

#### 8. Interview reminders

-   Prefer generic collections over legacy `ArrayList`, `Hashtable`, and
    similar non-generic types.
-   `List<T>.Count` is the number of elements; an array's `Length` is
    its element count.
-   Dictionary keys must be unique; set elements must be unique
    according to the equality comparer.
-   `Queue<T>` is FIFO; `Stack<T>` is LIFO;
    `PriorityQueue<TElement, TPriority>` removes the lowest priority
    first by default.
-   A heap is a data-structure concept;
    `PriorityQueue<TElement, TPriority>` is the built-in heap-backed
    priority queue. There is no separate standard `MaxHeap<T>` type.
-   `byte` is often appropriate for binary data, `int` for typical
    integer values, `long` for larger counts, `double` for general
    floating-point math, and `decimal` for decimal financial
    calculations.
-   `uint`/`ulong` are not automatically better for non-negative values;
    use them when their unsigned semantics or range is actually needed.
-   Use `TryParse`, `TryGetValue`, `TryDequeue`, and similar `Try...`
    APIs when failure is an expected possibility.

---

<a id="senior-csharp-code-review-refactoring-interview-guide-md"></a>

## Senior C#/.NET Developer: Code Review, Refactoring & Interview Guide

A practical reference for senior-developer interviews (“viva”) and for reviewing code and mentoring a .NET team. Examples target modern C# and ASP.NET Core; adapt them to the project's language version and conventions.

#### 1. How to approach a code review

Review in this order:

1. **Correctness:** Does the code meet the requirement? Are edge cases handled?
2. **Security and privacy:** Is authorization enforced? Are inputs validated? Are secrets and sensitive data kept out of logs?
3. **Data integrity:** Are transactions, concurrency, and duplicate requests handled correctly?
4. **Performance:** Are database calls, allocations, network calls, and payload sizes reasonable?
5. **Maintainability:** Are names clear, responsibilities focused, and abstractions justified?
6. **Tests:** Are important success, failure, boundary, and authorization cases covered?
7. **Consistency:** Does the change follow established project conventions?

##### Useful review comments

Prefer specific, respectful comments that explain the risk and suggest an action.

- **Blocking:** “This endpoint checks authentication but not ownership of the requested record. Could we verify the current user is authorized to access this resource?”
- **Suggestion:** “Could we project only the fields returned by the API? This query currently loads more columns than the response needs.”
- **Question:** “What should happen if two requests update this record at the same time?”
- **Nit:** “Could we rename `data` to `activeUsers` to make its purpose clearer?”

Do not request changes just to impose personal style. Separate correctness/security blockers from optional improvements.

#### 2. Modern C# patterns: old vs. current

##### 2.1 Short-circuiting

`&&` skips the right-hand expression when the left side is `false`; `||` skips it when the left side is `true`.

```csharp
// Safe: IsActive is checked only when user is not null.
if (user is not null && user.IsActive)
{
    SendEmail(user);
}

// Also short-circuits: the fallback is evaluated only when needed.
var displayName = user?.Name ?? "Guest";
```

**Review note:** Do not put important side effects in complicated Boolean expressions. Prefer a normal `if` when it makes the behavior clearer.

##### 2.2 Ternary operator

```csharp
// Old
string label;
if (isActive)
    label = "Active";
else
    label = "Inactive";

// Modern: suitable for one simple choice
var label = isActive ? "Active" : "Inactive";
```

Avoid nested ternaries when they make the expression difficult to scan.

##### 2.3 Null-conditional and null-coalescing operators

```csharp
// Old
string name = "Guest";
if (user is not null && user.Name is not null)
    name = user.Name;

// Modern
string name = user?.Name ?? "Guest";
```

- `?.` safely accesses a member when the receiver may be null.
- `??` supplies a fallback when the left operand is null.
- `??=` assigns a fallback only when the variable is null.

```csharp
options ??= new AppOptions();
```

Use nullable reference types (`string?`, `User?`) intentionally; do not use null-conditional operators to hide invalid states that should be rejected.

##### 2.4 Type patterns

Switch expressions are shown above (section 2.4 and Case 3). Pattern matching can also combine a type check with variable capture:

```csharp
if (value is Customer customer && customer.IsActive)
{
    Process(customer);
}
```

##### 2.5 Expression-bodied members

Use for simple expressions, not for long workflows.

```csharp
public decimal CalculateTotal(decimal price, int quantity)
    => price * quantity;
```

##### 2.6 Dictionary lookup

```csharp
// Avoid two lookups
if (rates.ContainsKey(country))
    return amount * rates[country];

// One lookup
return rates.TryGetValue(country, out var rate)
    ? amount * rate
    : amount * defaultRate;
```

##### 2.7 String building

For a small fixed number of interpolations, normal interpolation is clear. For repeated appends in a loop, consider `StringBuilder`.

```csharp
var builder = new StringBuilder();
foreach (var item in items)
    builder.AppendLine(item.Name);

var result = builder.ToString();
```

Do not optimize string concatenation without considering readability and measuring real bottlenecks.

#### 3. Refactoring: recognize smells and choose a safe change

| Code smell | Refactoring direction |
|---|---|
| Long method | Extract cohesive methods with meaningful names |
| Large class / many reasons to change | Split by responsibility |
| Duplicated business rules | Centralize the rule in an appropriate domain/application component |
| Deep nesting | Use guard clauses where they improve clarity |
| Magic values | Use named constants, enums, or configuration |
| Tight coupling | Introduce an abstraction only where it creates a useful seam |
| Primitive obsession | Consider a value object when the value has domain rules |
| Over-engineering | Remove unnecessary patterns and abstractions |
| Repeated database calls | Review query shape, batching, and loading strategy |

##### Example: guard clauses and a focused method

```csharp
// Before
public decimal CalculateDiscount(Order? order)
{
    if (order != null)
    {
        if (order.TotalAmount > 1_000m)
            return order.TotalAmount * 0.10m;
        else
            return order.TotalAmount * 0.05m;
    }

    throw new ArgumentNullException(nameof(order));
}

// After
public decimal CalculateDiscount(Order order)
{
    ArgumentNullException.ThrowIfNull(order);

    var rate = order.TotalAmount > 1_000m ? 0.10m : 0.05m;
    return order.TotalAmount * rate;
}
```

The method parameter is non-nullable in the refactored version, but the runtime guard still protects callers that pass null. Keep the exception behavior and business rules consistent with the existing contract.

#### 4. Async, EF Core, and performance

##### Async I/O

```csharp
// Avoid blocking asynchronous work with .Result or .Wait()
var body = await httpClient.GetStringAsync(url, cancellationToken);
```

- Use asynchronous APIs for database, HTTP, and file I/O.
- Pass `CancellationToken` through layers when cancellation is supported.
- Do not wrap naturally asynchronous I/O in `Task.Run`.
- Use `Task.WhenAll` for independent async operations only when concurrency and downstream capacity are appropriate.
- Avoid unbounded fan-out that can overload a database or external service.

##### EF Core query shape

See the projection example in section 2.6 above; EF Core performance is covered in depth in [EF Core, LINQ, and SQL](05-efcore-linq-and-sql.md). Key review checks:

- Project only the fields needed by the caller.
- Use `AsNoTracking()` for read-only entity queries when change tracking is unnecessary.
- Filter and sort in SQL; avoid materializing a large table before filtering.
- Check for N+1 queries and unnecessary `Include`s.
- Add indexes based on query patterns and execution plans, not guesswork.
- Use pagination for large result sets; consider keyset pagination when offset paging becomes expensive.
- Use transactions when multiple writes must succeed or fail together.
- Do not share one `DbContext` across concurrent operations; it is not thread-safe.
- Measure before introducing compiled queries, pooling, or low-level optimizations.

#### 5. REST API, security, and reliability

A senior review should ask:

- Is authentication configured correctly, and is **authorization** checked for this specific resource?
- Are request DTOs validated at the API boundary?
- Are responses consistent, and are errors returned using a suitable format such as Problem Details?
- Are status codes appropriate (`400`, `401`, `403`, `404`, `409`, `429`, `5xx`)?
- Is pagination and filtering bounded to prevent excessive work?
- Are rate limits needed for the endpoint?
- Are secrets stored in an approved secret store or managed identity flow, not source control?
- Are logs useful without exposing credentials, tokens, or sensitive personal data?
- Are retries limited to transient failures and safe to repeat?
- Do state-changing operations need idempotency or concurrency control?
- Are CORS and CSRF protections appropriate for the actual authentication model?

**Important:** Parameterized EF Core queries help prevent SQL injection, but raw SQL and dynamic query construction still need careful review. Input validation is not a substitute for authorization.

#### 6. Clean Architecture and CQRS

Use boundaries to protect business rules, not merely to create folders. CQRS does **not** automatically require microservices, event sourcing, or a message broker. Layers and trade-offs are explained in [Architecture and Design Patterns](06-architecture-and-design-patterns.md).

#### 7. Testing and safe refactoring workflow

1. Clarify expected behavior and edge cases.
2. Add or identify characterization tests around existing behavior.
3. Make one small refactoring at a time.
4. Keep behavior changes separate from structure-only refactoring where possible.
5. Run unit tests, integration tests, analyzers, and relevant security checks.
6. Review the diff for accidental scope changes and generated-file noise.
7. Measure performance changes with representative workloads.
8. Deploy progressively when risk warrants it; monitor logs, metrics, and errors.
9. Roll back or fix forward according to the team's release strategy.

A refactor should preserve externally observable behavior unless the change explicitly includes a requirement fix.

#### 8. Senior-level interview questions and model answers

##### Q1. How do you ensure code quality in a large project?
**Answer:** Establish shared conventions, automated formatting and analyzers, meaningful tests, CI quality gates, and a consistent PR process. Review correctness, security, data integrity, performance, and maintainability. Track recurring defects and improve the system rather than relying only on manual review.

##### Q2. What code smells do you look for?
**Answer:** Long methods, large classes, duplication, deep nesting, unclear naming, hidden side effects, excessive coupling, and abstractions that add complexity without value. I confirm the underlying problem before choosing a refactoring.

##### Q3. How do you balance readability and performance?
**Answer:** Start with the clearest correct implementation, measure using profiling or representative benchmarks, identify the bottleneck, and optimize that part. Keep a test or benchmark to verify the improvement and avoid speculative micro-optimization.

##### Q4. How do you handle breaking API changes?
**Answer:** Prefer additive, backward-compatible changes. If breaking changes are unavoidable, version the contract where appropriate, communicate a migration plan, support a transition period if feasible, update contract tests and documentation, and monitor adoption before retiring the old version.

##### Q5. Refactoring vs. rewriting?
**Answer:** Refactoring changes internal structure while preserving behavior. Rewriting replaces an implementation and carries more risk because existing behavior and edge cases can be lost. I prefer incremental refactoring when tests and architecture allow it; a rewrite needs a clear business and technical case.

##### Q6. How do you refactor a tightly coupled class?
**Answer:** Identify its responsibilities and dependencies, add tests around current behavior, separate cohesive responsibilities, inject external dependencies through constructors, and introduce interfaces at meaningful boundaries. Avoid creating abstractions solely to satisfy a pattern.

##### Q7. How do you optimize EF Core?
**Answer:** Inspect generated SQL and execution plans, project only needed fields, filter in the database, use no-tracking reads when appropriate, address N+1 queries, apply suitable indexes and pagination, and measure. I also check transaction scope and concurrency behavior.

##### Q8. When is a design pattern overkill?
**Answer:** When it adds indirection or concepts without solving a current, credible problem. Choose the simplest design that meets current requirements while leaving reasonable room for likely change.

##### Q9. What is the difference between `Task.WhenAll` and `Parallel.ForEach`?
**Answer:** `Task.WhenAll` coordinates multiple tasks, often asynchronous I/O tasks. `Parallel.ForEach` is for parallel synchronous work; for async iteration, consider `Parallel.ForEachAsync` with an explicit degree of parallelism. Choose based on workload and resource limits.

##### Q10. How do you prevent race conditions when updating data?
**Answer:** Define the consistency requirement first. Depending on the case, use database transactions, optimistic concurrency tokens (such as a row version), unique constraints, appropriate isolation, or atomic updates. Application-level checks alone may not protect against concurrent requests.

##### Q11. How would you investigate a slow API?
**Answer:** Establish a baseline, inspect latency percentiles and traces, separate time spent in the application, database, and external dependencies, check query plans and N+1 behavior, review allocations and serialization, then make and measure one targeted change.

##### Q12. How do you mentor a team through code review?
**Answer:** Explain the impact rather than just saying “wrong,” distinguish blockers from suggestions, ask questions when context is unclear, share examples and team conventions, and follow up on recurring patterns through pairing, tests, analyzers, or documentation.

#### 9. A concise PR checklist

- [ ] Requirement and edge cases are covered.
- [ ] Authorization and input validation are correct.
- [ ] No secrets or sensitive information are exposed.
- [ ] Errors and logs are useful and safe.
- [ ] Async calls do not block; cancellation is handled where relevant.
- [ ] Queries are bounded and fetch only necessary data.
- [ ] Transactions and concurrency are handled appropriately.
- [ ] Tests cover success, failure, boundary, and access-control cases.
- [ ] Public API and database changes are backward-compatible or have a migration plan.
- [ ] The change is focused, readable, and consistent with project conventions.
- [ ] CI checks pass; performance claims are measured.

#### Final principle

**A senior developer does more than make code shorter.** They protect behavior, security, data integrity, and operability while making the design easier for the next person to understand and change.

---

<a id="shallow-vs-deep-clone-md"></a>

## Shallow Clone vs. Deep Clone in C#

Cloning creates a new object based on an existing object. The key difference is whether referenced objects are shared or copied.

#### 1. Shallow clone: `MemberwiseClone()`

`Object.MemberwiseClone()` is a **protected** method provided by .NET. It creates a new instance and copies the current object's fields:

- **Value-type fields** (such as `int` and `bool`) are copied by value.
- **Reference-type fields** (such as `Address`) copy the reference, so the original and clone point to the same referenced object.

##### Example

```csharp
public sealed class Address
{
    public string City { get; set; } = "";
}

public class Person
{
    public string Name { get; set; } = "";
    public Address Address { get; set; } = new();

    public Person ShallowClone() =>
        (Person)MemberwiseClone();
}

var original = new Person
{
    Name = "Alice",
    Address = new Address { City = "New York" }
};

var clone = original.ShallowClone();

clone.Name = "Bob";
clone.Address.City = "London";

Console.WriteLine(original.Name);         // Alice
Console.WriteLine(original.Address.City); // London
Console.WriteLine(ReferenceEquals(original.Address, clone.Address)); // True
```

`Name` is a `string`. Assigning a different string to `clone.Name` does not change `original.Name`. The `Address` property, however, points to the same `Address` instance in both objects, so changing its `City` is visible through both references.

> `MemberwiseClone()` is shallow: it copies fields, not the entire referenced object graph.

#### 2. Deep clone

A **deep clone** creates a separate copy of the object and of the mutable objects it references, recursively as required by the application's data model.

##### Example

```csharp
public class Person
{
    public string Name { get; set; } = "";
    public Address Address { get; set; } = new();

    public Person DeepClone() =>
        new()
        {
            Name = Name,
            Address = new Address
            {
                City = Address.City
            }
        };
}

public sealed class Address
{
    public string City { get; set; } = "";
}

var original = new Person
{
    Name = "Alice",
    Address = new Address { City = "New York" }
};

var clone = original.DeepClone();
clone.Address.City = "London";

Console.WriteLine(original.Address.City); // New York
Console.WriteLine(clone.Address.City);    // London
Console.WriteLine(ReferenceEquals(original.Address, clone.Address)); // False
```

The `Address` object is explicitly created again, so changes to the cloned address do not affect the original. This implementation is deep only for the fields shown; if the model gains more mutable reference-type fields, those must be copied too.

#### 3. Comparison

| Aspect | Shallow clone (`MemberwiseClone`) | Deep clone |
|---|---|---|
| New top-level object | Yes | Yes |
| Value-type fields | Copied | Copied |
| Reference-type fields | References are copied and shared | Referenced mutable objects are copied as defined by the cloning logic |
| Nested changes independent? | Not necessarily | Yes, for objects that are deeply copied |
| Built into `Object`? | Yes | No universal built-in deep-clone method |
| Implementation effort | Minimal | Requires explicit copying or a suitable cloning strategy |

#### 4. When should you use each?

- **Use a shallow clone** when sharing referenced objects is intentional or when the object is designed so that sharing is safe.
- **Use a deep clone** when the clone must be independently mutable, including its nested objects.
- **Consider immutable types or records** when you need safer value-like data handling. A C# record's `with` expression is a *shallow copy*; nested reference-type members remain shared unless you explicitly replace or copy them.

#### 5. Important considerations

- `MemberwiseClone()` is `protected`, so you normally expose it through a method inside the class or a derived class.
- `ICloneable` exists in .NET, but its `Clone()` contract does not specify whether the result is shallow or deep. For public APIs, a clearly named method such as `ShallowClone()` or `DeepClone()` is less ambiguous.
- JSON serialization/deserialization can sometimes be used to copy simple DTO graphs, but it may change behavior for types with cycles, special reference relationships, unsupported members, custom converters, or non-serializable state. It should not be treated as a universal deep-clone solution.
- Deep cloning object graphs with cycles or shared references needs care: a correct implementation may need a map of already-cloned objects to preserve relationships and avoid infinite recursion.

**In short:** `MemberwiseClone()` creates a new outer object but shares referenced objects. A deep clone also creates independent copies of the nested objects that need to be independent.

---

<a id="get-property-using-type-csharp-md"></a>

## Reflection: Get a Property by Name Using `Type`

When the property name is only known at **runtime** (for example, from configuration, a sort-by query parameter, or a CSV column), use reflection. `Type.GetProperty()` returns a `PropertyInfo`, which can read or write the value.

```csharp
using System;
using System.Reflection;

public class MyClass
{
    public int MyProperty { get; set; }
}

public class Program
{
    public static void Main()
    {
        var obj = new MyClass { MyProperty = 42 };
        string propertyName = "MyProperty";

        Type type = obj.GetType();                                   // 1. runtime type
        PropertyInfo? propertyInfo = type.GetProperty(propertyName); // 2. find property

        if (propertyInfo is not null)
        {
            object? value = propertyInfo.GetValue(obj);              // 3. read value
            Console.WriteLine($"The value of {propertyName} is: {value}");
        }
        else
        {
            Console.WriteLine($"Property '{propertyName}' was not found."); // 4. null if missing
        }
    }
}
```

**How it works**

1. `obj.GetType()` returns the object's runtime type.
2. `type.GetProperty(name)` searches for a **public** property with that name. It returns `null` if none exists.
3. `propertyInfo.GetValue(obj)` reads the value from that specific instance.

**If you only have the `Type`** (no instance yet), you can find the property, but you still need an instance to read it:

```csharp
Type type = typeof(MyClass);
PropertyInfo? propertyInfo = type.GetProperty("MyProperty");

var obj = new MyClass { MyProperty = 42 };
object? value = propertyInfo?.GetValue(obj);   // 42
propertyInfo?.SetValue(obj, 100);              // write works the same way
```

**Interview notes**

- Name matching is case-sensitive by default. For case-insensitive matching, use `type.GetProperty(name, BindingFlags.Public | BindingFlags.Instance | BindingFlags.IgnoreCase)`.
- Reflection is **less type-safe** (typos fail at runtime, not compile time) and **slower** than direct access. In hot paths, cache the `PropertyInfo`, or compile a delegate or expression tree once and reuse it.
- Prefer `nameof(MyClass.MyProperty)` over a hard-coded string when the name is known at compile time.
- Reflection can break with trimming/Native AOT unless the members are preserved.

---

## Additional Questpond question sets

### OOP / SOLID

###### 1. What is a method signature? (PDF p. 4)
**Answer:** In C#, a method signature consists of its <b>name and parameter types</b>, order, and passing modes (`ref`, `out`, or `in`). Return type and parameter names are not part of the signature, so methods cannot be overloaded by return type alone.

###### 2. What is the order of constructor execution? (PDF p. 4)
**Answer:** For a derived object, static initialization runs first (base type before derived type); then instance field initializers run base-to-derived, followed by instance constructors base-to-derived. A derived constructor's `base(...)` constructor call happens before its own body.

###### 3. What is a destructor, and how many can there be in a class? (PDF p. 4)
**Answer:** C#'s destructor syntax (`~Type()`) declares a finalizer, called nondeterministically by the garbage collector before reclaiming an object. A class can declare at most one finalizer, and it cannot be called directly.

###### 4. Why do you need a destructor when GC is there? (PDF p. 4)
**Answer:** GC reclaims managed memory, but it does not itself release external resources such as native handles. Prefer `IDisposable` and `using` for deterministic cleanup; a finalizer is a last-resort safety net for directly owned unmanaged resources.

###### 5. What happens when a constructor is private? (PDF p. 4)
**Answer:** Code outside the type cannot instantiate it directly. This is useful for static-only classes, controlled factory creation, or singleton-style construction; the type itself can still create instances.

###### 6. If an abstract class has a concrete method, can its child class have a method with the same name and modify it? (PDF p. 4)
**Answer:** Yes. If the base method is `virtual` or `abstract`, the child can override it using `override`. Otherwise a same-named method hides it (optionally with `new`), and calls through a base reference still use the base implementation.

###### 7. Abstract vs. static class (PDF p. 4)
**Answer:** An abstract class can have instance state, constructors, and concrete or abstract members, and is intended as a base type. A static class cannot be instantiated or inherited and contains only static members.

###### 8. Abstract class vs. interface (PDF p. 4)
**Answer:** An abstract class can share implementation and instance state but a class can inherit only one base class. An interface defines a capability/contract that a type can implement alongside other interfaces; modern C# also allows default interface implementations, but interfaces are not a substitute for class instance state.

###### 9. Why does an abstract class have constructors? (PDF p. 4)
**Answer:** The abstract class cannot be instantiated directly, but its constructor initializes the base portion of each concrete derived object and can enforce invariants or accept shared initialization data.

###### 10. When should you use an abstract class versus an interface? Give a real-time C# example. (PDF p. 5)
**Answer:** Use an interface for a capability shared by otherwise unrelated types, such as `IExportable.Export()`. Use an abstract class when related types need shared state or common implementation, such as `Employee` holding an employee ID and implementing shared payroll behavior.

###### 11. Implicit and explicit interface implementation (PDF p. 5)
**Answer:** An implicit implementation uses a public member with the matching signature and is callable through the class or interface. An explicit implementation is qualified (`void IFoo.Run()`), is callable only through an `IFoo` reference, and lets a class resolve same-signature members from multiple interfaces.

###### 12. What is a default interface method, and why do we need it? (PDF p. 5)
**Answer:** It is an interface member with a body. It lets an interface evolve with a default behavior without forcing every existing implementer to add the method; implementations can still provide their own behavior.

###### 13. Polymorphism in C# (PDF p. 5)
**Answer:** Polymorphism lets callers use a common base or interface while the actual implementation varies by runtime type. For example, calling `Draw()` on `IShape` invokes the implementation for the concrete shape.

###### 14. Method overriding with an example (PDF p. 5)
**Answer:** A base class marks a method `virtual` (or `abstract`), and a derived class provides an `override`. For example, `Animal.Speak()` can be virtual and `Dog.Speak()` can override it to return “Woof.”

###### 15. Difference between `virtual`, `override`, and `new` in C# (PDF p. 5)
**Answer:** `virtual` permits derived customization; `override` replaces a virtual/abstract base implementation with runtime dispatch; `new` hides a same-named member and dispatch is determined by the reference's compile-time type.

###### 16. Method hiding with an example (PDF p. 5)
**Answer:** A derived method with the same name/signature hides a base method, typically declared with `new`. If `Base b = new Derived()`, calling a hidden nonvirtual method uses `Base`'s version; calling through `Derived` uses the derived version.

###### 17. Is return type considered in method overloading? (PDF p. 5)
**Answer:** No. Overloads must differ by their parameter list (including parameter types/order or passing modes); changing only the return type causes a compile-time ambiguity/error.

###### 18. Why and when use method hiding? Give a real-time scenario. (PDF p. 5)
**Answer:** Hiding can provide a type-specific member when a base member is nonvirtual or when changing virtual dispatch would be unsafe, but it often surprises callers. For example, a derived reporting type may expose a specialized `Print()` only when referenced as that derived type; prefer composition or overriding when polymorphic behavior is intended.

###### 19. How do you extend a sealed class? (PDF p. 5)
**Answer:** You cannot derive from a sealed class. Use composition/wrapping, or add extension methods for stateless helper operations; extension methods do not alter the type or access its private state.

###### 20. What is an extension method? Write an example. (PDF p. 5)
**Answer:** It is a static method that can be invoked with instance-method syntax on a receiver type. Example: `public static bool IsBlank(this string? s) => string.IsNullOrWhiteSpace(s);` in a non-nested static class.

###### 21. What is the SOLID principle and why implement it? (PDF p. 5)
**Answer:** SOLID is a set of design principles—SRP, OCP, LSP, ISP, and DIP—that help keep code cohesive, extensible, substitutable, and loosely coupled. Apply them where they reduce change risk and improve testability; avoid adding abstractions without a real need.

###### 22. State all the SOLID principles with an example. (PDF p. 5)
**Answer:** **SRP:** separate invoice calculation from emailing. **OCP:** add a new discount strategy without editing stable checkout logic. **LSP:** any `IStorage` implementation honors the same read/write contract. **ISP:** split a large device interface into `IPrintable` and `IScannable`. **DIP:** business services depend on `IRepository`, not a concrete database class.

###### 23. What is the Liskov Substitution Principle? Explain with an example. (PDF p. 5)
**Answer:** A subtype must be usable wherever its base type is expected without violating that type's behavioral promises. If a `ReadOnlyFile` inherits a mutable `File` but throws on `Write`, it breaks callers' expectations; model read-only behavior with a narrower interface instead.

###### 24. Difference between Open/Closed and Liskov principles (PDF p. 5)
**Answer:** Open/Closed says stable code should allow extension without repeated modification, often through polymorphism. Liskov says those extensions must preserve the base contract so substitutes remain correct. OCP concerns change structure; LSP concerns behavioral correctness.

###### 25. Difference between Liskov Substitution and Interface Segregation (PDF p. 5)
**Answer:** LSP requires implementations/subtypes to honor the contract clients rely on. ISP says clients should not be forced to depend on methods they do not use, so split broad interfaces into focused contracts. ISP helps define smaller contracts; LSP governs valid implementations of them.

###### 26. Calculate salary of an employee based on grades (Strategy pattern) (PDF p. 5)
**Answer:** Define `ISalaryStrategy.Calculate(basePay)`, provide one implementation per grade/pay rule, and inject the selected strategy into an employee/payroll service. This keeps grade-specific calculations separate and permits adding a grade without embedding conditionals throughout payroll code.

---

### C#

##### Theory questions

###### 1. Value type vs. reference type (PDF p. 5)
**Answer:** A value-type variable holds its value directly (or inline in its containing object); assignment copies the value. A reference-type variable holds a reference to an object; assignment copies the reference, so both variables can refer to the same object. Structs and enums are value types; classes, arrays, and delegates are reference types.

###### 2. `ref` vs. `out`; give a real-time example and usage (PDF p. 5)
**Answer:** Both pass a variable by reference. `ref` requires the caller to initialize it and allows reading/writing; `out` need not be initialized, and the method must assign it before returning. Use `TryParse(string, out int)` to return success and a parsed value.

###### 3. Can `ref` be used in async programming? (PDF p. 5)
**Answer:** `ref`, `in`, and `out` parameters are not allowed on `async` methods because state may have to survive across suspension points. Return a result object/tuple or use an appropriate mutable object instead.

###### 4. `const` vs. `readonly` (PDF p. 5)
**Answer:** `const` is a compile-time constant, implicitly static, and must be initialized at declaration. `readonly` is assigned at declaration or in an instance/static constructor and can hold a value determined at runtime; it is not deeply immutable when it references a mutable object.

###### 5. What is the garbage collector? (PDF p. 5)
**Answer:** The .NET GC automatically reclaims managed objects that are no longer reachable, using generations to optimize collection. It does not guarantee prompt cleanup of unmanaged resources, which should normally be released with `IDisposable`.

###### 6. How do you remove unmanaged code/garbage in unmanaged code? (PDF p. 5)
**Answer:** Release each unmanaged allocation or handle using its matching API (for example, `free` for `malloc`, `CloseHandle` for a Windows handle). In .NET, wrap owned native resources in `SafeHandle`/`IDisposable` and dispose deterministically; a finalizer is only a fallback.

###### 7. `throw` vs. `throw ex` (PDF p. 5)
**Answer:** Inside a catch block, `throw;` rethrows while preserving the original stack trace. `throw ex;` resets the visible stack origin to that line, obscuring the original location; use `throw;` unless intentionally throwing a new exception with the original as its inner exception.

###### 8. What are access modifiers? (PDF p. 5)
**Answer:** Access modifiers control visibility: `public`, `private`, `protected`, `internal`, `protected internal`, and `private protected` (with file-local `file` for top-level types in modern C#). Choose the narrowest visibility that supports intended use.

###### 9. What is a partial class? Give a real-time scenario. (PDF p. 5)
**Answer:** `partial` lets one type declaration be split across files and combined at compile time. It is useful when generated code and hand-written code must coexist (for example, a generated model plus custom methods); both parts are one type, not separate runtime classes.

###### 10. What are generics? Have you implemented any in your project? (PDF p. 5)
**Answer:** Generics parameterize types or methods, enabling reusable, type-safe code without routine boxing/casting, such as `List<T>` or `Repository<T>`. In a project, a generic repository or result wrapper can centralize shared behavior while retaining compile-time type checks.

###### 11. What are collections and their types? (PDF p. 5)
**Answer:** Collections store and organize groups of values. Common families include generic lists/sequences (`List<T>`, `Queue<T>`), key/value maps (`Dictionary<TKey,TValue>`), sets (`HashSet<T>`), concurrent collections, and legacy non-generic collections.

###### 12. What are `Dictionary`, `List`, and `Hashtable`? (PDF p. 5)
**Answer:** `List<T>` is an indexed, ordered dynamic array; `Dictionary<TKey,TValue>` maps unique keys to values with generic type safety; `Hashtable` is a legacy non-generic key/value map that stores `object` and may require boxing/casts. Prefer generic collections in new code.

###### 13. What are `Stack` and `Queue`? (PDF p. 5)
**Answer:** A stack is last-in, first-out (LIFO), commonly used for undo/history; a queue is first-in, first-out (FIFO), useful for ordered work processing. Generic forms are `Stack<T>` and `Queue<T>`.

###### 14. Difference between generic and non-generic collections (PDF p. 6)
**Answer:** Generic collections enforce element types at compile time and avoid most value-type boxing. Non-generic collections store `object`, need runtime casts, and can fail with invalid types; they are mainly encountered in legacy APIs.

###### 15. What is a concurrent collection? (PDF p. 6)
**Answer:** It is a collection designed for safe concurrent access by multiple threads, such as `ConcurrentDictionary<TKey,TValue>` or `ConcurrentQueue<T>`. Use it when shared concurrent mutation is needed, while still reasoning about atomicity across multi-step operations.

###### 16. What are delegates and built-in delegates? (PDF p. 6)
**Answer:** A delegate is a type-safe reference to method(s) with a matching signature. Common generic delegates are `Action` (no result), `Func` (result), and `Predicate<T>` (Boolean test); events are commonly backed by delegates.

###### 17. What is LINQ? (PDF p. 6)
**Answer:** Language Integrated Query provides composable query operations for in-memory collections and providers such as EF Core. It supports query and method syntax; execution may be immediate or deferred depending on the operator and provider.

###### 18. `Single` vs. `SingleOrDefault`; `First` vs. `FirstOrDefault`; `Select` vs. `SelectMany` (PDF p. 6)
**Answer:** `Single` requires exactly one match and throws for zero or multiple; `SingleOrDefault` permits zero but still throws for multiple. `First` returns the first match and throws if none; `FirstOrDefault` returns a default when none. `Select` maps each item to one result; `SelectMany` maps each item to a sequence and flattens the sequences.

###### 19. LINQ query practice: left join, right join, inner join, and get the third highest (PDF p. 6)
**Answer:** An inner join keeps matching pairs; a left join keeps all left rows and fills missing right values with defaults; a right join is expressed by reversing the inputs to a left join. For the third-highest distinct salary in SQL, use `DENSE_RANK() OVER (ORDER BY Salary DESC)` and filter rank 3; in LINQ, sort distinct salaries descending and use `Skip(2).FirstOrDefault()`.

###### 20. What is the effect of combining `AsParallel` with LINQ queries? (PDF p. 6)
**Answer:** `AsParallel()` opts an `IEnumerable<T>` query into PLINQ, allowing work to run concurrently across cores. It can improve sufficiently large CPU-bound queries, but adds overhead, may change result ordering unless `AsOrdered()` is used, and is not appropriate for every query or side-effecting operation.

###### 21. `IEnumerable` vs. `ToList` (PDF p. 6)
**Answer:** `IEnumerable<T>` represents an enumerable sequence and may execute lazily each time it is iterated. `ToList()` immediately enumerates and materializes results into a list, using memory but giving a stable snapshot of those results.

###### 22. `IQueryable` vs. `ToList` (PDF p. 6)
**Answer:** `IQueryable<T>` builds a provider-translatable expression that is typically executed by the data source when enumerated. `ToList()` executes the query immediately and brings the resulting rows into memory; filter and project before materializing to avoid unnecessary transfer.

###### 23. `IQueryable` vs. `IEnumerable` (PDF p. 6)
**Answer:** `IEnumerable<T>` is generally enumerated in application code, while `IQueryable<T>` carries an expression tree that a provider may translate (for example, to SQL). Provider translation has limits, so keep query expressions translatable and materialize deliberately.

###### 24. What is the `yield` keyword? (PDF p. 6)
**Answer:** `yield return` produces values lazily from an iterator, and `yield break` ends it. The compiler generates the iterator state machine, so values are produced as the consumer advances rather than being collected up front.

###### 25. `break` vs. `continue` vs. `return` (PDF p. 6)
**Answer:** `break` exits the nearest loop or switch; `continue` skips to the next loop iteration; `return` exits the current method (optionally returning a value).

###### 26. Reflection in C# (PDF p. 6)
**Answer:** Reflection inspects assembly/type/member metadata at runtime and can invoke members dynamically. It supports serializers, plugin systems, and tooling, but can reduce performance and type safety; cache metadata or use source generation where suitable.

###### 27. What is TPL? (PDF p. 6)
**Answer:** The Task Parallel Library is .NET's task-oriented API for asynchronous and parallel work, including `Task`, continuations, cancellation, and parallel loops. It abstracts much of the low-level thread management.

###### 28. TPL vs. thread (PDF p. 6)
**Answer:** A `Thread` is a lower-level execution resource with explicit lifecycle management. TPL tasks represent units of work and normally use the thread pool for CPU-bound work, providing composition, continuations, and cancellation; a task is not necessarily a dedicated thread.

###### 29. Async vs. thread (PDF p. 6)
**Answer:** Asynchrony lets an operation yield while waiting, improving scalability for I/O; it does not inherently create a thread. A thread is an execution resource. CPU-bound work needs execution capacity (often a pool thread), whereas asynchronous I/O can wait without occupying one.

###### 30. How do you start a task? (PDF p. 6)
**Answer:** Prefer invoking an `async` method or using `Task.Run` for CPU-bound work. For I/O, call the API's asynchronous method directly (for example, `await client.GetAsync(...)`); avoid `Task.Run` merely to wrap naturally asynchronous I/O.

###### 31. Does `async` create threads? (PDF p. 6)
**Answer:** No. `async` enables compiler-generated state-machine behavior and `await` continuation; an I/O operation may complete without tying up a thread. `Task.Run` can queue CPU-bound work to a thread-pool thread.

###### 32. What are `async` and `await`? (PDF p. 6)
**Answer:** `async` allows a method to use `await` and normally return `Task`, `Task<T>`, or `ValueTask<T>`. `await` asynchronously waits for an awaitable to complete, then resumes the method, propagating results or exceptions without synchronously blocking the caller.

###### 33. Difference between `async`/`await` and TPL (PDF p. 6)
**Answer:** TPL is the broader task and parallel-work API; `async`/`await` is language syntax for composing awaitable operations, often represented by TPL tasks. Async is primarily useful for nonblocking waits; parallelism is about doing work simultaneously.

###### 34. How does `async`/`await` work under the hood? (PDF p. 6)
**Answer:** The compiler transforms an async method into a state machine. At an incomplete `await`, it records state and registers a continuation; when the awaitable completes, that continuation resumes the state machine, eventually completing the returned task.

###### 35. `ValueTask` vs. `Task` (PDF p. 6)
**Answer:** `Task` is a reference type and is the normal, simpler choice. `ValueTask<T>` can avoid allocation when results are often available synchronously, but is more complex and has consumption constraints; use it only after measuring a meaningful hot-path benefit.

###### 36. What is thread-pool starvation? (PDF p. 6)
**Answer:** It occurs when pool threads are blocked or consumed faster than the pool can make them available, delaying queued work. Avoid sync-over-async and unnecessary blocking, use asynchronous I/O, and diagnose thread-pool queues/latency under load.

###### 37. What is `IAsyncStateMachine`? (PDF p. 6)
**Answer:** It is the runtime interface implemented by compiler-generated async state machines, with `MoveNext()` advancing execution and `SetStateMachine()` supporting state-machine setup. Application code normally uses `async`/`await` rather than implementing it directly.

###### 38. What is the structure of the query pipeline in EF? (PDF p. 6)
**Answer:** LINQ builds an expression tree over `IQueryable`; EF Core's provider translates supported parts to a database query, sends it through the driver, then materializes returned rows into entities/projections and applies tracking if enabled. Deferred queries execute when enumerated or materialized.

###### 39. What latest .NET version have you used, and what special feature have you seen? (PDF p. 6)
**Answer:** Give the version you have actually used, then name a feature and its practical effect. For example: “I used .NET 8; its first-party keyed dependency-injection services help resolve implementations by key.” Do not claim experience you do not have.

###### 40. Why is there no explicit `Main` in a new .NET 6/7 console `Program.cs`, and how does execution start? (PDF p. 6)
**Answer:** Top-level statements let the compiler synthesize the entry-point `Main` method around statements in `Program.cs`. The runtime still starts at that generated entry point; top-level statements may also access command-line arguments and use `await`.

###### 41. What is a concurrency issue, how do you handle it, and is it related to performance? (PDF p. 6)
**Answer:** It is an incorrect or conflicting outcome caused by overlapping operations (such as lost updates or races). Use suitable synchronization, atomic operations, immutable data, or optimistic concurrency with conflict handling. Concurrency can improve throughput but excessive contention or coordination can hurt performance.

###### 42. How do you optimize memory? (PDF p. 6)
**Answer:** Measure allocations and retention with a profiler first. Reduce unnecessary copies/large temporary objects, stream large data, reuse buffers carefully, dispose unmanaged resources, avoid retaining object graphs, and use pooling only when measurements justify its complexity.

###### 43. If the database is optimized but the application still has performance issues, what steps would you take? (PDF p. 6)
**Answer:** Establish latency/throughput baselines and profile CPU, allocation/GC, locks, network, serialization, and external dependencies. Find the dominant cost, reduce redundant work and payloads, use async I/O and caching where correct, then load-test and compare against the baseline.

###### 44. How can you optimize an application with heavy database interactions—legacy application, Web API, or React front end? (PDF p. 6)
**Answer:** Reduce round trips and over-fetching, batch writes, paginate/stream large results, and use caching only with explicit freshness/invalidation rules. In legacy code, instrument and refactor hotspots incrementally; in APIs optimize queries and payloads; in React avoid duplicate requests and cache server state appropriately. Measure end-to-end before and after.

###### 45. How would you diagnose a production issue with very limited information? (PDF p. 6)
**Answer:** Confirm impact and timeline, preserve evidence, check recent changes and health dashboards, then correlate logs, metrics, and traces using request/correlation IDs. Form and test hypotheses safely, mitigate customer impact (rollback/feature flag if appropriate), identify root cause, and document prevention and follow-up.

###### 46. Which code will perform better? (PDF pp. 6–7; code image continues on p. 7)
**Prompt examples:**
```csharp
string longString = "";
for (int i = 0; i < 1000; i++)
{
    longString += "one more ";
}

StringBuilder sBuilder = new StringBuilder();
for (int i = 0; i < 1000; i++)
{
    sBuilder.Append("Add This");
}
string str = sBuilder.ToString();
```
**Answer:** The `StringBuilder` approach is generally faster and allocates less for repeated concatenation in a loop. C# strings are immutable, so each `+=` creates a new string and copies prior content; `StringBuilder` reuses a mutable buffer and creates the final string once. The sample appends different text in each version, so a fair benchmark should use equivalent output and representative input sizes.
