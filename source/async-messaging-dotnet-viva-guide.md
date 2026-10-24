# Asynchronous Messaging in .NET — Senior Developer & Architect Viva Guide

This guide brings together **MediatR, MassTransit, RabbitMQ, Kafka, and publish–subscribe (pub/sub)**. It focuses on what each tool does, how messages flow, when to choose each approach, and the reliability concerns expected in a senior-level design discussion.

## 1. The key distinction

| Technology | Main role | Scope | Typical use |
|---|---|---|---|
| **MediatR** | In-process mediator | One application process | Route a command/query to a handler; notify local handlers |
| **MassTransit** | .NET messaging framework | Application ↔ broker | Define consumers, publish/send messages, retries, and sagas using a consistent API |
| **RabbitMQ** | Message broker | Across processes/services | Route messages to queues; distribute commands and events |
| **Apache Kafka** | Distributed event-streaming platform | Across processes/services | Store and distribute partitioned event streams; support replay and high-throughput processing |

**Remember:** MediatR does not itself provide durable, cross-service messaging. MassTransit is not a broker; it is a framework that works with transports such as RabbitMQ and, through its Kafka rider, Kafka. RabbitMQ and Kafka have different delivery and storage models, so they are not interchangeable in every design.

## 2. Commands, events, and notifications

- A **command** asks a specific handler/service to perform an action, such as `CreateOrder` or `ChargePayment`.
- An **event** states that something has already happened, such as `OrderCreated` or `PaymentAuthorized`. It can have multiple subscribers.
- A **notification** is a broad term for information sent to interested parties. In MediatR, `INotification` means an in-process notification; in distributed messaging, an event may be published through a broker.

Naming an event in the past tense (`OrderCreated`) makes its meaning clearer: it describes a fact, rather than instructing another service what to do.

## 3. MediatR: in-process request/notification handling

MediatR helps decouple code **inside one .NET process**. It routes requests to a handler and can dispatch notifications to multiple local handlers. It does not automatically persist messages or deliver them to another service.

### Request/command example

```csharp
using MediatR;

public sealed record CreateOrderCommand(Guid CustomerId, decimal Total)
    : IRequest<Guid>;

public sealed class CreateOrderHandler
    : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        // Validate and save the order using the application's domain/repository layer.
        await Task.CompletedTask;
        return Guid.NewGuid();
    }
}
```

A controller or endpoint can inject `ISender` and call `Send`:

```csharp
var orderId = await sender.Send(
    new CreateOrderCommand(customerId, total), cancellationToken);
```

### Local notification example

```csharp
public sealed record OrderCreatedLocalNotification(Guid OrderId) : INotification;

public sealed class WriteAuditEntryHandler
    : INotificationHandler<OrderCreatedLocalNotification>
{
    public Task Handle(
        OrderCreatedLocalNotification notification,
        CancellationToken cancellationToken)
    {
        // Write an in-process audit entry or trigger local application behavior.
        return Task.CompletedTask;
    }
}
```

Dispatch it with `IPublisher`:

```csharp
await publisher.Publish(
    new OrderCreatedLocalNotification(orderId), cancellationToken);
```

Register handlers from the relevant assembly, using the registration API supported by your installed MediatR version:

```csharp
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<CreateOrderHandler>());
```

### When MediatR is appropriate

Use it for application-layer CQRS handlers, local orchestration, and separation between controllers and use cases. Do **not** use `IPublisher.Publish` as a replacement for a durable broker event: the call is in-process and is not a cross-service delivery guarantee.

## 4. Publish–subscribe in RabbitMQ

RabbitMQ pub/sub is built around **exchanges, bindings, queues, and consumers**:

1. A producer publishes a message to an exchange.
2. The exchange routes it according to its type and binding rules.
3. Each matching queue receives a copy of the message.
4. Consumers attached to the same queue compete to process messages from that queue.

### Main exchange types

| Exchange type | Routing behavior | Typical use |
|---|---|---|
| **Fanout** | Routes to every bound queue; routing key is ignored | Broadcast to all interested services |
| **Direct** | Routes when the routing key exactly matches the binding key | Route to a specific category/severity |
| **Topic** | Routes using patterns: `*` matches one word; `#` matches zero or more words | Subscribe to selected event categories, such as `order.*` |
| **Headers** | Routes using message headers instead of the routing key | Specialized header-based routing |

### Example topology

Suppose `OrderService` publishes `OrderCreated`. Three services need their own copy:

```text
                         ┌─> inventory.queue ─> Inventory Service
OrderService ─> exchange ├─> billing.queue   ─> Billing Service
                         └─> email.queue     ─> Notification Service
```

Each service has a **separate queue**. If Inventory Service has three replicas consuming `inventory.queue`, those replicas are competing consumers: a given delivery goes to one replica, rather than being broadcast to all three. Broadcasting happens across the independently bound queues.

### Reliability points

- Use durable topology and persistent messages where survival across broker restarts is required.
- Consumer acknowledgements allow RabbitMQ to know when a delivery has been processed; unacknowledged messages can be redelivered after a failure.
- Publisher confirms help the producer determine whether RabbitMQ accepted a publish. They are different from consumer acknowledgements.
- Redelivery can create duplicates. Design consumers to be **idempotent**.
- Configure bounded retries and a dead-letter strategy for messages that repeatedly fail. Avoid infinite requeue loops.
- If no queue is bound when a message is published, ordinary exchange routing does not create a historical copy for a future subscriber. Plan the queue topology before relying on delivery.

## 5. Publish–subscribe in Apache Kafka

Kafka pub/sub is built around **topics, partitions, consumer groups, offsets, and retention**:

1. A producer appends a record to a topic.
2. The topic is split into partitions, each an ordered log.
3. Consumers read records and track their positions using offsets.
4. Each consumer group tracks its own progress, so different groups can independently process the same topic.

### Example topology

```text
                         ┌─ group: inventory-service ─> Inventory Service
OrderService ─> topic ───┼─ group: billing-service   ─> Billing Service
  order-created          └─ group: analytics-service ─> Analytics Service
```

Each group can read the same records independently. Within a consumer group, partitions are assigned among its consumers, so adding consumers increases parallelism only up to the available partition count (under the standard consumer-group model). Ordering is guaranteed within a partition, not across the entire topic. Choosing a key such as `OrderId` can keep related records in the same partition.

### Reliability and replay points

- Kafka normally retains records for a configured time or size policy, regardless of whether a consumer has read them.
- An offset records a consumer group's position. A consumer can often reset an offset and reprocess retained records.
- Offset commit timing matters: committing before business processing finishes can lose work; committing afterwards can result in a record being processed again after a crash.
- Duplicates are possible, so idempotent processing is still important.
- Kafka supports strong processing guarantees for certain Kafka-to-Kafka workflows, but a database update or external API call is not automatically part of a Kafka transaction.

## 6. RabbitMQ vs Kafka for pub/sub

| Concern | RabbitMQ | Kafka |
|---|---|---|
| Core model | Broker routes messages to queues | Consumers read a retained, partitioned log |
| Pub/sub unit | One queue per independent subscriber; exchange bindings decide routing | One topic; each consumer group has independent progress |
| Scaling consumers | Multiple consumers compete on a queue | Consumers in a group split topic partitions |
| Message lifetime | Common queue workflows remove a message after acknowledgement | Records remain until retention/compaction removes them |
| Replay | Not the default queue model; design a separate archive/stream if needed | Natural use case through offsets while records are retained |
| Routing | Rich exchange and binding rules | Topic/partition/key based; routing is usually designed by the producer and topic structure |
| Common fit | Commands, work distribution, service integration, flexible routing | High-volume event streams, analytics, replay, multiple independent consumers |

Neither is universally better. Choose based on delivery semantics, replay needs, throughput, ordering, operational maturity, and team experience—not only a benchmark.

## 7. MassTransit with RabbitMQ: end-to-end example

MassTransit provides a .NET abstraction for publishing, sending, consuming, retry behavior, and orchestration. The following is an illustrative RabbitMQ setup; use secure credentials from configuration or a secret store in real environments.

### Message contract

```csharp
public sealed record OrderCreated(
    Guid OrderId,
    Guid CustomerId,
    decimal Total,
    DateTime CreatedAtUtc);
```

In a multi-service solution, put public message contracts in a small shared contracts assembly/package, or use a deliberate schema/versioning strategy. Avoid publishing database entities directly.

### Consumer

```csharp
using MassTransit;

public sealed class OrderCreatedConsumer : IConsumer<OrderCreated>
{
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        var message = context.Message;

        // Call the service's application layer. Make the operation idempotent.
        Console.WriteLine($"Order created: {message.OrderId}");
        await Task.CompletedTask;
    }
}
```

### Dependency injection and RabbitMQ configuration

```csharp
using MassTransit;

builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", host =>
        {
            host.Username("guest");
            host.Password("guest");
        });

        // Creates receive endpoints for the registered consumers.
        cfg.ConfigureEndpoints(context);
    });
});
```

The `guest` credentials are for a local development example, not a production configuration. MassTransit starts and manages the bus through its hosting integration; older examples may show a separate `AddMassTransitHostedService` registration that is not needed in current versions.

### Publisher

```csharp
using MassTransit;

public sealed class OrderApplicationService(IPublishEndpoint publishEndpoint)
{
    public Task PublishOrderCreatedAsync(
        Guid orderId,
        Guid customerId,
        decimal total,
        CancellationToken cancellationToken)
    {
        return publishEndpoint.Publish(
            new OrderCreated(
                orderId,
                customerId,
                total,
                DateTime.UtcNow),
            cancellationToken);
    }
}
```

`Publish` expresses an event: interested consumers can receive it through their broker topology. `Send` is typically used when directing a message to a known endpoint/queue, such as a command. Exact topology and naming conventions depend on transport configuration.

## 8. Where Kafka and MassTransit fit

MassTransit supports Kafka through a **Kafka rider** and the Confluent .NET client. However, Kafka remains topic/partition/log based; it does not acquire RabbitMQ's exchange-and-queue routing model simply because MassTransit is used. Check the current MassTransit documentation and compatibility notes before choosing its Kafka integration for a production system.

For a direct Kafka implementation in .NET, the commonly used client is Confluent.Kafka. Choose the client approach based on required Kafka features, operational standards, and whether your team benefits from the MassTransit abstraction.

## 9. Reliability patterns an architect should mention

### The dual-write problem and the Outbox pattern

Imagine saving an order to SQL Server and then publishing `OrderCreated`:

```text
Save order to database  ── success
Publish event to broker ── failure
```

The order exists, but downstream services never receive the event. Reversing the order creates the opposite failure risk. The **Transactional Outbox** pattern addresses this by writing the business change and an outbox record in the same database transaction. A background publisher sends the outbox event to the broker and marks it as dispatched. Consumers still need idempotency because publishing can be retried.

### Other essential controls

- **Idempotent consumer:** Processing the same event twice must not create a second payment, duplicate record, or repeated side effect. Use a stable event/message ID and a durable deduplication mechanism where necessary.
- **Retry policy:** Retry transient failures with bounded attempts and backoff; do not retry permanent validation failures forever.
- **Dead-letter handling:** Preserve failed messages, record the error, alert, and define a safe replay procedure.
- **Eventual consistency:** Different services update at different times. The UI and business workflow must tolerate intermediate states.
- **Saga/process manager:** Coordinate a multi-service business process with local transactions and compensating actions instead of expecting one distributed database transaction.
- **Observability:** Include correlation/trace IDs, structured logs, queue depth or consumer lag, retry counts, dead-letter counts, and message processing latency.
- **Contract evolution:** Version event schemas compatibly; treat published events as long-lived public contracts.

## 10. Common senior-level viva questions

**Q: Is MediatR asynchronous messaging?**  
It supports asynchronous method calls within a process, but it is not a distributed message broker and does not by itself provide durable delivery or cross-process communication.

**Q: What is the difference between `Send` and `Publish`?**  
In MediatR, `Send` dispatches a request to one request handler, while `Publish` dispatches an in-process notification to its notification handlers. In MassTransit, `Send` generally targets a known endpoint and `Publish` distributes an event to interested subscribers through the transport topology.

**Q: In RabbitMQ, how do I make three services each receive an event?**  
Publish to an exchange and bind a separate queue for each service. Three consumers on the same queue are competing consumers, not three independent subscribers.

**Q: In Kafka, how do three services each receive an event?**  
Give each service a distinct consumer group. If all three share one group, they divide the work instead of each receiving an independent copy.

**Q: Does a message broker guarantee exactly-once business processing?**  
Do not assume so. Delivery may be repeated, and external side effects complicate exactly-once claims. Design for idempotency and use transactional patterns such as an Outbox where required.

**Q: When would you choose Kafka over RabbitMQ?**  
Prefer Kafka when retained event history, replay, high-throughput streams, and multiple independent downstream consumers are central requirements. Prefer RabbitMQ when broker routing, task distribution, acknowledgements, and queue-oriented workflows are the natural fit. Validate the decision against actual requirements and operations.

**Q: How do you keep SQL writes and event publishing consistent?**  
Use a Transactional Outbox, then make consumers idempotent. A database commit and broker publish are not one atomic transaction by default.

**Q: How do you handle a poison message?**  
Apply bounded retries, route it to a dead-letter/error workflow, capture diagnostic context, alert operators, and replay only after fixing the cause.

## 11. Quick decision guide

- **Controller → application use case inside one API:** MediatR or direct dependency injection, depending on whether mediator indirection adds value.
- **One service needs to command a known downstream endpoint:** broker send/command pattern, commonly through MassTransit.
- **Several services react independently to a business fact:** publish an event through RabbitMQ or Kafka.
- **Need rich queue routing and task distribution:** evaluate RabbitMQ first.
- **Need retained event history, replay, and streaming analytics:** evaluate Kafka first.
- **Need local database changes and reliable event publication:** use an Outbox regardless of which broker is selected.

## Further reading

- [RabbitMQ — Exchanges and routing](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [RabbitMQ — Consumer acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/confirms)
- [Apache Kafka — Introduction](https://kafka.apache.org/intro)
- [Apache Kafka — Consumer offset tracking](https://kafka.apache.org/43/implementation/distribution/)
- [MassTransit — Bus configuration](https://masstransit.io/documentation/configuration)
- [MassTransit — RabbitMQ transport](https://masstransit.io/documentation/configuration/transports/rabbitmq)
- [MassTransit — Kafka rider](https://masstransit.io/documentation/configuration/transports/kafka)
- [MediatR — Repository and registration examples](https://github.com/LuckyPennySoftware/MediatR)

---

**Viva takeaway:** MediatR decouples handlers inside a process; MassTransit simplifies .NET messaging; RabbitMQ routes messages into queues; Kafka stores records in partitioned topics that consumer groups read independently. In a reliable architecture, also explain idempotency, retries, dead-letter handling, event versioning, observability, and the Outbox pattern.
