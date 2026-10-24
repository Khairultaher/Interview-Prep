# .NET Dependency Injection (DI)

## Senior Developer & Solution Architect Interview Guide

## 1. Core concepts

**Dependency Injection (DI)** is a technique for supplying an object
with the dependencies it needs instead of having the object create them
itself.

``` csharp
public interface IEmailSender
{
    Task SendAsync(string to, string subject, CancellationToken cancellationToken = default);
}

public sealed class NotificationService(IEmailSender emailSender)
{
    public Task NotifyAsync(string email, CancellationToken cancellationToken = default) =>
        emailSender.SendAsync(email, "Notification", cancellationToken);
}
```

`NotificationService` depends on the abstraction `IEmailSender`; the
composition root decides which implementation to provide.

-   **Inversion of Control (IoC):** a broad principle in which control
    over object creation or flow is moved away from the component.
-   **Dependency Injection:** one way to implement IoC.
-   **Service Locator:** a component asks a container/provider for
    dependencies itself. This hides dependencies and is usually less
    desirable than constructor injection.

### Why DI matters

-   Reduces coupling and makes implementations replaceable.
-   Improves unit testing by allowing fakes or mocks.
-   Centralizes object construction and lifetime management.
-   Supports clean architecture when dependencies point toward
    abstractions and inner layers.

DI does not automatically make code well-designed. Interfaces for every
class, excessive registrations, and service classes with too many
dependencies can still create poor designs.

## 2. Built-in .NET container: registration and resolution

The built-in container is configured through `IServiceCollection`, which
holds `ServiceDescriptor` registrations. The host builds an
`IServiceProvider`, which resolves those registrations.

``` csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();
app.MapControllers();
app.Run();
```

Conceptually, resolution works as follows:

1.  Find a registration for the requested service type.
2.  Determine how to create its implementation.
3.  Resolve constructor dependencies recursively.
4.  Reuse or create instances according to their lifetimes.
5.  Track disposable instances created by the container so they can be
    disposed at the appropriate time.

The built-in container caches compiled resolution plans/call sites for
efficiency. Exact implementation details can change between .NET
versions; avoid claiming that every resolution always compiles an
expression tree or emits IL.

### Constructor selection and failures

The container normally uses a public constructor whose parameters it can
resolve. Ambiguous applicable constructors can cause an exception.
Missing registrations for required constructor parameters also cause
resolution to fail.

Prefer a single, explicit public constructor for DI-managed services.

## 3. Service lifetimes

  ---------------------------------------------------------------------------
  Lifetime          Instance reuse    Typical use           Main risk
  ----------------- ----------------- --------------------- -----------------
  `Transient`       A new instance    Lightweight,          Repeated
                    each time it is   stateless services    allocations;
                    requested                               disposable
                                                            transient
                                                            instances can be
                                                            retained by a
                                                            long-lived scope

  `Scoped`          One instance per  EF Core `DbContext`,  Incorrect use
                    scope             request-level         outside a scope
                                      business services     or capture by a
                                                            singleton

  `Singleton`       One instance for  Thread-safe caches,   Shared mutable
                    the root          immutable             state,
                    container's       configuration-based   concurrency bugs,
                    lifetime          services              captive
                                                            dependencies
  ---------------------------------------------------------------------------

In a typical ASP.NET Core app, a request creates a scope. A scoped
service is shared within that request scope, not universally across all
possible execution environments. Background workers and manually created
scopes have their own scope boundaries.

``` csharp
builder.Services.AddTransient<IClock, SystemClock>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddSingleton<IApplicationCache, ApplicationCache>();
```

### Lifetime rules to remember

-   A singleton must be safe for concurrent access.
-   Do not inject a scoped service directly into a singleton.
-   A transient dependency injected into a singleton is effectively held
    for the singleton's lifetime.
-   Scoped does not mean "one per thread"; it means "one per scope."
-   `AddDbContext<TContext>()` registers the context as scoped by
    default.

## 4. Captive dependencies and scope validation

A **captive dependency** occurs when a longer-lived service retains a
shorter-lived dependency.

``` csharp
public sealed class ReportWorker(IOrderDbContext context)
{
    // If ReportWorker is singleton and the context is scoped,
    // the context can be captured for too long.
}
```

This can cause stale state, incorrect sharing, concurrency issues, and
disposal problems. In Development, the default host commonly enables
scope validation, which can detect scoped services resolved from the
root provider or injected into singletons.

### Correct pattern for a background service

Create a scope for each unit of work and resolve scoped dependencies
from that scope:

``` csharp
public sealed class OrderWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<OrderWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var scope = scopeFactory.CreateAsyncScope();

            var processor = scope.ServiceProvider
                .GetRequiredService<IOrderProcessor>();

            await processor.ProcessNextAsync(stoppingToken);

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

Use `IServiceScopeFactory` when a long-lived component genuinely needs
to start a scoped unit of work. Avoid using `IServiceProvider` as a
general-purpose service locator throughout application code.

## 5. `IServiceProvider`, scopes, and disposal

-   **`IServiceProvider`:** resolves registered services from a
    provider.
-   **`IServiceScopeFactory`:** creates a new scope.
-   **`IServiceScope`:** exposes a scoped provider and manages scope
    disposal.
-   **Root provider:** owns singleton instances and root-resolved
    disposable services.
-   **Request scope:** manages scoped instances and disposable services
    created in that scope.

The container disposes container-created disposable services according
to their lifetime/scope. Do not manually dispose a service that DI owns.
If your code creates an object directly with `new`, your code generally
owns its disposal.

Avoid calling `BuildServiceProvider()` during service registration. It
can create a second container, duplicate singleton instances, and split
disposal/lifetime management. Let the host build the application
provider.

## 6. Injection styles

### Constructor injection --- default choice

``` csharp
public sealed class InvoiceService(IInvoiceRepository repository)
{
    public Task<Invoice?> GetAsync(int id, CancellationToken ct) =>
        repository.GetAsync(id, ct);
}
```

Makes required dependencies explicit and supports straightforward
testing.

### Method injection

Useful when a dependency is needed only for one ASP.NET Core endpoint:

``` csharp
app.MapGet("/health/details", (IDiagnosticService diagnostics) =>
    diagnostics.GetDetails());
```

MVC actions can also use `[FromServices]` for endpoint-specific
dependencies.

### Factory registration

Use a factory when construction needs configuration or a runtime choice:

``` csharp
builder.Services.AddScoped<IReportGenerator>(sp =>
{
    var options = sp.GetRequiredService<IOptions<ReportOptions>>().Value;
    return new ReportGenerator(options.Format);
});
```

Keep factories simple and avoid hiding a large dependency graph inside
them.

## 7. Options pattern

Use strongly typed options for configuration rather than scattering
configuration lookups throughout services.

``` csharp
public sealed class EmailOptions
{
    public const string SectionName = "Email";
    public string Sender { get; set; } = "";
}

builder.Services
    .AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .Validate(o => !string.IsNullOrWhiteSpace(o.Sender),
        "Email:Sender is required.")
    .ValidateOnStart();
```

-   `IOptions<T>`: suitable for stable options; does not provide
    per-request reloading semantics.
-   `IOptionsSnapshot<T>`: scoped snapshot, commonly used in
    request-scoped services.
-   `IOptionsMonitor<T>`: singleton-friendly access to current options
    and change notifications where supported.

Choose based on whether configuration reloads and whether the consuming
service is singleton or scoped.

## 8. `HttpClient` and typed clients

Do not repeatedly instantiate and dispose `HttpClient` for each request.
Use `IHttpClientFactory`, which manages handler pooling and lifetime.

``` csharp
builder.Services.AddHttpClient<IPaymentClient, PaymentClient>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["PaymentApi:BaseUrl"]!);
});
```

Typed clients keep HTTP behavior behind an abstraction. Be careful not
to capture request-specific or scoped state inside long-lived handlers.
Configure timeouts, authentication, resilience, and logging
deliberately.

## 9. DI in Clean Architecture and CQRS

A typical dependency direction is:

``` text
API / Presentation
       |
       v
Application (use cases, interfaces)
       |
       v
Domain (business rules)

Infrastructure implements Application interfaces
and is wired up at the composition root.
```

The Application layer defines interfaces such as `IOrderRepository`;
Infrastructure implements them with EF Core or external services. The
API/host wires registrations together.

``` csharp
// Infrastructure registration extension
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("Default")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        return services;
    }
}
```

In CQRS, handlers are often registered as transient or scoped depending
on the library and their dependencies. The important rule is that a
handler must not outlive its scoped dependencies. Keep business logic in
use cases/handlers rather than putting it into the DI registration
layer.

## 10. Common pitfalls and design guidance

1.  **Too many constructor parameters:** may indicate a class has too
    many responsibilities. Refactor around cohesive behavior rather than
    merely grouping dependencies into a "manager."
2.  **Circular dependency:** for example, `A -> B -> A`. The built-in
    container cannot construct the cycle; redesign the responsibilities
    or introduce an appropriate abstraction.
3.  **Service locator:** resolving arbitrary dependencies inside
    business methods hides the dependency graph and complicates tests.
    Use explicit injection unless runtime resolution is genuinely
    required.
4.  **Duplicate registrations:** for a single service type, normal
    resolution generally uses the last registration; `IEnumerable<T>`
    returns all registrations in registration order. Be intentional
    about overrides and decorators.
5.  **Registering the same implementation with different lifetimes:**
    separate registrations can produce separate instances. If one shared
    instance is required, register the implementation once and map
    abstractions to that same instance deliberately.
6.  **Thread safety:** singleton services may be used concurrently.
    Protect shared mutable state or avoid it.
7.  **Disposal:** do not dispose DI-owned services manually; dispose
    scopes you create.
8.  **Premature abstraction:** not every class needs an interface. Add
    abstractions where they establish a useful boundary, enable
    substitution, or improve isolation.
9.  **`BuildServiceProvider()` during registration:** avoid creating a
    second container.
10. **`DbContext` concurrency:** do not run parallel operations on the
    same EF Core context. Use separate scopes/contexts for independent
    concurrent work.

## 11. Testing DI and application behavior

Unit tests can instantiate a class directly with a fake dependency.
Integration tests can use the actual service provider and override
selected registrations.

``` csharp
var services = new ServiceCollection();
services.AddLogging();
services.AddScoped<IUserService, UserService>();
services.AddScoped<IUserRepository, FakeUserRepository>();

using var provider = services.BuildServiceProvider(
    new ServiceProviderOptions
    {
        ValidateScopes = true,
        ValidateOnBuild = true
    });

using var scope = provider.CreateScope();
var service = scope.ServiceProvider.GetRequiredService<IUserService>();
```

In a real ASP.NET Core integration test, prefer the test
host/WebApplicationFactory configuration mechanisms to replace
registrations. Avoid building multiple providers in production setup
merely to test resolution.

## 12. Senior/architect interview questions

### Q1. Is DI the same as IoC?

No. IoC is the broader principle of transferring control; DI is one
technique for supplying dependencies.

### Q2. Why is `DbContext` scoped by default?

It represents a short unit of work, tracks entity state, and is not
thread-safe. A scope usually aligns with a web request or a discrete
background operation.

### Q3. Can a singleton depend on a transient?

Technically yes, but the transient instance injected into the singleton
is retained by it. The transient is therefore not recreated for each
operation. Ensure it is safe to retain and does not depend on
shorter-lived services.

### Q4. Can a singleton use a scoped service?

Not by direct constructor injection. Create an explicit scope with
`IServiceScopeFactory` for each unit of work and resolve the scoped
service from that scope.

### Q5. What happens if a service is registered more than once?

For a single service resolution, the last registration normally wins.
Resolving `IEnumerable<T>` returns all registrations. Registration order
can therefore matter.

### Q6. What happens with circular dependencies?

Resolution fails with an exception. Break the cycle by clarifying
responsibilities, introducing an event or abstraction where appropriate,
or restructuring the workflow.

### Q7. Why avoid `BuildServiceProvider()` inside registration?

It creates another provider and can duplicate singleton instances or
cause separate disposal scopes. Build the final provider through the
host.

### Q8. How does the container handle disposal?

It tracks disposable instances it creates and disposes them when their
owning scope/provider is disposed. Code that creates an object outside
DI owns its lifecycle.

### Q9. Is the built-in container always sufficient?

It supports common registrations, lifetimes, factories, open generics,
and enumerable registrations. A third-party container may be justified
for specific advanced features, but adds integration and maintenance
costs.

### Q10. How would you diagnose a DI error in production?

Read the complete exception and service type chain; check registrations,
constructor parameters, lifetime mismatches, environment-specific
configuration, and scope boundaries. Add a small integration test that
builds the host/provider with validation enabled.

## 13. Quick revision checklist

-   [ ] Explain DI, IoC, and service locator distinctly.
-   [ ] Explain transient, scoped, and singleton with examples.
-   [ ] Explain captive dependencies and how to fix them.
-   [ ] Describe root provider, request scope, and disposal.
-   [ ] Explain why `DbContext` is scoped and not thread-safe.
-   [ ] Know `IServiceProvider` vs `IServiceScopeFactory`.
-   [ ] Avoid `BuildServiceProvider()` during registration.
-   [ ] Understand duplicate registrations and `IEnumerable<T>`.
-   [ ] Explain options lifetimes and `IHttpClientFactory`.
-   [ ] Explain how DI supports Clean Architecture and CQRS.
-   [ ] Discuss testing, concurrency, and production troubleshooting.

## Interview answer framework

For architecture questions, structure your answer in this order:

1.  **Definition** --- explain the concept precisely.
2.  **Mechanics** --- describe what .NET does.
3.  **Trade-offs** --- explain benefits and costs.
4.  **Failure mode** --- give a real pitfall.
5.  **Production example** --- connect it to an API, EF Core, a worker,
    or a CQRS handler.
