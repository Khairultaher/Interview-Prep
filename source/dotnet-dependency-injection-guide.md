# Dependency Injection in .NET: Lifetimes, Scopes, and Advanced Patterns

A practical guide to the built-in .NET dependency injection (DI) container, with examples for ASP.NET Core and desktop applications such as WPF and Windows Forms.

## Contents

1. [What dependency injection solves](#1-what-dependency-injection-solves)
2. [How the built-in container works](#2-how-the-built-in-container-works)
3. [Service lifetimes](#3-service-lifetimes)
4. [What is a logical scope?](#4-what-is-a-logical-scope)
5. [Lifetime mismatches and captive dependencies](#5-lifetime-mismatches-and-captive-dependencies)
6. [Circular dependencies](#6-circular-dependencies)
7. [Parameterized injection and factories](#7-parameterized-injection-and-factories)
8. [Scoped services in background work](#8-scoped-services-in-background-work)
9. [Dependency injection in desktop apps](#9-dependency-injection-in-desktop-apps)
10. [Validation, disposal, and common mistakes](#10-validation-disposal-and-common-mistakes)
11. [Quick reference](#11-quick-reference)

---

## 1. What dependency injection solves

A class has a *dependency* when it needs another object to do its work. Dependency injection means those objects are supplied from outside rather than constructed directly inside the class.

Without DI:

```csharp
public sealed class OrderService
{
    private readonly SqlOrderRepository _repository = new();

    public Task<Order?> GetAsync(int id) =>
        _repository.GetAsync(id);
}
```

`OrderService` is tightly coupled to `SqlOrderRepository`. Replacing the repository, testing the service in isolation, or configuring its dependencies becomes harder.

With constructor injection:

```csharp
public interface IOrderRepository
{
    Task<Order?> GetAsync(int id);
}

public sealed class OrderService(IOrderRepository repository)
{
    private readonly IOrderRepository _repository = repository;

    public Task<Order?> GetAsync(int id) =>
        _repository.GetAsync(id);
}
```

Register the mapping at the composition root—usually `Program.cs`:

```csharp
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<OrderService>();
```

The container supplies `IOrderRepository` when it creates `OrderService`. The class declares what it needs; registration determines which implementation is supplied.

**Why use DI?**

- **Loose coupling:** depend on abstractions instead of specific implementations.
- **Testability:** substitute a fake or mock implementation in tests.
- **Centralized configuration:** define implementations and lifetimes in one place.
- **Explicit dependencies:** constructor parameters show what a class needs.
- **Lifecycle management:** the container tracks and disposes services it creates when their lifetime ends.

DI is a technique, not a guarantee of good design. A class with too many constructor parameters may still have too many responsibilities.

## 2. How the built-in container works

### 2.1 Registration

When you write:

```csharp
services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

the service collection records a descriptor containing information such as:

- The service type (`IOrderRepository`)
- The implementation type (`SqlOrderRepository`)
- The lifetime (`Scoped`)

Registrations are configuration; this line normally does **not** immediately create `SqlOrderRepository`.

### 2.2 Building the provider

In a typical application:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<OrderService>();

var app = builder.Build();
```

`Build()` creates the service provider used to resolve registered services. The built-in provider prepares resolution machinery and caches information needed to construct service call sites. It does not eagerly instantiate every registered service.

Avoid building a second provider manually with `services.BuildServiceProvider()` during registration. Doing so can create duplicate singleton instances and separate disposal tracking. Register services first, then let the application host build the provider.

### 2.3 Resolving a service

Suppose a controller depends on `OrderService`, which depends on `IOrderRepository`:

```csharp
public sealed class OrdersController(OrderService orderService)
    : ControllerBase
{
    // Use orderService in action methods.
}
```

Conceptually, the container follows the dependency graph:

```text
OrdersController
    └── OrderService
          └── IOrderRepository
                └── SqlOrderRepository
```

When the controller is created, the framework asks the container for its dependencies. The container selects the registered implementation, resolves that implementation's constructor parameters recursively, and reuses instances according to their lifetimes.

This is a conceptual model rather than a promise about every internal implementation detail. The built-in container uses optimized resolution paths; application code should rely on its documented behavior, not internal implementation classes.

### 2.4 Constructor selection

Prefer one public constructor per DI-managed class. If a class has multiple public constructors, the container attempts to choose a suitable constructor based on the services it can resolve; ambiguous choices can cause an exception. A single, explicit constructor is easier to understand and maintain.

### 2.5 What happens when a service is disposable?

If the container creates a disposable service, it tracks it for disposal at the end of the applicable lifetime:

- Transient disposable: typically disposed when the scope that resolved it is disposed.
- Scoped disposable: disposed when its scope is disposed.
- Singleton disposable: disposed when the root provider is disposed.

Do not manually dispose a service that was injected and is owned by the container. If you create an object yourself, you generally own its disposal unless ownership is explicitly transferred.

## 3. Service lifetimes

.NET's built-in container supports three standard lifetimes.

### 3.1 Transient

```csharp
services.AddTransient<IFormatter, JsonFormatter>();
```

A new instance is created each time the service is resolved. If a transient is a dependency of a singleton, however, it is created when the singleton is constructed and then held by that singleton. So “transient” does not mean that every method call creates a new instance.

**Good fits:** small, stateless helpers and lightweight services.

### 3.2 Scoped

```csharp
services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

One instance is reused within a particular DI scope. A new scope gets a different instance.

In a typical ASP.NET Core request, the framework creates a request scope, so a scoped service is usually shared within that request and disposed at its end. A scope is not inherently an HTTP request: in desktop apps, console apps, and background workers, the application defines scopes explicitly.

**Good fits:** EF Core `DbContext`, unit-of-work services, and services that share state during one operation.

### 3.3 Singleton

```csharp
services.AddSingleton<IClock, SystemClock>();
```

One instance is used for the lifetime of the root service provider. It may be resolved from many requests or scopes.

**Good fits:** thread-safe shared services, immutable configuration-derived helpers, and caches designed for concurrent access.

Singleton services must be designed for concurrency. Avoid storing request- or operation-specific state in a singleton, and do not directly inject scoped services into one.

### 3.4 Comparison

| Lifetime | Instance reuse | Typical disposal boundary | Common use |
|---|---|---|---|
| Transient | New instance per resolution, subject to dependency graph behavior | Scope/provider that tracks it | Lightweight stateless helper |
| Scoped | One per scope | End of scope | `DbContext`, unit of work |
| Singleton | One per root provider | Provider/host shutdown | Thread-safe shared service |

### 3.5 Example: observing instance reuse

```csharp
public sealed class InstanceMarker
{
    public Guid Id { get; } = Guid.NewGuid();
}

// Registration:
services.AddTransient<TransientMarker>();
services.AddScoped<ScopedMarker>();
services.AddSingleton<SingletonMarker>();
```

Resolve the same scoped service twice from one scope and its `Id` remains the same. Resolve it from a second scope and the `Id` differs. A singleton's `Id` stays the same across both scopes. A transient normally has a different `Id` on each separate resolution.

## 4. What is a logical scope?

A **scope** is a boundary for scoped-service reuse and disposal. “Logical scope” is an informal way to describe the operation or activity represented by that boundary; it is not a separate special DI lifetime.

Create a scope with `IServiceScopeFactory` or `IServiceProvider.CreateScope()`:

```csharp
using var scope = serviceScopeFactory.CreateScope();

var repository =
    scope.ServiceProvider.GetRequiredService<IOrderRepository>();

await repository.GetAsync(42);
```

All resolutions of `IOrderRepository` from that scope reuse the same scoped instance. Disposing the scope disposes its scoped disposable services.

Examples of useful scope boundaries:

- ASP.NET Core: normally one HTTP request.
- WPF/Windows Forms: one user operation, dialog, or window workflow.
- Background worker: one batch or unit of work.
- Console application: one processing operation.

A scope does not automatically mean “one thread,” “one window,” or “one database transaction.” Those are design choices. A scope controls DI instance reuse and disposal; database transactions must be started and managed separately.

## 5. Lifetime mismatches and captive dependencies

A *captive dependency* occurs when a longer-lived service holds a reference to a shorter-lived service beyond the shorter-lived service's intended lifetime.

### 5.1 The classic error: singleton consumes scoped service

```csharp
services.AddSingleton<ReportService>();
services.AddScoped<AppDbContext>();
```

```csharp
public sealed class ReportService(AppDbContext dbContext)
{
    // ...
}
```

`ReportService` is created once, but `AppDbContext` is intended to belong to a scope. The singleton would retain one context for too long. With scope validation enabled, the container reports an error similar to:

> Cannot consume scoped service from singleton.

Even where validation is disabled, the design is unsafe: the context may be used across unrelated operations, may not be disposed when expected, and is not thread-safe.

**Best fix:** make the consumer scoped if it represents one operation:

```csharp
services.AddScoped<ReportService>();
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### 5.2 Can a scoped service depend on a singleton?

Yes, provided the singleton is safe to share and does not retain operation-specific state.

```csharp
services.AddSingleton<IClock, SystemClock>();
services.AddScoped<OrderService>();
```

The scoped `OrderService` can use the singleton clock.

### 5.3 Can a singleton depend on a transient?

Technically, yes. But the transient is resolved when the singleton is created and is then retained as a field for the singleton's lifetime. It will not be recreated for every operation. If that transient is stateful or not thread-safe, this may cause bugs.

### 5.4 Can a transient depend on a scoped service?

This can work when the transient is resolved inside a scope. But do not resolve a scoped dependency from the root provider or let a longer-lived object retain the transient that captured it. Keep the whole dependency graph within the intended scope.

### 5.5 Lifetime compatibility guide

| Consumer | Dependency | Guidance |
|---|---|---|
| Singleton | Singleton | Usually fine if thread-safe |
| Singleton | Scoped | Invalid design; usually throws with scope validation |
| Singleton | Transient | Allowed, but transient is effectively retained by singleton |
| Scoped | Singleton | Usually fine if singleton is thread-safe |
| Scoped | Scoped | Fine; same instance within that scope |
| Scoped | Transient | Allowed; transient created when resolved |
| Transient | Scoped | Fine only when resolved within a valid scope |
| Transient | Singleton | Usually fine if singleton is thread-safe |

### 5.6 Scope validation

For a generic host, enable validation in development where applicable. `Host.CreateApplicationBuilder` and ASP.NET Core's development hosting configuration typically enable useful validation defaults. You can also configure provider validation explicitly in custom setups. Validation can detect many captive dependencies and missing or invalid registrations early, but it cannot prove the whole application's concurrency or lifecycle design is correct.

## 6. Circular dependencies

A circular dependency exists when the dependency graph loops back to a service already being constructed.

```csharp
public sealed class ServiceA(ServiceB serviceB);

public sealed class ServiceB(ServiceA serviceA);
```

The container cannot construct either service first, so resolution fails with a circular-dependency exception.

### Preferred solution: change the design

Often the two classes have responsibilities that should be separated. Extract the shared behavior into a third service:

```text
ServiceA ──► SharedService
ServiceB ──► SharedService
```

Or let one higher-level coordinator call both services instead of having them call each other.

### Event/callback alternative

If one component only needs to react to something another component does, an event, callback, mediator, or domain event may remove the direct cycle. Choose the simplest mechanism that fits the architecture.

### Avoid using `IServiceProvider` to hide the cycle

Resolving one service manually from `IServiceProvider` may defer the failure or obscure dependencies rather than fix the design. The built-in container also does not automatically support `Lazy<T>` injection as a general circular-dependency solution. A factory can be appropriate for a genuine runtime creation requirement, but should not merely conceal a dependency cycle.

## 7. Parameterized injection and factories

Constructor injection is ideal when dependencies are known when the object is created. Sometimes an object also needs a value that is known only at runtime, such as a file name, customer ID, or export format.

### 7.1 Values from configuration: Options pattern

Use options for application configuration rather than hard-coding values in a factory.

`appsettings.json`:

```json
{
  "Email": {
    "Sender": "noreply@example.com",
    "RetryCount": 3
  }
}
```

Settings class and registration:

```csharp
public sealed class EmailOptions
{
    public string Sender { get; set; } = "";
    public int RetryCount { get; set; }
}

// Program.cs
builder.Services.Configure<EmailOptions>(
    builder.Configuration.GetSection("Email"));
```

Inject `IOptions<EmailOptions>` (or `IOptionsSnapshot<T>` / `IOptionsMonitor<T>` when their respective semantics are needed) into the service. Options are for configuration values, not arbitrary per-call values such as a user-selected file path.

### 7.2 Factory registration with a fixed value

```csharp
services.AddTransient<IFileExporter>(sp =>
    new FileExporter("csv"));
```

This supplies a fixed `"csv"` value every time the factory creates the service. It is not a runtime parameter that changes per request.

### 7.3 Runtime parameters: define a factory abstraction

```csharp
public interface IReportExporterFactory
{
    IReportExporter Create(string format);
}

public sealed class ReportExporterFactory : IReportExporterFactory
{
    private readonly IServiceProvider _services;

    public ReportExporterFactory(IServiceProvider services) =>
        _services = services;

    public IReportExporter Create(string format) =>
        format.ToLowerInvariant() switch
        {
            "csv" => ActivatorUtilities.CreateInstance<CsvReportExporter>(_services),
            "json" => ActivatorUtilities.CreateInstance<JsonReportExporter>(_services),
            _ => throw new ArgumentOutOfRangeException(
                nameof(format), format, "Unsupported format.")
        };
}
```

Register implementations and the factory:

```csharp
services.AddTransient<CsvReportExporter>();
services.AddTransient<JsonReportExporter>();
services.AddTransient<IReportExporterFactory, ReportExporterFactory>();
```

This example chooses an implementation based on a runtime value while allowing `ActivatorUtilities` to supply registered constructor dependencies. The factory must be used within the correct scope if the created object depends on scoped services. The caller should also understand who owns and disposes the created object; `ActivatorUtilities.CreateInstance` does not by itself make an object a container-owned scoped registration.

For simpler designs, use a delegate factory or a typed factory with a clear contract. Avoid spreading arbitrary service-locator calls throughout application code.

### 7.4 `ActivatorUtilities`

`ActivatorUtilities.CreateInstance<T>(provider, explicitArgument)` can combine explicit construction arguments with dependencies resolved from DI:

```csharp
var exporter = ActivatorUtilities.CreateInstance<CsvReportExporter>(
    serviceProvider,
    outputPath);
```

For example, the constructor could accept `string outputPath` plus `ILogger<CsvReportExporter>`. The explicit argument supplies the runtime path and DI supplies the logger. Ensure argument types and constructor selection are unambiguous.

## 8. Scoped services in background work

Hosted services such as `BackgroundService` are registered as singletons. They should not directly capture a scoped repository or `DbContext` in their constructor.

Instead, create a scope for each unit of work:

```csharp
public sealed class CleanupWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<CleanupWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using (var scope = scopeFactory.CreateAsyncScope())
            {
                var cleanup =
                    scope.ServiceProvider.GetRequiredService<ICleanupService>();

                await cleanup.RunAsync(stoppingToken);
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

This pattern creates and disposes a fresh scope for each iteration. `CreateAsyncScope()` is useful when scoped services may implement `IAsyncDisposable`. Handle cancellation and exceptions according to the worker's needs.

## 9. Dependency injection in desktop apps

Unlike ASP.NET Core, WPF and Windows Forms do not automatically create a DI scope for each request. The app should decide whether a scope represents a button operation, a dialog, a window, or another unit of work.

### 9.1 Configure the host

A WPF application can use the Generic Host to configure DI, logging, and configuration:

```csharp
// App.xaml.cs (illustrative WPF example)
public partial class App : Application
{
    public IHost Host { get; }

    public App()
    {
        Host = Host.CreateDefaultBuilder()
            .ConfigureServices((context, services) =>
            {
                services.AddSingleton<MainWindow>();
                services.AddScoped<IOrderService, OrderService>();
                services.AddDbContext<AppDbContext>(options =>
                    options.UseSqlServer(
                        context.Configuration.GetConnectionString("Default")));
            })
            .Build();
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await Host.StartAsync();

        var mainWindow = Host.Services.GetRequiredService<MainWindow>();
        mainWindow.Show();

        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await Host.StopAsync();
        Host.Dispose();

        base.OnExit(e);
    }
}
```

This is an illustrative pattern; adapt startup and shutdown handling to your app's actual window lifecycle and exception-handling policy. Do not use `async void` for ordinary application methods—event overrides are one of the few contexts where it is conventional.

### 9.2 Scope per operation

For a button action that performs a discrete database operation:

```csharp
private readonly IServiceScopeFactory _scopeFactory;

public MainWindow(IServiceScopeFactory scopeFactory)
{
    InitializeComponent();
    _scopeFactory = scopeFactory;
}

private async void LoadOrders_Click(object sender, RoutedEventArgs e)
{
    await using var scope = _scopeFactory.CreateAsyncScope();

    var service =
        scope.ServiceProvider.GetRequiredService<IOrderService>();

    var orders = await service.GetOrdersAsync();

    // Update the UI on the UI thread if needed.
}
```

The scope keeps the operation's scoped services together and disposes them when the operation completes. In real UI code, handle exceptions and disable or otherwise guard the button if concurrent clicks would be problematic.

### 9.3 Scope per window or dialog

A window-specific scope can be appropriate when its view model and services should live for the window's lifetime. Create the scope when opening the window and dispose it when the window closes. Do not create a scope in a method and immediately dispose it if the window continues using services obtained from that scope.

A simple ownership wrapper can keep the scope alive:

```csharp
public sealed class ScopedWindow<TWindow> : IDisposable
    where TWindow : Window
{
    private readonly AsyncServiceScope _scope;

    public TWindow Window { get; }

    public ScopedWindow(IServiceScopeFactory scopeFactory)
    {
        _scope = scopeFactory.CreateAsyncScope();
        Window = _scope.ServiceProvider.GetRequiredService<TWindow>();
        Window.Closed += OnClosed;
    }

    private async void OnClosed(object? sender, EventArgs e)
    {
        Window.Closed -= OnClosed;
        await _scope.DisposeAsync();
    }

    public void Dispose()
    {
        Window.Closed -= OnClosed;
        _scope.DisposeAsync().AsTask().GetAwaiter().GetResult();
    }
}
```

This is a sketch, not a universal window-management abstraction: avoid mixing synchronous and asynchronous disposal paths without a clear ownership policy. A simpler app can explicitly manage a scope in its window-opening code and dispose it from the window's `Closed` event.

### 9.4 Do not resolve scoped services from the root provider

This is a common desktop-app mistake:

```csharp
// Avoid for scoped services:
var service = appHost.Services.GetRequiredService<IOrderService>();
```

If `IOrderService` is scoped, resolving it from the root provider can either fail under validation or effectively retain it for too long when validation is disabled. Create a scope and resolve it from `scope.ServiceProvider`.

## 10. Validation, disposal, and common mistakes

- **Do not inject scoped services into singletons.** Use a scoped consumer or create a scope for each unit of work.
- **Do not keep a scoped `DbContext` for the whole desktop-app lifetime.** A context is designed for a short unit of work and is not thread-safe.
- **Do not call `BuildServiceProvider()` during service registration.** Let the host build the provider once.
- **Do not use `IServiceProvider` everywhere.** Prefer constructor injection; use factories or scopes at explicit boundaries.
- **Do not assume transient means “per method call.”** A transient captured by a singleton lives as long as that singleton holds it.
- **Do not forget scope disposal.** Prefer `using` / `await using` for operation scopes.
- **Do not assume a scope starts a transaction.** Manage database transactions separately.
- **Do not ignore thread safety.** Singleton services may be called concurrently.
- **Do not use `Lazy<T>` as a default fix for circular dependencies.** Refactor the dependency graph.
- **Do validate at startup.** Scope and dependency validation can expose problems earlier.

## 11. Quick reference

| Question | Practical answer |
|---|---|
| What does DI do? | Creates objects and supplies their declared dependencies |
| When are services created? | Usually when first resolved, not merely when registered |
| What is a scope? | A boundary for scoped-instance reuse and disposal |
| Is a scope always an HTTP request? | No; that is the usual ASP.NET Core convention |
| Which lifetime suits `DbContext`? | Scoped in typical applications |
| Why is scoped-in-singleton problematic? | The singleton can retain a scoped object too long |
| How should a background worker use a scoped service? | Create a scope per unit of work |
| How should a desktop app use scoped services? | Create and dispose scopes around operations or window lifetimes |
| How do I pass runtime values? | Use a factory or `ActivatorUtilities` |
| What causes circular dependency errors? | A dependency graph that loops back to a service being constructed |
| What is the best circular-dependency fix? | Refactor responsibilities to remove the cycle |

---

## Final takeaway

Think of the DI container as an object-construction and lifetime-management system. **Registration** tells it what to create, **resolution** builds the dependency graph, **lifetimes** determine reuse, and **scopes** define where scoped objects are shared and disposed. In desktop applications especially, deliberately choose the boundary for each unit of work and keep scoped services—particularly EF Core contexts—inside that boundary.
