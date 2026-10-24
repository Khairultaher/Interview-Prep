# Architecture and Design Patterns

This guide combines the distinct source materials below. Original explanations, examples, and interview questions are retained; exact duplicate source files are excluded and documented in the library index.

## Included source materials

- [23-design-patterns-csharp-senior-viva.md](#23-design-patterns-csharp-senior-viva-md)
- [clean-architecture-cqrs-vs-mvc-dotnet8.md](#clean-architecture-cqrs-vs-mvc-dotnet8-md)
- [senior_dotnet_architect_viva_prep.md](#senior-dotnet-architect-viva-prep-md)
---

<a id="23-design-patterns-csharp-senior-viva-md"></a>

## Source: 23 GoF Design Patterns in C# — Senior .NET Developer / Architect Viva Guide

*Imported from `23-design-patterns-csharp-senior-viva.md`.*

### 23 GoF Design Patterns in C# — Senior .NET Developer / Architect Viva Guide

A practical viva-preparation guide to the **23 Gang of Four (GoF) design patterns**, grouped into **Creational**, **Structural**, and **Behavioral** patterns. Each pattern includes its intent, a .NET application scenario, a C# example, and a senior-level consideration.

> **Code note:** Examples are intentionally small and focused on the pattern. Some snippets assume the surrounding application interfaces or services already exist; they are not intended to form one giant compilable project.

#### Contents

- [1. Creational patterns](#1-creational-patterns)
  1. [Singleton](#1-singleton)
  2. [Factory Method](#2-factory-method)
  3. [Abstract Factory](#3-abstract-factory)
  4. [Builder](#4-builder)
  5. [Prototype](#5-prototype)
- [2. Structural patterns](#2-structural-patterns)
  6. [Adapter](#6-adapter)
  7. [Bridge](#7-bridge)
  8. [Composite](#8-composite)
  9. [Decorator](#9-decorator)
  10. [Facade](#10-facade)
  11. [Flyweight](#11-flyweight)
  12. [Proxy](#12-proxy)
- [3. Behavioral patterns](#3-behavioral-patterns)
  13. [Chain of Responsibility](#13-chain-of-responsibility)
  14. [Command](#14-command)
  15. [Interpreter](#15-interpreter)
  16. [Iterator](#16-iterator)
  17. [Mediator](#17-mediator)
  18. [Memento](#18-memento)
  19. [Observer](#19-observer)
  20. [State](#20-state)
  21. [Strategy](#21-strategy)
  22. [Template Method](#22-template-method)
  23. [Visitor](#23-visitor)
- [Common viva comparisons](#common-viva-comparisons)
- [Architecture-level guidance](#architecture-level-guidance)

---

### 1. Creational patterns

Creational patterns separate **how objects are created** from the code that uses them. They are useful when construction varies, has many steps, or must be controlled consistently.

#### 1. Singleton

**Intent:** Ensure a class has one shared instance and provide a single access point to it.

**.NET application scenario:** A process-wide immutable metadata registry or a shared service whose lifetime should be one instance per application process.

```csharp
public sealed class RuntimeMetadata
{
    private static readonly Lazy<RuntimeMetadata> _instance =
        new(() => new RuntimeMetadata());

    public static RuntimeMetadata Instance => _instance.Value;

    private RuntimeMetadata() { }

    public string ApplicationName => "Clinical Trial Registry";
}
```

In ASP.NET Core, dependency injection is usually preferable to a hand-written global singleton:

```csharp
builder.Services.AddSingleton<IApplicationMetadata, ApplicationMetadata>();
```

**Senior-level consideration:** `AddSingleton` means one instance per DI container, not one instance across every process, server, or container replica. Singleton services must be thread-safe and should not directly depend on scoped services such as `DbContext`. Avoid putting mutable request- or tenant-specific state in a singleton.

#### 2. Factory Method

**Intent:** Define a creation method in a base type and let concrete creators choose which product to instantiate.

**.NET application scenario:** A reporting module exports data in different formats, while each exporter owns its own export implementation.

```csharp
public interface IReportExporter
{
    Task ExportAsync(Stream destination, CancellationToken cancellationToken);
}

public abstract class ReportJob
{
    protected abstract IReportExporter CreateExporter();

    public Task RunAsync(Stream destination, CancellationToken cancellationToken) =>
        CreateExporter().ExportAsync(destination, cancellationToken);
}

public sealed class CsvReportJob : ReportJob
{
    protected override IReportExporter CreateExporter() => new CsvReportExporter();
}

public sealed class JsonReportJob : ReportJob
{
    protected override IReportExporter CreateExporter() => new JsonReportExporter();
}
```

`CsvReportExporter` and `JsonReportExporter` implement `IReportExporter`; their serialization details are omitted here.

**Senior-level consideration:** Factory Method uses inheritance to vary creation. If the choice is simply based on configuration or a runtime key, a small factory service registered with DI may be simpler than a class hierarchy. Do not confuse the GoF Factory Method with every method named `Create` or with the often-used “Simple Factory” idiom.

#### 3. Abstract Factory

**Intent:** Create a **family of related objects** without coupling client code to their concrete classes.

**.NET application scenario:** A multi-cloud platform needs a compatible blob-storage client and message-queue client for the selected provider.

```csharp
public interface IBlobStore
{
    Task PutAsync(string name, Stream content, CancellationToken cancellationToken);
}

public interface IMessageQueue
{
    Task PublishAsync(string message, CancellationToken cancellationToken);
}

public interface ICloudServicesFactory
{
    IBlobStore CreateBlobStore();
    IMessageQueue CreateMessageQueue();
}

public sealed class AzureServicesFactory : ICloudServicesFactory
{
    public IBlobStore CreateBlobStore() => new AzureBlobStore();
    public IMessageQueue CreateMessageQueue() => new AzureServiceBusQueue();
}

public sealed class AwsServicesFactory : ICloudServicesFactory
{
    public IBlobStore CreateBlobStore() => new S3BlobStore();
    public IMessageQueue CreateMessageQueue() => new SqsMessageQueue();
}
```

Concrete adapters are omitted. Application services receive an `ICloudServicesFactory` rather than knowing which cloud SDK is being used.

**Senior-level consideration:** Use this when products must vary **together as a compatible family**. A Factory Method generally creates one product; an Abstract Factory coordinates several related products. Consider deployment configuration and DI registration as the selection mechanism rather than scattering provider checks throughout the codebase.

#### 4. Builder

**Intent:** Construct a complex object step by step, separating construction from the final representation.

**.NET application scenario:** Assemble a report request with required fields and optional filters without a constructor containing many positional parameters.

```csharp
public sealed record ReportRequest(
    string ReportName,
    DateOnly From,
    DateOnly To,
    IReadOnlyList<string> Statuses,
    bool IncludeTotals);

public sealed class ReportRequestBuilder
{
    private string? _reportName;
    private DateOnly? _from;
    private DateOnly? _to;
    private readonly List<string> _statuses = [];
    private bool _includeTotals;

    public ReportRequestBuilder ForReport(string name)
    {
        _reportName = name;
        return this;
    }

    public ReportRequestBuilder Between(DateOnly from, DateOnly to)
    {
        if (to < from) throw new ArgumentException("End date must not precede start date.");
        _from = from;
        _to = to;
        return this;
    }

    public ReportRequestBuilder WithStatus(string status)
    {
        _statuses.Add(status);
        return this;
    }

    public ReportRequestBuilder IncludeTotals(bool include = true)
    {
        _includeTotals = include;
        return this;
    }

    public ReportRequest Build()
    {
        if (string.IsNullOrWhiteSpace(_reportName) || _from is null || _to is null)
            throw new InvalidOperationException("Report name and date range are required.");

        return new ReportRequest(
            _reportName, _from.Value, _to.Value, _statuses.ToArray(), _includeTotals);
    }
}
```

Example usage:

```csharp
var request = new ReportRequestBuilder()
    .ForReport("SubjectEnrollment")
    .Between(new DateOnly(2026, 1, 1), new DateOnly(2026, 3, 31))
    .WithStatus("Active")
    .IncludeTotals()
    .Build();
```

**Senior-level consideration:** Builder is valuable when there are many optional settings, validation rules, or multiple construction steps. It is unnecessary for a simple object that can be expressed clearly with a constructor, object initializer, or immutable record. A separate Director class is optional, not mandatory.

#### 5. Prototype

**Intent:** Create a new object by copying an existing object, called the prototype.

**.NET application scenario:** Duplicate a report definition or tenant template, preserving its settings while allowing independent changes.

```csharp
public sealed class ReportDefinition
{
    public required string Name { get; init; }
    public List<string> Columns { get; init; } = [];
    public Dictionary<string, string> Parameters { get; init; } = [];

    public ReportDefinition Clone(string newName) => new()
    {
        Name = newName,
        Columns = [.. Columns],
        Parameters = new Dictionary<string, string>(Parameters)
    };
}
```

**Senior-level consideration:** The example performs a copy of the collections but not of objects nested inside them. `MemberwiseClone()` is shallow: reference-type members remain shared. Decide explicitly whether the required semantics are shallow copy, deep copy, or immutable sharing. C# records and `with` expressions can be useful, but they also do not automatically deep-copy referenced objects.

---

### 2. Structural patterns

Structural patterns describe how objects and classes are connected to form larger components while limiting coupling between parts.

#### 6. Adapter

**Intent:** Convert an existing interface into the interface expected by the application.

**.NET application scenario:** Integrate a legacy SMS gateway or third-party SDK into a notification abstraction already used by the application.

```csharp
public interface INotificationSender
{
    Task SendAsync(string recipient, string message, CancellationToken cancellationToken);
}

// Existing third-party API that cannot be changed.
public sealed class LegacySmsClient
{
    public Task SendTextAsync(string phone, string text) => Task.CompletedTask;
}

public sealed class LegacySmsAdapter(LegacySmsClient client) : INotificationSender
{
    public Task SendAsync(
        string recipient, string message, CancellationToken cancellationToken)
    {
        cancellationToken.ThrowIfCancellationRequested();
        return client.SendTextAsync(recipient, message);
    }
}
```

**Senior-level consideration:** An adapter translates an incompatible contract; it should not become a general-purpose service that accumulates business logic. Keep vendor-specific details behind the application's own interface so that testing and provider replacement are easier.

#### 7. Bridge

**Intent:** Separate an abstraction from its implementation so both can evolve independently.

**.NET application scenario:** Different notification types (for example, order alerts and security alerts) need to work with different delivery channels (email, SMS, or push).

```csharp
public interface IMessageChannel
{
    Task SendAsync(string recipient, string body, CancellationToken cancellationToken);
}

public sealed class EmailChannel : IMessageChannel
{
    public Task SendAsync(string recipient, string body, CancellationToken cancellationToken) =>
        Task.CompletedTask; // Call the email provider.
}

public sealed class SmsChannel : IMessageChannel
{
    public Task SendAsync(string recipient, string body, CancellationToken cancellationToken) =>
        Task.CompletedTask; // Call the SMS provider.
}

public abstract class Notification(IMessageChannel channel)
{
    protected IMessageChannel Channel { get; } = channel;
    public abstract Task SendAsync(string recipient, CancellationToken cancellationToken);
}

public sealed class SecurityAlert(IMessageChannel channel) : Notification(channel)
{
    public override Task SendAsync(string recipient, CancellationToken cancellationToken) =>
        Channel.SendAsync(recipient, "A security event requires your attention.", cancellationToken);
}

public sealed class OrderUpdate(IMessageChannel channel) : Notification(channel)
{
    public override Task SendAsync(string recipient, CancellationToken cancellationToken) =>
        Channel.SendAsync(recipient, "Your order status has changed.", cancellationToken);
}
```

**Senior-level consideration:** The two dimensions can vary independently: notification type and transport. Adapter makes one interface fit another; Bridge intentionally designs two independent variation axes. This can prevent a class explosion such as `EmailSecurityAlert`, `SmsSecurityAlert`, `EmailOrderUpdate`, and `SmsOrderUpdate`.

#### 8. Composite

**Intent:** Represent part-whole hierarchies as trees and let clients treat an individual object and a group uniformly.

**.NET application scenario:** A document management system represents folders containing files and other folders.

```csharp
public interface IFileNode
{
    long GetSizeBytes();
}

public sealed class FileNode(long sizeBytes) : IFileNode
{
    public long GetSizeBytes() => sizeBytes;
}

public sealed class FolderNode : IFileNode
{
    private readonly List<IFileNode> _children = [];

    public void Add(IFileNode child) => _children.Add(child);

    public long GetSizeBytes() => _children.Sum(child => child.GetSizeBytes());
}
```

**Senior-level consideration:** Composite is appropriate for naturally recursive structures such as folder trees, organizational charts, and UI component trees. Decide whether leaf objects should expose child-management methods; forcing meaningless operations on every leaf can weaken the design. Validate cycles if arbitrary nodes can be attached to one another.

#### 9. Decorator

**Intent:** Add behavior to an object by wrapping it, without changing the wrapped object's class.

**.NET application scenario:** Add logging, metrics, retry, or caching around an application service while preserving its interface.

```csharp
public interface ICustomerQuery
{
    Task<Customer?> GetAsync(Guid id, CancellationToken cancellationToken);
}

public sealed class LoggingCustomerQuery(
    ICustomerQuery inner,
    ILogger<LoggingCustomerQuery> logger) : ICustomerQuery
{
    public async Task<Customer?> GetAsync(Guid id, CancellationToken cancellationToken)
    {
        logger.LogInformation("Loading customer {CustomerId}", id);
        var customer = await inner.GetAsync(id, cancellationToken);
        logger.LogInformation("Customer lookup completed: {Found}", customer is not null);
        return customer;
    }
}
```

`Customer` is the application's customer model. The inner `ICustomerQuery` can be the real database-backed implementation or another decorator.

**Senior-level consideration:** Decorators compose behaviors around a stable contract and are commonly used with DI. Be explicit about decorator ordering—for example, whether caching sits inside or outside logging. In .NET, libraries such as Scrutor can help register decorators, but the pattern itself does not require a library. Avoid duplicating framework features without a clear reason.

#### 10. Facade

**Intent:** Provide a simple entry point to a complex subsystem.

**.NET application scenario:** A checkout use case coordinates inventory, payment, and order persistence behind one application-facing operation.

```csharp
public sealed class CheckoutFacade(
    IInventoryService inventory,
    IPaymentService payments,
    IOrderRepository orders)
{
    public async Task<Guid> CheckoutAsync(
        CheckoutRequest request, CancellationToken cancellationToken)
    {
        await inventory.ReserveAsync(request.Items, cancellationToken);
        var paymentId = await payments.ChargeAsync(
            request.CustomerId, request.Total, cancellationToken);
        return await orders.CreateAsync(request, paymentId, cancellationToken);
    }
}
```

**Senior-level consideration:** A facade simplifies access to a subsystem; it does not automatically provide transactionality or distributed consistency. If reservation succeeds but payment fails, the application needs compensation, a transaction where applicable, or a workflow such as a saga. Keep the facade focused rather than allowing it to become a god object.

#### 11. Flyweight

**Intent:** Reduce memory use by sharing common, reusable state across many fine-grained objects while keeping per-use state external.

**.NET application scenario:** A document renderer handles thousands of repeated icons. The icon resource is shared; each occurrence supplies its own position.

```csharp
using System.Collections.Concurrent;

public sealed class IconResource(string name, byte[] imageBytes)
{
    public string Name { get; } = name;
    public byte[] ImageBytes { get; } = imageBytes;
}

public sealed class IconResourceFactory
{
    private readonly ConcurrentDictionary<string, IconResource> _cache = new();

    public IconResource Get(string name, Func<byte[]> loadImage) =>
        _cache.GetOrAdd(name, key => new IconResource(key, loadImage()));
}

public sealed record IconPlacement(IconResource Resource, int X, int Y);
```

**Senior-level consideration:** Intrinsic state is shared (icon resource); extrinsic state varies by use (coordinates). Shared state should usually be immutable or treated as read-only. Consider memory limits and cache eviction if the set of resource keys can grow without bound.

#### 12. Proxy

**Intent:** Stand in for another object to control access to it, possibly adding lazy loading, authorization, remote access, or caching.

**.NET application scenario:** A document service checks access rights before calling a slower storage service.

```csharp
public interface IDocumentContent
{
    Task<Stream> OpenReadAsync(Guid documentId, CancellationToken cancellationToken);
}

public sealed class AuthorizedDocumentProxy(
    IDocumentContent realContent,
    IDocumentAccessPolicy accessPolicy,
    ICurrentUser currentUser) : IDocumentContent
{
    public Task<Stream> OpenReadAsync(Guid documentId, CancellationToken cancellationToken)
    {
        accessPolicy.EnsureCanRead(currentUser.UserId, documentId);
        return realContent.OpenReadAsync(documentId, cancellationToken);
    }
}
```

**Senior-level consideration:** A proxy controls access to the real subject; a decorator primarily adds responsibilities, although both use wrapping and can look structurally similar. Do not rely on a proxy alone for security if the underlying service can be accessed through another path. EF Core lazy-loading proxies are a framework feature related to proxying, but they are not a reason to enable lazy loading indiscriminately.

---

### 3. Behavioral patterns

Behavioral patterns organize communication, responsibilities, and algorithms among objects.

#### 13. Chain of Responsibility

**Intent:** Pass a request through a sequence of handlers; each handler can process it, reject it, or pass it onward.

**.NET application scenario:** An order passes through independent validation checks before it can be accepted.

```csharp
public sealed record OrderRequest(Guid CustomerId, decimal Total);

public interface IOrderCheck
{
    Task CheckAsync(OrderRequest request, CancellationToken cancellationToken);
}

public sealed class PositiveTotalCheck : IOrderCheck
{
    public Task CheckAsync(OrderRequest request, CancellationToken cancellationToken)
    {
        if (request.Total <= 0)
            throw new InvalidOperationException("Order total must be positive.");

        return Task.CompletedTask;
    }
}

public sealed class CustomerEligibilityCheck(ICustomerEligibility eligibility) : IOrderCheck
{
    public Task CheckAsync(OrderRequest request, CancellationToken cancellationToken) =>
        eligibility.EnsureEligibleAsync(request.CustomerId, cancellationToken);
}

public sealed class OrderValidationPipeline(IEnumerable<IOrderCheck> checks)
{
    public async Task ValidateAsync(OrderRequest request, CancellationToken cancellationToken)
    {
        foreach (var check in checks)
            await check.CheckAsync(request, cancellationToken);
    }
}
```

**Senior-level consideration:** ASP.NET Core middleware is a familiar chain-like pipeline: each middleware can act before and after calling the next delegate, or short-circuit the pipeline. This sample uses a sequence of validators rather than linked `SetNext` objects. Define handler ordering clearly and distinguish a validation pipeline from a chain where each handler decides whether to pass the request onward.

#### 14. Command

**Intent:** Encapsulate an operation as an object so it can be queued, logged, retried, scheduled, or executed by an invoker.

**.NET application scenario:** A background worker executes invoice-generation work submitted by an API.

```csharp
public interface IApplicationCommand
{
    Task ExecuteAsync(CancellationToken cancellationToken);
}

public sealed class GenerateInvoiceCommand(
    Guid orderId,
    IInvoiceGenerator generator) : IApplicationCommand
{
    public Task ExecuteAsync(CancellationToken cancellationToken) =>
        generator.GenerateAsync(orderId, cancellationToken);
}

public sealed class CommandWorker
{
    public Task ExecuteAsync(IApplicationCommand command, CancellationToken cancellationToken) =>
        command.ExecuteAsync(cancellationToken);
}
```

**Senior-level consideration:** The Command pattern turns an action into an object, which can make queuing, auditing, retries, and undo possible. A **CQRS command** is often just a message expressing an intention (for example, `CreateOrderCommand`); a separate handler performs it. Libraries such as MediatR may dispatch such messages, but the message/handler setup and the GoF Command pattern are related ideas rather than exact synonyms. Retries require idempotency and clear failure handling.

#### 15. Interpreter

**Intent:** Represent a small grammar as objects and evaluate expressions in that grammar.

**.NET application scenario:** Evaluate simple, predefined eligibility rules or filter expressions stored as structured data.

```csharp
public interface IRuleExpression
{
    bool Evaluate(IReadOnlyDictionary<string, decimal> values);
}

public sealed class GreaterThanExpression(string key, decimal threshold) : IRuleExpression
{
    public bool Evaluate(IReadOnlyDictionary<string, decimal> values) =>
        values.TryGetValue(key, out var value) && value > threshold;
}

public sealed class AndExpression(IRuleExpression left, IRuleExpression right) : IRuleExpression
{
    public bool Evaluate(IReadOnlyDictionary<string, decimal> values) =>
        left.Evaluate(values) && right.Evaluate(values);
}
```

Example: `new AndExpression(new GreaterThanExpression("Age", 18), new GreaterThanExpression("Score", 70))` expresses a small rule without compiling arbitrary source code.

**Senior-level consideration:** Interpreter is suitable for a small, stable language. For complex expressions, consider a parser, a carefully constrained expression tree, or a dedicated rules engine. Never use unrestricted `eval`-style execution for user-provided rules; validate allowed operators and inputs.

#### 16. Iterator

**Intent:** Traverse a collection sequentially without exposing the collection's internal representation.

**.NET application scenario:** Stream database-backed results page by page so a large export does not have to load every row into memory at once.

```csharp
using System.Runtime.CompilerServices;

public sealed class OrderStream(IOrderRepository repository)
{
    public async IAsyncEnumerable<OrderSummary> ReadAllAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var pageNumber = 0;

        while (true)
        {
            var page = await repository.GetPageAsync(pageNumber++, cancellationToken);
            if (page.Count == 0)
                yield break;

            foreach (var order in page)
                yield return order;
        }
    }
}
```

**Senior-level consideration:** `IEnumerable<T>`, `IEnumerator<T>`, iterator blocks using `yield return`, and `IAsyncEnumerable<T>` are built-in .NET forms of the Iterator idea. Async iteration can reduce memory pressure, but consumers must enumerate correctly and cancellation should propagate. Offset paging can become inefficient or inconsistent on changing datasets; keyset paging may be better for large production queries.

#### 17. Mediator

**Intent:** Centralize communication among collaborating objects so they do not need direct references to one another.

**.NET application scenario:** A controller sends a query to a handler through a mediator rather than constructing or calling the handler directly.

```csharp
// Example using the MediatR library; package and registration are assumed.
public sealed record GetOrderByIdQuery(Guid OrderId) : IRequest<OrderDto?>;

public sealed class OrdersController(ISender sender) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderDto>> Get(Guid id, CancellationToken cancellationToken)
    {
        var order = await sender.Send(new GetOrderByIdQuery(id), cancellationToken);
        return order is null ? NotFound() : Ok(order);
    }
}
```

`GetOrderByIdQueryHandler` implements the library's corresponding request-handler interface and performs the retrieval.

**Senior-level consideration:** The mediator reduces direct coupling between senders and handlers, and pipeline behaviors can centralize validation, logging, or transaction boundaries. It can also hide the call graph and encourage excessive small messages or handlers. Use it where indirect dispatch improves the architecture, not automatically for every method call. A mediator coordinates peers; a facade gives a simpler entry point to a subsystem.

#### 18. Memento

**Intent:** Capture and restore an object's previous state without exposing all of its internal implementation details.

**.NET application scenario:** A report designer supports undo by saving snapshots of the document state.

```csharp
public sealed class ReportEditor
{
    private string _content = string.Empty;

    public void ReplaceContent(string content) => _content = content;
    public string ReadContent() => _content;

    public Snapshot Save() => new(_content);
    public void Restore(Snapshot snapshot) => _content = snapshot.Content;

    public sealed record Snapshot(string Content);
}
```

**Senior-level consideration:** This example snapshots a string, so the state is immutable and copying is straightforward. For a large object graph, snapshots can consume significant memory; consider immutable models, diffs, checkpoints, or a command-based undo history. A UI undo snapshot is not the same as a database transaction or a durable audit trail.

#### 19. Observer

**Intent:** Define a one-to-many dependency so subscribers are notified when a publisher emits an event.

**.NET application scenario:** When an order is placed, separate handlers can trigger an email, update analytics, or notify another component.

```csharp
public sealed record OrderPlaced(Guid OrderId, Guid CustomerId);

public sealed class OrderService
{
    public event EventHandler<OrderPlaced>? OrderPlaced;

    public void PublishOrderPlaced(OrderPlaced order) =>
        OrderPlaced?.Invoke(this, order);
}

public sealed class OrderEmailSubscriber(IEmailSender emailSender)
{
    public void Subscribe(OrderService service) => service.OrderPlaced += OnOrderPlaced;

    private async void OnOrderPlaced(object? sender, OrderPlaced order)
    {
        await emailSender.SendOrderConfirmationAsync(order.OrderId, order.CustomerId);
    }
}
```

**Senior-level consideration:** C# events are a direct Observer-style mechanism for in-process notification. The `async void` event handler is shown only to illustrate the event shape; in production, prefer an explicit asynchronous event-dispatch abstraction when the publisher must await subscribers or observe failures. Unsubscribe long-lived subscribers when appropriate. In distributed systems, use durable messaging or an outbox when notification delivery must survive process failure.

#### 20. State

**Intent:** Let an object change its behavior according to its current state by delegating behavior to state-specific objects.

**.NET application scenario:** An order can be submitted, shipped, or cancelled, with different valid operations in each state.

```csharp
public interface IOrderState
{
    void Ship(Order order);
    void Cancel(Order order);
}

public sealed class PendingOrderState : IOrderState
{
    public void Ship(Order order) => order.ChangeState(new ShippedOrderState());
    public void Cancel(Order order) => order.ChangeState(new CancelledOrderState());
}

public sealed class ShippedOrderState : IOrderState
{
    public void Ship(Order order) => throw new InvalidOperationException("Order is already shipped.");
    public void Cancel(Order order) => throw new InvalidOperationException("A shipped order cannot be cancelled here.");
}

public sealed class CancelledOrderState : IOrderState
{
    public void Ship(Order order) => throw new InvalidOperationException("Cancelled orders cannot be shipped.");
    public void Cancel(Order order) => throw new InvalidOperationException("Order is already cancelled.");
}

public sealed class Order
{
    private IOrderState _state = new PendingOrderState();

    public void Ship() => _state.Ship(this);
    public void Cancel() => _state.Cancel(this);
    internal void ChangeState(IOrderState state) => _state = state;
}
```

**Senior-level consideration:** State models state-dependent behavior; Strategy makes an algorithm interchangeable by choice. For a persisted business workflow, the state transition should also be represented and validated in durable domain data, with concurrency rules where multiple requests may update the same order. A full state-machine library may be clearer for many states and transitions.

#### 21. Strategy

**Intent:** Define a family of algorithms behind a common interface and select one without changing the client that uses it.

**.NET application scenario:** A payment workflow supports different fee-calculation policies, selected based on the payment method or tenant configuration.

```csharp
public interface IFeeStrategy
{
    decimal Calculate(decimal amount);
}

public sealed class CardFeeStrategy : IFeeStrategy
{
    public decimal Calculate(decimal amount) => amount * 0.025m + 0.30m;
}

public sealed class BankTransferFeeStrategy : IFeeStrategy
{
    public decimal Calculate(decimal amount) => 1.00m;
}

public sealed class PaymentPricing(IFeeStrategy feeStrategy)
{
    public decimal GetTotal(decimal amount) => amount + feeStrategy.Calculate(amount);
}
```

**Senior-level consideration:** Strategy favors composition over condition-heavy methods with many `if` or `switch` branches. In a real application, select the strategy through DI or a factory based on a validated key. Validate currency, rounding rules, limits, and provider-specific constraints; the pattern itself does not solve business correctness.

#### 22. Template Method

**Intent:** Define the stable skeleton of an algorithm in a base class and allow subclasses to customize selected steps.

**.NET application scenario:** Different import formats follow the same high-level sequence—read, validate, transform, and persist—but customize parsing and mapping.

```csharp
public abstract class ImportJob
{
    public async Task RunAsync(Stream input, CancellationToken cancellationToken)
    {
        var records = await ReadAsync(input, cancellationToken);
        Validate(records);
        var entities = Transform(records);
        await PersistAsync(entities, cancellationToken);
    }

    protected abstract Task<IReadOnlyList<string[]>> ReadAsync(
        Stream input, CancellationToken cancellationToken);

    protected virtual void Validate(IReadOnlyList<string[]> records)
    {
        if (records.Count == 0)
            throw new InvalidOperationException("The import contains no records.");
    }

    protected abstract IReadOnlyList<ImportEntity> Transform(
        IReadOnlyList<string[]> records);

    protected abstract Task PersistAsync(
        IReadOnlyList<ImportEntity> entities, CancellationToken cancellationToken);
}
```

**Senior-level consideration:** Template Method centralizes the algorithm's order and uses inheritance for variation. It works when the sequence is genuinely stable. In modern .NET applications, composition with injected steps can be easier to test and extend when the pipeline needs runtime configuration or multiple combinations. Ensure overridable steps cannot silently violate required invariants.

#### 23. Visitor

**Intent:** Add operations to a stable object structure without adding each operation to every element class.

**.NET application scenario:** A document or syntax tree may need separate operations for validation, rendering, or calculating a summary.

```csharp
public interface IReportNodeVisitor
{
    void Visit(TextNode node);
    void Visit(TableNode node);
}

public interface IReportNode
{
    void Accept(IReportNodeVisitor visitor);
}

public sealed class TextNode(string text) : IReportNode
{
    public string Text { get; } = text;
    public void Accept(IReportNodeVisitor visitor) => visitor.Visit(this);
}

public sealed class TableNode(int rowCount, int columnCount) : IReportNode
{
    public int RowCount { get; } = rowCount;
    public int ColumnCount { get; } = columnCount;
    public void Accept(IReportNodeVisitor visitor) => visitor.Visit(this);
}

public sealed class ReportWordCountVisitor : IReportNodeVisitor
{
    public int WordCount { get; private set; }

    public void Visit(TextNode node) =>
        WordCount += node.Text.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;

    public void Visit(TableNode node) =>
        WordCount += node.RowCount * node.ColumnCount; // Illustrative table counting rule.
}
```

**Senior-level consideration:** Visitor makes it easy to add new operations over a stable set of node types, but adding a new node type requires changing the visitor interface and its implementations. It suits relatively stable object structures with changing operations, such as compilers and document processing. For frequently changing element types, ordinary polymorphism or pattern matching may be simpler.

---

### Common viva comparisons

| Comparison | Key distinction | Example cue |
|---|---|---|
| Factory Method vs Abstract Factory | Factory Method specializes creation of a product; Abstract Factory creates a compatible family of products. | One report exporter vs a cloud provider's blob-store and queue clients. |
| Builder vs Factory | Builder assembles a complex object step by step; a factory chooses or creates a product. | A validated report request vs selecting a CSV exporter. |
| Adapter vs Bridge | Adapter fits an existing incompatible API to an expected interface; Bridge separates two planned dimensions of variation. | Legacy SMS SDK vs notification type × delivery channel. |
| Decorator vs Proxy | Decorator adds responsibilities; Proxy controls access to the subject. Their structure can be very similar. | Service metrics vs document access checks. |
| Facade vs Mediator | Facade simplifies access to a subsystem; Mediator coordinates communication among collaborating objects. | Checkout entry point vs request dispatch to handlers. |
| Strategy vs State | Strategy chooses an interchangeable algorithm; State changes available behavior based on lifecycle state. | Fee calculation vs order status transitions. |
| Strategy vs Template Method | Strategy varies an algorithm through composition; Template Method varies steps through inheritance while preserving sequence. | Injected fee policy vs fixed import workflow. |
| Chain of Responsibility vs Decorator | A chain passes a request through handlers that may stop or continue; decorators wrap the same operation to add behavior. | Validation pipeline vs logging around a query service. |
| Observer vs Mediator | Observer broadcasts notifications to subscribers; Mediator centralizes interaction and routing among components. | Order-placed event vs coordinated component interactions. |
| Composite vs Decorator | Composite organizes children into a tree; Decorator wraps one object to add behavior. | Folder contents vs logging around a service. |
| Command vs Strategy | Command represents an action/request; Strategy represents an algorithm or policy used to perform a task. | Generate-invoice work item vs fee-calculation rule. |
| Prototype vs shallow copy | Prototype creates an object from an existing one; copy depth must be chosen deliberately. | Cloning a report definition without sharing mutable collections. |

### Architecture-level guidance

#### How to answer a pattern question in a viva

For any pattern, structure the answer in this order:

1. **Problem:** What coupling, complexity, or variation does it address?
2. **Participants:** Which interfaces and concrete types collaborate?
3. **Application example:** Identify a realistic use case in an API, application service, integration, or domain workflow.
4. **Trade-off:** State what extra abstraction or complexity the pattern introduces.
5. **Alternative:** Explain when a simpler .NET feature or composition would be preferable.

#### Applying patterns in ASP.NET Core and Clean Architecture

- **Dependency Injection is a tool, not a design pattern substitute.** It helps compose implementations, lifetimes, and decorators at the application boundary.
- **Prefer composition where it reduces coupling.** Inheritance-oriented patterns such as Template Method and Factory Method are useful, but not every variation needs a class hierarchy.
- **Keep domain rules explicit.** State and Strategy can model behavior, but they do not replace validation, transaction boundaries, authorization, or concurrency control.
- **Use framework support intentionally.** ASP.NET Core middleware, `IEnumerable<T>`, `IAsyncEnumerable<T>`, C# events, EF Core proxies, and DI lifetimes already embody or support ideas related to several patterns.
- **Avoid pattern overuse.** A pattern is justified when it makes a real variation point, dependency boundary, or algorithm easier to understand and maintain—not merely because a class can be named after it.
- **Consider operational behavior.** In production systems, evaluate cancellation, retries, idempotency, telemetry, resource disposal, caching, concurrency, and failure recovery alongside the class design.

#### Final reminder

The goal of a senior-level answer is not just to recite each pattern's definition. Explain **why it fits the problem, what it costs, how it integrates with .NET's built-in capabilities, and what could go wrong in production**.

---

<a id="clean-architecture-cqrs-vs-mvc-dotnet8-md"></a>

## Source: Why Choose Clean Architecture with CQRS over MVC in .NET 8?

*Imported from `clean-architecture-cqrs-vs-mvc-dotnet8.md`.*

### Why Choose Clean Architecture with CQRS over MVC in .NET 8?

#### Introduction

Traditional ASP.NET Core MVC can be an excellent choice for straightforward applications. For systems with complex business rules, multiple workflows, and long-term maintenance needs, **Clean Architecture with CQRS** can provide clearer boundaries and a more organized codebase.

One important distinction: MVC is a presentation pattern, while Clean Architecture is an architectural approach and CQRS is a way to separate read operations from write operations. They are not direct alternatives; MVC can also be used as the presentation layer in a Clean Architecture application.

#### MVC vs. Clean Architecture with CQRS

| Area | Traditional MVC-style application | Clean Architecture with CQRS |
|---|---|---|
| Main focus | Organizes request handling around controllers and views or API actions | Organizes the system around business rules and application use cases |
| Typical structure | Controllers, services, repositories, and data models | Domain, Application, Infrastructure, and API/Presentation projects or layers |
| Read and write operations | Often handled through shared services or controller workflows | Represented separately as queries (reads) and commands (writes) |
| Business rules | Can become mixed into controllers or infrastructure if boundaries are not enforced | Kept in application/domain code, away from HTTP and database implementation details |
| Testing | Straightforward when logic is already separated; harder when controllers contain business rules | Use cases and domain rules can be tested independently of HTTP and persistence |
| Cross-cutting concerns | Often implemented through middleware, filters, decorators, or services | Can use those same mechanisms, plus CQRS pipeline behaviors for validation, logging, and similar concerns |
| Read performance | Usually sufficient for ordinary CRUD workloads | Allows tailored read models and independent optimization when justified |
| Complexity | Simpler to start and often needs fewer abstractions | More layers, handlers, and conventions to learn and maintain |
| Best fit | Small applications, prototypes, and uncomplicated CRUD systems | Applications with substantial business rules, multiple use cases, or long-term growth needs |

#### What does “business logic mixed with HTTP concerns” mean?

It means that business decisions are made directly inside a controller and become coupled to web-specific details such as `HttpContext`, `ModelState`, or `IActionResult`.

For example, a controller that calculates an order total, applies discount rules, decides whether an order is allowed, and saves it is doing more than handling HTTP.

##### Example: too much responsibility in a controller

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    var total = request.Items.Sum(x => x.Price * x.Quantity);

    if (total > 1000)
        return BadRequest("Order total exceeds the limit.");

    await _orderRepository.SaveAsync(request);

    return Ok();
}
```

The example combines HTTP validation/response handling, a business rule, and persistence orchestration in one method.

##### Example: controller delegates to a use case

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(
    [FromBody] CreateOrderCommand command,
    CancellationToken cancellationToken)
{
    var result = await _sender.Send(command, cancellationToken);
    return Ok(result);
}
```

The command handler coordinates the use case, while domain/application code enforces business rules. The controller remains responsible for translating the HTTP request into an application call and mapping the outcome to an HTTP response.

> The second example is not automatically better just because it uses a handler. The benefit comes from putting responsibilities in the right place and keeping the business rules independent of HTTP.

#### Why consider this approach in .NET 8?

- **Clear boundaries:** The Domain and Application layers do not need to depend on ASP.NET Core controllers or EF Core implementation details.
- **Focused use cases:** Each command or query represents a specific application operation, making code easier to locate and review.
- **Independent testing:** Business rules and handlers can be tested without hosting the web application.
- **Consistent request processing:** CQRS pipeline behaviors can centralize validation, logging, authorization checks, and performance measurement.
- **Room to evolve:** Read models, persistence strategies, and delivery mechanisms can change with less impact when dependencies point inward.

#### Important trade-offs

Clean Architecture with CQRS is not automatically faster, more scalable, or more maintainable for every project.

- It introduces extra types, folders, and abstractions.
- CQRS does **not** require separate databases or microservices.
- Separate read/write scaling is possible, but usually requires additional infrastructure and design.
- MVC is not inherently poorly structured. MVC with thin controllers and well-separated services can be a sound architecture.
- MediatR is one option for dispatching requests, not a requirement. Consider its licensing and project needs before adopting it.

#### A practical rule of thumb

Choose a simpler MVC-style structure when the application is mostly straightforward CRUD and the extra layers would add ceremony without much benefit.

Consider Clean Architecture with CQRS when the application has complex workflows, significant business rules, many distinct use cases, or a need to keep the core logic independent from web and persistence technologies.

#### Conclusion

For .NET 8, the decision is not simply **MVC versus Clean Architecture with CQRS**. MVC can remain the API presentation layer while Clean Architecture organizes dependencies and CQRS separates read and write use cases. Choose the level of structure that solves the application's actual complexity rather than adding patterns by default.

---

<a id="senior-dotnet-architect-viva-prep-md"></a>

## Source: Senior .NET Developer & Solution Architect — Viva Preparation

*Imported from `senior_dotnet_architect_viva_prep.md`.*

### Senior .NET Developer & Solution Architect — Viva Preparation

This guide consolidates the topics discussed so far into one reference. Repeated explanations have been merged, and examples focus on the concepts most useful in a senior developer or solution architect viva.

> **Version context:** Examples generally target .NET 8 / C# 12. Check package APIs and hosting features against the exact .NET version used by your project.

#### Contents

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

### 1. High-performance REST APIs

#### 1.1 What are the best practices for optimizing a .NET API?

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

##### Example: Efficient EF Core read query

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

#### 1.2 How would you handle millions of concurrent API requests?

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

#### 1.3 How do you minimize memory allocations?

- Avoid unnecessary materialization and duplicate copies of collections.
- Stream large data when the client and transport support it.
- Use `ArrayPool<T>` for temporary reusable buffers in measured hot paths.
- Consider `Span<T>` / `Memory<T>` for efficient buffer processing.
- Avoid boxing and repeated string creation where profiling shows impact.
- Reuse serializer options and long-lived HTTP clients.
- Do not introduce low-level complexity without benchmarks.

##### Example: rent and return a buffer safely

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

#### 1.4 `Task`, `ValueTask<T>`, and `IAsyncEnumerable<T>`

| Type | Meaning | Typical use |
|---|---|---|
| `Task` / `Task<T>` | Represents one asynchronous operation, optionally with a result | Most service and I/O methods |
| `ValueTask<T>` | Can represent a result without allocating a new `Task<T>` in some cases | Specialized hot paths where synchronous completion is common and measured |
| `IAsyncEnumerable<T>` | Represents a sequence that can be consumed asynchronously, item by item | Streaming records or results |

`ValueTask<T>` is not automatically faster. It has usage constraints and can be more complex; use `Task<T>` by default unless measurement supports the alternative.

##### Example: asynchronous streaming

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

#### 1.5 How do you optimize JSON serialization?

- Prefer `System.Text.Json` when it meets the requirements.
- Reuse `JsonSerializerOptions`.
- Return DTOs rather than unnecessarily large domain graphs.
- Avoid serializing fields or properties the client does not need.
- For high-throughput applications, evaluate source-generated JSON metadata.
- For large payloads, consider streaming rather than building a giant string first.
- Benchmark representative payloads; do not claim a universal performance multiplier.

##### Example: source-generated JSON metadata

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

#### 1.6 Caching at different levels

- **Browser/client cache:** suitable for client-owned data with correct cache directives.
- **CDN/reverse proxy:** useful for cacheable public responses near users.
- **In-process cache (`IMemoryCache`):** fast, but each API instance has its own cache.
- **Distributed cache (for example Redis):** shared across API instances.
- **Database cache/index/materialized view:** can improve expensive data access, but requires careful data consistency design.

##### Cache-aside example

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

#### 1.7 `Span<T>` and `Memory<T>`

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

#### 1.8 How do you handle long-running requests?

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

#### 1.9 `IHostedService` vs `BackgroundService`

- `IHostedService` defines `StartAsync` and `StopAsync`.
- `BackgroundService` is a base class for implementing a long-running task through `ExecuteAsync`.

Use `BackgroundService` for a worker loop. Use `IHostedService` directly when you need custom start/stop behavior. Hosted services are generally singletons; create a DI scope for scoped services such as EF Core `DbContext`.

#### 1.10 Request throttling / rate limiting

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

### 2. Async/await, threads, and operating-system I/O

#### 2.1 Does `async` / `await` use ThreadPool threads?

**Viva answer:** `async` and `await` are language features for composing asynchronous operations. They do not inherently create a new thread.

- **I/O-bound operation:** the OS/runtime can wait for I/O without occupying a worker thread for the entire wait.
- **CPU-bound operation:** the work must execute on a CPU. `Task.Run` can schedule it on a ThreadPool thread, but should not be used automatically inside every ASP.NET Core request.
- **Continuation:** after an awaited operation completes, the remaining code runs according to the captured context/scheduler rules. In ASP.NET Core, continuations normally run on ThreadPool threads because there is no classic ASP.NET synchronization context.

##### I/O-bound example

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

##### CPU-bound example

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

#### 2.2 How does the OS kernel handle I/O asynchronously?

A simplified flow:

1. Application requests a network or file operation.
2. The runtime invokes OS facilities appropriate for the platform and operation.
3. The kernel and/or device processes the operation.
4. Completion is reported through OS mechanisms.
5. .NET completes the associated asynchronous operation and schedules its continuation.

The exact mechanism differs by OS and I/O type. Windows uses completion mechanisms such as I/O Completion Ports for many socket and file scenarios. Linux commonly uses readiness notification mechanisms such as `epoll` for sockets; newer APIs such as `io_uring` support other I/O models. Do not state that all .NET I/O always uses one specific OS mechanism.

##### File I/O example

```csharp
public static async Task<string> ReadTextAsync(
    string path,
    CancellationToken cancellationToken)
{
    return await File.ReadAllTextAsync(path, cancellationToken);
}
```

The caller can await the operation without blocking its thread while the asynchronous operation is pending. Actual behavior can vary by OS, filesystem, device, and API; not every operation is guaranteed to be truly asynchronous at the hardware level.

#### 2.3 Task Scheduler and `SynchronizationContext`

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

#### 2.4 What does `ConfigureAwait(false)` do, and how can it help?

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

##### When it matters

- In reusable libraries that must work in UI and other environments, avoiding context capture can prevent unnecessary context switches.
- In WPF/WinForms, it can prevent a library continuation from needing the UI thread when the continuation does not use UI objects.
- In ASP.NET Core, there is normally no classic synchronization context to return to, so it usually provides little or no performance benefit.

##### Important correction

`ConfigureAwait(false)` does **not** mean “always switch to a ThreadPool thread,” and it does not itself make the underlying I/O faster. It means the continuation does not require the captured context. The continuation may run wherever the completion machinery schedules it.

Use it selectively in library code where context affinity is unnecessary. Do not use it in UI code before directly updating UI controls unless you explicitly marshal back to the UI thread.

#### 2.5 `async void` vs `async Task`

Use `async Task` or `async Task<T>` for ordinary asynchronous methods so callers can await completion and observe exceptions. `async void` is mainly for event handlers whose signature requires `void`.

```csharp
public async Task SaveAsync(CancellationToken cancellationToken)
{
    await Task.Delay(10, cancellationToken);
}
```

---

### 3. ASP.NET Core security

#### 3.1 Authentication and authorization

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

#### 3.2 Role-based and claim-based authorization

```csharp
app.MapGet("/admin", () => Results.Ok())
   .RequireAuthorization(policy =>
       policy.RequireRole("Admin"));

app.MapGet("/reports", () => Results.Ok())
   .RequireAuthorization("Reports.Read");
```

Roles represent broad job functions; claims/policies can express finer permissions. For access to a particular resource, use resource-based authorization so a user cannot access another tenant's or another customer's record merely by changing an ID.

#### 3.3 OAuth 2.0 provider / IdentityServer-style products

A dedicated identity provider centralizes token issuance, client registration, scopes, consent, key rotation, and SSO/OpenID Connect integration. Choose a supported product and lifecycle; do not start a new project on an obsolete identity-server version without checking its support status.

OAuth 2.0 is primarily an authorization framework; OpenID Connect adds an identity layer. Use authorization code with PKCE for interactive clients and client credentials for service-to-service clients where appropriate. Never put a confidential client secret in browser or mobile code.

#### 3.4 CSRF, XSS, and SQL injection

- **CSRF:** especially relevant when browsers automatically attach cookies. Use anti-forgery tokens and appropriate `SameSite`, `Secure`, and `HttpOnly` cookie settings. Bearer tokens explicitly supplied in an `Authorization` header have a different CSRF profile, but browser storage introduces other risks.
- **XSS:** encode output, sanitize allowed rich HTML, avoid unsafe DOM insertion, and deploy a suitable Content Security Policy. Input validation alone does not prevent XSS.
- **SQL injection:** use parameterized queries and ORM parameters. Do not concatenate untrusted values into SQL. For dynamic sort columns or identifiers, use an allowlist because SQL parameters cannot stand in for identifiers.

```csharp
var user = await db.Users
    .AsNoTracking()
    .SingleOrDefaultAsync(u => u.Email == email, cancellationToken);
```

#### 3.5 HSTS

HTTP Strict Transport Security tells supporting browsers to use HTTPS for future requests to a host. It helps prevent downgrade attacks after the browser learns the policy. Enable it in production when HTTPS is correctly configured, and be cautious with `includeSubDomains` and preload because those choices affect every covered subdomain.

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();
```

#### 3.6 Secret management

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

#### 3.7 Zero Trust

Do not automatically trust a request because it originates inside a network. Authenticate identities, authorize every relevant action, enforce least privilege, protect service-to-service traffic, validate inputs, monitor activity, and assume credentials or network segments can be compromised.

#### 3.8 API keys for third-party integrations

Treat API keys as secrets, not as a complete user identity system. Store them securely, transmit them only over TLS, scope them narrowly, rotate them, monitor use, and support revocation. Where possible, use OAuth client credentials or signed requests. Never put keys in URLs because URLs often appear in logs and browser history.

#### 3.9 JWT best practices and pitfalls

- Validate signature, issuer, audience, lifetime, and allowed algorithms.
- Use short-lived access tokens and a secure refresh-token strategy when needed.
- Keep sensitive information out of the payload; a signed JWT is usually readable, not encrypted.
- Plan for key rotation and revocation.
- Enforce authorization on the API, not only in the frontend.
- Do not accept a token merely because it can be decoded.

#### 3.10 Safe error handling

Return a generic error response and a correlation/trace ID to the caller. Keep detailed exceptions in access-controlled logs. Avoid exposing SQL, stack traces, filesystem paths, secrets, internal hostnames, or personal data.

Use ASP.NET Core exception-handling middleware and `ProblemDetails` for consistent error responses.

---

### 4. Clean Architecture

#### 4.1 Why use it?

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

#### 4.2 Domain entity example

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

#### 4.3 Repository pattern: use it deliberately

A repository can express persistence needs in domain terms and make a use case independent of a particular data-access implementation. However, wrapping every EF Core operation in a generic repository often adds boilerplate and hides useful query capabilities. EF Core's `DbContext` already implements Unit of Work-like behavior and its `DbSet` has repository-like characteristics.

Use repositories where they add a meaningful abstraction, not as a mandatory wrapper around every table.

#### 4.4 Transactions across repositories

When repositories use the same scoped `DbContext`, a single `SaveChangesAsync` normally persists their tracked changes in one database transaction when the provider supports transactions.

```csharp
await orderRepository.AddAsync(order, cancellationToken);
await inventoryRepository.ReserveAsync(order, cancellationToken);
await unitOfWork.SaveChangesAsync(cancellationToken);
```

For multiple databases or services, a local database transaction cannot make the whole distributed workflow atomic. Use a saga and/or transactional outbox, plus idempotency and compensating actions where appropriate.

#### 4.5 Dependency inversion and DI

Application defines an interface; Infrastructure implements it; the API composition root wires the concrete implementation.

```csharp
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IUnitOfWork, EfUnitOfWork>();
```

Avoid service-locator patterns and avoid making Domain depend on the DI container.

#### 4.6 Logging and testing

Use `ILogger<T>` and structured logging. Add correlation/trace context, but do not log secrets or sensitive request bodies by default.

Test the Application layer by testing use-case behavior and business decisions. Use fakes or mocks for external abstractions, and use integration tests against the real database provider for important persistence behavior. EF Core's in-memory provider is not a faithful substitute for relational database behavior.

#### 4.7 Clean vs traditional layered architecture

Both separate concerns. Clean Architecture is distinguished primarily by its dependency direction and protection of business rules. A layered architecture can also enforce inward dependencies; the name alone does not guarantee quality.

---

### 5. CQRS and event-driven architecture

#### 5.1 What is CQRS?

Command Query Responsibility Segregation separates operations that change state (commands) from operations that read state (queries). It does not require two databases or a message broker.

Use CQRS when read and write needs differ significantly, business commands have meaningful validation, or different read models improve performance. Avoid adding its complexity to a straightforward CRUD application without a clear benefit.

#### 5.2 Simple CQRS example

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

#### 5.3 Eventual consistency

If a command updates the write model and an asynchronous event updates a read model, there can be a period when reads show the old state. Make this explicit in product behavior. Use retries, idempotent consumers, monitoring, and a way to rebuild read models. For workflows requiring immediate confirmation, return the command result or read from the authoritative store where appropriate.

#### 5.4 Event sourcing

Event sourcing stores the sequence of domain events as the source of truth and derives current state by replaying events or using snapshots. It is useful when historical state and auditability are central. It adds complexity around event schema evolution, replay, snapshots, privacy/deletion requirements, and operational tooling. A normal audit log is not automatically event sourcing.

#### 5.5 Outbox and saga

**Transactional outbox:** write business data and an outgoing event record in the same local database transaction. A background publisher sends the event later. Consumers must tolerate duplicates.

**Saga:** coordinate a business process across multiple services through steps and compensating actions. It is not a distributed ACID transaction.

#### 5.6 Kafka vs RabbitMQ

- Kafka is a distributed event log suited to high-throughput streams, replay, and multiple consumer groups.
- RabbitMQ is a message broker suited to routing work to queues and acknowledgement-based consumption.
- Selection depends on ordering, throughput, replay, routing, delivery semantics, operations, and team expertise.

Neither automatically guarantees end-to-end exactly-once business effects. Design for duplicate delivery and idempotent processing.

#### 5.7 CQRS for a high-frequency trading platform

For a real trading system, correctness and risk controls take precedence over a generic “use Redis and Kafka” answer. Separate market-data ingestion, strategy computation, order management, risk checks, execution, and audit. Use bounded queues, explicit ordering semantics, deterministic risk checks, durable audit records, and strict latency measurements. Choose in-memory structures and transport only after requirements and benchmarks are clear. Kafka may suit market-data distribution and replay, but it is not automatically the lowest-latency component for every critical path.

---

### 6. Microservices and distributed systems

#### 6.1 Monolith vs modular monolith vs microservices

- **Monolith:** one deployable application; simplest operational model for many products.
- **Modular monolith:** one deployment with explicit module boundaries and controlled dependencies.
- **Microservices:** independently deployable services aligned to business capabilities, typically with their own data ownership.

Start with a modular monolith when the domain and team structure do not yet justify distributed complexity. Choose microservices when independent deployment/scaling, team autonomy, or isolation of failure domains provides a measurable benefit.

#### 6.2 Service communication

- **Synchronous:** REST or gRPC when the caller needs an immediate response.
- **Asynchronous:** broker/event communication when work can proceed independently or consumers need decoupling.

Every synchronous hop adds latency and a failure dependency. Use timeouts, bounded retries with backoff and jitter, circuit breaking where suitable, and cancellation. Do not retry non-idempotent operations blindly.

#### 6.3 API Gateway

A gateway can provide routing, TLS termination, rate limiting, authentication integration, and request aggregation. It should not become a giant business-logic monolith or the only place where authorization is enforced. Services must still enforce their own access rules.

#### 6.4 Service discovery

In Kubernetes, service DNS and cluster networking commonly provide discovery. Other environments may use a service registry or platform-specific naming. Prefer the simplest discovery mechanism supported by the deployment platform.

#### 6.5 Docker and Kubernetes

- Docker containers package the application and runtime dependencies consistently.
- Kubernetes orchestrates containers, scheduling, service discovery, health checks, rollout, and scaling.
- Kubernetes adds operational complexity; it is not automatically required for every .NET application.

Use a multi-stage Docker build and a supported .NET runtime image. Pin production image versions rather than relying on `latest`.

#### 6.6 Authentication between microservices

Use workload identity or managed identity where available, short-lived credentials, TLS, and audience-scoped tokens. Use service-to-service authorization and least privilege. A gateway-authenticated request should not automatically be trusted by every internal service.

#### 6.7 gRPC

gRPC uses Protocol Buffers contracts and commonly HTTP/2 for efficient typed communication and streaming. It is often a good fit for internal service-to-service calls. REST remains useful for browser-facing APIs, broad interoperability, and easy debugging. Choose based on clients, tooling, latency, streaming, and contract needs.

#### 6.8 Distributed caching

Use Redis or another shared cache when instances need common cached values. Define key structure, TTL, invalidation, tenant isolation, stampede prevention, failure behavior, and sensitive-data policy. A cache is not the source of truth unless the system is deliberately designed that way.

---

### 7. System design and deployment

#### 7.1 Design a high-scale e-commerce API

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

#### 7.2 Cloud deployment choices

Choose based on operations, scale, networking, compliance, cost, and team expertise:

- Managed application platform for simpler operations
- Container apps or managed container services for containerized workloads
- Kubernetes when orchestration flexibility and workload scale justify its overhead
- Serverless functions for event-triggered jobs with suitable execution characteristics

Use managed identity, secret management, health checks, autoscaling, private networking where required, and infrastructure as code.

#### 7.3 Zero-downtime deployment

Common techniques:

- Rolling deployment with readiness checks
- Blue-green deployment with traffic switching
- Canary release with gradual traffic increase
- Feature flags for separating deployment from feature activation

Zero downtime also requires backward-compatible database changes: add new schema first, deploy code that supports both old and new shapes, migrate data, then remove old schema in a later release. Avoid destructive schema changes in the same step as the first new-code deployment.

#### 7.4 Horizontal vs vertical scaling

- **Vertical:** give one instance more CPU/RAM; simple, but bounded by machine limits.
- **Horizontal:** add instances; improves capacity and resilience, but requires statelessness or externalized state, load balancing, and careful handling of shared dependencies.

Scaling API replicas will not solve a database bottleneck unless database capacity and query behavior are also addressed.

#### 7.5 Terraform and CI/CD

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

#### 7.6 Logging, metrics, tracing, and observability

- **Logs:** discrete events and diagnostic details.
- **Metrics:** numeric time series such as request rate, latency, errors, CPU, queue depth.
- **Traces:** a request's path across services and dependencies.

Use OpenTelemetry instrumentation and export to a supported backend. Propagate trace context across HTTP and messaging. Monitor service-level indicators such as latency percentiles, error rates, and saturation. Avoid logging access tokens, secrets, and sensitive payloads.

#### 7.7 Global APIs

Use edge routing/CDN for suitable content, regional deployments where justified, health-aware traffic routing, and data residency-aware storage. Decide whether data is globally replicated, regionally owned, or partitioned by tenant. Multi-region writes and failover are data-consistency problems as much as networking problems.

---

### 8. Advanced C# coding patterns

#### 8.1 Custom `IAsyncEnumerable<T>`

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

#### 8.2 Priority queue

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

#### 8.3 Thread-safe cache

`ConcurrentDictionary` provides thread-safe dictionary operations, but its value factory may run more than once under contention. It is not an expiration policy by itself.

```csharp
private readonly ConcurrentDictionary<string, ProductDto> _cache = new();

public ProductDto GetOrAdd(string key, Func<string, ProductDto> factory)
    => _cache.GetOrAdd(key, factory);
```

For expiration, size limits, and cache policies, use `IMemoryCache` or a distributed cache as appropriate.

#### 8.4 Fibonacci: memoization and iterative alternative

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

#### 8.5 Optimize slow LINQ / EF Core queries

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

#### 8.6 Request/response body logging middleware

Be cautious: body logging can expose passwords, access tokens, health data, and personal information. It can also add buffering and memory costs. Prefer structured metadata and log bodies only with explicit redaction, size limits, content-type filters, and a justified requirement.

A safer default is to log method, route, status, elapsed time, trace ID, and a request/correlation ID—not raw bodies.

#### 8.7 Worker queue with `System.Threading.Channels`

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

#### 8.8 `Parallel.ForEach` vs `Task.WhenAll`

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

#### 8.9 `System.Text.Json`

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

### 9. Senior-level interview answer framework

For each architecture or performance question, answer in this order:

1. **Clarify requirements:** traffic, latency, consistency, security, budget, and operational constraints.
2. **State the principle:** explain the pattern in one or two sentences.
3. **Describe the design:** components, responsibilities, and data flow.
4. **Discuss trade-offs:** complexity, cost, latency, consistency, and maintainability.
5. **Cover failure modes:** timeouts, retries, duplicate messages, cache staleness, overload, and recovery.
6. **Explain measurement:** metrics, load tests, traces, query plans, and acceptance thresholds.
7. **Give an example:** a realistic .NET implementation or a project scenario.

#### Common senior-level follow-up questions

- How would you prove this is faster? Which benchmark and production metric would you use?
- What happens when Redis, the database, or the message broker is unavailable?
- How do you prevent duplicate processing?
- What is the consistency model, and can the business accept stale reads?
- How does the design behave under overload?
- How are secrets, tenant boundaries, and permissions enforced?
- How will you deploy a schema change without downtime?
- What is the rollback plan?
- What is the simplest design that meets the requirements?

#### Final advice

Avoid absolute statements such as “async always makes an API faster,” “microservices are always more scalable,” or “`ConfigureAwait(false)` always switches to a ThreadPool thread.” Senior-level answers explain the conditions, trade-offs, and failure modes, and show how to validate decisions with measurements.
