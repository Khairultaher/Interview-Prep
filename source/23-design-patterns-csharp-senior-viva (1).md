# 23 GoF Design Patterns in C# — Senior .NET Developer / Architect Viva Guide

A practical viva-preparation guide to the **23 Gang of Four (GoF) design patterns**, grouped into **Creational**, **Structural**, and **Behavioral** patterns. Each pattern includes its intent, a .NET application scenario, a C# example, and a senior-level consideration.

> **Code note:** Examples are intentionally small and focused on the pattern. Some snippets assume the surrounding application interfaces or services already exist; they are not intended to form one giant compilable project.

## Contents

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

# 1. Creational patterns

Creational patterns separate **how objects are created** from the code that uses them. They are useful when construction varies, has many steps, or must be controlled consistently.

## 1. Singleton

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

## 2. Factory Method

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

## 3. Abstract Factory

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

## 4. Builder

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

## 5. Prototype

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

# 2. Structural patterns

Structural patterns describe how objects and classes are connected to form larger components while limiting coupling between parts.

## 6. Adapter

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

## 7. Bridge

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

## 8. Composite

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

## 9. Decorator

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

## 10. Facade

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

## 11. Flyweight

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

## 12. Proxy

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

# 3. Behavioral patterns

Behavioral patterns organize communication, responsibilities, and algorithms among objects.

## 13. Chain of Responsibility

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

## 14. Command

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

## 15. Interpreter

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

## 16. Iterator

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

## 17. Mediator

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

## 18. Memento

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

## 19. Observer

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

## 20. State

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

## 21. Strategy

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

## 22. Template Method

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

## 23. Visitor

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

# Common viva comparisons

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

# Architecture-level guidance

## How to answer a pattern question in a viva

For any pattern, structure the answer in this order:

1. **Problem:** What coupling, complexity, or variation does it address?
2. **Participants:** Which interfaces and concrete types collaborate?
3. **Application example:** Identify a realistic use case in an API, application service, integration, or domain workflow.
4. **Trade-off:** State what extra abstraction or complexity the pattern introduces.
5. **Alternative:** Explain when a simpler .NET feature or composition would be preferable.

## Applying patterns in ASP.NET Core and Clean Architecture

- **Dependency Injection is a tool, not a design pattern substitute.** It helps compose implementations, lifetimes, and decorators at the application boundary.
- **Prefer composition where it reduces coupling.** Inheritance-oriented patterns such as Template Method and Factory Method are useful, but not every variation needs a class hierarchy.
- **Keep domain rules explicit.** State and Strategy can model behavior, but they do not replace validation, transaction boundaries, authorization, or concurrency control.
- **Use framework support intentionally.** ASP.NET Core middleware, `IEnumerable<T>`, `IAsyncEnumerable<T>`, C# events, EF Core proxies, and DI lifetimes already embody or support ideas related to several patterns.
- **Avoid pattern overuse.** A pattern is justified when it makes a real variation point, dependency boundary, or algorithm easier to understand and maintain—not merely because a class can be named after it.
- **Consider operational behavior.** In production systems, evaluate cancellation, retries, idempotency, telemetry, resource disposal, caching, concurrency, and failure recovery alongside the class design.

## Final reminder

The goal of a senior-level answer is not just to recite each pattern's definition. Explain **why it fits the problem, what it costs, how it integrates with .NET's built-in capabilities, and what could go wrong in production**.
