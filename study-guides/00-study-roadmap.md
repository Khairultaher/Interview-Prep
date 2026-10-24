# Senior .NET Study Roadmap and Interview Strategy

Use this roadmap to plan preparation across .NET runtime, C#, APIs, data, architecture, security, distributed systems, operations, and system design. Study the focused topic guides from the library index alongside this sequence.

## Senior .NET Developer and Solution Architect Roadmap

> **Purpose:** A structured learning path based on the topics discussed
> in the previous chat. Repeated subjects have been consolidated so each
> concept has one primary home. Related topics are cross-referenced
> rather than taught repeatedly.

### 1. .NET Runtime and Request Execution

-   **CLR (Common Language Runtime):** What the CLR does, managed
    execution, assemblies, metadata, and runtime services.
-   **Compilation and execution:** C# compilation, IL, JIT compilation,
    tiered compilation, and ReadyToRun.
-   **Memory fundamentals:** Stack vs. heap, object lifetime,
    references, and allocation.
-   **Garbage collection:** Generations 0/1/2, large object heap,
    workstation vs. server GC, allocation pressure, and finalization.
-   **ASP.NET Core request pipeline:** Middleware order, routing,
    endpoint execution, filters, and exception handling.
-   **Kestrel and IIS:** Hosting models, reverse proxying, request flow,
    process recycling, and deployment considerations.
-   **Dependency injection:** Container resolution, service lifetimes
    (`Transient`, `Scoped`, `Singleton`), scope validation, and
    disposal.
-   **Configuration and options:** Configuration providers,
    environment-specific settings, and the Options pattern.

### 2. C# Language and Concurrency

-   **Core language features:** Generics, delegates, events, LINQ,
    records, pattern matching, and nullable reference types.
-   **Async programming:** `Task`, `async`/`await`, continuations,
    cancellation tokens, exception propagation, and common
    deadlock/blocking mistakes.
-   **Thread Pool:** Work-item scheduling, thread injection, starvation,
    and diagnosing blocked worker threads.
-   **Concurrency vs. parallelism:** When to use asynchronous I/O,
    `Parallel`, `Task.Run`, and dedicated threads.
-   **Synchronization:** Locks, `SemaphoreSlim`, concurrent collections,
    race conditions, and thread safety.
-   **Advanced memory APIs:** `Span<T>`, `Memory<T>`, `ArrayPool<T>`,
    object pooling, and allocation-conscious code.
-   **Resource cleanup:** `IDisposable`, `IAsyncDisposable`, `using`,
    and safe handling of unmanaged resources.

### 3. Performance Diagnosis and Optimization

-   **Measure before optimizing:** Establish baselines and distinguish
    CPU, memory, database, network, and queueing bottlenecks.
-   **Profiling tools:** Visual Studio Profiler, `dotnet-counters`,
    `dotnet-trace`, PerfView, and memory profilers.
-   **Benchmarking:** BenchmarkDotNet, representative workloads, and
    avoiding misleading microbenchmarks.
-   **GC and allocation tuning:** Reduce unnecessary allocations; use
    pooling only when measurement justifies it.
-   **HTTP performance:** Compression, payload sizing, streaming,
    connection reuse, and HTTP protocol considerations.
-   **Load and stress testing:** k6 or JMeter, throughput, latency
    percentiles, saturation, and capacity limits.
-   **Production observability:** Metrics, traces, logs, dashboards,
    alerts, and service-level indicators.

### 4. ASP.NET Core API Design

-   **HTTP and REST fundamentals:** Resource-oriented routes, verbs,
    status codes, headers, content negotiation, and API versioning.
-   **Request validation and errors:** DTOs, validation, consistent
    problem details, exception handling, and safe error responses.
-   **Pagination and filtering:** Offset vs. keyset pagination, stable
    ordering, filtering, sorting, and response-size limits.
-   **Idempotency and concurrency:** Safe retries for writes,
    idempotency keys, ETags, and optimistic concurrency.
-   **Caching and HTTP semantics:** Cache-Control, ETags, conditional
    requests, output caching, and cache invalidation. Distributed cache
    implementation is covered in Section 9.
-   **Rate limiting and throttling:** Partitioning limits by user,
    client, or IP; handling HTTP 429; and coordinating limits across
    instances.
-   **Alternative API styles:** GraphQL trade-offs, gRPC contracts and
    Protobuf, SignalR/WebSockets for real-time communication.
-   **Background work:** When to move long-running work out of the
    request lifecycle; durable jobs and hosted services are covered in
    Section 8.

### 5. Data Access and Database Engineering

-   **EF Core fundamentals:** Change tracking, migrations,
    relationships, query translation, and `DbContext` lifetime.
-   **Query performance:** Projection, `AsNoTracking`, avoiding N+1
    queries, split queries, compiled queries, and inspecting generated
    SQL.
-   **Pagination and large datasets:** Efficient ordering, keyset
    pagination, batching, and streaming where appropriate.
-   **SQL Server fundamentals:** Execution plans, clustered/nonclustered
    indexes, statistics, sargability, and index trade-offs.
-   **Transactions and concurrency:** ACID, isolation levels, locks,
    deadlocks, optimistic concurrency, and retrying transient database
    errors.
-   **Connection management:** ADO.NET connection pooling, command
    timeouts, EF Core context pooling, and their distinct purposes.
-   **Dapper vs. EF Core:** Choosing based on complexity, control,
    maintainability, and measured performance.
-   **Database scaling:** Read replicas, partitioning, sharding, and
    when these approaches are warranted.
-   **SQL vs. NoSQL:** Data-model and consistency trade-offs; choose
    document databases such as MongoDB or Cosmos DB for concrete
    workload needs.

### 6. Architecture, Design Principles, and Domain Modeling

-   **SOLID and clean code:** Cohesion, coupling, abstractions, and
    maintainable boundaries.
-   **Dependency inversion and Clean Architecture:** Domain,
    application, infrastructure, and presentation responsibilities.
-   **Common design patterns:** Strategy, Factory, Decorator, Adapter,
    Repository, and Unit of Work---use only where they solve a real
    problem.
-   **Domain-Driven Design (DDD):** Entities, value objects, aggregates,
    aggregate roots, domain services, and bounded contexts.
-   **CQRS:** Separate command and query models when the domain or
    workload benefits; CQRS does not automatically require separate
    databases or event sourcing.
-   **Domain events:** Represent meaningful domain facts and coordinate
    in-process reactions without coupling unrelated domain logic.
-   **Modular monolith vs. microservices:** Boundaries, deployment
    independence, operational cost, and criteria for splitting services.

### 7. Security and Identity

-   **Authentication vs. authorization:** Identity verification and
    permission enforcement.
-   **OAuth 2.0 and OpenID Connect:** Authorization Code with PKCE,
    client credentials, scopes, audiences, and choosing an identity
    provider.
-   **JWT validation:** Issuer, audience, signature, expiry, key
    rotation, claims, and secure token handling. A JWT is signed, not
    automatically encrypted.
-   **Authorization models:** Roles, claims, policies, resource-based
    authorization, and tenant/data-level access checks.
-   **Password and secret handling:** Use established identity libraries
    and password hashing; keep secrets in a secret manager such as Azure
    Key Vault.
-   **Web/API threats:** SQL injection, XSS, CSRF, SSRF, broken access
    control, insecure deserialization, and dependency vulnerabilities.
-   **Transport and browser security:** HTTPS, CORS, CSP, security
    headers, cookies, SameSite, and antiforgery protections where
    applicable.
-   **API protection:** Rate limits, input validation, least privilege,
    safe error handling, and audit logging.
-   **Security operations and compliance:** Dependency scanning, threat
    modeling, logging without sensitive data, and mapping controls to
    relevant obligations such as HIPAA or GDPR.

### 8. Background Processing and Reliable Messaging

-   **Hosted services and job schedulers:**
    `IHostedService`/`BackgroundService`, Hangfire, and Quartz.NET;
    understand persistence, retries, and job ownership.
-   **Message-broker concepts:** Queues vs. topics/pub-sub, competing
    consumers, delivery acknowledgements, dead-letter queues, and back
    pressure.
-   **Choose a broker by workload:**
    -   **RabbitMQ:** Broker-based routing and work queues.
    -   **Kafka:** Durable event streams, partitions, consumer groups,
        and replay.
    -   **Azure Service Bus:** Managed enterprise queues/topics,
        dead-lettering, and sessions.
    -   **Azure Event Grid:** Event routing and notifications, rather
        than a general replacement for every queue.
-   **Messaging libraries:** MassTransit or similar abstractions where
    they simplify consumers, retries, and transport integration.
-   **Delivery semantics:** At-most-once, at-least-once, and practical
    implications; design consumers to be idempotent.
-   **Outbox and inbox patterns:** Persist business changes and outgoing
    events atomically; deduplicate incoming messages.
-   **Ordering and schema evolution:** Partitioning/session keys, event
    contracts, compatibility, and versioning.

### 9. Distributed Systems and Microservices

-   **Synchronous vs. asynchronous communication:** REST/gRPC for
    request-response; messages/events for decoupled workflows.
-   **API gateway and reverse proxy:** Routing, cross-cutting policies,
    YARP/Ocelot, and Azure API Management; avoid duplicating business
    authorization only at the gateway.
-   **Service discovery and networking:** DNS, platform-native
    discovery, health checks, and when a service mesh is justified.
-   **Resilience:** Timeouts, bounded retries with backoff and jitter,
    circuit breakers, bulkheads, and carefully designed fallbacks. Avoid
    retry storms and retries for non-idempotent operations.
-   **Distributed transactions:** Why ordinary database transactions do
    not span independent services; Saga orchestration/choreography and
    compensating actions.
-   **Consistency and failure modes:** Eventual consistency, duplicate
    delivery, partial failure, clock assumptions, and CAP trade-offs.
-   **Event sourcing:** Store domain events as the source of truth only
    when auditability, temporal reconstruction, or domain requirements
    justify its complexity. It is distinct from ordinary event-driven
    messaging.
-   **Distributed caching:** `IMemoryCache`
    vs. Redis/`IDistributedCache`, cache-aside, TTLs, invalidation,
    stampede prevention, and stale-data trade-offs.
-   **Multi-tenancy:** Tenant isolation, tenant resolution, per-tenant
    authorization, and database/schema/shared-database trade-offs.

### 10. Observability and Production Reliability

-   **Structured logging:** Semantic properties, log levels, scopes, and
    avoiding secrets or sensitive data in logs.
-   **Correlation and trace context:** Propagate trace context across
    HTTP and messaging; use correlation IDs where useful.
-   **OpenTelemetry:** Instrumentation, traces, metrics, exporters, and
    distributed request analysis.
-   **Health checks:** Liveness vs. readiness, dependency checks, and
    avoiding overly aggressive health probes.
-   **Failure diagnosis:** Use logs, metrics, traces, dumps, counters,
    and dependency telemetry together.
-   **Operational readiness:** SLOs, alert thresholds, incident
    response, runbooks, graceful shutdown, and deployment rollback.

### 11. Cloud, Containers, and Delivery

-   **Azure hosting choices:** App Service, Azure Container Apps,
    Functions, AKS, and choosing the least operationally complex fit.
-   **Docker:** Multi-stage builds, non-root execution, image size,
    configuration, secrets, and container health.
-   **Kubernetes fundamentals:** Pods, Deployments, Services, Ingress,
    probes, resource requests/limits, autoscaling, and rollout
    strategies.
-   **Networking and identity:** Private networking, managed identities,
    least-privilege RBAC, and secure service-to-service access.
-   **CI/CD:** Build/test/package/release pipelines, environment
    promotion, approvals, rollback, and artifact traceability.
-   **Infrastructure as Code:** Bicep, Terraform, or ARM; repeatable
    environments and drift management.
-   **Configuration and secrets:** Separate configuration from images;
    avoid committing credentials or embedding secrets in containers.
-   **Deployment strategies:** Rolling, blue-green, and canary releases;
    database migration compatibility during rollouts.

### 12. Architecture and System Design Practice

-   **Requirements first:** Functional requirements, quality attributes,
    constraints, assumptions, and measurable targets.
-   **Capacity and bottleneck analysis:** Estimate traffic, storage,
    throughput, concurrency, and dependencies.
-   **Architecture trade-offs:** Simplicity vs. flexibility, consistency
    vs. availability, synchronous vs. asynchronous flow, and operational
    burden.
-   **Data and integration design:** Ownership, boundaries, contracts,
    failure handling, and data lifecycle.
-   **High availability and scaling:** Horizontal/vertical scaling, load
    balancing, redundancy, failover, and disaster recovery.
-   **Design reviews:** Explain decisions, alternatives, risks,
    observability, security, and cost---not just diagrams.
-   **Practice designs:** Order processing, document processing,
    notification delivery, multi-tenant SaaS, and audit-heavy systems.

### Suggested Study Order

1.  Sections 1--3: Runtime, concurrency, and performance foundations.
2.  Sections 4--5: API and database depth.
3.  Sections 6--7: Architecture and security.
4.  Sections 8--10: Messaging, distributed systems, and operations.
5.  Section 11: Cloud and delivery.
6.  Section 12: Apply the concepts through system-design exercises.

### Learning Method

For each topic, be able to explain:

1.  **What it is** and which problem it solves.
2.  **How it works internally** in .NET or the surrounding platform.
3.  **When to use it** and when not to.
4.  **Failure modes and trade-offs.**
5.  **How to measure, debug, secure, and test it.**
6.  **A practical implementation** using current .NET APIs and
    production-minded patterns.

------------------------------------------------------------------------

**Deduplication note:** Security, caching, API performance, CQRS,
microservices communication, messaging, resilience, and observability
appeared repeatedly in the previous sequence. They are now each
organized under one primary section; related references elsewhere are
intentionally brief rather than repeating the full topic.
