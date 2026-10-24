# Senior C#/.NET Developer: Code Review, Refactoring & Interview Guide

A practical reference for senior-developer interviews (“viva”) and for reviewing code and mentoring a .NET team. Examples target modern C# and ASP.NET Core; adapt them to the project's language version and conventions.

## 1. How to approach a code review

Review in this order:

1. **Correctness:** Does the code meet the requirement? Are edge cases handled?
2. **Security and privacy:** Is authorization enforced? Are inputs validated? Are secrets and sensitive data kept out of logs?
3. **Data integrity:** Are transactions, concurrency, and duplicate requests handled correctly?
4. **Performance:** Are database calls, allocations, network calls, and payload sizes reasonable?
5. **Maintainability:** Are names clear, responsibilities focused, and abstractions justified?
6. **Tests:** Are important success, failure, boundary, and authorization cases covered?
7. **Consistency:** Does the change follow established project conventions?

### Useful review comments

Prefer specific, respectful comments that explain the risk and suggest an action.

- **Blocking:** “This endpoint checks authentication but not ownership of the requested record. Could we verify the current user is authorized to access this resource?”
- **Suggestion:** “Could we project only the fields returned by the API? This query currently loads more columns than the response needs.”
- **Question:** “What should happen if two requests update this record at the same time?”
- **Nit:** “Could we rename `data` to `activeUsers` to make its purpose clearer?”

Do not request changes just to impose personal style. Separate correctness/security blockers from optional improvements.

## 2. Modern C# patterns: old vs. current

### 2.1 Short-circuiting

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

### 2.2 Ternary operator

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

### 2.3 Null-conditional and null-coalescing operators

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

### 2.4 Pattern matching and switch expressions

```csharp
// Old
string GetStatus(int code)
{
    switch (code)
    {
        case 200: return "OK";
        case 400: return "Bad Request";
        case 500: return "Server Error";
        default: return "Unknown";
    }
}

// Modern
static string GetStatus(int code) => code switch
{
    200 => "OK",
    400 => "Bad Request",
    500 => "Server Error",
    _ => "Unknown"
};
```

Pattern matching can combine type checks and variable capture:

```csharp
if (value is Customer customer && customer.IsActive)
{
    Process(customer);
}
```

### 2.5 Expression-bodied members

Use for simple expressions, not for long workflows.

```csharp
public decimal CalculateTotal(decimal price, int quantity)
    => price * quantity;
```

### 2.6 Records and value-based equality

Use records for data-centric values where value equality is useful, such as commands, query results, and DTOs.

```csharp
public sealed record CreateUserRequest(string Name, string Email);
```

A record is **not automatically deeply immutable**: referenced objects or collection properties can still be mutable. Use ordinary classes for entities with identity and lifecycle when that fits the domain model better.

### 2.7 Dictionary lookup

```csharp
// Avoid two lookups
if (rates.ContainsKey(country))
    return amount * rates[country];

// One lookup
return rates.TryGetValue(country, out var rate)
    ? amount * rate
    : amount * defaultRate;
```

### 2.8 String building

For a small fixed number of interpolations, normal interpolation is clear. For repeated appends in a loop, consider `StringBuilder`.

```csharp
var builder = new StringBuilder();
foreach (var item in items)
    builder.AppendLine(item.Name);

var result = builder.ToString();
```

Do not optimize string concatenation without considering readability and measuring real bottlenecks.

## 3. Refactoring: recognize smells and choose a safe change

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

### Example: guard clauses and a focused method

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

### Example: dependency injection

```csharp
public interface IEmailSender
{
    Task SendAsync(string recipient, string subject, CancellationToken ct);
}

public sealed class OrderNotifier(IEmailSender emailSender)
{
    public Task NotifyAsync(string email, CancellationToken ct) =>
        emailSender.SendAsync(email, "Order confirmed", ct);
}
```

Register the implementation in the composition root. DI helps testing and decoupling, but do not create interfaces for every class without a clear benefit.

## 4. Async, EF Core, and performance

### Async I/O

```csharp
// Avoid blocking asynchronous work with .Result or .Wait()
var body = await httpClient.GetStringAsync(url, cancellationToken);
```

- Use asynchronous APIs for database, HTTP, and file I/O.
- Pass `CancellationToken` through layers when cancellation is supported.
- Do not wrap naturally asynchronous I/O in `Task.Run`.
- Use `Task.WhenAll` for independent async operations only when concurrency and downstream capacity are appropriate.
- Avoid unbounded fan-out that can overload a database or external service.

### EF Core query shape

```csharp
var users = await dbContext.Users
    .AsNoTracking()
    .Where(user => user.IsActive)
    .OrderBy(user => user.Name)
    .Select(user => new UserListItem(user.Id, user.Name))
    .ToListAsync(cancellationToken);
```

Key review checks:

- Project only the fields needed by the caller.
- Use `AsNoTracking()` for read-only entity queries when change tracking is unnecessary.
- Filter and sort in SQL; avoid materializing a large table before filtering.
- Check for N+1 queries and unnecessary `Include`s.
- Add indexes based on query patterns and execution plans, not guesswork.
- Use pagination for large result sets; consider keyset pagination when offset paging becomes expensive.
- Use transactions when multiple writes must succeed or fail together.
- Do not share one `DbContext` across concurrent operations; it is not thread-safe.
- Measure before introducing compiled queries, pooling, or low-level optimizations.

## 5. REST API, security, and reliability

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

## 6. Clean Architecture and CQRS

Use boundaries to protect business rules, not merely to create folders.

- **Domain:** Core business concepts and invariants; should not depend on infrastructure.
- **Application:** Use cases, orchestration, and application contracts.
- **Infrastructure:** EF Core, external services, messaging, file storage, and implementations.
- **Presentation:** HTTP endpoints, authentication integration, request/response mapping.

CQRS separates read and write models when that separation solves a real problem. It does **not** automatically require microservices, event sourcing, separate databases, or a message broker. For simple CRUD features, a simpler design may be better.

## 7. Testing and safe refactoring workflow

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

## 8. Senior-level interview questions and model answers

### Q1. How do you ensure code quality in a large project?
**Answer:** Establish shared conventions, automated formatting and analyzers, meaningful tests, CI quality gates, and a consistent PR process. Review correctness, security, data integrity, performance, and maintainability. Track recurring defects and improve the system rather than relying only on manual review.

### Q2. What code smells do you look for?
**Answer:** Long methods, large classes, duplication, deep nesting, unclear naming, hidden side effects, excessive coupling, and abstractions that add complexity without value. I confirm the underlying problem before choosing a refactoring.

### Q3. How do you balance readability and performance?
**Answer:** Start with the clearest correct implementation, measure using profiling or representative benchmarks, identify the bottleneck, and optimize that part. Keep a test or benchmark to verify the improvement and avoid speculative micro-optimization.

### Q4. How do you handle breaking API changes?
**Answer:** Prefer additive, backward-compatible changes. If breaking changes are unavoidable, version the contract where appropriate, communicate a migration plan, support a transition period if feasible, update contract tests and documentation, and monitor adoption before retiring the old version.

### Q5. Refactoring vs. rewriting?
**Answer:** Refactoring changes internal structure while preserving behavior. Rewriting replaces an implementation and carries more risk because existing behavior and edge cases can be lost. I prefer incremental refactoring when tests and architecture allow it; a rewrite needs a clear business and technical case.

### Q6. How do you refactor a tightly coupled class?
**Answer:** Identify its responsibilities and dependencies, add tests around current behavior, separate cohesive responsibilities, inject external dependencies through constructors, and introduce interfaces at meaningful boundaries. Avoid creating abstractions solely to satisfy a pattern.

### Q7. How do you optimize EF Core?
**Answer:** Inspect generated SQL and execution plans, project only needed fields, filter in the database, use no-tracking reads when appropriate, address N+1 queries, apply suitable indexes and pagination, and measure. I also check transaction scope and concurrency behavior.

### Q8. When is a design pattern overkill?
**Answer:** When it adds indirection or concepts without solving a current, credible problem. Choose the simplest design that meets current requirements while leaving reasonable room for likely change.

### Q9. What is the difference between `Task.WhenAll` and `Parallel.ForEach`?
**Answer:** `Task.WhenAll` coordinates multiple tasks, often asynchronous I/O tasks. `Parallel.ForEach` is for parallel synchronous work; for async iteration, consider `Parallel.ForEachAsync` with an explicit degree of parallelism. Choose based on workload and resource limits.

### Q10. How do you prevent race conditions when updating data?
**Answer:** Define the consistency requirement first. Depending on the case, use database transactions, optimistic concurrency tokens (such as a row version), unique constraints, appropriate isolation, or atomic updates. Application-level checks alone may not protect against concurrent requests.

### Q11. How would you investigate a slow API?
**Answer:** Establish a baseline, inspect latency percentiles and traces, separate time spent in the application, database, and external dependencies, check query plans and N+1 behavior, review allocations and serialization, then make and measure one targeted change.

### Q12. How do you mentor a team through code review?
**Answer:** Explain the impact rather than just saying “wrong,” distinguish blockers from suggestions, ask questions when context is unclear, share examples and team conventions, and follow up on recurring patterns through pairing, tests, analyzers, or documentation.

## 9. A concise PR checklist

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

## Final principle

**A senior developer does more than make code shorter.** They protect behavior, security, data integrity, and operability while making the design easier for the next person to understand and change.
