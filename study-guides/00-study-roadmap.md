# Study Roadmap and Interview Strategy

This is one consolidated study guide combining the senior .NET learning roadmap with the HTTP/API interview question bank. The roadmap provides the study sequence; the second part adds MCQs, API standards, testing guidance, and senior-level scenarios.

## Guide contents

1. [Senior .NET learning roadmap](#part-i-senior-net-developer-and-solution-architect-roadmap)
2. [HTTP and API interview practice](#part-ii-http-and-api-interview-question-bank)
---

## Part I: Senior .NET Developer and Solution Architect Roadmap

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
## Part II: HTTP and API Interview Question Bank

### HTTP, Request/Response, Status Codes, Authentication, Headers, Middleware, and Testing

> Target: Senior .NET Developer, Staff Engineer, and Solution Architect
> interviews\
> Examples: ASP.NET Core / .NET 8+, xUnit, `WebApplicationFactory`

### Contents

1.  [50 unique MCQs with answers](#1-50-unique-mcqs-with-answers)
2.  [How ASP.NET Core builds an HTTP
    response](#2-how-aspnet-core-builds-an-http-response)
3.  [Enterprise API standards
    checklist](#3-enterprise-api-standards-checklist)
4.  [Testing authentication, headers, and
    middleware](#4-testing-authentication-headers-and-middleware)
5.  [Senior-level scenario
    questions](#5-senior-level-scenario-questions)

------------------------------------------------------------------------

### 1. 50 Unique MCQs with Answers

Each question tests a distinct concept. The answer and a short
explanation follow each question.

#### HTTP fundamentals

**1. Which HTTP method is both safe and idempotent?**

A. POST\
B. GET\
C. PATCH\
D. CONNECT

**Answer: B --- GET.** Safe means the method is intended not to change
server state; idempotent means repeating the same request has the same
intended effect.

**2. What does idempotency mean for an HTTP operation?**

A. It always returns 200\
B. It can never fail\
C. Repeating it has the same intended effect as doing it once\
D. It must return an empty body

**Answer: C.** An idempotent operation may return different responses on
later calls, but its intended server-state effect remains the same.

**3. Which statement about POST is correct?**

A. POST is always idempotent\
B. POST is commonly used to create a resource or trigger processing\
C. POST must never have a response body\
D. POST is a safe method

**Answer: B.** POST is commonly used for resource creation and
command-style processing. It is not inherently safe or idempotent.

**4. Which method is normally used to replace a resource
representation?**

A. GET\
B. PUT\
C. HEAD\
D. OPTIONS

**Answer: B --- PUT.** PUT generally expresses replacement of the target
resource state and is defined as idempotent.

**5. Which statement about PATCH is most accurate?**

A. PATCH always replaces the whole resource\
B. PATCH is always idempotent\
C. PATCH applies partial modifications; idempotency depends on the patch
operation\
D. PATCH cannot contain a body

**Answer: C.** A patch that sets a field to a fixed value can be
idempotent; an operation that increments a value generally is not.

**6. What is the main purpose of the OPTIONS method?**

A. Delete a resource\
B. Discover communication options supported by a target or for CORS
preflight\
C. Create a resource\
D. Refresh a JWT

**Answer: B.** Browsers use OPTIONS for CORS preflight requests when
required.

**7. What is the main purpose of the HEAD method?**

A. Return headers without the response representation body\
B. Return a partial JSON body\
C. Create a resource\
D. Authenticate a user

**Answer: A.** HEAD is similar to GET but does not return the
representation body.

**8. What is the difference between a URI path and a query string?**

A. There is no difference\
B. The path identifies the target route/resource; query parameters
commonly qualify or filter the request\
C. Query strings are always encrypted\
D. Paths can only contain numbers

**Answer: B.** For example, `/orders/42` identifies an order while
`/orders?status=pending` filters a collection.

#### Status codes and response semantics

**9. Which status code most appropriately indicates that a resource was
created successfully?**

A. 200\
B. 201\
C. 202\
D. 204

**Answer: B --- 201 Created.** A `Location` header identifying the
created resource is often appropriate.

**10. When is 202 Accepted appropriate?**

A. The request is invalid\
B. Processing has been accepted but has not necessarily completed\
C. The resource is definitely created and fully processed\
D. The user is unauthenticated

**Answer: B.** It is useful for asynchronous jobs; provide a way to
check processing status when appropriate.

**11. When should an API return 204 No Content?**

A. When authentication fails\
B. When the operation succeeds and there is no response representation
to return\
C. Whenever JSON serialization fails\
D. When a resource is created

**Answer: B.** A 204 response must not include response content.

**12. What does 304 Not Modified normally indicate?**

A. The request body is malformed\
B. A conditional request can reuse a cached representation\
C. The server is overloaded\
D. The user lacks permission

**Answer: B.** It is used with conditional requests such as
`If-None-Match` or `If-Modified-Since`.

**13. Which status code means the request syntax or request content is
invalid?**

A. 400\
B. 401\
C. 403\
D. 503

**Answer: A --- 400 Bad Request.** Malformed JSON and invalid request
structure commonly result in 400.

**14. What should an API normally return when valid authentication
credentials are absent or invalid?**

A. 301\
B. 401\
C. 404\
D. 409

**Answer: B --- 401 Unauthorized.** Despite the name, it concerns
authentication and normally includes an applicable `WWW-Authenticate`
challenge.

**15. What is the usual distinction between 401 and 403?**

A. 401 means forbidden; 403 means not found\
B. 401 means authentication is missing or invalid; 403 means access is
refused\
C. They are interchangeable\
D. 403 means the server crashed

**Answer: B.** A valid identity can still be denied access by
authorization policy.

**16. When is 404 Not Found appropriate?**

A. Whenever a request is slow\
B. When the target resource is not found, or the API intentionally
conceals its existence\
C. Only when a database is offline\
D. Whenever validation fails

**Answer: B.** Security policy may intentionally use 404 to avoid
revealing a protected resource's existence.

**17. When is 409 Conflict useful?**

A. For a state conflict, such as a duplicate unique value or
incompatible current resource state\
B. For every server exception\
C. For missing authentication\
D. For a successful delete

**Answer: A.** The response should explain the conflict in a stable,
client-safe format.

**18. Which status code is commonly used for a request that violates
semantic validation rules, when the API adopts that convention?**

A. 202\
B. 304\
C. 422\
D. 502

**Answer: C --- 422 Unprocessable Content.** APIs may instead use 400
for validation errors; consistency and documented contracts matter.

**19. What does 429 Too Many Requests communicate?**

A. A resource was deleted\
B. A rate limit was exceeded\
C. Authentication succeeded\
D. A gateway failed

**Answer: B.** A `Retry-After` header can communicate when the client
may retry.

**20. Which code indicates that a server acting as a gateway/proxy
received an invalid response from an upstream server?**

A. 400\
B. 401\
C. 502\
D. 204

**Answer: C --- 502 Bad Gateway.** 503 generally indicates temporary
unavailability; 504 indicates an upstream timeout.

#### Headers, caching, and content negotiation

**21. What does `Content-Type` describe?**

A. The format of the representation being sent in the message\
B. The user's role\
C. The response's cache lifetime only\
D. The server's hostname

**Answer: A.** For a JSON request, it is commonly `application/json`.

**22. What does the `Accept` request header express?**

A. The body format the client is sending\
B. Response media types the client can accept\
C. The user's password\
D. The server's TLS version

**Answer: B.** Content negotiation may use `Accept` to select a response
representation.

**23. Which header commonly carries a bearer access token?**

A. `Location`\
B. `Authorization`\
C. `ETag`\
D. `Accept-Language`

**Answer: B.** Example: `Authorization: Bearer <access-token>`.

**24. Which header can identify the URI of a newly created resource?**

A. `Location`\
B. `Content-Length`\
C. `Origin`\
D. `Vary`

**Answer: A.** It is commonly returned with 201 Created.

**25. What does `Cache-Control: no-store` mean?**

A. The response must be stored indefinitely\
B. Caches should not store the request/response\
C. The response is public\
D. The client must retry

**Answer: B.** It is particularly important for sensitive responses;
apply cache policy deliberately.

**26. What is an ETag commonly used for?**

A. Authentication\
B. Representation validation and conditional requests\
C. Compressing JSON\
D. Routing to a controller

**Answer: B.** ETags can support cache revalidation and optimistic
concurrency.

**27. Which request header is commonly used with an ETag for conditional
retrieval?**

A. `If-None-Match`\
B. `Set-Cookie`\
C. `Server`\
D. `Allow`

**Answer: A.** A matching validator can allow the server to return 304.

**28. Which header can prevent a cached representation from being reused
when a selected request header differs?**

A. `Vary`\
B. `Location`\
C. `Authorization`\
D. `Content-Length`

**Answer: A.** For example, `Vary: Accept-Encoding` distinguishes
compressed and uncompressed representations.

**29. What is the purpose of `WWW-Authenticate`?**

A. Advertise an authentication challenge associated with a 401 response\
B. Specify the JSON schema\
C. Return the user's role list\
D. Enable compression

**Answer: A.** The challenge depends on the authentication scheme.

**30. Which statement about custom correlation headers is best?**

A. Trust any client-provided ID without validation\
B. Use a documented, validated correlation mechanism and integrate it
with structured logging/tracing\
C. Put secrets in the correlation ID\
D. Use it as the only authorization check

**Answer: B.** W3C Trace Context (`traceparent`) is the standard
distributed-tracing mechanism; custom correlation IDs may supplement it.

#### ASP.NET Core request processing

**31. Which component accepts HTTP connections in a typical ASP.NET Core
deployment?**

A. Entity Framework Core\
B. Kestrel\
C. MediatR\
D. AutoMapper

**Answer: B --- Kestrel.** A reverse proxy or cloud gateway may sit in
front of it.

**32. What is middleware in ASP.NET Core?**

A. A database table\
B. A component in the HTTP pipeline that can inspect, modify,
short-circuit, or pass on a request\
C. A DTO mapper only\
D. A type of HTTP status code

**Answer: B.** Middleware can run logic before and after the next
component.

**33. What happens when middleware does not call
`await next(context)`?**

A. The pipeline always restarts\
B. Downstream middleware/endpoints are not executed\
C. The response is automatically 404\
D. Authentication is automatically skipped but the endpoint still runs

**Answer: B.** This is called short-circuiting the pipeline.

**34. Why does middleware order matter?**

A. It never matters\
B. Components depend on earlier components having established required
state, such as routing, authentication, or exception handling\
C. It only affects CSS\
D. It determines SQL table order

**Answer: B.** For example, authentication should run before
authorization.

**35. What does `return Ok(value)` from an MVC controller typically
create?**

A. `NotFoundResult`\
B. `OkObjectResult`\
C. `FileResult`\
D. `ChallengeResult`

**Answer: B.** MVC later executes the result and formats the object for
the response.

**36. What is the main benefit of `ActionResult<T>` over an untyped
`IActionResult` in many controller actions?**

A. It disables HTTP status codes\
B. It communicates the success payload type while still allowing
alternative action results\
C. It bypasses serialization\
D. It makes every response 200

**Answer: B.** It also improves API metadata and readability.

**37. What does model binding do?**

A. Converts request data from sources such as route, query, and body
into action parameters or models\
B. Automatically grants authorization\
C. Creates database indexes\
D. Compresses responses

**Answer: A.** Binding and validation are related but distinct steps.

**38. In an `[ApiController]` controller, what commonly happens when
model validation fails?**

A. The API always returns 200\
B. Automatic HTTP 400 validation response, unless behavior is
customized\
C. The process exits\
D. Authentication is disabled

**Answer: B.** The automatic behavior can be configured through API
behavior options.

**39. What is the default JSON formatter in modern ASP.NET Core Web
APIs?**

A. `BinaryFormatter`\
B. `System.Text.Json`-based JSON formatting\
C. XML only\
D. Java serialization

**Answer: B.** Newtonsoft.Json can be configured when its features are
required.

**40. Why should global exception handling generally be centralized?**

A. To expose full stack traces\
B. To produce consistent, safe error responses and centralized logging\
C. To eliminate all validation\
D. To turn every error into 404

**Answer: B.** Exception handling should not leak implementation details
to clients.

#### Authentication, security, and reliability

**41. What is the difference between authentication and authorization?**

A. Authentication identifies/validates the principal; authorization
determines permitted actions\
B. They are identical\
C. Authorization validates JSON syntax\
D. Authentication handles database migrations

**Answer: A.** ASP.NET Core commonly uses authentication middleware
followed by authorization middleware.

**42. What is the main risk of storing a long-lived bearer token in
browser-accessible local storage?**

A. It prevents routing\
B. JavaScript running through an XSS vulnerability may steal it\
C. It automatically expires in one second\
D. It disables HTTPS

**Answer: B.** Token storage decisions depend on the application threat
model; HttpOnly cookies introduce their own CSRF considerations.

**43. Why is HTTPS required for bearer-token APIs?**

A. It makes all requests idempotent\
B. It protects credentials and traffic in transit\
C. It prevents all authorization bugs\
D. It replaces token validation

**Answer: B.** HTTPS does not replace server-side authentication and
authorization checks.

**44. What should a production API avoid returning to an untrusted
client?**

A. Documented error codes\
B. A trace identifier\
C. Stack traces, secrets, and sensitive internal details\
D. A validation summary

**Answer: C.** Log diagnostic details securely on the server and return
a safe error contract.

**45. Which response header is commonly used to tell a client when it
may retry after throttling or temporary unavailability?**

A. `Retry-After`\
B. `Content-Disposition`\
C. `Origin`\
D. `ETag`

**Answer: A.** Clients should still use bounded retries and backoff.

**46. Why can automatic retries be dangerous for a non-idempotent
POST?**

A. POST requests cannot contain JSON\
B. The first attempt may have succeeded even if its response was lost,
so a retry could duplicate the operation\
C. POST always returns 404\
D. Retries bypass TLS

**Answer: B.** Use idempotency keys or a domain-specific deduplication
strategy when duplicate execution is unacceptable.

#### Testing and architecture

**47. What is the strongest general choice for testing the real ASP.NET
Core routing, middleware, serialization, and status-code contract?**

A. A controller unit test alone\
B. An integration test using `WebApplicationFactory` and `HttpClient`\
C. A SQL-only test\
D. A static code search

**Answer: B.** It exercises the in-process application pipeline, though
it does not automatically test the deployed network infrastructure.

**48. What is the purpose of a test authentication handler?**

A. Test production cryptography by accepting every token\
B. Supply a controlled test principal so authorization policies can be
tested without issuing real tokens\
C. Disable all authorization tests\
D. Replace the production authentication scheme in production

**Answer: B.** Register it only in the test host and explicitly set the
claims/roles needed for each test.

**49. Which test best verifies that a middleware adds a response header
even when the endpoint throws?**

A. Test only a successful controller method\
B. An integration test that triggers the exception and inspects the
final response\
C. A database migration test\
D. A compile-only check

**Answer: B.** Middleware order and whether the response has started
affect whether headers can still be changed.

**50. What is the difference between a unit test and an integration test
for an API?**

A. Unit tests exercise a focused unit in isolation; integration tests
verify interactions and contracts across components\
B. They are the same\
C. Integration tests cannot use HTTP\
D. Unit tests must connect to production

**Answer: A.** Use both: unit tests for focused logic and integration
tests for externally observable behavior.

------------------------------------------------------------------------

### 2. How ASP.NET Core Builds an HTTP Response

#### High-level flow

``` text
Client
  |
  v
Kestrel / hosting server
  |
  v
Middleware pipeline
  |-- Exception handling
  |-- HTTPS redirection / forwarded headers (as configured)
  |-- Routing
  |-- CORS (as configured)
  |-- Authentication
  |-- Authorization
  v
Endpoint execution
  |-- Minimal API handler or MVC controller
  |-- Binding and validation (MVC)
  |-- Application/business logic
  v
Result execution / output formatting
  |-- Select status code
  |-- Set response headers
  |-- Serialize response body
  v
Response sent to client
```

Exact ordering depends on application configuration. Reverse proxies and
gateways may perform additional work before the request reaches Kestrel.

#### Example controller

``` csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    [HttpGet("{id:int}")]
    [ProducesResponseType<OrderDto>(StatusCodes.Status200OK)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> GetById(
        int id,
        CancellationToken cancellationToken)
    {
        var order = await _service.GetByIdAsync(id, cancellationToken);

        if (order is null)
        {
            return NotFound();
        }

        return Ok(order);
    }
}
```

#### What happens at `return Ok(order)`?

1.  The action returns an `OkObjectResult`, containing the value.
2.  MVC executes the result after the action completes.
3.  The framework selects an output formatter based on the result,
    configured formatters, and content negotiation.
4.  The response status is set to 200.
5.  The selected formatter serializes the object, typically using
    `System.Text.Json`.
6.  Response headers such as `Content-Type` are set as appropriate.
7.  The server sends the response. Once the response has started,
    changing status or headers is generally no longer possible.

#### Middleware that observes the final status

``` csharp
public sealed class RequestMetricsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestMetricsMiddleware> _logger;

    public RequestMetricsMiddleware(
        RequestDelegate next,
        ILogger<RequestMetricsMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startedAt = Stopwatch.GetTimestamp();

        try
        {
            await _next(context);
        }
        finally
        {
            var elapsed = Stopwatch.GetElapsedTime(startedAt);

            _logger.LogInformation(
                "HTTP {Method} {Path} responded {StatusCode} in {ElapsedMs} ms",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                elapsed.TotalMilliseconds);
        }
    }
}
```

Register it early enough to observe the endpoint's outcome:

``` csharp
app.UseMiddleware<RequestMetricsMiddleware>();
```

Add `using System.Diagnostics;` for `Stopwatch`.

#### Important internals to explain in interviews

-   **`HttpContext`** represents the current HTTP exchange; do not
    retain it beyond the request lifetime.
-   **Middleware** wraps downstream execution and can short-circuit, add
    headers, or handle exceptions.
-   **Routing** selects an endpoint using the configured route patterns
    and HTTP metadata.
-   **Authentication** establishes the request principal;
    **authorization** evaluates access policies.
-   **Model binding and validation** map incoming data into
    parameters/models and validate them.
-   **Action results** represent the intended HTTP outcome; result
    execution writes status, headers, and content.
-   **Response start** is a key boundary: headers/status cannot normally
    be changed after the response has started.
-   **Streaming responses** may start sending data before the entire
    body has been produced; exception handling is more limited after
    that point.

------------------------------------------------------------------------

### 3. Enterprise API Standards Checklist

Use this checklist during architecture reviews, pull-request reviews,
and production readiness assessments.

#### API contract and resource design

-   [ ] Use resource-oriented, consistent routes, such as
    `/api/orders/{id}`.
-   [ ] Use HTTP methods according to their semantics.
-   [ ] Document query parameters, pagination, filtering, sorting, and
    limits.
-   [ ] Define API versioning and a deprecation policy.
-   [ ] Publish an OpenAPI specification and keep it aligned with
    runtime behavior.
-   [ ] Define request and response DTOs separately from persistence
    entities.
-   [ ] Avoid exposing database entities or internal implementation
    details directly.
-   [ ] Define concurrency behavior for updates, including ETags or
    version fields where appropriate.
-   [ ] Define idempotency behavior for operations that clients may
    safely retry.

#### Status codes and errors

-   [ ] Use status codes consistently: 200, 201, 202, 204, 400, 401,
    403, 404, 409, 429, and appropriate 5xx codes.
-   [ ] Return 201 for successful creation when appropriate; include
    `Location` where useful.
-   [ ] Never put an application failure inside a 200 response merely to
    avoid non-2xx status codes.
-   [ ] Use a consistent error format, preferably `ProblemDetails` /
    `ValidationProblemDetails`.
-   [ ] Include a safe trace identifier in error responses.
-   [ ] Never expose stack traces, secrets, SQL details, or internal
    hostnames in production errors.
-   [ ] Document which errors clients can retry and under what
    conditions.

#### Headers and caching

-   [ ] Validate `Content-Type` and support documented `Accept` values.
-   [ ] Apply explicit cache policies, especially to sensitive or
    user-specific data.
-   [ ] Use ETags or other concurrency validators where useful.
-   [ ] Define CORS origins, methods, and headers narrowly.
-   [ ] Propagate W3C trace context and use correlation identifiers
    where required.
-   [ ] Set security headers according to the deployment model.
-   [ ] Avoid trusting arbitrary client-supplied forwarding or identity
    headers.

#### Authentication and authorization

-   [ ] Require HTTPS in production.
-   [ ] Validate token issuer, audience, signature, lifetime, and
    required claims.
-   [ ] Apply authorization policies at the endpoint boundary.
-   [ ] Enforce object-level authorization, not just role checks.
-   [ ] Use least privilege for application identities and database
    access.
-   [ ] Protect secrets with an approved secret-management solution.
-   [ ] Consider CSRF protections when authentication relies on
    automatically sent cookies.
-   [ ] Rate-limit sensitive and abuse-prone endpoints.
-   [ ] Ensure logs do not contain bearer tokens, passwords, or
    sensitive personal data.

#### Validation and business rules

-   [ ] Validate input at the API boundary.
-   [ ] Enforce business invariants in the appropriate
    application/domain layer.
-   [ ] Enforce data integrity with database constraints where
    appropriate.
-   [ ] Set limits for request size, upload size, pagination size, and
    execution time.
-   [ ] Distinguish malformed input, business conflicts, and unexpected
    failures.
-   [ ] Avoid mass assignment by using explicit request DTOs and
    allowed-field mapping.

#### Resilience and performance

-   [ ] Use asynchronous I/O for I/O-bound operations and pass
    cancellation tokens through.
-   [ ] Configure timeouts for outbound HTTP and other remote calls.
-   [ ] Use bounded retries with backoff only for appropriate transient
    failures.
-   [ ] Avoid retrying non-idempotent operations unless deduplication is
    implemented.
-   [ ] Use circuit breakers and concurrency limits where justified.
-   [ ] Apply pagination and compression where useful.
-   [ ] Monitor latency percentiles, throughput, error rates, and
    dependency health.
-   [ ] Prevent unbounded parallelism and resource exhaustion.

#### Middleware and observability

-   [ ] Make middleware ordering intentional and documented.
-   [ ] Handle exceptions centrally and consistently.
-   [ ] Put request logging and metrics in the appropriate pipeline
    location.
-   [ ] Propagate trace context across service boundaries.
-   [ ] Use structured logs with route templates where possible; avoid
    logging sensitive bodies by default.
-   [ ] Add health checks that distinguish liveness from readiness where
    appropriate.
-   [ ] Ensure error handling still works when endpoints fail or
    cancellation occurs.

#### Testing and release quality

-   [ ] Unit-test business rules and isolated components.
-   [ ] Integration-test routing, serialization, middleware,
    authentication, authorization, and status codes.
-   [ ] Test invalid JSON, invalid models, missing resources, and
    unexpected exceptions.
-   [ ] Test CORS and security headers where applicable.
-   [ ] Test pagination limits, rate limits, and concurrency behavior.
-   [ ] Include negative authorization tests for cross-user and
    cross-tenant access.
-   [ ] Validate OpenAPI output and representative client contracts.
-   [ ] Run security, dependency, and performance checks in CI/CD.

------------------------------------------------------------------------

### 4. Testing Authentication, Headers, and Middleware

The following examples use **xUnit**, **ASP.NET Core integration
testing**, and **`Microsoft.AspNetCore.Mvc.Testing`**. Keep the test
host separate from production configuration.

#### 4.1 Packages

For a test project targeting the same .NET major version as the API, add
the matching `Microsoft.AspNetCore.Mvc.Testing` package and `xunit` /
test runner packages.

``` xml
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="YOUR_DOTNET_MATCHING_VERSION" />
  <PackageReference Include="xunit" Version="YOUR_XUNIT_VERSION" />
  <PackageReference Include="xunit.runner.visualstudio" Version="YOUR_RUNNER_VERSION" />
</ItemGroup>
```

Replace placeholders with versions appropriate to the project; keep
package versions aligned with the target framework and repository
standards.

#### 4.2 Integration test host

For a typical ASP.NET Core application with a `Program` entry point:

``` csharp
// In Program.cs, if the test project needs to reference the entry point:
public partial class Program { }
```

Test fixture:

``` csharp
using Microsoft.AspNetCore.Mvc.Testing;

public sealed class ApiFactory : WebApplicationFactory<Program>
{
}
```

Test:

``` csharp
using System.Net;
using Xunit;

public sealed class OrdersApiTests : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client;

    public OrdersApiTests(ApiFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task MissingResource_Returns404()
    {
        var response = await _client.GetAsync("/api/orders/999999");

        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

This exercises the in-process ASP.NET Core pipeline. It does not, by
itself, test a real reverse proxy, deployed TLS configuration, DNS, or
cloud gateway.

#### 4.3 Test unauthenticated access

Assume `/api/secure` requires authentication and the test host is
configured to use the intended authentication scheme.

``` csharp
[Fact]
public async Task SecureEndpoint_WithoutCredentials_Returns401()
{
    using var client = _factory.CreateClient();

    var response = await client.GetAsync("/api/secure");

    Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
}
```

If the endpoint uses a cookie scheme or a custom challenge policy,
verify the expected behavior for that scheme. A redirect to a login page
may be expected in some cookie-authenticated browser applications; APIs
usually prefer a 401 response.

#### 4.4 Test authorization with a controlled test scheme

A test authentication handler lets tests supply a principal without
creating real JWTs. Register it **only in the test host**.

``` csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;

public sealed class TestAuthHandler
    : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public const string Scheme = "Test";

    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder)
    {
    }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var role = Request.Headers["X-Test-Role"].FirstOrDefault();

        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, "test-user"),
            new(ClaimTypes.Name, "integration-test-user")
        };

        if (!string.IsNullOrWhiteSpace(role))
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var identity = new ClaimsIdentity(claims, Scheme);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

Configure the test host:

``` csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;

public sealed class AuthApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            services
                .AddAuthentication(options =>
                {
                    options.DefaultAuthenticateScheme = TestAuthHandler.Scheme;
                    options.DefaultChallengeScheme = TestAuthHandler.Scheme;
                })
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                    TestAuthHandler.Scheme,
                    _ => { });
        });
    }
}
```

If the application already configures a default scheme, ensure the test
scheme is selected for the endpoints under test. Some applications
require removing or replacing existing authentication registrations. Do
not use this test handler in production.

Example endpoint:

``` csharp
[Authorize(Roles = "Admin")]
[HttpGet("admin")]
public IActionResult AdminOnly() => Ok(new { message = "allowed" });
```

Tests:

``` csharp
[Fact]
public async Task AuthenticatedUserWithoutAdminRole_Returns403()
{
    using var client = _factory.CreateClient();
    client.DefaultRequestHeaders.Add("X-Test-Role", "User");

    var response = await client.GetAsync("/api/admin");

    Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
}

[Fact]
public async Task AdminRole_CanAccessAdminEndpoint()
{
    using var client = _factory.CreateClient();
    client.DefaultRequestHeaders.Add("X-Test-Role", "Admin");

    var response = await client.GetAsync("/api/admin");

    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
}
```

**Important:** This fake scheme validates authorization behavior, not
JWT signature, issuer, audience, expiry, or token parsing. Add separate
tests for the real token-validation configuration.

#### 4.5 Test response headers

Suppose the API promises to return a correlation header and JSON
content.

``` csharp
[Fact]
public async Task GetOrders_ReturnsCorrelationHeaderAndJson()
{
    using var client = _factory.CreateClient();

    var response = await client.GetAsync("/api/orders");

    Assert.True(response.Headers.Contains("X-Correlation-Id"));

    Assert.Equal(
        "application/json",
        response.Content.Headers.ContentType?.MediaType);
}
```

For headers added to the content rather than the response, inspect
`response.Content.Headers`. The correct collection depends on the
header.

#### 4.6 Example correlation middleware

``` csharp
public sealed class CorrelationIdMiddleware
{
    private const string HeaderName = "X-Correlation-Id";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var incoming = context.Request.Headers[HeaderName].FirstOrDefault();

        // Production code should validate incoming IDs and define a length/format policy.
        var correlationId = IsAcceptable(incoming)
            ? incoming!
            : Guid.NewGuid().ToString("N");

        context.TraceIdentifier = correlationId;
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[HeaderName] = correlationId;
            return Task.CompletedTask;
        });

        await _next(context);
    }

    private static bool IsAcceptable(string? value) =>
        !string.IsNullOrWhiteSpace(value) &&
        value.Length <= 100 &&
        value.All(c => char.IsAsciiLetterOrDigit(c) || c is '-' or '_');
}
```

Register early in the pipeline:

``` csharp
app.UseMiddleware<CorrelationIdMiddleware>();
```

`OnStarting` allows the header to be added immediately before response
headers are sent. It cannot change headers after the response has
already started.

#### 4.7 Test middleware on successful and failed requests

``` csharp
[Fact]
public async Task CorrelationMiddleware_ReturnsAnId()
{
    using var client = _factory.CreateClient();

    var response = await client.GetAsync("/api/orders");

    Assert.True(response.Headers.TryGetValues(
        "X-Correlation-Id", out var values));

    Assert.False(string.IsNullOrWhiteSpace(values!.Single()));
}
```

Test that an acceptable incoming ID is propagated:

``` csharp
[Fact]
public async Task CorrelationMiddleware_PropagatesAcceptableIncomingId()
{
    using var client = _factory.CreateClient();
    client.DefaultRequestHeaders.Add("X-Correlation-Id", "test-trace-123");

    var response = await client.GetAsync("/api/orders");

    Assert.Equal(
        "test-trace-123",
        response.Headers.GetValues("X-Correlation-Id").Single());
}
```

Also test an endpoint that throws and verify the error handler returns
the expected status and that the correlation header remains present. The
exception handler and correlation middleware must be ordered so the
correlation middleware can observe the final response. If the response
has already started, the server may not be able to replace it with a
standard error response.

#### 4.8 Test ProblemDetails for errors

Configure ProblemDetails/exception handling in the application, then
assert the contract rather than depending on fragile full-body text
matches.

``` csharp
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc;

[Fact]
public async Task MissingOrder_ReturnsNotFoundProblemDetails()
{
    using var client = _factory.CreateClient();

    var response = await client.GetAsync("/api/orders/999999");

    Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);

    var problem = await response.Content.ReadFromJsonAsync<ProblemDetails>();

    Assert.NotNull(problem);
    Assert.Equal(StatusCodes.Status404NotFound, problem.Status);
}
```

A 404 endpoint does not automatically return a ProblemDetails body in
every configuration. Configure the API's error contract explicitly and
adapt the test to the contract the application promises.

#### 4.9 Unit-test a controller result

Unit tests should isolate controller behavior and mock dependencies.
They do not execute routing, middleware, model binding, or
serialization.

``` csharp
[Fact]
public async Task GetById_WhenOrderDoesNotExist_ReturnsNotFound()
{
    var service = new Mock<IOrderService>();
    service
        .Setup(x => x.GetByIdAsync(42, It.IsAny<CancellationToken>()))
        .ReturnsAsync((OrderDto?)null);

    var controller = new OrdersController(service.Object);

    var result = await controller.GetById(42, CancellationToken.None);

    Assert.IsType<NotFoundResult>(result.Result);
}
```

This example assumes Moq, an `IOrderService` interface, and an
`OrdersController` whose action returns `Task<ActionResult<OrderDto>>`.
Adjust types and signatures to match the application.

#### 4.10 What to test in CI

-   [ ] Unauthenticated request → expected 401/challenge behavior.
-   [ ] Authenticated principal without required permission → 403 or
    documented concealment behavior.
-   [ ] Principal with permission → successful response.
-   [ ] Invalid token → rejected by the real authentication
    configuration.
-   [ ] Invalid request body → documented 400/422 response.
-   [ ] Missing resource → documented 404 response.
-   [ ] State conflict → documented 409 response.
-   [ ] Successful create → 201 and expected `Location`, when
    applicable.
-   [ ] Successful no-body operation → 204 with no response body.
-   [ ] Correlation/trace headers appear on success and handled errors.
-   [ ] CORS preflight behavior matches configured origins, methods, and
    headers.
-   [ ] Rate limiting returns 429 and the documented retry metadata,
    where applicable.
-   [ ] Unexpected exception → safe 5xx response without stack-trace
    disclosure.
-   [ ] Cross-tenant and cross-user access is denied.
-   [ ] Middleware ordering is covered by observable integration tests.

------------------------------------------------------------------------

### 5. Senior-Level Scenario Questions

Use these as verbal interview prompts. A strong answer should explain
the decision, trade-offs, HTTP contract, security implications, and how
to test it.

#### Scenario 1: A client retries a POST after a timeout

**Question:** The client did not receive a response, but the server may
have created the order. How do you prevent duplicate orders?

**Expected discussion:** - A timeout does not prove the server failed to
process the request. - POST is not inherently idempotent. - Use an
idempotency key or domain-level deduplication with durable storage and
suitable uniqueness constraints. - Define the key's scope, retention,
replay response, and concurrent-request behavior. - Integration-test
repeated and concurrent submissions.

#### Scenario 2: An API returns 200 for every error

**Question:** Why is this a contract problem?

**Expected discussion:** - HTTP clients, gateways, monitoring, caching,
and retry policies rely on status semantics. - Use meaningful status
codes and a stable ProblemDetails error format. - Keep machine-readable
error codes separate from human-readable detail. - Avoid changing
established contracts without versioning or a migration plan.

#### Scenario 3: A user has a valid token but can access another tenant's data

**Question:** Authentication works. What is missing?

**Expected discussion:** - Authentication does not grant access to every
resource. - Enforce tenant and object-level authorization at trusted
server-side boundaries. - Do not trust a tenant ID supplied only by the
request body or query string. - Test cross-tenant access with negative
integration tests. - Audit logs should help investigations without
recording sensitive tokens or unnecessary personal data.

#### Scenario 4: A response header disappears on exceptions

**Question:** How would you diagnose it?

**Expected discussion:** - Review middleware order and exception
handling. - Check whether the header is set before or after
`await next`. - Consider `Response.OnStarting` and whether the response
has already started. - Reproduce with an integration test that forces an
endpoint exception. - Ensure the error handler does not accidentally
overwrite the header.

#### Scenario 5: An endpoint is slow despite fast database queries

**Question:** What would you investigate?

**Expected discussion:** - End-to-end distributed traces and dependency
timings. - Serialization cost and payload size. - Downstream HTTP calls,
connection pooling, thread-pool starvation, locks, and queueing. -
Pagination, compression, caching, and asynchronous I/O. - Compare
latency percentiles, not just average duration.

#### Scenario 6: Should validation failures be 400 or 422?

**Expected discussion:** - 400 is commonly used for malformed requests
and is also the default for automatic `[ApiController]` model-validation
failures. - 422 can distinguish syntactically valid content that fails
semantic validation. - Both conventions can work if documented and
consistently implemented. - Consider client compatibility and the
existing API contract before changing behavior.

#### Scenario 7: How do you test authorization without real tokens?

**Expected discussion:** - Use a test-only authentication scheme with
controlled claims and roles to exercise authorization policies. -
Separately test production JWT validation for signature, issuer,
audience, and lifetime. - Never allow the fake handler into production
configuration. - Test missing credentials, insufficient permission, and
allowed access.

------------------------------------------------------------------------

### Final interview recap

-   **Methods** express intent; safe and idempotent are different
    properties.
-   **Status codes** communicate outcomes to clients and infrastructure.
-   **Headers** carry metadata for authentication, negotiation, caching,
    tracing, and security.
-   **Middleware** composes cross-cutting behavior around endpoint
    execution; ordering matters.
-   **Unit tests** isolate logic; **integration tests** verify the
    observable HTTP contract.
-   **Architecture-level answers** include failure modes, security
    boundaries, operational impact, and test strategy.