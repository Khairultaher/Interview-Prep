# Senior .NET Developer & Solution Architect — Viva Preparation

This guide consolidates the topics discussed so far into one reference. Repeated explanations have been merged, and examples focus on the concepts most useful in a senior developer or solution architect viva.

> **Version context:** Examples generally target .NET 8 / C# 12. Check package APIs and hosting features against the exact .NET version used by your project.

## Contents

1. High-performance REST APIs
2. Async/await, threads, and operating-system I/O
3. `ConfigureAwait(false)`
4. ASP.NET Core security
5. Clean Architecture
6. CQRS and event-driven architecture
7. Microservices and distributed systems
8. System design and deployment
9. Advanced C# coding patterns
10. Senior-level interview answer framework

---

# 1. High-performance REST APIs

## 1.1 What are the best practices for optimizing a .NET API?

**Viva answer:** I optimize based on measurements, not assumptions. I first establish a baseline for latency, throughput, error rate, CPU, memory, database time, and dependency latency. Then I address the largest bottleneck.

Key practices:

- Use asynchronous APIs for I/O-bound work.
- Keep database queries selective; project only required columns.
- Add indexes based on actual query plans.
- Paginate large result sets, preferably with keyset pagination for deep pages.
- Avoid N+1 database queries and unnecessary eager loading.
- Cache stable, frequently requested data where staleness is acceptable.
- Limit request sizes and validate input.
- Use response compression when payload size justifies its CPU cost.
- Reuse `HttpClient` through `IHttpClientFactory`.
- Apply timeouts, cancellation, and bounded concurrency.
- Measure with load tests and production telemetry.

### Example: Efficient EF Core read query

```csharp
app.MapGet("/products", async (
    AppDbContext db,
    CancellationToken cancellationToken) =>
{
    var products = await db.Products
        .AsNoTracking()
        .Where(p => p.IsActive)
        .OrderBy(p => p.Name)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))
        .Take(100)
        .ToListAsync(cancellationToken);

    return Results.Ok(products);
});

public sealed record ProductDto(int Id, string Name, decimal Price);
```

`AsNoTracking()` avoids change-tracking overhead for a read-only query. `Select` limits retrieved columns. `Take` bounds the result size. These choices must be paired with suitable indexes and verified through profiling.

## 1.2 How would you handle millions of concurrent API requests?

First, clarify what “millions” means: millions of registered users, millions of requests per day, or millions of simultaneously active requests are very different workloads.

A scalable design typically uses:

1. A CDN/WAF or edge layer for suitable public traffic.
2. A load balancer or gateway.
3. Multiple stateless API instances that can scale horizontally.
4. Asynchronous I/O for database and network calls.
5. Database connection pooling, query optimization, and carefully designed capacity limits.
6. Distributed caching for shared cache entries.
7. A message broker and background workers for long-running work.
8. Rate limiting, timeouts, backpressure, and load shedding.
9. Multi-region deployment only when availability or latency requirements justify it.
10. Load tests that model realistic request mixes and dependency bottlenecks.

**Important:** `async` does not make CPU-bound work free and does not remove database limits. If every request requires a slow query, the database can still be the bottleneck. Scale each constrained dependency deliberately.

## 1.3 How do you minimize memory allocations?

- Avoid unnecessary materialization and duplicate copies of collections.
- Stream large data when the client and transport support it.
- Use `ArrayPool<T>` for temporary reusable buffers in measured hot paths.
- Consider `Span<T>` / `Memory<T>` for efficient buffer processing.
- Avoid boxing and repeated string creation where profiling shows impact.
- Reuse serializer options and long-lived HTTP clients.
- Do not introduce low-level complexity without benchmarks.

### Example: rent and return a buffer safely

```csharp
using System.Buffers;

byte[] buffer = ArrayPool<byte>.Shared.Rent(16 * 1024);

try
{
    // Use only the portion of the buffer that your operation owns.
    // The rented array may be larger than requested and may contain old data.
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
}
```

Clear pooled buffers when they may contain sensitive data. Always return a rented buffer, and never use it after returning it to the pool.

## 1.4 `Task`, `ValueTask<T>`, and `IAsyncEnumerable<T>`

| Type | Meaning | Typical use |
|---|---|---|
| `Task` / `Task<T>` | Represents one asynchronous operation, optionally with a result | Most service and I/O methods |
| `ValueTask<T>` | Can represent a result without allocating a new `Task<T>` in some cases | Specialized hot paths where synchronous completion is common and measured |
| `IAsyncEnumerable<T>` | Represents a sequence that can be consumed asynchronously, item by item | Streaming records or results |

`ValueTask<T>` is not automatically faster. It has usage constraints and can be more complex; use `Task<T>` by default unless measurement supports the alternative.

### Example: asynchronous streaming

```csharp
static async IAsyncEnumerable<int> CountAsync(
    int count,
    [System.Runtime.CompilerServices.EnumeratorCancellation]
    CancellationToken cancellationToken = default)
{
    for (int i = 1; i <= count; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await Task.Delay(50, cancellationToken);
        yield return i;
    }
}

await foreach (var number in CountAsync(5))
{
    Console.WriteLine(number);
}
```

Streaming can reduce buffering, but it does not guarantee zero allocations. HTTP clients, serializers, and application code still allocate some objects.

## 1.5 How do you optimize JSON serialization?

- Prefer `System.Text.Json` when it meets the requirements.
- Reuse `JsonSerializerOptions`.
- Return DTOs rather than unnecessarily large domain graphs.
- Avoid serializing fields or properties the client does not need.
- For high-throughput applications, evaluate source-generated JSON metadata.
- For large payloads, consider streaming rather than building a giant string first.
- Benchmark representative payloads; do not claim a universal performance multiplier.

### Example: source-generated JSON metadata

```csharp
using System.Text.Json.Serialization;

public sealed record ProductDto(int Id, string Name, decimal Price);

[JsonSerializable(typeof(ProductDto))]
[JsonSerializable(typeof(List<ProductDto>))]
internal partial class ApiJsonContext : JsonSerializerContext
{
}
```

Usage:

```csharp
var json = System.Text.Json.JsonSerializer.Serialize(
    new ProductDto(10, "Keyboard", 49.99m),
    ApiJsonContext.Default.ProductDto);
```

Source generation can reduce reflection requirements and help trimming/AOT scenarios; actual throughput depends on the model and workload.

## 1.6 Caching at different levels

- **Browser/client cache:** suitable for client-owned data with correct cache directives.
- **CDN/reverse proxy:** useful for cacheable public responses near users.
- **In-process cache (`IMemoryCache`):** fast, but each API instance has its own cache.
- **Distributed cache (for example Redis):** shared across API instances.
- **Database cache/index/materialized view:** can improve expensive data access, but requires careful data consistency design.

### Cache-aside example

```csharp
public sealed class ProductService(
    IMemoryCache cache,
    AppDbContext db)
{
    public async Task<ProductDto?> GetAsync(
        int id,
        CancellationToken cancellationToken)
    {
        var key = $"product:{id}";

        if (cache.TryGetValue(key, out ProductDto? cached))
            return cached;

        var product = await db.Products
            .AsNoTracking()
            .Where(p => p.Id == id)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price))
            .SingleOrDefaultAsync(cancellationToken);

        if (product is not null)
        {
            cache.Set(key, product, TimeSpan.FromMinutes(2));
        }

        return product;
    }
}
```

For multiple API replicas, an in-memory cache is not shared. Use a distributed cache when shared state is needed. Design invalidation, expiration, stampede protection, and tenant/user isolation explicitly.

## 1.7 `Span<T>` and `Memory<T>`

Both provide views over contiguous memory without necessarily copying the underlying data.

- `Span<T>` is a stack-only `ref struct`; it is ideal for synchronous parsing and buffer manipulation.
- `Memory<T>` can be stored in ordinary objects and used across `await` boundaries.
- Neither automatically makes every operation allocation-free.

```csharp
ReadOnlySpan<char> input = "12345".AsSpan();

if (int.TryParse(input, out int value))
{
    Console.WriteLine(value);
}
```

Use these types when they simplify buffer processing and profiling demonstrates a benefit.

## 1.8 How do you handle long-running requests?

Do not keep an HTTP request open for a job that may take minutes or hours unless the protocol and product explicitly require it.

A common design:

1. Validate the request.
2. Persist a job record.
3. Enqueue the job reliably.
4. Return `202 Accepted` with a job-status URL.
5. A background worker processes it with retries and idempotency.
6. The client polls the status endpoint or receives a notification.

```csharp
app.MapPost("/reports", async (
    CreateReportRequest request,
    IReportJobService jobs,
    CancellationToken cancellationToken) =>
{
    var jobId = await jobs.EnqueueAsync(request, cancellationToken);

    return Results.Accepted($"/reports/{jobId}", new { jobId });
});
```

Persist the job before acknowledging it, or use a reliable transactional-outbox design so a crash does not lose accepted work.

## 1.9 `IHostedService` vs `BackgroundService`

- `IHostedService` defines `StartAsync` and `StopAsync`.
- `BackgroundService` is a base class for implementing a long-running task through `ExecuteAsync`.

Use `BackgroundService` for a worker loop. Use `IHostedService` directly when you need custom start/stop behavior. Hosted services are generally singletons; create a DI scope for scoped services such as EF Core `DbContext`.

## 1.10 Request throttling / rate limiting

ASP.NET Core includes rate-limiting middleware. Choose a policy based on the protected resource: per-IP limits can help with anonymous traffic, while authenticated APIs may need per-user, per-tenant, or per-client quotas. A local limiter on each replica is not automatically a globally coordinated quota.

```csharp
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddFixedWindowLimiter("public-api", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
        limiter.AutoReplenishment = true;
    });
});

var app = builder.Build();
app.UseRateLimiter();

app.MapGet("/api/data", () => Results.Ok("ok"))
   .RequireRateLimiting("public-api");

app.Run();
```

For production, consider partitioned policies, gateway-level protection, `Retry-After` behavior, and monitoring of rejected requests.

---

# 2. Async/await, threads, and operating-system I/O

## 2.1 Does `async` / `await` use ThreadPool threads?

**Viva answer:** `async` and `await` are language features for composing asynchronous operations. They do not inherently create a new thread.

- **I/O-bound operation:** the OS/runtime can wait for I/O without occupying a worker thread for the entire wait.
- **CPU-bound operation:** the work must execute on a CPU. `Task.Run` can schedule it on a ThreadPool thread, but should not be used automatically inside every ASP.NET Core request.
- **Continuation:** after an awaited operation completes, the remaining code runs according to the captured context/scheduler rules. In ASP.NET Core, continuations normally run on ThreadPool threads because there is no classic ASP.NET synchronization context.

### I/O-bound example

```csharp
public async Task<string> GetPageAsync(
    HttpClient httpClient,
    CancellationToken cancellationToken)
{
    return await httpClient.GetStringAsync(
        "https://example.com",
        cancellationToken);
}
```

While the network response is pending, this does not require a ThreadPool thread to sit blocked waiting for the response. When completion is reported, .NET schedules the continuation as appropriate.

### CPU-bound example

```csharp
public Task<long> CalculateAsync(
    int count,
    CancellationToken cancellationToken)
{
    return Task.Run(() =>
    {
        long total = 0;

        for (int i = 0; i < count; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            total += i;
        }

        return total;
    }, cancellationToken);
}
```

This consumes CPU on a ThreadPool worker. In ASP.NET Core, use this only when there is a real reason to move CPU work elsewhere; it does not increase total CPU capacity and can make thread-pool pressure worse.

**Avoid** `.Result`, `.Wait()`, and sync-over-async in request paths. They block threads and can cause throughput collapse or deadlocks in environments with a synchronization context.

## 2.2 How does the OS kernel handle I/O asynchronously?

A simplified flow:

1. Application requests a network or file operation.
2. The runtime invokes OS facilities appropriate for the platform and operation.
3. The kernel and/or device processes the operation.
4. Completion is reported through OS mechanisms.
5. .NET completes the associated asynchronous operation and schedules its continuation.

The exact mechanism differs by OS and I/O type. Windows uses completion mechanisms such as I/O Completion Ports for many socket and file scenarios. Linux commonly uses readiness notification mechanisms such as `epoll` for sockets; newer APIs such as `io_uring` support other I/O models. Do not state that all .NET I/O always uses one specific OS mechanism.

### File I/O example

```csharp
public static async Task<string> ReadTextAsync(
    string path,
    CancellationToken cancellationToken)
{
    return await File.ReadAllTextAsync(path, cancellationToken);
}
```

The caller can await the operation without blocking its thread while the asynchronous operation is pending. Actual behavior can vary by OS, filesystem, device, and API; not every operation is guaranteed to be truly asynchronous at the hardware level.

## 2.3 Task Scheduler and `SynchronizationContext`

- The **Task Scheduler** is relevant to scheduling `Task` work, especially tasks explicitly scheduled through APIs such as `Task.Run`.
- A **SynchronizationContext** is an abstraction that can direct continuations to a particular environment, such as a UI thread.
- WPF/WinForms commonly have a UI synchronization context. After `await`, code normally returns to that context so it can update UI controls.
- ASP.NET Core does not use the classic ASP.NET synchronization context, so context capture usually is not a concern there.

```csharp
Console.WriteLine($"Before: {Environment.CurrentManagedThreadId}");

await Task.Delay(100);

Console.WriteLine($"After: {Environment.CurrentManagedThreadId}");
```

The thread ID may change across the `await`. Correct asynchronous code should not depend on a particular thread ID unless the framework explicitly requires thread affinity.

## 2.4 What does `ConfigureAwait(false)` do, and how can it help?

By default, an `await` may capture the current synchronization context (or a relevant scheduler) so the continuation can return to it. `ConfigureAwait(false)` tells that await not to require resuming on the captured context.

```csharp
public async Task<string> FetchAsync(HttpClient client)
{
    var content = await client
        .GetStringAsync("https://example.com")
        .ConfigureAwait(false);

    return content;
}
```

### When it matters

- In reusable libraries that must work in UI and other environments, avoiding context capture can prevent unnecessary context switches.
- In WPF/WinForms, it can prevent a library continuation from needing the UI thread when the continuation does not use UI objects.
- In ASP.NET Core, there is normally no classic synchronization context to return to, so it usually provides little or no performance benefit.

### Important correction

`ConfigureAwait(false)` does **not** mean “always switch to a ThreadPool thread,” and it does not itself make the underlying I/O faster. It means the continuation does not require the captured context. The continuation may run wherever the completion machinery schedules it.

Use it selectively in library code where context affinity is unnecessary. Do not use it in UI code before directly updating UI controls unless you explicitly marshal back to the UI thread.

## 2.5 `async void` vs `async Task`

Use `async Task` or `async Task<T>` for ordinary asynchronous methods so callers can await completion and observe exceptions. `async void` is mainly for event handlers whose signature requires `void`.

```csharp
public async Task SaveAsync(CancellationToken cancellationToken)
{
    await Task.Delay(10, cancellationToken);
}
```

---

# 3. ASP.NET Core security

## 3.1 Authentication and authorization

- **Authentication:** establishes who the caller is.
- **Authorization:** decides whether that caller may perform an action on a resource.

For APIs, use a trusted identity provider with OAuth 2.0 / OpenID Connect as appropriate. Validate token signature, issuer, audience, expiry, and relevant scopes or claims. Use HTTPS everywhere.

```csharp
builder.Services
    .AddAuthentication()
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Identity:Authority"];
        options.Audience = builder.Configuration["Identity:Audience"];
        options.RequireHttpsMetadata = true;
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Reports.Read", policy =>
        policy.RequireAuthenticatedUser()
              .RequireClaim("permission", "reports.read"));
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

Configure the actual authority and audience for your identity provider. Do not hardcode production signing keys into source code.

## 3.2 Role-based and claim-based authorization

```csharp
app.MapGet("/admin", () => Results.Ok())
   .RequireAuthorization(policy =>
       policy.RequireRole("Admin"));

app.MapGet("/reports", () => Results.Ok())
   .RequireAuthorization("Reports.Read");
```

Roles represent broad job functions; claims/policies can express finer permissions. For access to a particular resource, use resource-based authorization so a user cannot access another tenant's or another customer's record merely by changing an ID.

## 3.3 OAuth 2.0 provider / IdentityServer-style products

A dedicated identity provider centralizes token issuance, client registration, scopes, consent, key rotation, and SSO/OpenID Connect integration. Choose a supported product and lifecycle; do not start a new project on an obsolete identity-server version without checking its support status.

OAuth 2.0 is primarily an authorization framework; OpenID Connect adds an identity layer. Use authorization code with PKCE for interactive clients and client credentials for service-to-service clients where appropriate. Never put a confidential client secret in browser or mobile code.

## 3.4 CSRF, XSS, and SQL injection

- **CSRF:** especially relevant when browsers automatically attach cookies. Use anti-forgery tokens and appropriate `SameSite`, `Secure`, and `HttpOnly` cookie settings. Bearer tokens explicitly supplied in an `Authorization` header have a different CSRF profile, but browser storage introduces other risks.
- **XSS:** encode output, sanitize allowed rich HTML, avoid unsafe DOM insertion, and deploy a suitable Content Security Policy. Input validation alone does not prevent XSS.
- **SQL injection:** use parameterized queries and ORM parameters. Do not concatenate untrusted values into SQL. For dynamic sort columns or identifiers, use an allowlist because SQL parameters cannot stand in for identifiers.

```csharp
var user = await db.Users
    .AsNoTracking()
    .SingleOrDefaultAsync(u => u.Email == email, cancellationToken);
```

## 3.5 HSTS

HTTP Strict Transport Security tells supporting browsers to use HTTPS for future requests to a host. It helps prevent downgrade attacks after the browser learns the policy. Enable it in production when HTTPS is correctly configured, and be cautious with `includeSubDomains` and preload because those choices affect every covered subdomain.

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();
```

## 3.6 Secret management

- Use .NET User Secrets for local development; it is not a production secret vault.
- Use environment-specific secret stores such as Azure Key Vault in production.
- Prefer managed identity/workload identity over stored credentials when supported.
- Never commit secrets or log them.
- Rotate credentials and restrict permissions.

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:Uri"]!),
    new Azure.Identity.DefaultAzureCredential());
```

## 3.7 Zero Trust

Do not automatically trust a request because it originates inside a network. Authenticate identities, authorize every relevant action, enforce least privilege, protect service-to-service traffic, validate inputs, monitor activity, and assume credentials or network segments can be compromised.

## 3.8 API keys for third-party integrations

Treat API keys as secrets, not as a complete user identity system. Store them securely, transmit them only over TLS, scope them narrowly, rotate them, monitor use, and support revocation. Where possible, use OAuth client credentials or signed requests. Never put keys in URLs because URLs often appear in logs and browser history.

## 3.9 JWT best practices and pitfalls

- Validate signature, issuer, audience, lifetime, and allowed algorithms.
- Use short-lived access tokens and a secure refresh-token strategy when needed.
- Keep sensitive information out of the payload; a signed JWT is usually readable, not encrypted.
- Plan for key rotation and revocation.
- Enforce authorization on the API, not only in the frontend.
- Do not accept a token merely because it can be decoded.

## 3.10 Safe error handling

Return a generic error response and a correlation/trace ID to the caller. Keep detailed exceptions in access-controlled logs. Avoid exposing SQL, stack traces, filesystem paths, secrets, internal hostnames, or personal data.

Use ASP.NET Core exception-handling middleware and `ProblemDetails` for consistent error responses.

---

# 4. Clean Architecture

## 4.1 Why use it?

Clean Architecture separates business policy from delivery mechanisms and infrastructure. The main goal is to keep business rules testable and less dependent on frameworks, databases, and external services.

A typical project layout:

```text
src/
  MyApp.Domain/
  MyApp.Application/
  MyApp.Infrastructure/
  MyApp.Api/
tests/
  MyApp.UnitTests/
  MyApp.IntegrationTests/
```

The dependency rule is important: **source-code dependencies point inward**. The Domain should not depend on EF Core, ASP.NET Core, or infrastructure. The Application layer defines use cases and abstractions; Infrastructure implements technical details.

## 4.2 Domain entity example

```csharp
public sealed class Order
{
    private readonly List<OrderLine> _lines = [];

    public Guid Id { get; private set; } = Guid.NewGuid();
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();
    public string Status { get; private set; } = "Draft";

    public void AddLine(Guid productId, int quantity, decimal unitPrice)
    {
        if (quantity <= 0)
            throw new ArgumentOutOfRangeException(nameof(quantity));

        if (unitPrice < 0)
            throw new ArgumentOutOfRangeException(nameof(unitPrice));

        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("An empty order cannot be submitted.");

        Status = "Submitted";
    }
}

public sealed record OrderLine(Guid ProductId, int Quantity, decimal UnitPrice);
```

The entity protects invariants rather than allowing arbitrary callers to set every property freely.

## 4.3 Repository pattern: use it deliberately

A repository can express persistence needs in domain terms and make a use case independent of a particular data-access implementation. However, wrapping every EF Core operation in a generic repository often adds boilerplate and hides useful query capabilities. EF Core's `DbContext` already implements Unit of Work-like behavior and its `DbSet` has repository-like characteristics.

Use repositories where they add a meaningful abstraction, not as a mandatory wrapper around every table.

## 4.4 Transactions across repositories

When repositories use the same scoped `DbContext`, a single `SaveChangesAsync` normally persists their tracked changes in one database transaction when the provider supports transactions.

```csharp
await orderRepository.AddAsync(order, cancellationToken);
await inventoryRepository.ReserveAsync(order, cancellationToken);
await unitOfWork.SaveChangesAsync(cancellationToken);
```

For multiple databases or services, a local database transaction cannot make the whole distributed workflow atomic. Use a saga and/or transactional outbox, plus idempotency and compensating actions where appropriate.

## 4.5 Dependency inversion and DI

Application defines an interface; Infrastructure implements it; the API composition root wires the concrete implementation.

```csharp
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IUnitOfWork, EfUnitOfWork>();
```

Avoid service-locator patterns and avoid making Domain depend on the DI container.

## 4.6 Logging and testing

Use `ILogger<T>` and structured logging. Add correlation/trace context, but do not log secrets or sensitive request bodies by default.

Test the Application layer by testing use-case behavior and business decisions. Use fakes or mocks for external abstractions, and use integration tests against the real database provider for important persistence behavior. EF Core's in-memory provider is not a faithful substitute for relational database behavior.

## 4.7 Clean vs traditional layered architecture

Both separate concerns. Clean Architecture is distinguished primarily by its dependency direction and protection of business rules. A layered architecture can also enforce inward dependencies; the name alone does not guarantee quality.

---

# 5. CQRS and event-driven architecture

## 5.1 What is CQRS?

Command Query Responsibility Segregation separates operations that change state (commands) from operations that read state (queries). It does not require two databases or a message broker.

Use CQRS when read and write needs differ significantly, business commands have meaningful validation, or different read models improve performance. Avoid adding its complexity to a straightforward CRUD application without a clear benefit.

## 5.2 Simple CQRS example

```csharp
public sealed record GetProductQuery(int Id);
public sealed record ProductDto(int Id, string Name, decimal Price);

public sealed class GetProductHandler(AppDbContext db)
{
    public Task<ProductDto?> Handle(
        GetProductQuery query,
        CancellationToken cancellationToken) =>
        db.Products
            .AsNoTracking()
            .Where(p => p.Id == query.Id)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price))
            .SingleOrDefaultAsync(cancellationToken);
}
```

A mediator such as MediatR or another library can route requests to handlers, but CQRS is the architectural separation, not the mediator library. Check the current licensing and version terms before choosing a package.

## 5.3 Eventual consistency

If a command updates the write model and an asynchronous event updates a read model, there can be a period when reads show the old state. Make this explicit in product behavior. Use retries, idempotent consumers, monitoring, and a way to rebuild read models. For workflows requiring immediate confirmation, return the command result or read from the authoritative store where appropriate.

## 5.4 Event sourcing

Event sourcing stores the sequence of domain events as the source of truth and derives current state by replaying events or using snapshots. It is useful when historical state and auditability are central. It adds complexity around event schema evolution, replay, snapshots, privacy/deletion requirements, and operational tooling. A normal audit log is not automatically event sourcing.

## 5.5 Outbox and saga

**Transactional outbox:** write business data and an outgoing event record in the same local database transaction. A background publisher sends the event later. Consumers must tolerate duplicates.

**Saga:** coordinate a business process across multiple services through steps and compensating actions. It is not a distributed ACID transaction.

## 5.6 Kafka vs RabbitMQ

- Kafka is a distributed event log suited to high-throughput streams, replay, and multiple consumer groups.
- RabbitMQ is a message broker suited to routing work to queues and acknowledgement-based consumption.
- Selection depends on ordering, throughput, replay, routing, delivery semantics, operations, and team expertise.

Neither automatically guarantees end-to-end exactly-once business effects. Design for duplicate delivery and idempotent processing.

## 5.7 CQRS for a high-frequency trading platform

For a real trading system, correctness and risk controls take precedence over a generic “use Redis and Kafka” answer. Separate market-data ingestion, strategy computation, order management, risk checks, execution, and audit. Use bounded queues, explicit ordering semantics, deterministic risk checks, durable audit records, and strict latency measurements. Choose in-memory structures and transport only after requirements and benchmarks are clear. Kafka may suit market-data distribution and replay, but it is not automatically the lowest-latency component for every critical path.

---

# 6. Microservices and distributed systems

## 6.1 Monolith vs modular monolith vs microservices

- **Monolith:** one deployable application; simplest operational model for many products.
- **Modular monolith:** one deployment with explicit module boundaries and controlled dependencies.
- **Microservices:** independently deployable services aligned to business capabilities, typically with their own data ownership.

Start with a modular monolith when the domain and team structure do not yet justify distributed complexity. Choose microservices when independent deployment/scaling, team autonomy, or isolation of failure domains provides a measurable benefit.

## 6.2 Service communication

- **Synchronous:** REST or gRPC when the caller needs an immediate response.
- **Asynchronous:** broker/event communication when work can proceed independently or consumers need decoupling.

Every synchronous hop adds latency and a failure dependency. Use timeouts, bounded retries with backoff and jitter, circuit breaking where suitable, and cancellation. Do not retry non-idempotent operations blindly.

## 6.3 API Gateway

A gateway can provide routing, TLS termination, rate limiting, authentication integration, and request aggregation. It should not become a giant business-logic monolith or the only place where authorization is enforced. Services must still enforce their own access rules.

## 6.4 Service discovery

In Kubernetes, service DNS and cluster networking commonly provide discovery. Other environments may use a service registry or platform-specific naming. Prefer the simplest discovery mechanism supported by the deployment platform.

## 6.5 Docker and Kubernetes

- Docker containers package the application and runtime dependencies consistently.
- Kubernetes orchestrates containers, scheduling, service discovery, health checks, rollout, and scaling.
- Kubernetes adds operational complexity; it is not automatically required for every .NET application.

Use a multi-stage Docker build and a supported .NET runtime image. Pin production image versions rather than relying on `latest`.

## 6.6 Authentication between microservices

Use workload identity or managed identity where available, short-lived credentials, TLS, and audience-scoped tokens. Use service-to-service authorization and least privilege. A gateway-authenticated request should not automatically be trusted by every internal service.

## 6.7 gRPC

gRPC uses Protocol Buffers contracts and commonly HTTP/2 for efficient typed communication and streaming. It is often a good fit for internal service-to-service calls. REST remains useful for browser-facing APIs, broad interoperability, and easy debugging. Choose based on clients, tooling, latency, streaming, and contract needs.

## 6.8 Distributed caching

Use Redis or another shared cache when instances need common cached values. Define key structure, TTL, invalidation, tenant isolation, stampede prevention, failure behavior, and sensitive-data policy. A cache is not the source of truth unless the system is deliberately designed that way.

---

# 7. System design and deployment

## 7.1 Design a high-scale e-commerce API

A strong interview answer starts with requirements:

- Expected traffic and peak-to-average ratio
- Product read/write mix
- Inventory correctness and overselling rules
- Payment and order consistency requirements
- Availability and recovery objectives
- Data privacy, audit, and regulatory constraints
- Latency targets and geographic coverage

A possible architecture includes:

- CDN/WAF and gateway
- Stateless ASP.NET Core API instances
- Product/catalog read cache
- Relational database for orders and transactional state
- Inventory reservation with concurrency control
- Payment provider integration with idempotency keys
- Broker plus outbox for asynchronous workflows
- Background workers for email, fulfillment, and report generation
- Centralized logs, metrics, traces, and alerting

Do not start with microservices or sharding by default. First identify domain boundaries and actual bottlenecks. A modular monolith with a relational database can be a better initial architecture.

## 7.2 Cloud deployment choices

Choose based on operations, scale, networking, compliance, cost, and team expertise:

- Managed application platform for simpler operations
- Container apps or managed container services for containerized workloads
- Kubernetes when orchestration flexibility and workload scale justify its overhead
- Serverless functions for event-triggered jobs with suitable execution characteristics

Use managed identity, secret management, health checks, autoscaling, private networking where required, and infrastructure as code.

## 7.3 Zero-downtime deployment

Common techniques:

- Rolling deployment with readiness checks
- Blue-green deployment with traffic switching
- Canary release with gradual traffic increase
- Feature flags for separating deployment from feature activation

Zero downtime also requires backward-compatible database changes: add new schema first, deploy code that supports both old and new shapes, migrate data, then remove old schema in a later release. Avoid destructive schema changes in the same step as the first new-code deployment.

## 7.4 Horizontal vs vertical scaling

- **Vertical:** give one instance more CPU/RAM; simple, but bounded by machine limits.
- **Horizontal:** add instances; improves capacity and resilience, but requires statelessness or externalized state, load balancing, and careful handling of shared dependencies.

Scaling API replicas will not solve a database bottleneck unless database capacity and query behavior are also addressed.

## 7.5 Terraform and CI/CD

Terraform defines infrastructure declaratively and supports reviewable, repeatable changes. Protect state, use least privilege, review plans, and separate environments carefully.

A .NET pipeline commonly includes:

1. Restore and build.
2. Unit tests and integration tests.
3. Formatting, static analysis, dependency and security scanning.
4. Publish artifact/container image.
5. Sign/scan image where required.
6. Deploy to a test environment.
7. Smoke tests and health checks.
8. Approval or progressive rollout for production.
9. Rollback strategy and deployment audit trail.

## 7.6 Logging, metrics, tracing, and observability

- **Logs:** discrete events and diagnostic details.
- **Metrics:** numeric time series such as request rate, latency, errors, CPU, queue depth.
- **Traces:** a request's path across services and dependencies.

Use OpenTelemetry instrumentation and export to a supported backend. Propagate trace context across HTTP and messaging. Monitor service-level indicators such as latency percentiles, error rates, and saturation. Avoid logging access tokens, secrets, and sensitive payloads.

## 7.7 Global APIs

Use edge routing/CDN for suitable content, regional deployments where justified, health-aware traffic routing, and data residency-aware storage. Decide whether data is globally replicated, regionally owned, or partitioned by tenant. Multi-region writes and failover are data-consistency problems as much as networking problems.

---

# 8. Advanced C# coding patterns

## 8.1 Custom `IAsyncEnumerable<T>`

Prefer an async iterator unless you need to implement the interface manually:

```csharp
public static async IAsyncEnumerable<int> GenerateAsync(
    int max,
    [System.Runtime.CompilerServices.EnumeratorCancellation]
    CancellationToken cancellationToken = default)
{
    for (var i = 1; i <= max; i++)
    {
        await Task.Delay(10, cancellationToken);
        yield return i;
    }
}
```

## 8.2 Priority queue

.NET 6+ provides `PriorityQueue<TElement, TPriority>`, backed by a heap.

```csharp
var queue = new PriorityQueue<string, int>();

queue.Enqueue("low priority", 3);
queue.Enqueue("urgent", 1);
queue.Enqueue("normal", 2);

while (queue.TryDequeue(out var item, out var priority))
{
    Console.WriteLine($"{priority}: {item}");
}
```

By default, lower priority values are dequeued first. Enqueue and dequeue are `O(log n)`; peek is `O(1)`.

## 8.3 Thread-safe cache

`ConcurrentDictionary` provides thread-safe dictionary operations, but its value factory may run more than once under contention. It is not an expiration policy by itself.

```csharp
private readonly ConcurrentDictionary<string, ProductDto> _cache = new();

public ProductDto GetOrAdd(string key, Func<string, ProductDto> factory)
    => _cache.GetOrAdd(key, factory);
```

For expiration, size limits, and cache policies, use `IMemoryCache` or a distributed cache as appropriate.

## 8.4 Fibonacci: memoization and iterative alternative

Memoization avoids repeated work, but the simple dictionary example is not thread-safe and recursion can overflow the stack for large `n`. `long` also overflows beyond the representable Fibonacci range.

```csharp
public static long Fibonacci(int n)
{
    if (n < 0)
        throw new ArgumentOutOfRangeException(nameof(n));

    if (n > 92)
        throw new ArgumentOutOfRangeException(
            nameof(n), "The result does not fit in Int64.");

    if (n < 2)
        return n;

    long previous = 0;
    long current = 1;

    for (int i = 2; i <= n; i++)
    {
        long next = checked(previous + current);
        previous = current;
        current = next;
    }

    return current;
}
```

This iterative implementation uses `O(n)` time and `O(1)` extra space.

## 8.5 Optimize slow LINQ / EF Core queries

1. Inspect generated SQL and query plans.
2. Confirm filters can use appropriate indexes.
3. Project only needed columns.
4. Use `AsNoTracking()` for read-only entity queries.
5. Avoid client-side evaluation or materializing before filtering.
6. Avoid N+1 queries.
7. Use pagination and suitable ordering.
8. Consider compiled queries only after measurement.

```csharp
var page = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .ThenByDescending(o => o.Id)
    .Select(o => new OrderSummaryDto(o.Id, o.CreatedAt, o.Total))
    .Take(50)
    .ToListAsync(cancellationToken);
```

For deep pagination, keyset pagination using the last seen ordering values may perform better than large `Skip` values.

## 8.6 Request/response body logging middleware

Be cautious: body logging can expose passwords, access tokens, health data, and personal information. It can also add buffering and memory costs. Prefer structured metadata and log bodies only with explicit redaction, size limits, content-type filters, and a justified requirement.

A safer default is to log method, route, status, elapsed time, trace ID, and a request/correlation ID—not raw bodies.

## 8.7 Worker queue with `System.Threading.Channels`

Channels support asynchronous producer-consumer patterns with backpressure.

```csharp
using System.Threading.Channels;

var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = true,
    SingleWriter = false
});

async Task ProduceAsync(CancellationToken cancellationToken)
{
    for (int i = 0; i < 1000; i++)
        await channel.Writer.WriteAsync(i, cancellationToken);

    channel.Writer.Complete();
}

async Task ConsumeAsync(CancellationToken cancellationToken)
{
    await foreach (var item in channel.Reader.ReadAllAsync(cancellationToken))
    {
        await ProcessAsync(item, cancellationToken);
    }
}

static Task ProcessAsync(int item, CancellationToken cancellationToken)
{
    Console.WriteLine(item);
    return Task.CompletedTask;
}
```

This is an in-process queue. It does not survive process failure. Use a durable broker or persistent job store when work must not be lost.

## 8.8 `Parallel.ForEach` vs `Task.WhenAll`

- `Parallel.ForEach` is primarily suited to CPU-bound parallel loops.
- `Task.WhenAll` waits for multiple tasks and is often appropriate for concurrent I/O operations.
- Do not launch unbounded tasks for millions of items; use bounded concurrency.

```csharp
var tasks = ids.Select(async id =>
{
    return await client.GetStringAsync($"/items/{id}", cancellationToken);
});

var results = await Task.WhenAll(tasks);
```

For large input sets, limit concurrent calls with `SemaphoreSlim`, a bounded channel, or a concurrency-limiting library.

## 8.9 `System.Text.Json`

Use the built-in serializer and source generation when appropriate rather than writing a custom JSON serializer unless there is a specific, measured requirement.

```csharp
var options = new System.Text.Json.JsonSerializerOptions
{
    PropertyNamingPolicy = System.Text.Json.JsonNamingPolicy.CamelCase
};

string json = System.Text.Json.JsonSerializer.Serialize(product, options);
ProductDto? result =
    System.Text.Json.JsonSerializer.Deserialize<ProductDto>(json, options);
```

---

# 9. Senior-level interview answer framework

For each architecture or performance question, answer in this order:

1. **Clarify requirements:** traffic, latency, consistency, security, budget, and operational constraints.
2. **State the principle:** explain the pattern in one or two sentences.
3. **Describe the design:** components, responsibilities, and data flow.
4. **Discuss trade-offs:** complexity, cost, latency, consistency, and maintainability.
5. **Cover failure modes:** timeouts, retries, duplicate messages, cache staleness, overload, and recovery.
6. **Explain measurement:** metrics, load tests, traces, query plans, and acceptance thresholds.
7. **Give an example:** a realistic .NET implementation or a project scenario.

## Common senior-level follow-up questions

- How would you prove this is faster? Which benchmark and production metric would you use?
- What happens when Redis, the database, or the message broker is unavailable?
- How do you prevent duplicate processing?
- What is the consistency model, and can the business accept stale reads?
- How does the design behave under overload?
- How are secrets, tenant boundaries, and permissions enforced?
- How will you deploy a schema change without downtime?
- What is the rollback plan?
- What is the simplest design that meets the requirements?

## Final advice

Avoid absolute statements such as “async always makes an API faster,” “microservices are always more scalable,” or “`ConfigureAwait(false)` always switches to a ThreadPool thread.” Senior-level answers explain the conditions, trade-offs, and failure modes, and show how to validate decisions with measurements.
