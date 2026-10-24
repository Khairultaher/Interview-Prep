# Dependency Injection in .NET: Lifetimes, Runtime Parameters, and Features Through .NET 10

> A concise, practical reference for built-in .NET dependency injection (DI). Examples use modern C# and `Program.cs`.

## 1. Service lifetimes

| Lifetime | Instance behavior | Typical use |
|---|---|---|
| `Singleton` | One instance for the root service provider/application lifetime | Thread-safe shared services, caches, configuration-independent helpers |
| `Scoped` | One instance per scope; in ASP.NET Core, usually one per HTTP request | `DbContext`, request-oriented business services |
| `Transient` | A new instance each time the service is resolved | Lightweight, stateless services |

### Which lifetime can depend on which?

The table describes the usual constructor-injection combinations. `Singleton → Scoped` is the important combination to avoid.

| Consumer \ Dependency | Singleton | Scoped | Transient |
|---|---:|---:|---:|
| **Singleton** | ✅ | ❌ | ⚠️ Allowed, but the transient instance is effectively retained by the singleton |
| **Scoped** | ✅ | ✅ | ✅ |
| **Transient** | ✅ | ✅* | ✅ |

\* A transient service that depends on a scoped service must itself be resolved within a valid scope. Do not resolve that object from the root provider.

### Why is Singleton → Scoped unsafe?

A singleton can live for the entire application, while a scoped service is intended to live only for one scope. Injecting a scoped dependency into a singleton can retain the first scoped instance and reuse it across requests. This is called a **captive dependency** and can cause stale data, cross-request state leakage, or use-after-disposal problems.

```csharp
builder.Services.AddSingleton<ReportService>();
builder.Services.AddScoped<AppDbContext>();

public sealed class ReportService(AppDbContext db) // Invalid lifetime design
{
}
```

In development, scope validation generally detects this and throws an exception such as:

`Cannot consume scoped service 'AppDbContext' from singleton 'ReportService'.`

### Correct pattern for background services

If a singleton/background worker needs a scoped service, create a scope for each unit of work.

```csharp
public sealed class ReportWorker(IServiceScopeFactory scopeFactory)
{
    public async Task RunOnceAsync(CancellationToken cancellationToken)
    {
        await using var scope = scopeFactory.CreateAsyncScope();

        var reportService =
            scope.ServiceProvider.GetRequiredService<IReportService>();

        await reportService.GenerateAsync(cancellationToken);
    }
}
```

Avoid storing the scoped service in a singleton field. Also remember that singleton services must be thread-safe when accessed concurrently.

## 2. Passing parameters to a service

DI automatically supplies registered dependencies. For values that vary at runtime—such as a file path, report ID, or customer ID—choose the approach that matches the value's purpose.

### A. Factory registration for a fixed/configured value

```csharp
builder.Services.AddScoped<FileProcessor>(sp =>
    new FileProcessor(
        sp.GetRequiredService<ILogger<FileProcessor>>(),
        "C:\\data\\input.csv"));
```

Use this when the value is fixed at registration time. Avoid calling `BuildServiceProvider()` during registration; use the `IServiceProvider` supplied to the factory.

### B. Options pattern for application configuration

`appsettings.json`:

```json
{
  "FileProcessing": {
    "InputPath": "C:\\data\\input.csv"
  }
}
```

```csharp
public sealed class FileProcessingOptions
{
    public string InputPath { get; set; } = "";
}

// Program.cs
builder.Services.Configure<FileProcessingOptions>(
    builder.Configuration.GetSection("FileProcessing"));
builder.Services.AddScoped<FileProcessor>();
```

```csharp
using Microsoft.Extensions.Options;

public sealed class FileProcessor(IOptions<FileProcessingOptions> options)
{
    public string InputPath => options.Value.InputPath;
}
```

Use `IOptions<T>` for configuration that is effectively fixed after startup. Use `IOptionsSnapshot<T>` in scoped services when per-scope configuration refresh is useful, or `IOptionsMonitor<T>` when a long-lived service needs to observe supported configuration changes.

### C. Runtime values: use a factory or `ActivatorUtilities`

Do not register every possible runtime ID as a DI service. Let DI resolve normal dependencies and supply the runtime value explicitly.

```csharp
public sealed class ReportExporter(
    ILogger<ReportExporter> logger,
    int reportId)
{
    public Task ExportAsync() => Task.CompletedTask;
}
```

Create an instance with DI-provided dependencies plus a runtime argument:

```csharp
var exporter = ActivatorUtilities.CreateInstance<ReportExporter>(
    serviceProvider,
    reportId);
```

Call this from an appropriate scope if the service's dependency graph includes scoped services. In application code, prefer a small, explicit factory abstraction when object creation is part of your domain logic.

> The built-in container does not automatically provide arbitrary `Func<T>` factories for every service. Register a delegate yourself or use a dedicated factory type if that is the intended design.

## 3. Keyed services (introduced in .NET 8)

Keyed services allow multiple implementations of the same service interface to be registered under different keys.

```csharp
builder.Services.AddKeyedScoped<INotificationService, EmailNotificationService>("email");
builder.Services.AddKeyedScoped<INotificationService, SmsNotificationService>("sms");
```

Inject a specific implementation:

```csharp
using Microsoft.Extensions.DependencyInjection;

public sealed class AlertService(
    [FromKeyedServices("email")] INotificationService notificationService)
{
}
```

Or resolve by key when runtime selection is genuinely needed:

```csharp
var service = serviceProvider.GetRequiredKeyedService<INotificationService>("email");
```

Use keyed services for known implementation choices, such as storage providers or notification channels. For complex business selection, a strategy/factory abstraction may be easier to test and maintain.

## 4. What changed through .NET 10?

### .NET 8

- **Keyed DI services**: register and resolve implementations by key.
- The built-in DI container continues to support constructor injection, service factories, scopes, and the options pattern.

### .NET 9

- No new general-purpose built-in DI source generator replaced the standard container. Avoid treating community proposals or third-party container features as built-in .NET features.

### .NET 10

- The current DI documentation includes `KeyedService.AnyKey` behavior for keyed registrations. In particular, calling singular `GetKeyedService()` with `KeyedService.AnyKey` throws `InvalidOperationException`; use `AnyKey` with plural keyed-service enumeration where appropriate.
- .NET 10 also contains broader runtime, library, and ASP.NET Core improvements, but do not assume these represent a new DI lifetime model.
- The core lifetimes remain **Singleton, Scoped, and Transient**. There is no general built-in feature that makes injecting a scoped dependency directly into a singleton safe.

For version-specific details, consult the official documentation linked below.

## 5. Recommended practices

- Enable scope validation and fix lifetime mismatches rather than suppressing the error.
- Use `AddSingleton`, `AddScoped`, and `AddTransient` according to the actual lifetime and state requirements.
- Keep singleton services thread-safe; avoid mutable per-request state in singletons.
- Use the options pattern for configuration and factories for genuine runtime arguments.
- Prefer constructor injection over resolving services manually from `IServiceProvider`.
- Avoid `BuildServiceProvider()` during service registration.
- Do not manually dispose services created and owned by the DI container.
- For hosted/background services, create and dispose a scope for each unit of work when scoped dependencies are needed.

## Official references

- [Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Dependency injection guidelines](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines)
- [Dependency injection in ASP.NET Core (.NET 10)](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-10.0)
- [What's new in .NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview)
