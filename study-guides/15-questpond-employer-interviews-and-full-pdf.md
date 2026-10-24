# Questpond Employer-Specific Interview Notes and Full Question Bank

This reference preserves the employer-specific notes and the complete text extracted from `RealTimeInterviewQuestions.pdf`. Topic-focused Questpond question sets are also included in their matching study guides. Employer reports are retained here in full because many contain cross-topic questions, process details, and interview context that do not belong to just one technical section.

## Employer-specific notes

### Contents

- [GlobalLogic](#interview-source-14-globallogic-interview-questionsmd)
- [Accenture — first round](#interview-source-20-accenture-first-roundmd)
- [Capgemini — Azure lead](#interview-source-20-capgemini-azure-leadmd)
- [Capgemini](#interview-source-20-capgeminimd)
- [Carelon](#interview-source-20-carelonmd)
- [Cognizant](#interview-source-20-cognizantmd)
- [Deloitte](#interview-source-20-deloittemd)
- [EPAM](#interview-source-20-epammd)
- [GlobalData](#interview-source-20-globaldatamd)
- [LTIMindtree](#interview-source-20-lti-mindtreemd)
- [Mphasis](#interview-source-20-mphasismd)
- [Persistent — architect](#interview-source-20-persistent-architectmd)
- [Persistent — Azure/.NET](#interview-source-20-persistent-azure-dotnetmd)
- [Verisk](#interview-source-20-veriskmd)
- [Last-letter alphabet coding question](#interview-source-30-fn-lastletteralphabetmd)
- [Infosys](#interview-source-30-infosysmd)
- [TCS](#interview-source-30-tcsmd)
- [Virtusa](#interview-source-30-virtusamd)
- [Wipro](#interview-source-30-wipromd)
- [Wolters](#interview-source-30-woltersmd)

---

## Interview source: 14-globallogic-interview-questions.md

### GlobalLogic interview questions

Answers are concise interview-style summaries. Source pages refer to the PDF's 1-based pages.

#### 1. What are .NET components—CLR and CTS? *(PDF p.24)*

- **CLR (Common Language Runtime):** executes managed code, provides JIT compilation, garbage collection, exception handling, threading services and runtime type safety.
- **CTS (Common Type System):** defines the types and rules that .NET languages share, enabling compatible type use across languages.
- **BCL (Base Class Library):** reusable APIs for collections, I/O, networking, database access and other common tasks.
- **CLS (Common Language Specification):** a subset of CTS rules that language/library authors follow to improve cross-language interoperability.
- **JIT:** compiles IL into native machine code at runtime (with runtime optimizations).
- **Garbage collector:** reclaims managed heap objects that are no longer reachable; unmanaged resources still need deterministic disposal, usually `IDisposable`.

#### 2. What is type safety? *(PDF p.24)*

Type safety means values and operations are constrained by their types, preventing invalid operations such as treating an unrelated value as a valid object. .NET's type system and runtime enforce many checks; unsafe code and explicit casts can still require care.

#### 3. Which .NET 6 features have you used? *(PDF p.25)*

Examples include C# 10 global using directives and file-scoped namespaces, Hot Reload, minimal hosting, and supported Azure Functions workloads. State only the features actually used. .NET 6 is now out of support; use a supported .NET release for new production work.

#### 4. What is garbage collection and how does it work? *(PDF p.25)*

The CLR tracks managed objects and reclaims unreachable objects on the managed heap. Generational collection focuses on short-lived objects; finalization is nondeterministic and should not replace `Dispose` for timely release of unmanaged resources.

#### 5. Difference between an abstract class and an interface *(PDF p.25)*

An abstract class can hold shared state/implementation and instance constructors; a class can inherit only one base class. An interface defines a contract that multiple types can implement; a class can implement multiple interfaces. Modern C# interfaces can also contain default implementations, but they do not have instance constructors.

#### 6. Why use NuGet packages? *(PDF p.25)*

NuGet distributes reusable .NET libraries and tools, resolves dependencies, and supports package versioning. Review package trust, licensing, vulnerability advisories and compatibility; pin/manage versions appropriately.

#### 7. Difference between .NET Core and .NET Framework *(PDF p.25)*

.NET Framework is the older Windows-only implementation. .NET Core was the cross-platform, open-source successor; since .NET 5, the unified platform is named **.NET** (for example, .NET 8). Choose based on support, APIs, deployment and platform requirements.

#### 8. Explain SOLID principles *(PDF p.25)*

- **S — Single Responsibility:** a type has one cohesive reason to change.
- **O — Open/Closed:** extend behavior without repeatedly modifying stable code.
- **L — Liskov Substitution:** subtypes can stand in for base types without breaking expected behavior.
- **I — Interface Segregation:** prefer focused interfaces over broad contracts clients do not need.
- **D — Dependency Inversion:** high-level policy depends on abstractions, not low-level details.

#### 9. What is a tuple in C#? *(PDF p.25)*

A tuple groups a fixed number of values, often for local results or returning multiple values. `ValueTuple` supports named elements, e.g. `(int Min, int Max)`. For long-lived/domain data, a named record or class may communicate intent better.

#### 10. What are generic delegates and their types? *(PDF p.25)*

Generic delegates accept type parameters. Common built-ins are `Func<T..., TResult>` (returns a value), `Action<T...>` (returns `void`), and `Predicate<T>` (takes a value and returns `bool`). They are commonly used with callbacks, LINQ and lambdas.

#### 11. Why use `async`/`await` with `Task`? *(PDF p.26)*

They let asynchronous operations—especially I/O—be expressed sequentially without blocking the calling thread while waiting. In server applications this can improve thread utilization and scalability. `async` does not automatically make CPU-bound work faster; avoid `.Result`/`.Wait()` in asynchronous request paths.

#### 12. How do you create custom middleware? *(PDF p.26)*

Create a component that accepts a `RequestDelegate`, implements `InvokeAsync(HttpContext)`, and calls the next delegate when processing should continue. Register it in the correct pipeline order:

```csharp
public sealed class ExampleMiddleware
{
    private readonly RequestDelegate _next;
    public ExampleMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Before the next component, if needed.
        await _next(context);
        // After the next component, if needed.
    }
}
// In Program.cs: app.UseMiddleware<ExampleMiddleware>();
```

#### 13. How are microservices different from monolithic applications? *(PDF p.26)*

A monolith is deployed as one application unit; microservices split capabilities into independently deployable services. Independent scaling/releases and fault isolation can help, but distributed communication, data consistency, observability and operations become more complex.

#### 14. What are microservice architecture design patterns? *(PDF pp.26–27)*

Common patterns include API Gateway, service discovery, circuit breaker, retry with backoff, Saga (choreography **or** orchestration), database per service, event-driven messaging, CQRS, Strangler Fig migration and Sidecar. Select patterns to meet specific needs; they add complexity. The source repeats “choreography” under Saga types; the usual counterpart is orchestration.

#### 15. How do you implement fault tolerance in a microservice system? *(PDF p.27)*

Use timeouts, bounded retries with backoff/jitter, circuit breakers, bulkheads, health checks, redundancy and graceful degradation. Ensure consumers are idempotent, isolate failures, and monitor/alert. Retries must not amplify overload.

#### 16. What is the circuit breaker pattern? *(PDF p.27)*

It stops calls to a dependency after failures exceed a threshold, preventing repeated work and cascading failure. Typical states are **closed** (calls flow), **open** (calls fail fast), and **half-open** (limited probes test recovery).

#### 17. What is CQRS? *(PDF p.27)*

Command Query Responsibility Segregation separates the model for operations that change state (commands) from the model for reading state (queries). They may share storage or use separate read models; separate stores introduce consistency and synchronization trade-offs.

#### 18. What is service discovery in microservice architecture? *(PDF p.27)*

Service discovery locates healthy service instances dynamically as instances scale or move. A registry and client-side lookup or a discovery-aware proxy/load balancer can provide endpoints, avoiding hard-coded instance addresses. The PDF's answer beneath this question repeats its CQRS answer; this is the correct concept instead.

#### Continued on page 28

1. **Q (p. 28): What are the different ways of communication between microservices?**  
   **A:** Synchronous request/response (for example, HTTP/REST or gRPC) gives an immediate result; asynchronous messaging or events (queues/pub-sub) decouples producers and consumers. Choose based on latency, coupling, reliability, and throughput needs.
2. **Q (p. 28): What is the role of an API gateway in microservices?**  
   **A:** It provides a client-facing entry point that routes requests and can centralize authentication, throttling, TLS, and response aggregation. Keep business logic in the services.
3. **Q (p. 28): What is content negotiation?**  
   **A:** The client indicates acceptable representations, commonly with `Accept`; the server chooses a supported format and returns its media type. ASP.NET Core uses configured formatters.
4. **Q (p. 28): What are the ACID properties in an RDBMS?**  
   **A:** Atomicity (all-or-nothing), consistency (constraints remain valid), isolation (concurrent transactions do not interfere improperly), and durability (committed data survives failure).
5. **Q (p. 28): How do you measure database serviceability/availability?**  
   **A:** Track availability and SLOs, uptime, error rate, latency, connection saturation, replication lag, and recovery time against defined targets; monitor them continuously.
6. **Q (p. 28): What is replication?**  
   **A:** Maintaining copies of data on other database nodes. It can improve read scale and availability, but introduces consistency, lag, and failover considerations.
7. **Q (p. 28): How do you identify slow-running queries and optimize them?**  
   **A:** Find high-duration/high-read queries from Query Store or telemetry, inspect actual execution plans and statistics, then tune predicates, joins, indexes, and returned columns; measure the change.
8. **Q (p. 28): What is indexing and how does it improve performance?**  
   **A:** An index is an auxiliary ordered access structure that can avoid scanning every row. It speeds selected reads but consumes storage and adds write/maintenance cost.
9. **Q (p. 28): What are triggers and their types?**  
   **A:** Triggers run automatically on specified database events. SQL Server has DML triggers (AFTER and INSTEAD OF), DDL triggers, and logon triggers; use them sparingly because they hide side effects.
10. **Q (p. 28): What is the difference between COMMIT, ROLLBACK, and SAVEPOINT?**  
    **A:** `COMMIT` makes a transaction's changes durable; `ROLLBACK` undoes uncommitted changes; a savepoint marks a point to which part of a transaction can be rolled back.
11. **Q (p. 28): What is the difference between `@@IDENTITY` and `SCOPE_IDENTITY()`?**  
    **A:** `@@IDENTITY` returns the last identity generated in the session, including by a trigger. `SCOPE_IDENTITY()` restricts the result to the current scope and is generally safer after an insert.
12. **Q (p. 28): What are materialized views?**  
    **A:** They persist the result of a query for faster reads. In SQL Server, indexed views materialize results and require specific definitions and maintenance.
13. **Q (p. 28): What is the difference between RANK, DENSE_RANK, and “Entile” (likely NTILE)?**  
    **A:** `RANK` leaves gaps after ties; `DENSE_RANK` does not; `NTILE(n)` divides ordered rows into up to *n* approximately equal buckets. “Entile” appears to be a source typo for `NTILE`.
14. **Q (p. 28): What are the different types of NoSQL databases?**  
    **A:** Key-value, document, wide-column, and graph databases are common categories; some systems are multi-model.
15. **Q (p. 28): Why choose NoSQL over SQL?**  
    **A:** Choose it when its data model, flexible schema, horizontal scaling, or access patterns fit better. Relational databases remain preferable when relational constraints and rich transactions are central.
---

## Interview source: 20-accenture-first-round.md

### Accenture — 1st Round

Prompts are in the PDF order. The source labels this group “Accenture (1st Round).”

#### OOPS (p. 32)

- **Q: What are OOP concepts?**  
  **A:** Encapsulation, abstraction, inheritance, and polymorphism.
- **Q: Explain inheritance and multiple inheritance.**  
  **A:** Inheritance reuses/extends a base contract or implementation. C# classes have single class inheritance but can implement multiple interfaces, avoiding many multiple-inheritance ambiguities.
- **Q: What are types of polymorphism?**  
  **A:** Compile-time polymorphism includes overload resolution; runtime polymorphism uses overriding/virtual dispatch through a base or interface reference.
- **Q: How can the same method be called from multiple interfaces?**  
  **A:** Implement a shared public method when signatures/semantics match, or use explicit interface implementations (`((IFoo)obj).Method()` and `((IBar)obj).Method()`) when behavior differs.

#### C# (p. 32)

- **Q: Swap two values without a third variable.**  
  **A:** For integers: `a ^= b; b ^= a; a ^= b;`. In production, a temporary variable or tuple deconstruction (`(a,b)=(b,a)`) is clearer and avoids arithmetic overflow.
- **Q: What are delegates?**  
  **A:** Type-safe references to methods; used for callbacks and events. Built-ins include `Action`, `Func`, and `Predicate`.
- **Q: What is asynchronous programming?**  
  **A:** Allowing operations to complete without blocking the caller, especially I/O. In .NET use `Task` with `async`/`await`.
- **Q: What is multithreading?**  
  **A:** Executing work on multiple threads concurrently. Manage synchronization and shared state; async I/O is not inherently multithreading.

#### MVC / API (pp. 32–33)

- **Q: What is a DTO?**  
  **A:** A Data Transfer Object defines the data shape crossing a boundary, decoupling the API contract from domain or database entities.
- **Q: What is `ViewBag`?**  
  **A:** A dynamic, short-lived wrapper for `ViewData` used to pass values from a controller to a view; a strongly typed view model is safer for substantial data.
- **Q: Explain the MVC lifecycle.**  
  **A:** The framework routes the request, runs filters/binding/validation, invokes the action, executes the result/view, and writes the response; middleware surrounds this pipeline.
- **Q: What are action return types?**  
  **A:** MVC can return `IActionResult`/`ActionResult<T>` for varied HTTP responses, concrete results, or a model/view; APIs often use `ActionResult<T>`.
- **Q: How do you secure an API?**  
  **A:** Use TLS, validated authentication, policy/role authorization, input validation, least privilege, rate limits, safe errors, secret management, and monitoring.

#### SQL (p. 33)

- **Q: How do you design a database?**  
  **A:** Model entities and relationships, define keys/constraints, normalize to avoid anomalies, then choose indexes and any denormalization based on measured access patterns.
- **Q: What are constraints?**  
  **A:** Rules enforced by the database, including `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, and `NOT NULL`.
- **Q: What is a stored procedure?**  
  **A:** A named database routine that accepts parameters and executes SQL logic; use typed parameters and least-privilege permissions.
- **Q: What is an index and what is a clustered index?**  
  **A:** An index speeds selected access paths. A clustered index defines the row storage order/access structure; a table has at most one clustered index.

#### Other (p. 33)

- **Q: What are microservices?**  
  **A:** Independently deployable services organized around business capabilities; they improve independent scaling but bring distributed-system complexity.
- **Q: How do you host an application?**  
  **A:** Select a host (for example, IIS, App Service, container platform) based on runtime, availability, scale, networking, and operations; configure health checks, logging, and deployment.
- **Q: What are attributes?**  
  **A:** Metadata attached to code elements and consumed through reflection/framework conventions, such as routing or validation attributes.
---

## Interview source: 20-capgemini-azure-lead.md

### Capgemini Azure Lead interview questions

The source heading is “Capgemini Azure Lead” (p. 38).

- **Q: How do you handle state management in Azure Functions?**  
  **A:** Keep function instances stateless; persist state in an appropriate durable store or use Durable Functions for orchestrated state. Do not rely on instance memory across invocations.
- **Q: How do you deploy APIM in multiple regions with budget constraints?**  
  **A:** Confirm latency/availability requirements and APIM tier capabilities, then compare regional units, gateways, and traffic routing costs. Use a primary region if requirements permit; monitor failover and avoid overprovisioning.
- **Q: How do you handle deduplication in Service Bus?**  
  **A:** Enable duplicate detection with a stable `MessageId` within its configured window, and retain idempotency in consumers because detection is bounded and does not replace safe processing.
- **Q: How do you ensure message reliability in Service Bus?**  
  **A:** Use durable entities, peek-lock processing, idempotent handlers, bounded retries, dead-letter inspection, duplicate detection as needed, and monitoring; design producer transaction/outbox behavior when necessary.
- **Q: Backend service is inside a VNet/private endpoint but APIM APIs are public. How do you ensure connectivity?**  
  **A:** Place APIM in a supported VNet integration mode/tier, configure private DNS and routing/firewall rules, and restrict backend access to APIM's private path. Verify product-tier and regional capability requirements.
- **Q: Can an App Service Plan for Azure Functions always provide pre-warmed instances?**  
  **A:** It depends on the hosting plan and scale settings. Premium plans support pre-warmed instances; configure minimum/prewarmed capacity and validate cold-start and cost behavior. Consumption plans do not guarantee an always-warm instance.
- **Q: How do partition keys work in Cosmos DB?**  
  **A:** A partition key distributes items and defines logical partitions; choose a high-cardinality, evenly distributed key aligned with common queries and transactional boundaries to avoid hot partitions.
- **Q: What are replay tools?**  
  **A:** Tools/processes that re-send captured requests or messages to reproduce and diagnose behavior. Use sanitized data, controlled environments, idempotency, and avoid replaying production side effects unintentionally.
---

## Interview source: 20-capgemini.md

### Capgemini interview questions

The PDF uses the headings “CapGemini” and “Capgemini Azure Lead”; these are preserved separately in `20-capgemini-azure-lead.md`.

#### CapGemini — 5 years' experience (p. 37)

- **Q: Explain D in SOLID with a written example.**  
  **A:** Dependency Inversion: high-level policy depends on abstractions, not concrete infrastructure. Example: `OrderService(IOrderRepository repository)` depends on an interface; a SQL implementation is injected at composition.
- **Q: Describe a design pattern you implemented (source example: Singleton).**  
  **A:** Explain the actual problem and trade-off. A Singleton centralizes one instance; in .NET, DI singleton registration is usually safer and clearer, while mutable state and test coupling are risks.
- **Q: How do you pass data between controllers?**  
  **A:** Prefer a shared application service or persisted state for durable/business data. `TempData` can carry small one-time redirect data; avoid controller-to-controller coupling.
- **Q: What is AutoMapper?**  
  **A:** A library that maps between object types using configured conventions/profiles. Keep important transformations explicit and test mappings.
- **Q: What is `Bundles.config`?**  
  **A:** In older ASP.NET MVC, it configured CSS/JavaScript bundling and minification. Modern ASP.NET Core commonly uses frontend build tooling or static asset pipelines instead.
- **Q: How do you implement caching?**  
  **A:** Choose a cache layer and key/TTL/invalidation policy, avoid caching user-sensitive data incorrectly, handle stampedes, and measure hit rate and staleness.
- **Q: Difference between List and Dictionary?**  
  **A:** `List<T>` is ordered by index and allows duplicates; `Dictionary<TKey,TValue>` provides key-based lookup with unique keys.
- **Q: How do you implement JWT security?**  
  **A:** Use a trusted issuer and secure signing keys, validate issuer/audience/signature/expiry, apply authorization policies, use HTTPS, and never store secrets in token claims.
- **Q: What are API return types?**  
  **A:** `ActionResult<T>` or `IActionResult` supports typed payloads and status results; return clear status codes and DTOs.
- **Q: How do you handle application-level exceptions?**  
  **A:** Central middleware catches unexpected errors, logs diagnostics with a trace ID, and returns safe Problem Details; explicitly map expected domain failures.
- **Q: SQL query for 2nd highest salary.**  
  **A:** `SELECT salary FROM (SELECT salary,DENSE_RANK() OVER(ORDER BY salary DESC) r FROM Employee) x WHERE r=2;` This returns the second distinct salary.
- **Q: Delete one of the duplicate records in SQL.**  
  **A:** Identify duplicates with `ROW_NUMBER() OVER (PARTITION BY business_key ORDER BY id)` and delete rows where `rn > 1`, inside a reviewed transaction. Define which row to retain.
- **Q: How can you optimize query performance?**  
  **A:** Use actual plans and Query Store, improve indexes/statistics and predicates, reduce data returned, and validate with realistic measurements.
- **Q: How do you create a directory in C#?**  
  **A:** `Directory.CreateDirectory(path);` It creates missing parent directories and succeeds if the directory already exists; validate/authorize the path.
- **Q: Explain Angular interpolation binding.**  
  **A:** `{{ expression }}` renders a component expression as text in a template; Angular escapes text output rather than treating it as HTML.
- **Q: Explain pure and impure pipes.**  
  **A:** A pure pipe runs when its primitive input or object reference changes; an impure pipe runs frequently during change detection and can be expensive.
- **Q: Give duplicates from a list using LINQ.**  
  **A:** `var duplicates = values.GroupBy(x => x).Where(g => g.Count() > 1).Select(g => g.Key);`
- **Q: Explain SQL joins.**  
  **A:** Joins combine rows by a condition; inner joins keep matches, outer joins also retain unmatched rows from selected sides, and cross join forms combinations.
- **Q (p. 37): What is the output of `SELECT * FROM Employees WHERE 1=0`?**  
  **A:** No rows, because the predicate is always false; the result still has the selected columns.
- **Q: Abstract class vs interface?**  
  **A:** An abstract class can hold state and shared implementation but only one base class can be inherited; a class can implement multiple interfaces that define contracts (and may include default members).
---

## Interview source: 20-carelon.md

### Carelon interview questions

- **Q (p. 31): What current .NET version is used in your project?**  
  **A:** State the project's actual target framework and runtime/deployment version; distinguish it from the newest available release.
- **Q (p. 32): What are the latest .NET features?**  
  **A:** Name features from the exact .NET/C# release used in the project and explain a practical benefit; confirm versions against official release notes rather than claiming features from another version.
- **Q (p. 32): Performance difference between .NET and .NET Framework?**  
  **A:** Modern .NET often offers newer runtime, JIT, GC, and cross-platform improvements, but results depend on workload and migration details; benchmark the application.
- **Q (p. 32): `ref` types vs value types?**  
  **A:** Value types hold values and copy on assignment; reference types hold references to objects. Both may be stored in different memory areas depending on context; value/reference semantics matter more than stack/heap shorthand.
- **Q (p. 32): Explain JWT working and expiry.**  
  **A:** A JWT has a signed header and claims payload; the API validates signing key, issuer, audience, and `exp`. Expiry is checked by the validator; tokens are not encrypted by default.
- **Q (p. 32): How can multiple views show the same data?**  
  **A:** Share a view model/data service and render it in separate views or partials; avoid coupling views to persistence entities.
- **Q (p. 32): Explain AJAX/jQuery.**  
  **A:** AJAX sends asynchronous HTTP requests from the browser without a full-page reload; jQuery offers helpers such as `$.ajax`, though modern `fetch` is a native alternative.
- **Q (p. 32): How do you handle errors?**  
  **A:** Validate at boundaries, handle expected failures explicitly, and use centralized exception handling to log details and return consistent safe errors.
- **Q (p. 32): How do you display dropdown data efficiently?**  
  **A:** Query only IDs and display labels, filter/cache small reference sets, and use search or paging for large option sets.
- **Q (p. 32): Write a letter-count program.**  
  **A:** In C#: `var counts = text.Where(char.IsLetter).GroupBy(char.ToLowerInvariant).ToDictionary(g => g.Key, g => g.Count());` State whether comparison is case-insensitive.
- **Q (p. 32): Write a LINQ join.**  
  **A:** `var result = from a in first join b in second on a.Id equals b.FirstId select new { a, b };`
---

## Interview source: 20-cognizant.md

### Cognizant interview questions

#### Project & Process (p. 33)

- **Q: What were your roles and responsibilities?**  
  **A:** Summarize real ownership, decisions, delivery, collaboration, and measurable outcomes; distinguish personal contributions from team work.
- **Q: How do you plan tasks?**  
  **A:** Break work into testable deliverables, identify dependencies/risks, estimate with the team, and track progress against acceptance criteria.
- **Q: Explain Agile.**  
  **A:** Agile delivers in short feedback cycles with prioritized work, collaboration, inspection, and adaptation; Scrum is one framework, not a synonym for all Agile.
- **Q: How do you estimate work?**  
  **A:** Clarify scope and acceptance criteria, decompose the work, identify uncertainty/dependencies, estimate collaboratively, and revise as evidence changes.
- **Q: How do you interact with clients?**  
  **A:** Clarify goals and constraints, communicate risks early in plain language, confirm decisions, and give regular progress updates.
- **Q: How do you help junior developers?**  
  **A:** Pair and explain reasoning, provide actionable review feedback, offer progressively independent tasks, and create a safe path for questions.

#### .NET / API (pp. 33–34)

- **Q: Explain DI and its lifetimes.**  
  **A:** Register abstractions and inject dependencies; transient is per resolution, scoped per request/scope, singleton per container. Avoid lifetime capture errors.
- **Q: Explain async/await.**  
  **A:** Await task-based operations without blocking threads during I/O; propagate cancellation and errors and avoid `.Result`/`.Wait()`.
- **Q: Authentication vs authorization?**  
  **A:** Authentication establishes identity; authorization checks permission for an action/resource.
- **Q: What is API versioning?**  
  **A:** Managing contract evolution through explicit versions (URL, header, or media type), compatibility policy, documentation, and planned deprecation.
- **Q: What is rate limiting?**  
  **A:** Restricting request volume per identity/key/window to protect service capacity and fairness; return `429` and optionally `Retry-After`.
- **Q: How can you validate requests before the controller?**  
  **A:** Use model binding/validation, endpoint filters or middleware for cross-cutting checks, and authorization policies; retain domain validation in the application layer.
- **Q: How do you handle exceptions?**  
  **A:** Handle expected failures deliberately and centrally convert unexpected exceptions to a safe Problem Details response while logging the full diagnostic context.

#### SQL / EF (p. 34)

- **Q: How do you prevent SQL injection?**  
  **A:** Parameterize every value, use ORM parameter binding, validate inputs, and grant least privilege. Avoid concatenating untrusted input into SQL.
- **Q: Explain stored procedures.**  
  **A:** Named server-side routines with parameters and controlled permissions, useful for reusable database logic; they do not automatically guarantee better performance.
- **Q: What is a CTE?**  
  **A:** A named query expression for one statement, useful for clarity and recursive queries.
- **Q: What are SQL functions?**  
  **A:** Routines returning a scalar or table-valued result; SQL Server functions have restrictions compared with procedures and should be assessed for query cost.
- **Q: What are join types?**  
  **A:** `INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, and `CROSS`; choose based on which unmatched rows must remain.
- **Q: How do you load parent-child data in EF?**  
  **A:** Use `Include`/`ThenInclude` for eager loading, explicit loading when requested, or project to a DTO. Avoid N+1 queries and excessive joins.
- **Q: How do you handle a scoped DB connection expiring?**  
  **A:** Do not cache connections beyond their unit of work; open/use/dispose through the provider pool, configure timeouts/retries for transient faults, and retry only safe operations.

#### Azure / DevOps (p. 34)

- **Q: Which Azure services have you used?**  
  **A:** Name only services actually used and explain purpose, integration, security, and operational responsibilities.
- **Q: Explain CI/CD.**  
  **A:** CI builds and tests each change; CD promotes a versioned artifact through environments with approvals, secure configuration, deployment checks, and rollback.
- **Q: What is OAuth?**  
  **A:** An authorization framework for delegated access using access tokens; OpenID Connect adds an authentication layer.
---

## Interview source: 20-deloitte.md

### Deloitte interview questions

#### Concepts (p. 34)

- **Q: What are SOLID principles?**  
  **A:** Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion.
- **Q: What is Dependency Injection?**  
  **A:** Supplying a class's dependencies from outside rather than constructing concrete collaborators internally; register and resolve them through a container or composition root.
- **Q: Explain the JWT mechanism.**  
  **A:** The issuer signs a token containing claims; the API validates signature, issuer, audience, and expiry, then uses claims for authorization. A signed JWT is not encrypted by default.
- **Q: Authentication vs authorization?**  
  **A:** Authentication verifies identity; authorization decides what an authenticated identity may do.
- **Q: What is asynchronous programming?**  
  **A:** Task-based operations that yield during waits rather than blocking a thread, especially useful for I/O.
- **Q: How do you handle exceptions?**  
  **A:** Catch errors where recovery is possible, preserve stack traces, clean up deterministically, and centralize safe response mapping and logging.
- **Q: What is CQRS?**  
  **A:** Command Query Responsibility Segregation separates updates from reads, allowing distinct models/paths when complexity and scale justify it.

#### Coding (p. 34)

- **Q: Implement a Singleton.**  
  **A:** Prefer DI registration (`AddSingleton<TService, TImpl>()`). If a class singleton is required: `public sealed class S { private static readonly Lazy<S> Instance = new(() => new S()); private S() {} public static S Current => Instance.Value; }`
- **Q: Explain delegates: `Func` vs `Action`.**  
  **A:** `Func<... , TResult>` represents a delegate returning `TResult`; `Action<...>` returns `void`.
- **Q: Write an anagram program.**  
  **A:** Normalize case/whitespace, then compare character counts: `string.Join("", a.Where(char.IsLetterOrDigit).Select(char.ToLowerInvariant).OrderBy(c => c)) == string.Join("", b.Where(char.IsLetterOrDigit).Select(char.ToLowerInvariant).OrderBy(c => c))`. Apply the same normalization to both.
- **Q: Find the 3rd largest number.**  
  **A:** For the third distinct value, use `numbers.Distinct().OrderByDescending(x => x).Skip(2).FirstOrDefault()` (guard for fewer than three); for O(n), maintain three distinct maxima in one pass.

#### Testing (p. 34–35)

- **Q: What unit-testing framework have you used?**  
  **A:** Name the framework actually used—such as xUnit, NUnit, or MSTest—and describe assertions, fixtures, and test organization.
- **Q: What is NUnit?**  
  **A:** A .NET unit-testing framework with test attributes, assertions, setup/teardown, and parameterized tests.
- **Q: Mocking vs stubbing?**  
  **A:** A stub supplies predetermined responses; a mock additionally verifies expected interactions. Use test doubles only where they isolate a meaningful boundary.
---

## Interview source: 20-epam.md

### EPAM interview questions

This combines the two EPAM-labeled groups in the source while retaining their separate headings. Answers are concise; citations indicate source pages.

#### EPAM: coding and concepts (p. 31)

- **Q: Count repeated characters and output alphabet + number.**  
  **A:** Group characters and count them, then format each pair, e.g. `text.GroupBy(c => c).Select(g => $"{g.Key}{g.Count()}")`; specify case and whitespace rules.
- **Q: LINQ group and sum.**  
  **A:** `var totals = items.GroupBy(x => x.Key).Select(g => new { Key=g.Key, Total=g.Sum(x => x.Amount) });`
- **Q: Count letters in a string.**  
  **A:** `text.Where(char.IsLetter).GroupBy(char.ToLowerInvariant).ToDictionary(g => g.Key, g => g.Count());` Decide whether counts are case-sensitive.
- **Q: LINQ join two tables.**  
  **A:** `var q = from a in left join b in right on a.Id equals b.LeftId select new { a, b };`
- **Q: Explain parallel programming.**  
  **A:** Run independent CPU-bound work concurrently using appropriate parallel APIs, bound degree of parallelism, avoid shared mutable state, and measure overhead. Use async I/O rather than parallel threads for I/O waits.
- **Q: Explain DI lifetimes, specifically Scoped.**  
  **A:** Transient creates per resolution, Singleton once per container, and Scoped once per request/scope. A scoped service must not be captured by a singleton.
- **Q: Authentication and authorization?**  
  **A:** Authentication validates who a caller is; authorization evaluates permissions/claims for a resource or operation.
- **Q: Logging?**  
  **A:** Emit structured, levelled events through `ILogger`, include correlation context, and avoid secrets or sensitive data.
- **Q: Middleware?**  
  **A:** An ordered request/response pipeline component that can call the next delegate or terminate the pipeline.
- **Q: REST?**  
  **A:** A resource-oriented, stateless API style using HTTP method/status semantics and representations.
- **Q: Routing?**  
  **A:** Selecting the endpoint from request path, method, and route constraints, conventionally or via attributes.
- **Q: Async/await?**  
  **A:** Awaiting a `Task` yields control while asynchronous work is pending, improving I/O scalability without blocking a thread.

#### EPAM interview assessment and Azure follow-up (pp. 41–47)

The source includes assessment notes as well as interview prompts. The notes below are retained as answer-ready topics; no claim is made that they were questions verbatim.

- **Assessment topic (p. 41): .NET Core vs .NET Framework—runtime, hosting, and performance.**  
  **A:** Modern .NET is cross-platform and modular with flexible hosting; .NET Framework is Windows-only and often required by legacy dependencies. Benchmark representative workloads and check API compatibility before migration.
- **Assessment topic (p. 41): Dependency Injection concepts and practical patterns.**  
  **A:** Depend on abstractions, register services with appropriate lifetimes, and inject them through constructors; avoid service-locator usage and lifetime mismatches.
- **Assessment topic (p. 41): Application logging with `ILogger` and recommended practices.**  
  **A:** Use structured templates, appropriate levels, scopes/correlation IDs, centralized sinks, and redaction of sensitive values.
- **Assessment topic (p. 41): ASP.NET Core middleware pipeline and short-circuiting.**  
  **A:** Middleware runs in registration order on the request and unwinds in reverse for the response. It may short-circuit by not invoking `next`; this is distinct from a circuit breaker.
- **Assessment topic (p. 41): JWT authentication, token validation, and middleware configuration.**  
  **A:** Register bearer authentication and validate signature, issuer, audience, lifetime, and signing keys; place authentication before authorization.
- **Assessment topic (p. 41): Model binding and model validation.**  
  **A:** Binding maps route/query/header/body data into parameters or models; validation checks constraints and returns clear client errors before business processing.
- **Assessment topic (p. 41): `IDisposable` and resource management.**  
  **A:** Dispose owned unmanaged or disposable resources deterministically, typically with `using`; do not rely on finalization for ordinary cleanup.
- **Assessment topic (p. 41): `async`/`await`, Task vs Thread.**  
  **A:** `Task` represents an operation; a thread is an execution resource. Await I/O asynchronously; reserve parallel execution for independent CPU work.
- **Assessment topic (p. 42): `Parallel.ForEach` and multithreading.**  
  **A:** Use it for independent CPU-bound iterations, control concurrency, and ensure thread-safe shared state; it is not the default for asynchronous I/O.
- **Assessment topic (p. 42): `ref` vs `out`.**  
  **A:** `ref` requires a prior value; `out` requires assignment by the callee before return.
- **Assessment topic (p. 42): Structured try/catch/finally.**  
  **A:** Catch only errors that can be handled meaningfully, preserve stack traces with `throw;`, and place cleanup in `finally` or `using`.
- **Assessment topic (p. 42): Aggregate sales by `ProductId` using LINQ.**  
  **A:** `var totals = sales.GroupBy(x => x.ProductId).Select(g => new { ProductId=g.Key, Total=g.Sum(x => x.Amount) });`
- **Assessment topic (p. 42): LINQ filtering, grouping, aggregation, projection.**  
  **A:** Chain `Where`, `GroupBy`, aggregate operators such as `Sum`, and `Select`; defer execution until enumeration where applicable.
- **Assessment topic (p. 42): DRY, clean code, and maintainable design.**  
  **A:** Keep responsibilities cohesive, use clear names, remove meaningful duplication, and avoid abstractions that obscure simple behavior.
- **Assessment topic (p. 42): Technical documentation, coding standards, and diagrams.**  
  **A:** Document decisions, APIs, setup, and operational behavior; use concise diagrams (for example, sequence/deployment diagrams) and keep them versioned with the code.
- **Assessment topic (p. 42): Decorator pattern.**  
  **A:** A decorator wraps an object implementing the same interface to add behavior without changing the wrapped type; useful for cross-cutting concerns such as metrics.
- **Assessment topic (p. 42): Singleton and Factory patterns.**  
  **A:** A singleton has one shared instance and requires thread-safe construction; a factory centralizes creation and can select implementations behind an abstraction.
- **Assessment topic (p. 42): EF Code First and migrations.**  
  **A:** Define the model in code, generate reviewed schema migrations, and apply them through a controlled deployment process.
- **Assessment topic (p. 42): EF tracking vs no-tracking queries.**  
  **A:** Tracking enables change detection but costs memory/CPU; `AsNoTracking` is generally preferable for read-only projections/entities.
- **Assessment topic (p. 42): EF architecture and real-time usage scenarios.**  
  **A:** EF Core maps entities and LINQ expressions to provider queries; use it for productive domain persistence, while profiling query translation and selecting lower-level access for specialized hot paths.
- **Assessment topic (p. 42): SQL `WHERE` vs `HAVING` and execution order.**  
  **A:** `WHERE` filters rows before grouping; `HAVING` filters aggregates/groups. Conceptually, `FROM`/`JOIN`, `WHERE`, `GROUP BY`, `HAVING`, `SELECT`, then `ORDER BY`.
- **Assessment topic (p. 43): SQL NULL comparisons and function behavior.**  
  **A:** `NULL` means unknown; test it with `IS NULL`, not `= NULL`. Functions may propagate NULL; use `COALESCE` or another explicit policy.
- **Assessment topic (p. 43): Structure an UPDATE with CASE.**  
  **A:** `UPDATE T SET Value = CASE WHEN condition THEN new_value ELSE Value END WHERE ...;` Limit rows with a deliberate predicate and test the affected set.
- **Assessment topic (p. 43): AAA, mocking, unit vs integration tests.**  
  **A:** Arrange dependencies/data, Act, Assert. Mock external collaborators to isolate a unit; integration tests exercise real boundaries such as a database or HTTP host.
- **Assessment topic (p. 43): Azure Functions, triggers, bindings.**  
  **A:** A trigger starts a function; bindings declaratively connect inputs/outputs to services. Keep execution idempotent and account for retries and time limits.
- **Assessment topic (p. 43): Azure Key Vault and Application Insights.**  
  **A:** Use managed identity and least privilege for secrets; instrument applications with telemetry and correlation in Application Insights.
- **Q (p. 43): What are the fundamentals of Azure Application Gateway?**  
  **A:** It is a regional Layer 7 web traffic load balancer supporting routing, TLS termination, and WAF capabilities.
- **Q (p. 43): Function Apps vs App Services?**  
  **A:** App Service hosts web apps/APIs continuously; Functions hosts event-triggered functions with function-oriented scaling and bindings. Functions can run on App Service plans too.
- **Q (p. 43): Explain Authentication Approach—AD.**  
  **A:** Register the app with Microsoft Entra ID, request/validate OAuth 2.0 or OpenID Connect tokens, and authorize using app roles/scopes and claims.
- **Q (p. 43): What is JWT?**  
  **A:** A compact token format with header, claims payload, and signature. A signed JWT is not confidential unless separately encrypted.
- **Q (p. 43): OAuth 2.0 vs OpenID Connect?**  
  **A:** OAuth 2.0 delegates authorization to access resources; OpenID Connect adds authentication and an ID token on top of OAuth 2.0.
- **Q (p. 43): Explain client-credentials flow for API authentication.**  
  **A:** A confidential service authenticates to the identity provider and requests an app-only access token for a target scope; the API validates it and enforces application permissions.
- **Q (p. 43): What are system-assigned, managed, and user identities?**  
  **A:** A system-assigned managed identity is tied to one Azure resource's lifecycle; a user-assigned managed identity is a reusable separately managed resource. Both avoid application-held credentials. “User identity” is an end-user identity, not a managed identity.
- **Q (p. 43): How does a .NET Web API integrate with SQL Server and Blob Storage in Azure?**  
  **A:** Use managed identity where supported, grant narrowly scoped database/storage permissions, configure endpoints through settings, and access them via EF/ADO.NET and the Azure Storage SDK.
- **Q (p. 43): What are ways to run a recurring daily background job?**  
  **A:** Use a timer-triggered Azure Function, WebJob, or a scheduler/orchestrator such as Logic Apps; design for retries, idempotency, monitoring, and time-zone/DST rules.
- **Q (p. 43): What are WebJobs?**  
  **A:** Background programs/scripts hosted alongside an App Service, triggered continuously, on demand, or by a schedule, depending on configuration and plan.
- **Q (p. 44): Can two WebJobs run in the same App Service Plan?**  
  **A:** Yes. They can share a plan (and potentially an App Service host); isolate them when resource contention, security, deployment, or scaling requirements warrant it.
- **Q (p. 44): How do you decide whether a separate App Service Plan is needed?**  
  **A:** Compare CPU/memory and scale needs, availability boundaries, deployment ownership, networking, cost, and noisy-neighbor risk; a plan is the compute/scaling boundary.
- **Q (p. 44): How do you track issues in Functions or App Services?**  
  **A:** Correlate Application Insights traces, exceptions, dependencies, and metrics with Azure Monitor alerts and logs; use operation IDs and investigate around the incident window.
- **Q (p. 44): Have you used Azure Monitor to track Application Insights issues?**  
  **A:** Use Azure Monitor workbooks, alerts, metrics, and Log Analytics queries on Application Insights telemetry; define actionable thresholds and ownership.
- **Q (p. 44): Explain Publisher/Subscriber and how to trace message flow and endpoints.**  
  **A:** Publishers emit events to a broker; subscribers receive them independently. Track correlation/message IDs, subscription identity, endpoint/configuration, delivery attempts, and telemetry across producer and consumer.
- **Q (p. 44): Explain a scenario using Service Bus or Storage Queues.**  
  **A:** Use a queue to buffer work between an API and a background worker, smoothing bursts and enabling retries. Choose Service Bus for richer broker features; use Storage Queues for simpler, cost-effective queueing.
- **Q (p. 44): One Service Bus topic has multiple subscribers; how do you trigger based on a field when topic parameters are the same?**  
  **A:** Use subscription filters (SQL or correlation filters) against message properties, or split subscriptions/topics by routing needs. Set the relevant property when publishing.
- **Q (p. 44): How should code process a queue, and how do Service Bus queue and dead-letter concepts work?**  
  **A:** Receive with peek-lock, validate and process idempotently, then complete; abandon/defer or allow lock expiry for retry. After delivery limits or explicit rejection, inspect and remediate dead-letter messages.
- **Q (p. 44): In how many ways can Service Bus be triggered?**  
  **A:** Typical options include an Azure Functions Service Bus trigger, SDK receiver/processor hosted in an app, or a Logic Apps connector; subscriptions and queues are distinct entity choices.
- **Q (p. 44): How do you access Key Vault from .NET at class level?**  
  **A:** Prefer a managed-identity-backed `SecretClient` (`DefaultAzureCredential`) injected into a service; retrieve the named secret asynchronously and avoid logging its value.
- **Q (p. 44): Describe your AKS deployment: Helm, ingress, configuration, and CI/CD.**  
  **A:** Build and scan an image, push it to ACR, deploy versioned Helm values/manifests through CI/CD, configure secrets via secure integration, and expose services through ingress with health probes and rollback.
- **Q (p. 44): What is ACR used for?**  
  **A:** Azure Container Registry stores and distributes container images and related artifacts; deploy immutable, tagged images rather than mutable local builds.
- **Q (p. 44): Container vs pod?**  
  **A:** A container packages and runs a process; a Kubernetes pod is the smallest schedulable unit and can contain one or more co-located containers sharing networking and volumes.
- **Q (p. 45): What do Helm charts do and why use them?**  
  **A:** A chart templates Kubernetes manifests with configurable values, packaging and versioning a repeatable deployment.
- **Q (p. 45): What CLI tests a Helm chart locally—is it Docker or something else?**  
  **A:** Use `helm lint` and `helm template` to validate/render locally; `helm install --dry-run` can check a release. Docker builds images but does not validate a Helm chart.
- **Q (p. 45): Any experience creating a VNet or networking concepts?**  
  **A:** Describe an actual design accurately. Core concepts include address spaces/subnets, NSGs, routes, DNS, peering, gateways, private endpoints, and least-privilege network paths.
- **Q (p. 45): Explain DI types: Scoped, Singleton, Transient.**  
  **A:** Scoped is one instance per request/scope; Singleton per container lifetime; Transient per resolution. Avoid capturing shorter-lived services in longer-lived ones.
- **Q (p. 45): How do you handle CORS issues?**  
  **A:** Configure an explicit origin/method/header policy on the server, place CORS middleware correctly, and allow credentials only with explicit trusted origins.
- **Q (p. 45): One API endpoint is consumed by multiple applications; how do you manage it (APIM)?**  
  **A:** Publish/version the API in APIM, apply per-consumer products/subscriptions or identity-based policies, quotas, monitoring, and a controlled deprecation plan.
- **Q (p. 45): How can APIM validate each request without writing API code?**  
  **A:** Apply inbound policies such as JWT validation, schema/parameter checks, rate limits, and IP restrictions; keep authorization and business validation aligned with the backend.
- **Q (p. 45): Explain a middleware scenario.**  
  **A:** Correlation-ID middleware can read or create a request ID, add it to logging scope and response headers, then call the next delegate.
- **Q (p. 45): How does middleware check authentication and authorization before the API executes?**  
  **A:** Register authentication middleware before authorization and endpoint execution. Authentication builds the principal; authorization evaluates endpoint policy and can short-circuit with 401/403.
- **Q (p. 45): Difference between sync and async; give an example.**  
  **A:** Synchronous code waits while an operation completes; async I/O (such as `await httpClient.GetAsync`) releases the request thread while waiting and resumes when the result is available.
- **Q (p. 45): Can async be used without await?**  
  **A:** A method may be marked `async` without `await`, but this normally produces a warning and an unnecessary state machine. Return the `Task` directly when no awaiting/transformation is needed.
- **Q (p. 46): An API must call an external API, wait for its result, then queue work based on the response. What should you do?**  
  **A:** Await the external call with timeout/cancellation, validate the response, then publish a durable message containing the required data and correlation ID; make publishing/processing resilient and idempotent.
- **Q (p. 46): What is a concurrent list vs a normal list?**  
  **A:** `List<T>` is not thread-safe for concurrent mutation. Use `ConcurrentBag<T>`, `ConcurrentQueue<T>`, or a lock/concurrent design based on required ordering and operations.
- **Q (p. 46): Difference between `.Result` and `ConfigureAwait`?**  
  **A:** `.Result` synchronously blocks on a task and can cause deadlocks/starvation; `ConfigureAwait(false)` controls whether a continuation captures a synchronization context and does not itself block.
- **Q (p. 46): What are possible approaches to connect to SQL?**  
  **A:** Use ADO.NET/SqlClient directly, an ORM such as EF Core, or a micro-ORM such as Dapper; parameterize queries, manage connections safely, and choose by query/control needs.
- **Q (p. 46): How do you scale a database as the application scales?**  
  **A:** Tune queries/indexes, pool connections, scale compute/storage, add read replicas where appropriate, partition/shard when needed, and cache carefully. Database scaling is not automatic just because app replicas increase.
- **Q (p. 46): How do you ensure connection pooling?**  
  **A:** Reuse provider-managed pools, use short-lived connections disposed promptly, keep connection strings consistent, and configure pool limits/timeouts appropriate to load.
- **Q (p. 46): During load testing of pooling, what issues are common?**  
  **A:** Pool exhaustion, connection leaks, long-held transactions, timeouts, server connection limits, and latency spikes. Inspect active connections and ensure disposal and bounded concurrency.
- **Q (p. 46): Have you used LINQ?**  
  **A:** Explain practical use for filtering, projection, grouping, and aggregation; distinguish LINQ-to-Objects from provider-translated LINQ and inspect generated SQL where relevant.
- **Q (p. 46): `IEnumerable` vs `IQueryable`—difference and when to use?**  
  **A:** `IEnumerable` executes iteration in memory; `IQueryable` builds an expression for a provider such as EF to translate. Keep filtering/projection server-side until materialization.
- **Q (p. 46): What have you worked on in SQL Server?**  
  **A:** Tailor to real experience; discuss schema/query work, transactions, indexing, execution plans, performance measurement, and safe deployment rather than overstating expertise.
- **Q (p. 46): Explain scalar functions with a scenario.**  
  **A:** A scalar function returns one value per invocation, for example normalizing a code. In large queries, scalar functions can affect performance; prefer inline expressions or inline TVFs when appropriate.
- **Q (p. 47): How do you check why a query is taking time—execution plan or SQL Profiler?**  
  **A:** Start with Query Store and actual execution plans, IO/time statistics, and wait analysis. Profiler is legacy and can add overhead; use Extended Events for targeted tracing.
- **Q (p. 47): What is a CTE? Explain a scenario.**  
  **A:** A CTE names a query for one statement. Use it to make a multi-step query readable or to express recursion such as an organizational hierarchy.
- **Q (p. 47): What tools have you used for API unit testing—NUnit, xUnit, MSTest?**  
  **A:** Use a test framework such as xUnit, NUnit, or MSTest with assertions and mocks; test controller/service behavior and use integration tests for the HTTP/database boundary.
- **Q (p. 47): Any specific GitHub Copilot things to define?**  
  **A:** Explain how you use it for suggestions, tests, and exploration while reviewing generated code, checking correctness/security, and following organizational data policies.
- **Q (p. 47): Explain CQRS and Mediator design patterns.**  
  **A:** CQRS separates command/write models from query/read models; Mediator routes requests to handlers without direct caller-to-handler coupling. They can be combined but neither requires the other.

**Source artifact (p. 47):** “Web Page is Unavailable” appears immediately before the next company heading. It is not an interview prompt, so it has no answer.
---

## Interview source: 20-globaldata.md

### GlobalData interview questions

#### Project (p. 35)

- **Q: Which module did you work on?**  
  **A:** Describe the business purpose, your specific module ownership, interfaces, and an outcome using truthful project details.
- **Q: Which ORM did you use?**  
  **A:** State the actual ORM and explain how it mapped data, handled tracking/migrations, and when direct SQL was preferable.
- **Q: How did you query the database?**  
  **A:** Explain the actual data-access approach, parameterized query patterns, transaction boundaries, and how you measured query performance.

#### ASP.NET (p. 35)

- **Q: What is a constructor in an API controller?**  
  **A:** It initializes the controller's dependencies, typically received through constructor injection; keep it light and avoid business work.
- **Q: `app.Use` vs `app.Run`?**  
  **A:** `Use` adds middleware that can invoke the next delegate; `Run` adds terminal middleware and does not pass control onward.
- **Q: How do you fix a slow MVC page?**  
  **A:** Measure end-to-end and server/database timings, inspect query plans and N+1 behavior, reduce payload/render work, add justified caching/indexes, and remeasure.
- **Q: How do you validate before a controller?**  
  **A:** Use binding and automatic model validation, endpoint filters, or middleware for cross-cutting validation. Put business invariants in the application/domain layer.

#### SQL (p. 35)

- **Q: Get the 5th highest salary.**  
  **A:** For fifth distinct salary: `SELECT salary FROM (SELECT salary,DENSE_RANK() OVER(ORDER BY salary DESC) r FROM Employee) x WHERE r=5;`
- **Q: Find a department with zero employees.**  
  **A:** `SELECT d.Id FROM Department d LEFT JOIN Employee e ON e.DepartmentId=d.Id WHERE e.Id IS NULL;`
- **Q: What are joins?**  
  **A:** Operations combining rows by a predicate: inner and outer joins preserve different sets of matching/unmatched rows; cross join forms combinations.
- **Q: How do you optimize SQL?**  
  **A:** Inspect plans and workload metrics, update statistics, tune predicates/joins/indexes, project only needed data, and verify improvements.

#### Caching (p. 35)

- **Q: What are types of caching?**  
  **A:** Browser/client, CDN, reverse-proxy, process-local memory, and distributed caches. Select TTL/invalidation and consistency rules deliberately.
- **Q: Where is Redis data stored?**  
  **A:** In Redis server memory, optionally with configured persistence to disk; it is external to an application's process and can be shared across instances.
- **Q: How do you configure in-memory cache?**  
  **A:** Register `AddMemoryCache()` and inject `IMemoryCache`; set expiration/size limits and ensure cached data is safe for process-local scope.
---

## Interview source: 20-lti-mindtree.md

### LTI Mindtree interview questions

Each prompt is retained in source order; page citations refer to the PDF's printed page numbering (1-based).

#### .NET / C# (p. 29)

- **Q: Stack vs heap?**  
  **A:** The stack stores call frames and local values with deterministic lifetimes; the managed heap stores objects managed by the GC. “Value types live on the stack” is not universally true—location depends on context.
- **Q: What is the Garbage Collector and how does it work?**  
  **A:** The .NET GC reclaims unreachable managed objects, generally using generations 0–2 and compacting eligible memory. It does not replace deterministic disposal of unmanaged resources.
- **Q: If I don’t want to create an instance of a class, how can I do it without `static` or `abstract`?**  
  **A:** Give the class a private constructor; callers cannot instantiate it. This is appropriate only when construction should be controlled, such as a factory or singleton.
- **Q: How do you create Dependency Injection?**  
  **A:** Define an abstraction, implement it, register the mapping and lifetime with `IServiceCollection`, then request it through constructor injection.
- **Q: Where does an injected dependency reside?**  
  **A:** The DI container constructs/resolves it according to its registered lifetime. Its actual memory location is an implementation/runtime detail, not determined by injection.
- **Q: What is constructor chaining?**  
  **A:** One constructor invokes another constructor with `this(...)`, or a base constructor with `base(...)`, to reuse initialization logic.
- **Q: What is a partial class and what is it used for?**  
  **A:** `partial` lets a type declaration span files in the same compilation. It is useful for generated code plus hand-written code, but not as a substitute for good type boundaries.
- **Q: Can a partial class exist in different projects?**  
  **A:** No. Its parts must be compiled together into the same assembly and namespace/type identity.
- **Q: What are the types of `NullReferenceException`?**  
  **A:** It is one exception type, not a family of types. It occurs when dereferencing a null reference; identify which reference is null and fix the invalid state or validate it.
- **Q: Task vs Thread / multithreading?**  
  **A:** A `Thread` is an OS execution thread; a `Task` represents work or an asynchronous operation and is commonly scheduled on the thread pool. `async` I/O does not necessarily create a thread.
- **Q: `IEnumerable` vs `IQueryable` (including inheritance difference)?**  
  **A:** `IEnumerable<T>` iterates objects in-process; `IQueryable<T>` builds provider-translatable expression trees, often executed by a database. `IQueryable<T>` derives from `IEnumerable<T>`.
- **Q: Hashtable vs Dictionary / HashSet vs Dictionary?**  
  **A:** Prefer generic `Dictionary<TKey,TValue>` for typed key-value lookup; `Hashtable` is legacy and non-generic. `HashSet<T>` stores unique values without associated values; a dictionary maps unique keys to values.
- **Q: `ref` vs `out`?**  
  **A:** Both pass a variable by reference. `ref` must be initialized before the call; `out` need not be, but the callee must assign it before returning.
- **Q: What is `IDisposable` and where should it be used?**  
  **A:** It defines deterministic cleanup via `Dispose`, especially for resources such as streams, database connections, and handles. Use `using`/`await using`; implement it when owning disposable resources.
- **Q: .NET Framework vs .NET Core from a developer perspective?**  
  **A:** .NET Framework is the Windows-only legacy platform; modern .NET (formerly .NET Core) is cross-platform, open-source, and suited to new cloud and service development. Check API and hosting compatibility during migration.
- **Q: What are `async` and `await`?**  
  **A:** They compose asynchronous operations without blocking the caller while I/O is pending. Return `Task`/`Task<T>`, propagate cancellation, and await the operation.

#### Entity Framework / ADO.NET (p. 29)

- **Q: What is ADO.NET?**  
  **A:** .NET's lower-level data-access APIs for connections, commands, readers, and adapters, typically used with SQL providers.
- **Q: Can stored procedures be used with ADO.NET?**  
  **A:** Yes. Create a command with `CommandType.StoredProcedure`, set its name and typed parameters, then execute the appropriate reader/nonquery/scalar method.
- **Q: Difference between a SQL query and stored procedure in ADO.NET?**  
  **A:** A text command sends SQL text; a stored-procedure command invokes a database routine. Both can use parameters. Procedures can centralize reusable database logic and permissions; performance depends on the actual plan and workload.
- **Q: What are EF migration commands?**  
  **A:** Common CLI commands are `dotnet ef migrations add <Name>`, `dotnet ef database update`, and `dotnet ef migrations remove`; review generated migrations before applying them.
- **Q: What is `AsNoTracking`?**  
  **A:** It tells EF Core not to track returned entities for changes, reducing overhead for read-only queries. It is not appropriate if those entities will be updated through the same context.
- **Q: EF Code First vs Database First?**  
  **A:** Code First models and migrations drive schema evolution; Database First scaffolds a model from an existing schema. Select based on ownership and governance of the database.
- **Q: What is eager loading?**  
  **A:** Loading related data as part of the initial query, commonly with `Include`/`ThenInclude`, rather than loading it later. Use it intentionally to avoid excessive queries.

#### SQL (pp. 29–30)

- **Q: Views vs indexes?**  
  **A:** A view is a stored query definition; an index is an access structure. A normal view does not store its result; an indexed view can materialize it under SQL Server's constraints.
- **Q: What are index types?**  
  **A:** SQL Server supports clustered, nonclustered, unique, filtered, columnstore, and other specialized indexes. Choose by query pattern and write cost.
- **Q: What are triggers?**  
  **A:** Database routines invoked automatically by DML, DDL, or logon events. Keep behavior set-based and predictable; avoid hidden business logic where possible.
- **Q: Stored procedure vs function?**  
  **A:** Procedures can perform operations and return result sets/output parameters; functions return a value or table and are subject to invocation restrictions. Use each according to its purpose.
- **Q: What is a stored procedure?**  
  **A:** A named database routine that can accept parameters and run SQL logic, with permissions and reuse managed in the database.
- **Q: `WHERE` vs `HAVING`?**  
  **A:** `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation.
- **Q: What is a CTE?**  
  **A:** A named query expression introduced with `WITH`, available to the immediately following statement; useful for readable and recursive queries.
- **Q: Can SQL functions have multiple return types?**  
  **A:** A SQL Server function has a defined return type—scalar or table-valued. Use a table-valued function or a procedure/result contract when multiple values are needed.
- **Q: What are the types of joins?**  
  **A:** `INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, and `CROSS`; self-joins use a table more than once.
- **Q: Query: if commission is NULL, return 0.**  
  **A:** `SELECT COALESCE(commission, 0) AS commission FROM Sales;` (`ISNULL` is also available in SQL Server.)
- **Q: Get the 5th highest salary.**  
  **A:** For the fifth distinct salary: `SELECT salary FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS r FROM Employee) s WHERE r = 5;`. Use `ROW_NUMBER` instead if “fifth row” is intended.
- **Q: Find departments with zero employees.**  
  **A:** `SELECT d.DepartmentId FROM Department d LEFT JOIN Employee e ON e.DepartmentId=d.DepartmentId WHERE e.EmployeeId IS NULL;`
- **Q: How do you identify a missing index?**  
  **A:** Review execution plans, Query Store, and DMVs for expensive scans/lookup patterns; validate a candidate index against workload and write impact rather than blindly applying DMV suggestions.
- **Q: What are SQL optimization techniques?**  
  **A:** Measure first; inspect plans and statistics; use selective, sargable predicates, suitable indexes, narrow projections, and sound joins; avoid needless row-by-row work and remeasure.
- **Q: What keyword gets the last inserted record?**  
  **A:** Prefer `INSERT ... OUTPUT inserted.<column>`; in a separate statement, `SCOPE_IDENTITY()` safely returns the identity generated in the current scope.

#### ASP.NET Core / MVC / API (p. 30)

- **Q: Authentication vs authorization?**  
  **A:** Authentication establishes identity; authorization checks what that identity may do.
- **Q: Explain JWT working, expiry, and contents.**  
  **A:** A signed token carries claims in header/payload and a signature. The API validates signature, issuer, audience, and expiry; `exp` controls validity. A JWT is encoded, not encrypted by default.
- **Q: What is logging?**  
  **A:** Recording structured, levelled diagnostic events with context and correlation IDs. Avoid secrets and sensitive personal data.
- **Q: How do you handle exceptions?**  
  **A:** Handle expected errors locally; use centralized exception middleware to log details and return a safe, consistent problem response without exposing internals.
- **Q: What is middleware?**  
  **A:** A component in the HTTP pipeline that can inspect/modify requests and responses and invoke the next component, or short-circuit.
- **Q: What are REST principles?**  
  **A:** Resource-oriented URIs, stateless requests, uniform HTTP semantics, representations, and cacheability where appropriate.
- **Q: What is routing?**  
  **A:** Mapping an incoming URL and HTTP method to an endpoint/action, using conventional or attribute routes.
- **Q: What is model validation?**  
  **A:** Checking bound input against constraints such as Data Annotations or custom validators, and returning validation errors (typically 400 for APIs).
- **Q: What are HTTP methods?**  
  **A:** Common methods include GET (read), POST (create/process), PUT (replace), PATCH (partial update), and DELETE (remove); honor their semantics and idempotency.
- **Q: HTTP 200 vs 201?**  
  **A:** `200 OK` means success with a response; `201 Created` means a resource was created and should generally include its location.
- **Q: Can `[Post]` be used on a GET action?**  
  **A:** In ASP.NET Core use `[HttpPost]` to map POST. A GET endpoint should use `[HttpGet]`; do not overload method semantics.
- **Q: What are clean-code principles such as DRY?**  
  **A:** Keep code readable and cohesive, name clearly, minimize duplication without over-generalizing, and apply SOLID where it improves maintainability.
- **Q: How do you return an error message to a client?**  
  **A:** Return the correct status code and a stable, non-sensitive Problem Details body with an error code/message and trace identifier.
- **Q: Multiple views / same view for different controllers?**  
  **A:** Views can be shared in `Views/Shared`; otherwise MVC resolves controller-specific paths. Reuse a view model and shared view only when the UI contract is genuinely the same.
- **Q: How do you handle dropdown performance?**  
  **A:** Load only the required fields and filtered options, cache stable reference data when safe, and use remote search/paging for large lists.

#### Azure (p. 31)

- **Q: Azure Functions—how do you extract data from a request?**  
  **A:** Bind an HTTP request trigger, read route/query/header/body values using the request APIs, validate them, and return an appropriate status/result.
- **Q: How do you integrate Azure Key Vault with a .NET API?**  
  **A:** Assign a managed identity, grant least-privilege secret access, configure the Key Vault provider/SDK, and retrieve secrets without embedding credentials in code.
- **Q: How do you analyze an API issue from the cloud?**  
  **A:** Establish time, request/correlation ID, and impact; inspect health, logs, traces, metrics, dependencies, and recent deployments; isolate the failing layer, mitigate safely, then confirm recovery.

#### Architecture / Design (p. 31)

- **Q: Explain the Factory design pattern.**  
  **A:** A factory centralizes object creation and returns an abstraction, hiding concrete construction choices from callers.
- **Q: What are SOLID principles?**  
  **A:** Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion.
- **Q: Give an example of violating Open/Closed.**  
  **A:** A method with a growing `switch` on payment type must be edited for every new provider. Use a payment interface and register a new strategy instead, keeping existing flow closed to modification.
- **Q: Why cache, and how do memory cache and Redis differ?**  
  **A:** Caching reduces repeated work and latency. In-memory cache is process-local and lost on restart; Redis is shared/distributed and supports multiple instances, at added operational/network cost.
- **Q: What is microservices architecture?**  
  **A:** An application is split into independently deployable services organized around capabilities, communicating through APIs/events. It enables independent scaling but adds distributed-systems and operational complexity.
---

## Interview source: 20-mphasis.md

### Mphasis interview questions

The PDF spells the heading “Mhpasis.” This file normalizes the filename/name to Mphasis and preserves its two round subgroups.

#### 1st round (pp. 39–40)

- **Q: Introduce yourself; explain roles and responsibilities.**  
  **A:** Give a brief career summary, relevant skills, current scope, concrete ownership, and outcomes without overstating responsibility.
- **Q: Describe your project and technology stack.**  
  **A:** State the product purpose, architecture, languages/frameworks, database/cloud, and your contribution.
- **Q: What is polymorphism?**  
  **A:** A common abstraction can represent different implementations; runtime polymorphism commonly uses interface dispatch or virtual overrides.
- **Q: How do you call a base-class constructor?**  
  **A:** Use `: base(arguments)` in the derived constructor; it runs before the derived constructor body.
- **Q: Explain DI and its lifetimes.**  
  **A:** Inject abstractions; transient is per resolution, scoped per request/scope, singleton per container. Avoid retaining scoped services in singleton objects.
- **Q: `throw` vs `throw ex`?**  
  **A:** `throw;` rethrows while preserving the original stack trace; `throw ex;` resets the trace. Prefer `throw;` inside a catch.
- **Q: Explain the Garbage Collector.**  
  **A:** It reclaims unreachable managed objects, generally by generations, and may compact memory. Dispose unmanaged resources deterministically.
- **Q: What is caching and what types are there?**  
  **A:** Caching stores reusable results to reduce latency/work. Common types include in-process, distributed, browser, CDN, and database caches; define expiry/invalidation and consistency.
- **Q: What is middleware?**  
  **A:** A request/response pipeline component that can call next or terminate the pipeline.
- **Q: How do you create custom middleware?**  
  **A:** Implement `Invoke`/`InvokeAsync(HttpContext, RequestDelegate)` (or use `IMiddleware`), register it in the pipeline with `UseMiddleware<T>()`, and call `next` when continuing.
- **Q: `app.Use()` vs `app.Run()`?**  
  **A:** `Use` can call the next middleware; `Run` is terminal and does not receive a next delegate.
- **Q: What are query parameters?**  
  **A:** Key/value data after `?` in a URL, typically used for filtering, paging, or optional selection; do not put secrets there.
- **Q: How do you extract data from an HTTP request?**  
  **A:** Use route/query/header APIs or bind and deserialize the request body to a validated model; enforce size, type, and authorization limits.
- **Q: How do you do model validation (Data Annotations, Regex)?**  
  **A:** Add annotations/custom validators to input DTOs and validate after binding; use bounded, safe regex patterns and return structured validation errors.
- **Q: What is model binding?**  
  **A:** Mapping route, query, form, and body values to action parameters/models using configured binders and formatters.
- **Q: What is asynchronous programming?**  
  **A:** Awaiting nonblocking operations (especially I/O) so a thread can serve other work while the operation is pending.
- **Q: Task vs Thread?**  
  **A:** A Task represents work/completion; a Thread is an execution resource. Tasks may use thread-pool threads, while async I/O need not occupy one while waiting.
- **Q: What is multithreading?**  
  **A:** Running work on multiple threads concurrently; protect shared state and bound parallelism.
- **Q: What is `IDisposable` used for?**  
  **A:** Deterministic cleanup of owned resources; call via `using`/`await using`.
- **Q: Explain microservices architecture.**  
  **A:** Independently deployed services organized around capabilities and communicating over APIs/events; benefits include independent deployment/scaling but costs include distributed complexity.
- **Q: Which design patterns have you used in your project?**  
  **A:** Give a real example and why it helped (for example, Strategy to vary behavior behind an interface); discuss trade-offs rather than listing pattern names.
- **Q: What is CQRS?**  
  **A:** Separating command/write handling from query/read handling, potentially with different models; use only when read/write needs justify the complexity.
- **Q: What is API versioning?**  
  **A:** Exposing controlled contract versions and a compatibility/deprecation plan through a URL, header, or media type.
- **Q: A service keeps failing. How do you handle it?**  
  **A:** Detect with timeouts/health signals, use bounded retries with backoff for transient failures, circuit-break repeated failures, provide safe fallback/queueing, and alert/diagnose the root cause.
- **Q: How do services communicate?**  
  **A:** Synchronously via HTTP/gRPC or asynchronously via brokered messages/events; select based on coupling, latency, delivery guarantees, and consistency needs.
- **Q (p. 40): How do you call one service from another?**  
  **A:** Use a typed HTTP/gRPC client for synchronous calls or publish a message for decoupled work; set timeouts, propagate identity/correlation, and handle retries safely.

##### Coding (p. 40)

- **Q: Reverse the letters in each word without changing word order.**  
  **A:** Split while preserving separators, reverse characters within each word token, and join; for example, `Hello world` becomes `olleH dlrow`. Define punctuation/Unicode handling.
- **Q: Count occurrences of letters in a string.**  
  **A:** `var counts = text.Where(char.IsLetter).GroupBy(char.ToLowerInvariant).ToDictionary(g => g.Key,g => g.Count());` This is case-insensitive; remove normalization for case-sensitive counts.

##### SQL (p. 40)

- **Q: What are indexes and their types?**  
  **A:** Indexes speed selected access paths at storage/write cost. Common SQL Server types include clustered/nonclustered, unique, filtered, and columnstore.
- **Q: How many clustered indexes can a table have?**  
  **A:** At most one, because a table can have only one clustered row-ordering structure; a heap has none.
- **Q: CTE vs temporary table?**  
  **A:** A CTE is scoped to one statement and is a query expression; a temp table is materialized in `tempdb`, can be reused/indexed across statements, and has different statistics/storage behavior.
- **Q: What are window functions?**  
  **A:** Functions such as `ROW_NUMBER`, `RANK`, `LAG`, or `SUM() OVER(...)` compute values across related rows without collapsing them into one row per group.
- **Q: `RANK` vs `DENSE_RANK`?**  
  **A:** Both assign equal ranks to ties; `RANK` leaves gaps after ties, while `DENSE_RANK` does not.

#### 2nd round (pp. 40–41)

- **Q (p. 40): Introduce yourself and explain roles/responsibilities.**  
  **A:** Provide a concise role-focused summary of actual responsibilities, technical scope, collaboration, and outcomes.
- **Q (p. 40): Tell me about your current project, tech stack, and modules worked on.**  
  **A:** Explain the system's purpose, architecture and stack, the module you owned, interfaces, and impact.
- **Q (p. 40): Write a controller showing Repository Pattern using DI and Entity Framework.**  
  **A:** Define `IItemRepository`, inject it into the controller, and keep the EF `DbContext` in the repository/service: `[HttpGet] public async Task<ActionResult<IReadOnlyList<ItemDto>>> Get() => Ok(await repo.GetAllAsync());`. Keep query logic testable and avoid an unnecessary generic repository wrapper.
- **Q (p. 40): Which DI lifetimes did you use in your project?**  
  **A:** Typically DbContext/repositories are scoped per request, stateless lightweight services transient or scoped, and truly shared thread-safe services singleton. Explain actual registrations and lifetime constraints.
- **Q (p. 40): How would you write unit tests for this?**  
  **A:** Unit-test controller behavior with a substitute repository and assert result/status; separately test repository/query logic using an appropriate database integration test.
- **Q (p. 40): What unit-testing frameworks do you use?**  
  **A:** Name the actual framework (xUnit, NUnit, or MSTest) and any assertion/mock libraries used.
- **Q (p. 40): How have you used middleware/custom middleware in your project?**  
  **A:** Give a concrete cross-cutting example (exception mapping, correlation IDs), explain ordering, and state when it calls next or short-circuits.
- **Q (p. 41): How do you handle exceptions and return user-friendly messages to the front end?**  
  **A:** Map known errors to stable codes/statuses and safe messages; centralized middleware logs diagnostic details and returns Problem Details with trace ID, never stack traces.
- **Q (p. 41): How did you implement security in your project?**  
  **A:** Describe only actual controls: HTTPS, validated identity/tokens, authorization policies, input validation, secret management, least privilege, rate limits, and audit/monitoring.
- **Q (p. 41): Explain how JWT tokens work.**  
  **A:** The issuer signs header/claims; clients send the token as a bearer credential; APIs validate signature and standard claims before authorization. Signed JWT contents are readable unless encrypted.
- **Q (p. 41): Have you used CORS and how does it work? What requests does it block?**  
  **A:** CORS is a browser-enforced cross-origin policy signaled by response headers. A preflight `OPTIONS` check is used for certain non-simple requests; CORS does not block server-to-server traffic or replace authentication.
- **Q (p. 41): What deployment methods have you used in your project?**  
  **A:** Describe the actual pipeline and hosting: build/test, artifact or image creation, environment configuration, staged/rolling or slot deployment, health verification, and rollback strategy.
---

## Interview source: 20-persistent-architect.md

### Persistent — Architect interview questions

The source labels this section “Persistent - Architect interview questions asked to me.” Each numbered prompt below is from pp. 36–37.

1. **Q (p. 36): A client asks to modernize an application. What is your approach?**  
   **A:** Assess business goals, architecture, dependencies, security, data, operations, and constraints; define target outcomes and options; prioritize incremental migration, risks, cost, testing, and measurable milestones.
2. **Q (p. 36): Explain authentication flow in your Web application/Web API.**  
   **A:** A client authenticates with the identity provider, receives a token/session, sends credentials securely, and the API validates them before authorization policies evaluate claims. Log correlation, not secrets.
3. **Q (p. 36): Explain authentication generally (JWT/OAuth 2.0).**  
   **A:** OAuth 2.0 obtains delegated/app access tokens; OpenID Connect adds user authentication. The resource server validates token issuer, audience, signature, and lifetime; JWT is a token format, not an auth protocol.
4. **Q (p. 36): What are Azure deployment options for a Web API (App Service/AKS/Container Service)?**  
   **A:** App Service for managed web hosting, containers for packaging/portability, and AKS when Kubernetes orchestration/control is justified. Select by operational skill, scale, networking, and cost.
5. **Q (p. 36): Why choose App Service for a Web API?**  
   **A:** It provides managed hosting, deployment slots, TLS, autoscaling/monitoring integration, and lower operational overhead than Kubernetes for a conventional API.
6. **Q (p. 36): How do you design logging/tracing and observability across an application ecosystem?**  
   **A:** Use structured logs, distributed traces, metrics, consistent correlation/trace propagation, shared schemas, dashboards, alerts, and redaction; define SLOs and access controls.
7. **Q (p. 36): How do you access data in a Web API, and when choose Dapper, EF Core, or ADO.NET?**  
   **A:** EF Core for domain-oriented productivity and change tracking; Dapper for lightweight mapped SQL and control; ADO.NET for low-level control. Parameterize all queries and choose based on measured needs.
8. **Q (p. 36): How is Application Insights configured, and can multiple applications' telemetry be viewed together?**  
   **A:** Instrument each app with the SDK/agent and configure connection settings securely. Use a shared Log Analytics workspace/Application Insights workspace-based setup and Azure Monitor workbooks to correlate telemetry, respecting app-level access.
9. **Q (p. 36): How do you configure deployment pod counts/min/max for an App Service Web App/API? Can one config manage all apps?**  
   **A:** The prompt mixes Kubernetes pods with App Service instances. App Service scales out in instances, not pods; configure each app/plan's autoscale rules and centralize policy through IaC/templates where appropriate. For AKS, set replicas and HPA min/max per workload.
10. **Q (p. 36): How would you estimate/approach upgrading a legacy Angular app to the latest version?**  
    **A:** Inventory version/dependencies, review breaking-change migration guides, establish tests and build baselines, upgrade sequentially through supported versions, modernize deprecated APIs, and estimate in risk-sized increments.
11. **Q (p. 36): How do you conduct TDD for database objects/stored procedures?**  
    **A:** Write repeatable tests for inputs, outputs, edge cases, and side effects; run them against an isolated database/schema in CI, seed deterministic data, and version migrations with the procedure.
12. **Q (p. 36): What design patterns are used in Event-Driven Architecture?**  
    **A:** Common patterns include pub/sub, competing consumers, outbox, idempotent consumer, retry/dead-letter, saga, and event sourcing where justified.
13. **Q (p. 37): What is Key Vault used for in your project?**  
    **A:** Securely store and rotate secrets, keys, and certificates; grant applications least-privilege access through managed identities and audit retrieval.
---

## Interview source: 20-persistent-azure-dotnet.md

### Persistent Azure .NET role interview questions

Source heading: “Persistent Azure Dotnet Role” (pp. 38–39).

- **Q (p. 38): Explain middleware.**  
  **A:** Middleware is an ordered HTTP pipeline component that can process a request/response, invoke the next component, or short-circuit.
- **Q (p. 38): Explain exception handling.**  
  **A:** Use centralized exception middleware for consistent status/problem responses and structured diagnostics; catch locally only when recovery or translation is meaningful.
- **Q (p. 38): What performance optimization techniques do you use in a .NET API?**  
  **A:** Profile first; tune database calls, async I/O, allocations, serialization, caching, and payload size; use pagination and bounded concurrency, then validate under load.
- **Q (p. 38): Explain your project structure.**  
  **A:** Describe the actual separation of API, application/use cases, domain, infrastructure, and tests; explain dependency direction and where configuration/composition occurs.
- **Q (p. 38): How do you make an application more scalable and available?**  
  **A:** Remove single points of failure, use stateless instances and horizontal scale, health probes, resilient dependencies, queues/caching where justified, backups/DR, and SLO monitoring.
- **Q (p. 38): Did you take downtime during deployment? How much, and why?**  
  **A:** Answer from actual experience; explain the deployment strategy, measured downtime, dependency/schema constraints, and customer impact transparently.
- **Q (p. 38): How do you achieve zero downtime (not even 0.00001%)?**  
  **A:** Absolute zero downtime cannot be guaranteed; target a defined availability SLO using redundant instances, rolling/slot deployments, backward-compatible changes, health checks, and tested failover.
- **Q (p. 38): What is Azure Service Bus?**  
  **A:** A managed enterprise message broker offering queues and topics for decoupled, durable communication, with features such as peek-lock, sessions, and dead-lettering.
- **Q (p. 38): Azure Service Bus vs Event Hubs?**  
  **A:** Service Bus supports enterprise messaging/commands and per-message settlement; Event Hubs ingests high-throughput event streams with partitioned retention and consumer offsets.
- **Q (p. 38): What is an Azure Function App?**  
  **A:** A hosting resource for one or more event-triggered functions, configuration, identity, and related runtime settings.
- **Q (p. 38): How do you decide to use a Function App?**  
  **A:** Use it for event-driven or scheduled units of work with suitable execution/runtime limits; compare hosting cost, cold-start, duration, networking, and operational needs against App Service/containers.
- **Q (p. 38): How do you implement Azure Entra ID, and what configuration is needed?**  
  **A:** Register app/API, define scopes/roles, configure redirect/audience settings, grant consent, secure credentials or use managed identity, and validate tokens in the API.
- **Q (p. 38): How does authentication work with Entra ID?**  
  **A:** The client obtains a token from Entra ID using an appropriate OAuth/OIDC flow; it sends the access token to the API, which validates it and authorizes claims/scopes.
---

## Interview source: 20-verisk.md

### Verisk interview questions

#### Project / Basics (p. 35)

- **Q: What is your tech stack?**  
  **A:** Give a concise, accurate list of languages, frameworks, database, cloud, and delivery tooling, noting how each was used.
- **Q: Which SQL Server version did you use?**  
  **A:** State the actual version/compatibility level and mention features only if verified against that environment.
- **Q: How does the application connect to SQL?**  
  **A:** Configure a secure connection string/identity through environment settings or a secret store, use a provider/ORM with parameterization and pooling, and dispose connections promptly.

#### SQL (p. 36)

- **Q: How do you optimize a query?**  
  **A:** Capture the slow query, inspect actual plan and IO/time, check statistics/indexes and predicates, reduce rows/columns, and compare after change.
- **Q: Group patients by doctor for today.**  
  **A:** `SELECT DoctorId, COUNT(*) AS PatientCount FROM Appointment WHERE AppointmentTime >= CONVERT(date, GETDATE()) AND AppointmentTime < DATEADD(day,1,CONVERT(date,GETDATE())) GROUP BY DoctorId;` Use the relevant date/time zone and table/column names.

#### Coding (p. 36)

- **Q: Find the 3rd largest element using one loop.**  
  **A:** Maintain `first`, `second`, `third`; for each value, skip duplicates if distinct rank is wanted, shift values when it exceeds a maximum, and at end handle fewer than three values. This is O(n) time, O(1) space.

#### API (p. 36)

- **Q: Sketch a REST API skeleton.**  
  **A:** Define resource routes and DTOs, inject a service, use HTTP method attributes and `ActionResult<T>`, validate inputs, return correct status codes, and centralize errors. Example: `[HttpGet("{id}")] public async Task<ActionResult<ItemDto>> Get(int id)`.
- **Q: What is a REST API?**  
  **A:** An API organized around resources with stateless requests and standard HTTP methods/statuses and representations.
- **Q: How do you execute requests in parallel?**  
  **A:** For independent async operations, use `await Task.WhenAll(tasks)` with bounded concurrency, cancellation, and error handling; do not parallelize dependent operations.
---

## Interview source: 30-fn-lastletteralphabet.md

### FN (LastLetterAlphabet) — four-year questions

Prompts are transcribed in their source order. The source contains a literal ellipsis, misspellings, and an incomplete C# fragment; these are called out rather than silently completed as if the PDF were unambiguous. Source pages are 1-based.

#### SOLID, dependency management, and patterns

1. **Explain SOLID principles with a real example.** *(Source: p. 68.)*  
   **Answer:** For an order system: keep order calculation separate from notification (SRP); add new discount policies through abstractions (OCP); ensure implementations honor the same contract (LSP); split broad interfaces into client-specific ones (ISP); and make business policy depend on abstractions such as `IPaymentMethod` rather than concrete gateways (DIP).
2. **Explain the Open/Closed Principle using the Strategy pattern.** *(Source: p. 68.)*  
   **Answer:** Define a stable strategy contract and let a context delegate behavior to an injected strategy. Add a new strategy implementation without changing the context's algorithm; the context remains closed to modification while behavior is open to extension.
3. **Does the Factory pattern violate OCP?** *(Source: p. 68.)*  
   **Answer:** Not inherently. A factory centralizes object creation, but a factory with a growing hard-coded switch must change for each new type and can violate the spirit of OCP. Use registration/configuration, DI keyed services, or a factory abstraction; a small explicit factory may be the clearest design.
4. **What is Dependency Inversion Principle, and how does DI implement it?** *(Source: p. 68.)*  
   **Answer:** High-level policy and low-level details should depend on abstractions; abstractions should not depend on details. DI supplies implementations of those abstractions at runtime, commonly through constructor injection. DI is a mechanism that can support DIP, not the principle itself.
5. **Why use constructor dependency injection?** *(Source: p. 68.)*  
   **Answer:** Required dependencies are explicit, the object is valid when constructed, and tests can provide substitutes. It avoids hidden service-locator lookups; keep constructor dependencies focused to reveal overly broad responsibilities.
6. **How would you design an interest calculator using Strategy?** *(Source: p. 68.)*  
   **Answer:** Define `IInterestStrategy.Calculate(principal, rate, period, terms)`, implement strategies for the supported products/formulas, and let `InterestCalculator` delegate to the selected strategy. Validate inputs and make compounding/day-count rules explicit; select strategies through DI/registration.
7. **When do you use Factory vs Dependency Injection?** *(Source: p. 68.)*  
   **Answer:** DI composes application dependencies and manages their lifetimes. A factory is useful when runtime input determines which object/configuration to create or when creation is complex. A factory can itself be injected; avoid using either as a reason to hide dependencies.
8. **How do you create a concrete implementation of a strategy?** *(Source: p. 68.)*  
   **Answer:** Implement the strategy interface, supply its required dependencies via constructor, register it in DI (or a strategy registry), and select it using a validated key/business rule. Unit-test the implementation against the contract.

#### Banking design and business rules

9. **Design a simple banking system with Deposit and Withdraw.** *(Source: p. 68.)*  
   **Answer:** Model `Account` with identity, currency, and balance/ledger; expose application use cases `Deposit` and `Withdraw`; validate positive amounts and account state in domain logic; persist an immutable transaction/ledger entry and update balance atomically. Authorize ownership, use idempotency for retried requests, and audit changes.
10. **Where should business logic live: Controller, Service, or Domain?** *(Source: p. 68.)*  
    **Answer:** Controllers handle transport concerns; application services orchestrate use cases and transactions; domain entities/services enforce business invariants. Persistence belongs in infrastructure behind appropriate abstractions. Avoid putting domain rules in controllers.
11. **Why should a controller not contain business logic?** *(Source: p. 68.)*  
    **Answer:** It couples rules to HTTP, makes reuse/testing harder, and encourages duplicated behavior across endpoints. A thin controller validates/binds the request, invokes an application use case, and maps the result to HTTP.
12. **How do you handle different account types without `if`/`else`?** *(Source: p. 68.)*  
    **Answer:** First keep common invariants in a shared account model. If behavior genuinely varies, use polymorphic account types or focused policies/strategies such as `IWithdrawalPolicy`; select by account configuration/type. Avoid replacing a simple clear conditional with needless abstraction.
13. **How do you enforce rules such as minimum balance?** *(Source: p. 68.)*  
    **Answer:** Enforce the invariant in the domain/use-case transaction at the authoritative write boundary, checking the resulting balance before commit. Do not rely on UI validation; protect concurrent updates with a transaction plus locking/optimistic concurrency.

#### Resource management and security

14. **What is a resource leak?** *(Source: p. 68.)*  
    **Answer:** A resource such as a file handle, socket, database connection, or unmanaged memory remains held after it is no longer needed, eventually exhausting capacity or degrading performance.
15. **Why use the `using` keyword in C#?** *(Source: p. 68.)*  
    **Answer:** A `using` statement/declaration guarantees `Dispose()` runs when scope exits, including on exceptions, for `IDisposable` resources. `await using` does the same for `IAsyncDisposable`.
16. **What is SQL injection and how do you prevent it?** *(Source: p. 68.)*  
    **Answer:** Untrusted input changes the meaning of dynamically constructed SQL. Use parameterized queries/prepared statements or safe ORM parameters, validate/allowlist identifiers that cannot be parameterized, and apply least-privilege database permissions. Do not rely on string escaping as the primary defense.
17. **What is a race condition in banking transactions?** *(Source: p. 68.)*  
    **Answer:** Concurrent operations read/update shared account state in an order that causes a lost update or violates a balance invariant—for example, two withdrawals both see the same available balance. Use atomic conditional updates, suitable isolation/locking or optimistic concurrency, and idempotency.
18. **Why should we use database transactions?** *(Source: p. 68.)*  
    **Answer:** Transactions make related operations atomic and preserve consistency under failure and concurrency. Use the smallest appropriate transaction boundary; distributed services generally need a saga/outbox rather than a long cross-service database transaction.

#### SQL questions

19. **Show total balance across accounts for each customer.** *(Source: p. 68.)*  
    **Answer:** Assuming `Account(CustomerId, Balance)` and all balances use a common currency:
    ```sql
    SELECT CustomerId, SUM(Balance) AS TotalBalance
    FROM Account
    GROUP BY CustomerId;
    ```
    If currencies differ, group by currency too or convert using an explicit exchange-rate policy.
20. **How do you optimize a query for the latest transaction per account?** *(Source: p. 68.)*  
    **Answer:** Use a composite index such as `(AccountId, TransactionDate DESC, TransactionId DESC)` and select deterministically:
    ```sql
    WITH Ranked AS (
        SELECT t.*,
               ROW_NUMBER() OVER (
                   PARTITION BY AccountId
                   ORDER BY TransactionDate DESC, TransactionId DESC
               ) AS rn
        FROM dbo.Transaction AS t
    )
    SELECT *
    FROM Ranked
    WHERE rn = 1;
    ```
    Select only required columns and verify the actual execution plan. If only one known account is requested, an indexed `TOP (1)` query may be more efficient.

21. **The source contains an ellipsis (`…`).** *(Source: p. 68.)*  
    **Answer:** This is an editorial omission marker, not a recoverable interview question; no omitted wording can be inferred reliably.
22. **“diff transiet and scope lifetime method on .net core” — difference between Transient and Scoped lifetimes in .NET Core.** *(Source: p. 68.)*  
    **Answer:** Transient creates a new instance each time it is resolved; scoped creates one instance per DI scope (normally one HTTP request). Use scoped for request-bound state/EF `DbContext`, transient for independent lightweight services; do not inject scoped services directly into singletons.
23. **“what are performance level use on EF core” — what EF Core performance practices do you use?** *(Source: p. 68.)*  
    **Answer:** Profile generated SQL; project only needed columns, use no-tracking for read-only queries, avoid N+1 and premature materialization, paginate, use async I/O, batch writes where supported, and add database indexes based on actual workload. Benchmark before optimizing.
24. **Difference between eager loading and lazy loading.** *(Source: p. 68.)*  
    **Answer:** Eager loading retrieves related entities as part of the planned query (e.g., `Include`); lazy loading retrieves them when a navigation property is accessed. Lazy loading can trigger N+1 round trips; eager loading can overfetch, so choose based on query needs and inspect SQL.

#### OAuth, asynchronous code, and Angular

25. **How do you implement OAuth 2.0 security?** *(Source spells it “oath 2.o”; p. 69.)*  
    **Answer:** Use a trusted authorization server and the appropriate flow (typically authorization code with PKCE for browser/native clients; client credentials for service-to-service). Validate issuer, audience, signature, expiry, and scopes at the API; use TLS, least privilege, secure token storage, and refresh-token rotation. OAuth is delegated authorization; use OpenID Connect when user authentication/identity is needed.
26. **“Aync and await” — explain `async` and `await`.** *(Source: p. 69.)*  
    **Answer:** `async` enables a method to use `await` and normally return `Task`/`Task<T>`; `await` asynchronously observes a task's completion and propagates its result/error without blocking the current thread. It is most useful for I/O-bound work and does not itself create a thread.
27. **Angular — auth guard and interceptor.** *(Source: p. 69.)*  
    **Answer:** A route guard controls navigation based on current authentication/authorization state, but server APIs must still enforce authorization. An HTTP interceptor adds request metadata/tokens and centralizes responses/errors; do not treat client-side guards as security boundaries.

#### SQL grouping exercise

28. **Explain `GROUP BY`.** *(Source: p. 69.)*  
    **Answer:** It groups rows with equal values in specified columns so aggregate functions such as `COUNT`, `SUM`, or `MAX` can be calculated per group. Non-aggregated selected columns must be grouping columns (subject to database-specific functional-dependency rules).
29. **Explain the `HAVING` clause.** *(Source: p. 69.)*  
    **Answer:** `HAVING` filters groups after aggregation; `WHERE` filters input rows before grouping. Example:
    ```sql
    SELECT DepartmentId, COUNT(*) AS EmployeeCount
    FROM Employee
    GROUP BY DepartmentId
    HAVING COUNT(*) > 5;
    ```
30. **Given `id, dept, country` rows `(1, IT, USA)`, `(2, HR, USA)`, `(3, Account, USA)`, explain `SELECT country FROM tblcountry GROUP BY country`.** *(Source: p. 69.)*  
    **Answer:** It returns one row per distinct country; for these sample rows, a single row `USA`. `SELECT DISTINCT country FROM tblcountry` expresses the same intent when no aggregate is needed.

#### Incomplete string-counting code prompt

31. **The source gives `string str="sarbeswarsahoo"; int count=0; for(int i=0; i > str.length) { foreach(Listitem li in str[1]-1) { if(li ) { } char ch = li; count = li.count; console.Write(count + ) } }` and closing braces. What should it do?** *(Source: pp. 69–70.)*  
    **Answer:** The fragment is syntactically invalid/incomplete: `i > str.length` prevents the intended forward iteration, `Listitem` is unrelated to iterating a string, `str[1]-1` is not a collection, the condition/output are missing, and the final `+` has no operand. The likely intent is to count character occurrences in `"sarbeswarsahoo"`. A corrected C# solution that reports each distinct character and its frequency is:
    ```csharp
    string str = "sarbeswarsahoo";
    foreach (var group in str.GroupBy(char.ToLowerInvariant))
        Console.WriteLine($"{group.Key}: {group.Count()}");
    ```
    This produces `s: 3`, `a: 3`, `r: 2`, `b: 1`, `e: 1`, `w: 1`, `h: 1`, `o: 2` (group order follows first appearance in current LINQ-to-Objects execution). If the intended prompt was specifically “count each character” versus “find duplicates” or “count one selected character,” the source does not resolve that ambiguity.
---

## Interview source: 30-infosys.md

### Infosys AI interview experience

The source describes the interview format rather than listing a fixed question set. It says candidates may answer orally or type an answer, after which the interview AI responds; follow-up questions may depend on the resume and previous answers. *(Source: p. 62.)*

- **Expected .NET topics:** .NET Core vs .NET Framework, `async`/`await`, and related fundamentals.  
  **Interview-ready response:** Clearly contrast modern cross-platform .NET with the older Windows-focused .NET Framework; explain asynchronous I/O and its non-blocking behavior, and relate the answer to a real project only where accurate.
- **Resume-based prompts and follow-ups:**  
  **Interview-ready response:** Keep project claims consistent, describe personal contribution, trade-offs, and measurable outcomes; expect the next question to probe the details just stated.
- **String problems and small C# programs:**  
  **Interview-ready response:** Clarify case sensitivity, ordering, and input constraints first; explain complexity and test empty/single-character/repeated-character cases. A frequency map or `StringBuilder` is a common basis for string exercises.

The note reports attending two AI interviews with this pattern; it does not identify exact questions beyond these topic areas. *(Source: p. 62.)*
---

## Interview source: 30-tcs.md

### TCS interview groups

The two source groups below remain separate: the .NET Full Stack + AI 20+ LPA interview and the second developer's experience notes. Source page numbers are 1-based PDF pages.

#### TCS Consultancy — 20+ LPA (.NET Full Stack + AI)

##### .NET Core questions

1. **Explain the Singleton lifetime in dependency injection with a real-time example.** *(Source: p. 55.)*  
   **Answer:** One instance is created for the service provider and reused for its lifetime. A stateless configuration/cache-like service can be singleton; it must be thread-safe and must not capture scoped request services. Register with `AddSingleton<T>()`.

2. **Explain SOLID principles.** *(Source: p. 55.)*  
   **Answer:** Single Responsibility (one reason to change); Open/Closed (extend without modifying stable code); Liskov Substitution (subtypes preserve base contracts); Interface Segregation (small client-specific interfaces); Dependency Inversion (high-level policy depends on abstractions). Use cohesive services, substitutable implementations, and injected interfaces to apply them.

3. **Difference between `Hashtable` and `Dictionary`.** *(Source: p. 55.)*  
   **Answer:** `Hashtable` is a legacy non-generic collection storing object keys/values, requiring casts and boxing value types. `Dictionary<TKey,TValue>` is generic and type-safe, usually preferred, with configurable equality comparers. Neither should be concurrently mutated without synchronization.

4. **Primary vs secondary constructor in C#.** *(Source: p. 55.)*  
   **Answer:** A primary constructor is declared in the type declaration and its parameters are available throughout the type's body (C# 12 for classes/structs; earlier records had primary constructors). A secondary constructor is a traditional named-type constructor body and can chain to another constructor with `this(...)` or a base constructor with `base(...)`.

5. **`Convert.ToString()` vs `ToString()`.** *(Source: p. 55.)*  
   **Answer:** `Convert.ToString(null)` returns an empty string (and supports `IFormatProvider` overloads); calling an instance's `.ToString()` when that reference is null throws `NullReferenceException`. `Convert.ToString` also handles nullable/object values more safely.

6. **Differences among `First()`, `FirstOrDefault()`, `Single()`, and `SingleOrDefault()`.** *(Source: p. 55.)*  
   **Answer:** `First` returns the first match or throws if none; `FirstOrDefault` returns default if none. `Single` requires exactly one match and throws for zero or multiple; `SingleOrDefault` allows zero but throws for multiple. Use `Single` only when uniqueness is an invariant.

7. **What is middleware, and how do you write custom middleware?** *(Source: p. 55.)*  
   **Answer:** Middleware is an ordered HTTP pipeline component. Implement `InvokeAsync(HttpContext, RequestDelegate)`, perform pre-processing, `await next(context)` to continue, and post-process if appropriate. Register via `UseMiddleware<T>()` in the correct order; inject scoped dependencies into `InvokeAsync`, not a singleton middleware constructor.

8. **GET vs POST—which is better and why?** *(Source: p. 55.)*  
   **Answer:** Neither is universally better. GET is safe/idempotent and suitable for retrieval, with cacheable URL parameters; POST submits/processes data and is not inherently idempotent or cacheable. Select based on semantics, payload, privacy, and HTTP behavior, not a performance rule.

9. **How do you optimize API performance?** *(Source: p. 55.)*  
   **Answer:** Measure first; optimize database indexes/queries and avoid N+1; use async I/O, pagination, projection, caching where safe, compression, connection pooling, bounded concurrency, and efficient serialization. Add tracing/metrics and load tests to confirm improvement.

10. **How do you implement pagination in .NET Core?** *(Source: p. 56.)*  
    **Answer:** Validate page size and offset/cursor, apply ordering and paging in the database before materialization, return items plus paging metadata or a continuation cursor. Use keyset pagination for large/changing datasets to avoid expensive offsets and unstable pages.

11. **What are MVC filters?** *(Source: p. 56.)*  
    **Answer:** Hooks around MVC resource, action, exception, and result execution, plus authorization filters. They apply MVC-specific cross-cutting behavior; middleware is preferable for concerns spanning the whole HTTP pipeline.

12. **`IActionResult` vs `ActionResult`.** *(Source: p. 56.)*  
    **Answer:** `IActionResult` is an interface for HTTP action results. `ActionResult` is its abstract base class; `ActionResult<T>` combines a typed success body with action results such as `NotFound()`, improving API typing and metadata.

13. **Why might an API be slow, and how would you fix it?** *(Source: p. 56.)*  
    **Answer:** Profile end-to-end to isolate database, downstream calls, CPU, serialization, locks, or network bottlenecks. Fix the measured cause—e.g., query/index/N+1, missing pagination, synchronous I/O, excess payload, contention, or slow dependency—and validate under representative load.

##### AI-related questions

1. **Explain AI integration in your project.** *(Source: p. 56.)*  
   **Answer:** Describe the actual use case, model/API, data flow, privacy controls, grounding, evaluation, latency/cost controls, and human fallback. Do not claim production experience that is not real.
2. **How do you call an AI API from .NET?** *(Source: p. 56.)*  
   **Answer:** Use a typed `HttpClient` from `IHttpClientFactory`, async calls, managed identity or a securely stored secret, timeouts/cancellation, bounded retries for transient failures, request validation, and redacted telemetry. Avoid logging sensitive prompts/responses.
3. **What is prompt engineering?** *(Source: p. 56.)*  
   **Answer:** Designing and testing instructions/context/examples and output constraints to make model responses useful and reliable. It should be paired with grounding, input/output validation, evaluation, and safety controls rather than treated as a guarantee.
4. **Design an AI-powered chatbot in .NET.** *(Source: p. 56.)*  
   **Answer:** Use an API/orchestrator with authentication, rate limits, prompt construction, and a provider abstraction; retrieve authorized documents through a search/vector index (RAG), call the model, validate/cite results, and return a response. Store conversation state appropriately, redact PII, monitor cost/latency, and provide fallback/escalation.
5. **A chatbot gives wrong answers—how do you fix it?** *(Source: p. 56.)*  
   **Answer:** Reproduce and classify failures; check retrieval relevance, freshness, permissions, prompt ambiguity, and model limitations. Improve trusted grounding/chunking/ranking, constrain unsupported answers, add citations and abstention, evaluate with representative cases, and monitor regressions; do not rely solely on a longer prompt.

##### Scenario questions

1. **Authorization is applied to an API, but a specific condition inside the method should bypass it. How would you implement that in production?** *(Source: p. 56.)*  
   **Answer:** Do not disable authorization after entering the action. Define the narrowly allowed policy/resource condition and enforce it through a custom authorization requirement/handler or resource-based authorization before sensitive work. If an endpoint is genuinely public, expose a separate explicitly anonymous route and audit it; never allow caller-controlled input to bypass access control.

2. **Given `int a = 1`, you want to assign it dynamically at runtime, e.g. `var e = a`. Which type would you use and why?** *(Source: p. 56.)*  
   **Answer:** If the type is known at compile time, use `var e = a`; `e` is still statically typed as `int`. Use `dynamic` only when runtime binding is actually required (such as interoperating with dynamically shaped data), since errors then surface at runtime.

3. **An MVC API returns `{ "id": 1, "name": null, "age": 25 }`. How do you omit null values?** *(Source: p. 56.)*  
   **Answer:** Configure the JSON serializer to ignore nulls, for example with `System.Text.Json`:
   ```csharp
   builder.Services.AddControllers().AddJsonOptions(o =>
       o.JsonSerializerOptions.DefaultIgnoreCondition =
           System.Text.Json.Serialization.JsonIgnoreCondition.WhenWritingNull);
   ```
   For per-property behavior, apply `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]`.

##### SQL scenarios

1. **What happens with `SELECT * INTO newtable FROM oldtable WHERE 1 = 0`? Where is it used?** *(Source: p. 57.)*  
   **Answer:** It creates `newtable` from the selected columns' metadata and inserts zero rows. Useful for creating an empty table shaped from a query; indexes, constraints, and other table properties are not generally copied.

2. **What happens with `SELECT * INTO newtable FROM oldtable WHERE 1 = 1`? Where is it used?** *(Source: p. 57.)*  
   **Answer:** It creates `newtable` and inserts every row satisfying the always-true predicate—effectively all rows. It can make a quick copy, but is not a substitute for a deliberate schema/migration or backup strategy.

3. **Can you write SQL pagination?** *(Source: p. 57.)*  
   **Answer:**
   ```sql
   SELECT Id, Name
   FROM dbo.Items
   ORDER BY Id
   OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY;
   ```
   Require deterministic ordering and validate parameters; for deep pages consider keyset pagination.

4. **Insert values `Male='M'`, `Female='F'`, `Other='O'`.** *(Source: p. 57.)*  
   **Answer:** The prompt gives no target table/column, so assuming `GenderLookup(Gender, Code)`:
   ```sql
   INSERT INTO GenderLookup (Gender, Code)
   VALUES ('Male', 'M'), ('Female', 'F'), ('Other', 'O');
   ```

5. **Create `UserDetails` with default age 18.** *(Source: p. 57.)*  
   **Answer:**
   ```sql
   CREATE TABLE dbo.UserDetails
   (
       UserId int NOT NULL PRIMARY KEY,
       Age int NOT NULL CONSTRAINT DF_UserDetails_Age DEFAULT (18)
   );
   ```

6. **Find the third-highest tanker water level and duplicate tanker names.** *(Source: p. 57.)*  
   **Answer:** Assuming `Tanker(Name, WaterLevel)`, use distinct levels for the third-highest level (ties count once) and group names for duplicates:
   ```sql
   SELECT WaterLevel
   FROM (SELECT DISTINCT WaterLevel FROM Tanker) d
   ORDER BY WaterLevel DESC
   OFFSET 2 ROWS FETCH NEXT 1 ROW ONLY;

   SELECT Name, COUNT(*) AS DuplicateCount
   FROM Tanker
   GROUP BY Name
   HAVING COUNT(*) > 1;
   ```
   Use `DENSE_RANK()` if the result should include all rows at the third distinct level. The table/column names are assumptions because the prompt does not specify them.

##### Interview focus note

The report says the interview emphasized real-time scenarios, API performance, SQL problem-solving, and basic AI understanding. **Preparation answer:** Practice explaining a measured design decision and solving a query/code problem aloud, including assumptions and edge cases. *(Source: pp. 57–58.)*

#### TCS developer experience 2 — interview notes

##### First technical round

The source says the first technical round was important and began with basic but practical topics. These are recorded prompts/areas, including the two concrete coding/query prompts: *(Source: p. 58.)*

1. **Simple coding program: string splitting.**  
   **Answer:** Clarify delimiter and empty-entry behavior; in C#, `text.Split(delimiter, StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)` handles common whitespace-separated cases. Mention that delimiters and options depend on the input contract.
2. **Find the third-highest integer.**  
   **Answer:** If “third highest” means the third distinct value:
   ```csharp
   var third = values.Distinct().OrderByDescending(x => x).Skip(2).FirstOrDefault();
   ```
   Return an explicit nullable/result type to distinguish “not found” from a real default value.
3. **.NET Core basics.**  
   **Answer:** Be ready to explain the cross-platform .NET runtime, DI, configuration, middleware, hosting, async I/O, and differences from .NET Framework.
4. **Authentication vs OAuth.**  
   **Answer:** Authentication establishes identity; authorization determines permitted actions. OAuth 2.0 is an authorization framework for delegated access; OpenID Connect adds an identity layer for authentication.
5. **How do you expire a JWT?**  
   **Answer:** Set a short `exp` claim and validate issuer/audience/signature/time. A JWT is normally not revocable by itself; use short lifetimes plus refresh-token rotation/revocation or a denylist/session version when immediate revocation is needed.
6. **Custom middleware.**  
   **Answer:** Implement `InvokeAsync(HttpContext, RequestDelegate)`, call the next delegate when appropriate, and register in the required order; keep middleware behavior focused and testable.
7. **Uses of `async`/`await`.**  
   **Answer:** They compose asynchronous operations without blocking a thread during I/O; `await` propagates completion/errors. They do not make CPU-bound work automatically faster.
8. **Database query optimization.**  
   **Answer:** Inspect the actual execution plan and I/O/latency; use suitable indexes, selective predicates, sargable expressions, correct joins, limited projections, current statistics, and pagination. Measure before/after.
9. **CTE.**  
   **Answer:** A named query expression scoped to one statement, useful for readability and recursive queries. A CTE is not automatically materialized or a cache.
10. **Demerits of a SQL cursor.**  
    **Answer:** Row-by-row execution often adds overhead and holds resources longer than set-based operations. Use set-based SQL/window functions where possible; use a cursor only when ordered per-row procedural work is truly required.
11. **When to use `AsNoTracking` in Entity Framework?**  
    **Answer:** For read-only queries where entities will not be updated through that `DbContext`; it avoids change-tracker overhead. Do not use it when tracked updates/identity resolution are needed.
12. **Department-wise maximum salary query.**  
    **Answer:** To return all employees tied for the maximum salary in each department:
    ```sql
    SELECT DepartmentId, EmployeeId, Salary
    FROM (
        SELECT e.*,
               DENSE_RANK() OVER (
                   PARTITION BY DepartmentId ORDER BY Salary DESC
               ) AS rn
        FROM Employee e
    ) x
    WHERE rn = 1;
    ```
    Use `ROW_NUMBER()` instead if exactly one row per department is required.

##### Second round

- **Managerial discussion:** The note describes questions about project responsibilities and technical roles. *(Source: p. 58.)*  
  **Answer:** Explain scope and ownership, decisions, collaboration, delivery, incidents, and outcomes; distinguish your own work from the team's.
---

## Interview source: 30-virtusa.md

### Virtusa interview groups

This file keeps the Lead Software Engineer process and the separately dated 11 June 2026 microservices/Docker interview distinct. Source page numbers refer to the PDF's printed page sequence (1-based).

#### Lead Software Engineer interview process (10+ years)

##### Process notes

- **AI interview (MCQ):** Online multiple-choice assessment of programming basics and core concepts. **Answer approach:** Read for qualifiers and edge cases, then eliminate options that contradict language/runtime behavior. *(Source: p. 47; heading/context for the group.)*
- **L1 technical:** Coding, problem-solving, SQL, C#, and output questions. The note says exact output and a complete correct program are expected. **Preparation:** Compile/run code mentally or in an IDE; state assumptions and cover edge cases. *(Source: p. 47.)*
- **L2 techno-managerial:** Technical knowledge, practical experience, system design, real-time scenarios, project discussion, decisions, and problem handling. **Answer approach:** Explain the trade-offs and give a concrete example. *(Source: pp. 47–48.)*
- **Client technical round (HackerRank):** Hands-on coding focused on communication, problem-solving, coding efficiency, and logic. **Answer approach:** Clarify input constraints, explain the algorithm, then test boundary cases. *(Source: p. 48.)*
- **HR discussion:** Salary, availability, cultural fit, background, and career goals. **Answer approach:** Be candid, concise, and consistent about availability and compensation expectations. *(Source: p. 48.)*

##### SQL questions

1. **Employees/projects:** Given `Employees`, `Projects`, and `EmployeeProjects`, find employees who worked on at least two projects; identify the longest-duration project; return employee names with the longest project they worked on, returning any one in a tie. *(Source: pp. 48–49.)*  
   **Answer:** Assume `EmployeeProjects(EmployeeId, ProjectId)`, `Projects(ProjectId, ProjectName, StartDate, EndDate)`, and `Employees(EmployeeId, EmployeeName)`. First aggregate assignments for the two-project condition; for the longest assigned project, rank each employee's projects by duration and return rank 1. For example (SQL Server):
   ```sql
   WITH ProjectCounts AS (
       SELECT EmployeeId
       FROM EmployeeProjects
       GROUP BY EmployeeId
       HAVING COUNT(DISTINCT ProjectId) >= 2
   ),
   Ranked AS (
       SELECT ep.EmployeeId, p.ProjectName,
              ROW_NUMBER() OVER (
                  PARTITION BY ep.EmployeeId
                  ORDER BY DATEDIFF(day, p.StartDate, p.EndDate) DESC
              ) AS rn
       FROM EmployeeProjects ep
       JOIN Projects p ON p.ProjectId = ep.ProjectId
       JOIN ProjectCounts pc ON pc.EmployeeId = ep.EmployeeId
   )
   SELECT e.EmployeeName, r.ProjectName
   FROM Ranked r
   JOIN Employees e ON e.EmployeeId = r.EmployeeId
   WHERE r.rn = 1;
   ```
   If “longest project” means the globally longest project rather than each qualifying employee's longest, rank projects globally instead; the wording permits either interpretation.

2. **Customers with more than three orders:** `tblCustomer(CustId, CustName)`, `tbleOrderdtls(OrderId, CustId, OrderNumber)`; return customer ID, name, and order count. *(Source: p. 49.)*  
   **Answer:**
   ```sql
   SELECT c.CustId, c.CustName, COUNT(*) AS OrderCount
   FROM tblCustomer c
   JOIN tbleOrderdtls o ON o.CustId = c.CustId
   GROUP BY c.CustId, c.CustName
   HAVING COUNT(*) > 3;
   ```
   Count distinct `OrderId` instead if order detail rows can repeat an order.

3. **Laptop sales:** `Laptop(L_Id, Brand)` and `Sales(Sale_Id, L_Id, Quantity)`; show each brand and total quantity sold, including brands with no sales. *(Source: pp. 49–50.)*  
   **Answer:**
   ```sql
   SELECT l.Brand, COALESCE(SUM(s.Quantity), 0) AS TotalQuantity
   FROM Laptop l
   LEFT JOIN Sales s ON s.L_Id = l.L_Id
   GROUP BY l.L_Id, l.Brand;
   ```

##### C# questions — L1

1. **For `"ABC"`, generate all combinations and permutations** (the examples include `ABC, BCA, CAB, AB, BC, AC`). *(Source: p. 50.)*  
   **Answer:** The examples mix full-string permutations with order-preserving combinations and are not exhaustive. Assuming distinct input characters, enumerate every nonempty subset and every permutation of each subset:
   ```csharp
   static IEnumerable<string> Generate(string text)
   {
       for (int mask = 1; mask < (1 << text.Length); mask++)
       {
           var chars = Enumerable.Range(0, text.Length)
               .Where(i => (mask & (1 << i)) != 0)
               .Select(i => text[i]).ToArray();
           foreach (var item in Permute(chars, 0)) yield return item;
       }
   }

   static IEnumerable<string> Permute(char[] chars, int start)
   {
       if (start == chars.Length) { yield return new string(chars); yield break; }
       for (int i = start; i < chars.Length; i++)
       {
           (chars[start], chars[i]) = (chars[i], chars[start]);
           foreach (var item in Permute(chars, start + 1)) yield return item;
           (chars[start], chars[i]) = (chars[i], chars[start]);
       }
   }
   ```
   For `ABC`, this produces 3 one-character strings, 6 two-character strings, and 6 three-character strings (15 total). If “combination” means only order-preserving subsequences, emit each selected subset once instead; confirm the intended definition and duplicate-character behavior.

2. **Output/inheritance:** `A.Display()` prints `A`; `B : A` declares `Display()` printing `B`; evaluate `B b = new A(); b.Display();` and `A a = new B(); a.Display();`, including `virtual`/`override` variants. *(Source: pp. 50–51.)*  
   **Answer:** As written, `B b = new A()` is a compile-time error: an `A` instance cannot be assigned to `B`. Thus the full snippet does not run. The valid `A a = new B(); a.Display();` prints `A` when methods are non-virtual (the call is selected by the reference's static type). If `A.Display` is `virtual` and `B.Display` is `override`, it prints `B` by runtime dispatch. If the first assignment is corrected to `B b = new B()`, that call prints `B`.

3. **Sort `{4,0,7,5,2,8,0,0,0}` to `{0,0,0,0,4,7,5,2,8}`.** *(Source: p. 51.)*  
   **Answer:** This is a stable partition, not a full ascending sort: move zeros to the front and preserve nonzero order. For an array `a`, compact nonzeros, then fill the leading slots with zero:
   ```csharp
   int write = a.Length - 1;
   for (int read = a.Length - 1; read >= 0; read--)
       if (a[read] != 0) a[write--] = a[read];
   while (write >= 0) a[write--] = 0;
   ```
   This preserves the nonzero order and returns `{0,0,0,0,4,7,5,2,8}`.

4. **Write a program to find duplicate characters in a string.** *(Source: p. 51.)*  
   **Answer:** Count characters, then emit those with count greater than one; this example is case-sensitive:
   ```csharp
   foreach (var group in text.GroupBy(c => c).Where(g => g.Count() > 1))
       Console.WriteLine($"{group.Key}: {group.Count()}");
   ```
   Normalize characters before grouping if duplicates should be case-insensitive.

5. **What is concurrent programming?** *(Source: p. 51.)*  
   **Answer:** Structuring work so multiple tasks make progress during overlapping periods. It can use interleaving on one core or parallel execution on several cores; shared mutable state requires synchronization or a safer design.

6. **What is GraphQL and why is it used?** *(Source: p. 51.)*  
   **Answer:** A typed API query language and server runtime. Clients request precisely the fields they need through a schema, reducing over/under-fetching and often consolidating related reads. It adds schema/resolver and caching/security considerations; it is not automatically faster than REST.

7. **Explain CI/CD.** *(Source: p. 51.)*  
   **Answer:** Continuous integration frequently builds and tests changes. Continuous delivery keeps validated changes releasable; continuous deployment automatically releases them. Pipelines usually include build, tests, security/quality checks, artifact creation, deployment, and monitoring/rollback.

8. **Differences: .NET 6 vs .NET 8; `var` vs `dynamic`; `Task` vs `Thread`; `async` vs `await`.** *(Source: p. 51.)*  
   **Answer:**
   - **.NET 6 / .NET 8:** Both are .NET (not .NET Framework) releases; .NET 8 is the newer LTS generation with newer runtime, libraries, language/tooling support, and performance improvements. Choose based on support policy and dependency compatibility.
   - **`var` / `dynamic`:** `var` is compile-time type inference and remains statically typed; `dynamic` defers member binding/type checks to runtime.
   - **`Task` / `Thread`:** `Task` represents an asynchronous operation and is commonly scheduled on the thread pool; `Thread` is an OS execution thread with explicit lifetime/control. Most I/O-bound work should use async tasks, not dedicated threads.
   - **`async` / `await`:** `async` marks a method that can use asynchronous operations and typically returns `Task`/`Task<T>`; `await` asynchronously waits for a task without blocking the calling thread.

9. **Explain static class, ASP.NET middleware, and a temporary table; can temp tables be used in functions and triggers?** *(Source: pp. 51–52.)*  
   **Answer:** A C# static class cannot be instantiated and contains static members. ASP.NET Core middleware is a component in the HTTP request/response pipeline that can inspect, modify, short-circuit, or pass control onward. In SQL Server, a local `#temp` table is session-scoped and a `##temp` table is globally visible while it exists. SQL Server user-defined functions cannot create or use temporary tables; a trigger can use a temp table, although `inserted`/`deleted` and set-based logic are generally preferable.

10. **How do you handle exceptions in C#, React, and SQL?** *(Source: p. 52.)*  
    **Answer:** In C#, use targeted `try`/`catch` and `finally`/`using`, preserve stack traces with `throw;`, and log at the appropriate boundary. In React, catch async/event-handler failures explicitly; error boundaries handle render/lifecycle errors in descendants, not arbitrary async callbacks. In SQL, use `TRY...CATCH`, `THROW`, and transactions with rollback where required.

11. **REST API vs traditional API.** *(Source: p. 52.)*  
    **Answer:** “Traditional API” is not a precise category. REST is an architectural style for resource-oriented APIs using HTTP semantics, stateless requests, representations, and standard methods/status codes. An API can instead be RPC/SOAP or another style; compare the specific protocol and constraints rather than treating “API” and “REST” as opposites.

12. **How do you implement API versioning?** *(Source: p. 52.)*  
    **Answer:** Choose a stable strategy (URL path such as `/api/v2`, header, or media type), configure version selection, keep contracts explicit, test each supported version, document deprecation timelines, and avoid branching business logic unnecessarily.

13. **What are ASP.NET filters?** *(Source: p. 52.)*  
    **Answer:** MVC action-pipeline hooks for cross-cutting behavior such as authorization, resource/action execution, exception handling, and result processing. Use middleware for application-wide HTTP concerns and filters for MVC/action-specific concerns.

14. **Can you create custom middleware? How?** *(Source: p. 52.)*  
    **Answer:** Yes. Implement `Invoke`/`InvokeAsync(HttpContext, RequestDelegate)` in a class or register an inline delegate; call `await next(context)` when continuing, and register it in the correct order with `UseMiddleware<T>()`/`Use(...)`.

15. **What is an alternative to React `useEffect`?** *(Source: p. 52.)*  
    **Answer:** There is no universal replacement. Derive values during render or use `useMemo` for pure expensive calculations; handle user actions in event handlers; use framework/data-fetching tools or a query library for server state; use `useSyncExternalStore` for external stores. Effects remain appropriate for synchronizing with external systems.

##### JavaScript / React output questions

16. **What is printed by `console.log(true + 1)`, `console.log(1 + '2')`, and `console.log('2' + 1)`?** *(Source: p. 53.)*  
    **Answer:** `2`, `"12"`, `"21"` respectively. Boolean converts to number for numeric addition; when either operand of `+` becomes a string, concatenation is used.

17. **What is printed by `let c; console.log(a); console.log(b); console.log(c); var a; let b;`?** *(Source: p. 53.)*  
    **Answer:** The first log prints `undefined` because `var a` is hoisted and initialized to `undefined`. The second throws `ReferenceError` because `b` is in its temporal dead zone until its declaration executes; execution stops, so `c` is not logged.

##### C# question — L2

18. **Implement `public static string MergeStrings(string s1, string s2)`: alternate characters while both strings have characters, then append the remainder.** Examples: `ABC` + `123` → `A1B2C3`; `ABCDEFG` + `123` → `A1B2C3DEFG`; `ABC` + `123456` → `A1B2C3456`. *(Source: pp. 53–54.)*  
    **Answer:**
    ```csharp
    public static string MergeStrings(string s1, string s2)
    {
        var result = new System.Text.StringBuilder(s1.Length + s2.Length);
        int i = 0;
        while (i < s1.Length && i < s2.Length)
        {
            result.Append(s1[i]).Append(s2[i]);
            i++;
        }
        if (i < s1.Length) result.Append(s1, i, s1.Length - i);
        if (i < s2.Length) result.Append(s2, i, s2.Length - i);
        return result.ToString();
    }
    ```

#### Virtusa — 11 June 2026 (microservices and Docker focused)

##### General and background

- **Can you introduce yourself?** *(Source: p. 54.)*  
  **Answer:** Give a 60–90 second summary of relevant experience, current role, strongest .NET/cloud/microservices skills, one measurable accomplishment, and what role you are seeking. Personalize with truthful details.

##### Microservices architecture

- **Have you worked with microservices?** *(Source: p. 54.)*  
  **Answer:** Describe actual services you owned, their boundaries, communication, data ownership, deployment/monitoring, and one result. Do not claim experience you do not have.
- **Have you migrated a monolith to microservices?** *(Source: p. 54.)*  
  **Answer:** Explain the incremental strangler approach, first extracting a bounded capability behind an API/event contract, moving its data ownership, routing traffic, and monitoring rollback. If not done, distinguish adjacent experience from direct migration.
- **How do you identify which microservices to create from a monolith?** *(Source: p. 54.)*  
  **Answer:** Map domain capabilities and bounded contexts with domain experts; identify cohesive rules/data and independent change/scaling needs. Extract incrementally where a service boundary reduces coupling enough to justify operational and distributed-systems costs.
- **What advantages do microservices have over a monolith?** *(Source: p. 54.)*  
  **Answer:** Independent deployment/scaling, fault isolation, and team autonomy with technology flexibility. These trade off against network failure, consistency, observability, deployment, and operational complexity; a modular monolith may be better when those benefits are not needed.
- **What was your biggest microservices-project challenge and how did you solve it?** *(Source: p. 54.)*  
  **Answer:** Use a real STAR example. A strong example is eventual consistency: define ownership and versioned events, use an outbox/retry strategy, make consumers idempotent, expose status, and monitor lag; explain the actual outcome.

##### C# and .NET Core

- **Write C# to remove duplicate characters from a string; example `"Anand"` → `"AND"`.** *(Source: p. 54.)*  
  **Answer:** The example implies case-insensitive de-duplication and uppercase output, retaining first-seen order:
  ```csharp
  static string RemoveDuplicates(string input)
  {
      var seen = new HashSet<char>();
      var result = new System.Text.StringBuilder();
      foreach (char ch in input)
      {
          char key = char.ToUpperInvariant(ch);
          if (seen.Add(key)) result.Append(key);
      }
      return result.ToString();
  }
  ```
- **Why can .NET Core/.NET run faster than older .NET Framework?** *(Source: p. 54.)*  
  **Answer:** It is a modern, cross-platform runtime with ongoing JIT, GC, library, and networking optimizations; it also supports lightweight deployment and modern hosting. Performance depends on workload and configuration, so benchmark rather than promise a universal speedup.

##### Cloud and DevOps (Azure and Docker)

- **What Azure services have you used?** *(Source: p. 55.)*  
  **Answer:** Name only services actually used and describe the role, e.g., App Service/AKS for hosting, Azure SQL, Storage, Key Vault, Functions, Service Bus, Application Insights, and managed identity.
- **Give an example task using Azure Functions.** *(Source: p. 55.)*  
  **Answer:** For example, a queue-triggered function can validate an uploaded order event, call a downstream service, and record a result; explain trigger, retry/idempotency, secrets identity, and monitoring. Use a truthful project example.
- **How did you deploy the Azure Function?** *(Source: p. 55.)*  
  **Answer:** Build/test, publish through a CI/CD pipeline or `func azure functionapp publish`, configure app settings and identity/networking, deploy to a staging slot/environment, smoke-test, then promote while monitoring logs and metrics.
- **Have you worked with Dockerfiles?** *(Source: p. 55.)*  
  **Answer:** Explain how a Dockerfile restores/builds/publishes the app, copies only runtime artifacts into a small non-root runtime image, configures the entry point, and is scanned/tagged by CI. State actual experience accurately.
- **How did you deploy the microservice?** *(Source: p. 55.)*  
  **Answer:** Describe building a versioned image, pushing it to a registry, deploying with Kubernetes/AKS or the actual platform, applying configuration/secrets via managed mechanisms, health probes, rolling rollout, and observability/rollback.
---

## Interview source: 30-wipro.md

### Wipro interview groups

The first-round group (30 April 2026, nine years' experience) and second-round group (8 May 2026) are kept separate. Source page references are 1-based PDF pages.

#### First round — 30 April 2026

##### General and introduction

1. **Tell me about yourself.** *(Source: p. 62.)*  
   **Answer:** Give a concise, truthful summary of relevant experience, current responsibilities, key .NET/Angular strengths, one accomplishment, and the role you seek.
2. **How many years of experience do you have?** *(Source: p. 62.)*  
   **Answer:** State the accurate total and, if useful, distinguish total experience from experience in the role's core technologies.
3. **What technologies have you worked with (e.g., .NET Core, Angular, Entity Framework, SQL Server)?** *(Source: p. 62.)*  
   **Answer:** List technologies actually used, versions, and depth (production ownership, project use, or learning); give one concrete example rather than an unsupported list.
4. **Tell me about your current/last project, role, and responsibilities.** *(Source: p. 62.)*  
   **Answer:** Summarize users/problem, architecture, your ownership, collaboration, delivery, and measurable outcome.
5. **Which .NET and Angular versions have you used?** *(Source: p. 62.)*  
   **Answer:** State the versions used and what you implemented; distinguish those from versions merely studied.
6. **How many projects have you completed using .NET Core and Angular?** *(Source: p. 62.)*  
   **Answer:** Give the truthful count and briefly distinguish project scope/your contribution.
7. **Rate yourself from 1 to 5 in .NET Core, Angular, Web API, and Entity Framework.** *(Source: p. 62.)*  
   **Answer:** Give a defensible rating for each, anchored to evidence and gaps; do not claim a 5 without being prepared for advanced questions.
8. **What architecture and data flow did your last project follow?** *(Source: p. 63.)*  
   **Answer:** Trace a request from Angular through API/controller, application/service/domain/data access, database and back; mention auth, validation, error handling, and any event/message flow. Explain boundaries rather than listing the stack.

##### .NET Core and C#

1. **Abstract class vs interface; give a real project scenario for an abstract class.** *(Source: p. 63.)*  
   **Answer:** An interface defines a contract and supports multiple implementation; an abstract class can share state and implemented/protected behavior with single class inheritance. Use an abstract base only when implementations share a genuine invariant/template; otherwise prefer a focused interface plus composition.
2. **Have you used delegates? Give a real-time scenario.** *(Source: p. 63.)*  
   **Answer:** Delegates are type-safe references to methods and underpin callbacks/events. A real example could be an event handler, a strategy/callback passed to a reusable component, or a `Func<T,...>` predicate—state only an example actually used.
3. **`First`, `FirstOrDefault`, `Single`, `SingleOrDefault` differences.** *(Source: p. 63.)*  
   **Answer:** `First` requires at least one; `FirstOrDefault` permits none; `Single` requires exactly one; `SingleOrDefault` permits zero but rejects multiple. Use single variants to assert uniqueness.
4. **`readonly` vs `const` vs `static`.** *(Source: p. 63.)*  
   **Answer:** `const` is a compile-time constant, implicitly static; `readonly` can be assigned at declaration or in an instance constructor (or static constructor for static readonly); `static` makes a member/type belong to the type rather than an instance and says nothing by itself about mutability.
5. **`String` vs `StringBuilder`.** *(Source: p. 63.)*  
   **Answer:** `string` is immutable, so repeated concatenation can allocate many strings. `StringBuilder` is mutable and efficient for many appends; simple concatenation/interpolation is usually clearer for a small fixed number of parts.
6. **What is middleware and how do you create custom middleware in .NET Core?** *(Source: p. 63.)*  
   **Answer:** An ordered request/response pipeline component. Implement `InvokeAsync(HttpContext, RequestDelegate)`, call `next` to continue, and register with `UseMiddleware<T>()` in the correct order.
7. **What is Kestrel?** *(Source: p. 63.)*  
   **Answer:** ASP.NET Core's cross-platform web server. It can be internet-facing or run behind a reverse proxy/load balancer such as IIS, Nginx, or a cloud ingress.
8. **How do you handle sessions in .NET Core?** *(Source: p. 63.)*  
   **Answer:** Configure a distributed cache and session middleware, set an appropriately secure cookie, and call middleware in correct order. Session is ephemeral state, not a replacement for durable user data; for scalable apps prefer stateless claims or distributed storage where suitable.
9. **How do you implement API versioning?** *(Source: p. 63.)*  
   **Answer:** Select a URL/header/media-type strategy, configure version selection and route contracts, test/document versions, and deprecate old versions on a published schedule.
10. **What is content negotiation in ASP.NET Core Web API?** *(Source: p. 63.)*  
    **Answer:** The client and server select a representation format using request headers such as `Accept`; ASP.NET Core formatters serialize supported formats. Configure supported formatters and return an appropriate response (or 406 if required) when no acceptable format exists.

##### SOLID principles and architecture

1. **What is the Single Responsibility Principle?** *(Source: p. 63.)*  
   **Answer:** A module should have one cohesive responsibility/reason to change. Separate unrelated persistence, business policy, and delivery concerns while avoiding needless fragmentation.
2. **What is the Liskov Substitution Principle?** *(Source: p. 63.)*  
   **Answer:** An implementation of a base type must be usable wherever that type is expected without breaking its documented invariants or behavior. Do not strengthen preconditions or weaken postconditions unexpectedly.
3. **What is dependency injection, and why use it?** *(Source: p. 63.)*  
   **Answer:** Supply a component's dependencies externally, commonly through constructor injection and a DI container. It reduces concrete coupling and supports configuration/testing; it is not itself a guarantee of good design.
4. **Transient vs scoped vs singleton; where and why have you used each?** *(Source: p. 63.)*  
   **Answer:** Transient creates a new instance per resolution; scoped is one per request/scope; singleton is one per provider lifetime. Typical choices: stateless lightweight helper, request/EF `DbContext`, and thread-safe immutable shared service, respectively. Avoid capturing scoped services in singletons.
5. **Can a scoped service be injected into a singleton?** *(Source: p. 63.)*  
   **Answer:** Not directly as a constructor dependency: that promotes scoped state and causes a captive dependency. Create an explicit scope with `IServiceScopeFactory` for bounded work, or redesign lifetime boundaries.
6. **Design a loosely coupled mall billing system for Credit Card, Debit Card, Voucher, with future Cash/Coupons. How many classes/interfaces so adding a method requires minimal changes?** *(Source: pp. 63–64.)*  
   **Answer:** Define a `IPaymentMethod` contract (`Pay(amount, context)`), implementations such as `CreditCardPayment`, `DebitCardPayment`, and `VoucherPayment`, and a `CheckoutService` that uses the selected abstraction. Resolve the strategy by an injected registry/factory keyed by a validated payment type; add a new implementation/registration without changing checkout logic. Add validation/refund contracts only if all methods genuinely support them. Avoid a growing `if/else` or switch inside checkout.

##### Security and authentication

1. **What authentication and authorization mechanisms have you used?** *(Source: p. 64.)*  
   **Answer:** Distinguish how identity is verified (e.g., OIDC/JWT/cookie) from access decisions (roles, claims, policies/resource checks). Describe the actual mechanism, token validation, least privilege, and secure secret handling.
2. **How do you generate a JWT in Web API?** *(Source: p. 64.)*  
   **Answer:** Authenticate the user, issue claims with minimal data, set issuer/audience/expiry, and sign using a securely managed key or identity provider. Return over TLS; validate signature, issuer, audience, and lifetime on protected endpoints. Prefer a trusted OIDC provider rather than hand-rolling token issuance.

##### SQL Server and databases

1. **How do you improve SQL Server query performance?** *(Source: p. 64.)*  
   **Answer:** Measure the query and inspect actual execution plan, reads, waits, and statistics. Add suitable indexes, avoid unnecessary columns/rows and non-sargable predicates, fix join/cardinality issues, update statistics as appropriate, and remeasure.
2. **Clustered vs non-clustered index.** *(Source: p. 64.)*  
   **Answer:** A clustered index defines the table's row storage order/organization (one per table); a nonclustered index is a separate structure with keys and row locators (many possible). Both affect reads and write/storage cost.
3. **What is a covering index?** *(Source: p. 64.)*  
   **Answer:** An index containing all columns a query needs for its predicates, joins, and output, allowing it to be satisfied without a base-table lookup. Use included columns carefully to avoid oversized indexes.
4. **What is a CTE? Have you used one inside a stored procedure and why?** *(Source: p. 64.)*  
   **Answer:** A named expression scoped to one following statement, useful for readability, reuse within that statement, and recursion. It can be used in a stored procedure, but is not automatically materialized; use a temp table when intermediate materialization/indexing/statistics or reuse across statements is needed.
5. **Temporary table `#temp` vs table variable `@table`.** *(Source: p. 64.)*  
   **Answer:** Both are temporary relational structures. A `#temp` table is in `tempdb`, supports indexes/statistics and is often better for larger/intermediate sets; a table variable has different scope/compilation behavior and may suit small sets. Modern SQL Server has deferred compilation improvements, so benchmark actual workload.
6. **`ROW_NUMBER()` vs `DENSE_RANK()`.** *(Source: p. 64.)*  
   **Answer:** `ROW_NUMBER` assigns a unique sequential number even to tied sort values; `DENSE_RANK` gives ties the same rank and has no gaps after ties. Use a deterministic tiebreaker for stable row numbering.
7. **Coding challenge: find the second-highest salary department-wise from `Employee(EmployeeID, Salary, DepartmentID)` and `Department(DepartmentID, DepartmentName)`; show employee name, salary, department name.** *(Source: p. 64.)*  
   **Answer:** The stated schema lacks employee name; assume it is `EmployeeName`. Use `DENSE_RANK` to include ties at the second distinct salary:
   ```sql
   WITH Ranked AS (
       SELECT e.EmployeeName, e.Salary, e.DepartmentID,
              DENSE_RANK() OVER (
                  PARTITION BY e.DepartmentID ORDER BY e.Salary DESC
              ) AS SalaryRank
       FROM Employee e
   )
   SELECT r.EmployeeName, r.Salary, d.DepartmentName
   FROM Ranked r
   JOIN Department d ON d.DepartmentID = r.DepartmentID
   WHERE r.SalaryRank = 2;
   ```
   If exactly one employee per department is requested, use `ROW_NUMBER()` with a deterministic tie-breaker.

##### Entity Framework

1. **Difference between eager, explicit, and lazy loading in Entity Framework.** *(Source: p. 65.)*  
   **Answer:** Eager loading retrieves related data as part of the initial query (e.g., `Include`); explicit loading fetches it later when specifically requested; lazy loading fetches it on navigation-property access when enabled. Choose based on needed data and query shape; lazy loading can hide round trips.
2. **What is the N+1 query problem? Does it happen with eager or lazy loading?** *(Source: p. 65.)*  
   **Answer:** One query loads a parent list, followed by one additional query per parent for related data. It commonly results from lazy loading or per-row queries; eager loading/projection can prevent it, but a badly shaped eager query can produce excessive joins/data. Inspect generated SQL.

##### Angular client-side

1. **How can unrelated components transfer data without a direct parent-child relationship?** *(Source: p. 65.)*  
   **Answer:** Use a shared injectable service/store with observable state (or a state-management library); route/query state or a backend may be more appropriate for durable/shareable data. Keep ownership and subscription lifecycles clear.
2. **`Subject` vs `BehaviorSubject` vs `ReplaySubject`.** *(Source: p. 65.)*  
   **Answer:** `Subject` broadcasts only future emissions; `BehaviorSubject` requires and retains a current value for new subscribers; `ReplaySubject` replays a configurable buffer/window of past emissions.
3. **What is an Angular interceptor and why use it?** *(Source: p. 65.)*  
   **Answer:** An HTTP pipeline handler for cross-cutting request/response behavior such as adding auth headers, correlation IDs, or centralized error handling. Keep retries/auth refresh bounded and avoid intercepting the refresh request recursively.
4. **How do you keep tokens such as JWT in Angular?** *(Source: p. 65.)*  
   **Answer:** Prefer an OIDC authorization-code-with-PKCE flow and short-lived tokens; where architecture allows, a backend-for-frontend with secure, HttpOnly, SameSite cookies reduces JavaScript exposure. Avoid persistent browser storage for high-value bearer tokens because XSS can read it; apply CSRF controls for cookies.
5. **What is the `standalone` property in Angular components and why use it?** *(Source: p. 65.)*  
   **Answer:** A standalone component/directive/pipe can be used without declaring it in an NgModule; it imports its own dependencies and simplifies composition, lazy loading, and incremental migration.
6. **How do you improve Angular application performance?** *(Source: p. 65.)*  
   **Answer:** Profile first; lazy-load routes, use `OnPush`/signals appropriately, track list items, avoid unnecessary change detection and subscriptions, optimize bundle/images, paginate/virtualize large lists, and cache server data safely.
7. **Observable vs Promise.** *(Source: p. 65.)*  
   **Answer:** A Promise represents one eventual value and begins when created; an RxJS Observable can emit zero or many values, is generally lazy until subscribed, and supports cancellation/unsubscription and stream operators.

##### Cloud services — Azure / AWS

1. **Have you used Azure or AWS, and which services?** *(Source: p. 65.)*  
   **Answer:** Name only services actually used and explain each role, identity/networking, deployment, and monitoring. Examples might include App Service/AKS, Azure SQL, Storage, Key Vault, Functions, or AWS equivalents.
2. **Have you worked with Azure Functions or AWS Lambda?** *(Interviewer clarification; Source: p. 65.)*  
   **Answer:** State direct experience accurately; describe trigger, scaling, retries/idempotency, timeout limits, secrets/identity, and monitoring. If inexperienced, outline a small suitable event-driven use case and its trade-offs.

#### Second round — 8 May 2026

##### Architecture and design principles

1. **Explain your recent project's application architecture, beyond listing the stack.** *(Source: p. 66.)*  
   **Answer:** Explain components, boundaries, ownership, request/event flow, data stores, deployment, security, failure handling, and why those choices fit the requirements.
2. **Clean Architecture vs traditional three-layer architecture.** *(Source: p. 66.)*  
   **Answer:** Both separate concerns. Three-layer designs commonly organize presentation, business, and data layers, often with dependencies flowing downward. Clean Architecture emphasizes domain/application policies at the center and dependency inversion so infrastructure depends on inner abstractions. It is a more explicit dependency rule, not a synonym for “more layers.”
3. **What is Layered Architecture and how does it work?** *(Source: p. 66.)*  
   **Answer:** Divide responsibilities into layers (often presentation, application/business, and data/infrastructure), with defined dependencies and interfaces. Requests flow through layers; enforce boundaries and avoid leaking persistence/UI concerns into domain rules.

##### Angular client-side

1. **How do you refresh an expired token using an HTTP interceptor?** *(Source: p. 66.)*  
   **Answer:** On an eligible 401, call a refresh endpoint once, update credentials, and retry the original request once; if refresh fails, clear auth state and reauthenticate. Use secure token storage/refresh rotation, avoid retry loops, and exclude the refresh endpoint itself.
2. **When a 401 occurs while the user is active, how do `switchMap`, `catchError`, and other RxJS logic enable seamless refresh?** *(Source: p. 66.)*  
   **Answer:** Centralize refresh with a shared in-flight observable/queue so parallel 401s do not race. `catchError` detects 401; `exhaustMap` or a shared refresh subject can ensure one refresh, then `switchMap` attaches the new token and retries the original request once. Clear the in-flight state and propagate errors; guard against refresh recursion and replay only safe requests or requests with idempotency protection.
3. **How do you make a `UserService` global singleton so it maintains state across screens during CRUD?** *(Source: p. 66.)*  
   **Answer:** Provide it at root with `@Injectable({ providedIn: 'root' })`, hold shared state in a well-defined observable/signal/store, and update it consistently after CRUD responses. Do not assume singleton memory survives page reloads or multiple browser tabs.
4. **What does `@Injectable()` do, and what does `providedIn: 'root'` mean?** *(Source: p. 66.)*  
   **Answer:** `@Injectable` marks a class for Angular DI and can specify where it is provided. `providedIn: 'root'` registers one app-root-scoped instance (tree-shakable when unused), typically shared by the application.

##### .NET Core and C# server-side

1. **What is middleware in .NET Core?** *(Source: p. 66.)*  
   **Answer:** An ordered component in the HTTP request/response pipeline that can inspect, transform, short-circuit, or call the next component; registration order affects behavior.
2. **For a new Customer Module in an existing .NET Core API, what files/layers would you create?** *(Source: p. 66.)*  
   **Answer:** Add only needed boundaries: request/response DTOs and validation, controller/endpoint, application service/use case, domain rules/entities, repository abstraction/implementation if persistence warrants it, EF mapping/migration, DI registration, authorization, and unit/integration tests. Keep controller thin and avoid creating redundant layers mechanically.
3. **Where and how do you configure connection strings and app settings?** *(Source: p. 66.)*  
   **Answer:** Bind configuration through `IConfiguration`/options. Use `appsettings.json` for non-secret defaults and environment-specific configuration, environment variables/secret manager for local secrets, and Key Vault/managed identity or platform secret storage in production. Never commit production credentials.

##### LINQ and EF Core

1. **Write LINQ to fetch the third-highest salaried employee from a list of customers/employees.** *(Source: p. 67.)*  
   **Answer:** The wording likely means employees; assuming third distinct salary:
   ```csharp
   var employee = employees
       .OrderByDescending(e => e.Salary)
       .Select(e => new { Employee = e, e.Salary })
       .DistinctBy(x => x.Salary) // .NET 6+
       .Skip(2)
       .Select(x => x.Employee)
       .FirstOrDefault();
   ```
   For `IQueryable`, use a database-translatable grouped/ranked query instead of relying on `DistinctBy`; define how ties should be handled.
2. **How do you handle equal salaries for N-th highest with `Skip()`/`Take()`? Why might `FirstOrDefault()` fail?** *(Source: p. 67.)*  
   **Answer:** `Skip(n-1)` over employees counts rows, so ties can shift the intended rank; order distinct salary values or group by salary first. For all employees at the N-th distinct salary, get the N-th salary then filter by it. `FirstOrDefault()` returns only one row and cannot return all tied employees; it also makes missing results indistinguishable from default unless handled explicitly.

##### SQL Server and databases

1. **How do you design indexes for a large 30-column table? What factors/query patterns do you analyze?** *(Source: p. 67.)*  
   **Answer:** Start from workload: frequent predicates, joins, sorts/grouping, selectivity, cardinality, key order, included columns, covering needs, and write/update cost. Inspect execution plans and existing indexes, avoid indexing every column, account for composite leading-key order and filtered indexes, then measure storage and read/write impact.
2. **For a stored procedure over 1M+ rows with complex calculations, use a CTE or global temp table (`##temp`), and why?** *(Source: p. 67.)*  
   **Answer:** It depends on reuse and plan needs. A CTE is a statement-scoped logical expression, not necessarily materialized; use it for a single readable transformation. A local `#temp` table is often preferable when materializing/reusing intermediate results, adding indexes, or getting statistics helps complex multi-step work. Do not default to global `##temp`: it risks name/session collisions and unnecessary cross-session visibility.
3. **Remove duplicate records from `User(Email, UserID, UserName)` with no primary key, using a CTE and `PARTITION BY`.** *(Source: p. 67.)*  
   **Answer:** The prompt does not define what makes two rows duplicates or how to identify one physical row, which is essential because there is no key. Assuming duplicates mean same `Email` and `UserName`, and `UserID` can choose a survivor:
   ```sql
   WITH D AS (
       SELECT *,
              ROW_NUMBER() OVER (
                  PARTITION BY Email, UserName
                  ORDER BY UserID
              ) AS rn
       FROM dbo.[User]
   )
   DELETE FROM D WHERE rn > 1;
   ```
   Confirm duplicate semantics, use a transaction/backup, and add a real key/unique constraint after cleanup; if duplicate rows are identical and no unique locator exists, safe deletion is under-specified.
4. **How many types of indexes are available in SQL Server?** *(Source: p. 67.)*  
   **Answer:** There is no single useful fixed count because classification dimensions overlap. Common types/features include clustered, nonclustered, unique, filtered, columnstore (clustered/nonclustered), XML, spatial, full-text, hash indexes for memory-optimized tables, and indexes on views. Clarify whether the interviewer means rowstore types or all index families.

##### Azure cloud services

1. **Which Azure services have you used for deployment and hosting?** *(Source: p. 67.)*  
   **Answer:** Name actual services (for example App Service, AKS, Container Apps, Functions) and describe deployment, networking, identity, scaling, and monitoring in the project.
2. **HTTP Trigger vs Timer Trigger in Azure Functions.** *(Source: p. 67.)*  
   **Answer:** HTTP trigger starts from an HTTP request and can return an HTTP response; timer trigger runs on a schedule using a cron expression and has no caller request/response. Both need idempotency/retry-aware behavior.
3. **Write a basic HTTP-triggered Azure Function that adds a customer.** *(Source: p. 68.)*  
   **Answer:** A minimal isolated-worker .NET example, with storage/persistence intentionally represented by an injected service:
   ```csharp
   public sealed record CreateCustomer(string Name, string Email);

   public sealed class CustomersFunction(ICustomerService customers)
   {
       [Function("AddCustomer")]
       public async Task<HttpResponseData> Run(
           [HttpTrigger(AuthorizationLevel.Function, "post", Route = "customers")]
           HttpRequestData request,
           CancellationToken cancellationToken)
       {
           var input = await request.ReadFromJsonAsync<CreateCustomer>(
               cancellationToken: cancellationToken);
           if (input is null || string.IsNullOrWhiteSpace(input.Name) ||
               string.IsNullOrWhiteSpace(input.Email))
           {
               var bad = request.CreateResponse(HttpStatusCode.BadRequest);
               await bad.WriteStringAsync("Name and email are required.", cancellationToken);
               return bad;
           }

           var customer = await customers.AddAsync(input, cancellationToken);
           var response = request.CreateResponse(HttpStatusCode.Created);
           await response.WriteAsJsonAsync(customer, cancellationToken);
           return response;
       }
   }
   ```
   `ICustomerService` is an application dependency registered in the Functions host; validate input and enforce authorization/uniqueness in production.
---

## Interview source: 30-wolters.md

### Wolters interview

This interview is recorded as a conversational microservices, Azure, Kubernetes, and Docker discussion. The prompts below preserve its follow-ups; answers correct oversimplifications in the source where needed. Source pages are 1-based PDF pages.

#### Microservices and distributed design

1. **What is your approach to designing a microservice pattern—how do you break down a project and design the solution?** *(Source: p. 58.)*  
   **Answer:** Start with business capabilities and domain discovery, identify bounded contexts and data ownership, then define service contracts, communication patterns, reliability and security needs. Prefer a modular monolith until independent deployment/scaling justifies services; extract incrementally and observe the system.

2. **How do you define bounded contexts? What do you do when contexts overlap?** *(Source: pp. 58–59.)*  
   **Answer:** Use domain language, rules, ownership, and change boundaries to model each context with domain experts. Overlap usually indicates separate models or an explicit integration contract—not a shared writable database. Integrate through APIs/events or a well-defined anti-corruption layer; accept eventual consistency where appropriate.

3. **If there are overlaps, should there be a common database and is that event sourcing/eventual consistency? How do you capture the sequence of events?** *(Source: pp. 58–59.)*  
   **Answer:** A common database is not event sourcing and tends to couple services. Event sourcing stores domain events as the authoritative history and derives state/projections; eventual consistency is a consistency property, not the event log itself. For ordinary integration, publish versioned domain/integration events reliably (often via transactional outbox), make consumers idempotent, and monitor/replay failures.

4. **Explain practically how you implement the Saga pattern: how many services, separate or embedded coordinator, and compensation when a step fails?** *(Source: p. 59.)*  
   **Answer:** Model a business transaction as local transactions across the participating services, each with a compensating action where feasible. For a small flow, choreography can use events; for complex flows, an orchestrator/workflow service or durable orchestration in an owning service tracks state, retries, timeouts, and compensations. A saga is not a global ACID “commit all or nothing”; compensation is a new business action and may fail or need manual resolution.

5. **Where does saga logic sit, and who is responsible—the consumer, producer, or both?** *(Source: p. 59.)*  
   **Answer:** Saga ownership belongs to the business process coordinator/orchestrator or to participants in choreography. Producers own publishing valid facts; consumers own their local transaction and idempotent handling. Do not put a generic saga implementation at every service-to-service call; define one process owner and explicit participant contracts.

6. **What is event sourcing?** *(Source: p. 59.)*  
   **Answer:** Persist state changes as an append-only sequence of domain events; rebuild current state by replaying events and create read projections as needed. It supports audit/history but brings schema evolution, replay, projection, and operational complexity; it is not required merely to achieve eventual consistency.

#### Azure networking and database capacity

7. **How do you connect a .NET Core Web API to an Azure resource that is private and unavailable over the public internet?** *(Source: pp. 59–60.)*  
   **Answer:** Place the app in a network path that can reach the resource (e.g., VNet integration/private endpoint and private DNS, or a private AKS subnet); configure routing/firewall rules and least-privilege identity such as managed identity. Keep credentials in Key Vault when credentials are unavoidable; validate DNS, NSG/firewall, and identity separately.

8. **High request volume causes many SQL connections, but every request must be served. How do you design this under SQL Server's connection limit?** *(Source: pp. 59–60.)*  
   **Answer:** ADO.NET connection pooling reuses physical connections; open late and dispose promptly, configure pool sizes deliberately, and limit concurrent database work with backpressure/queues rather than dropping every excess request. Reduce database demand with efficient queries, caching, batching, read replicas where suitable, and scale/tune the database. CQRS may separate read/write models but does not itself cap connections or solve overload; design bounded queues, timeouts, and graceful admission/response behavior.

#### Kubernetes, AKS, and storage

9. **What is your design strategy for Kubernetes deployed in a VNet: how many node pools and microservices, and how do you deploy them?** *(Source: p. 60.)*  
   **Answer:** Size pools from workload/resource/security needs, not a fixed service count. Use separate system and user pools where appropriate; isolate GPU, bursty, or regulated workloads only when justified. Define deployments, resource requests/limits, probes, autoscaling, disruption budgets, network policy, ingress, secrets/identity, and CI/CD per service. In AKS/VNet, plan subnet capacity, routing, DNS, and private access.

10. **Can one pod contain multiple containers of different types?** *(Source: p. 60.)*  
    **Answer:** Yes. A pod can contain multiple cooperating containers (for example, an app plus a sidecar proxy/log shipper), potentially with different images. They share network namespace/IP and can share mounted volumes; unrelated applications should normally be separate pods for independent scaling and lifecycle.

11. **An external HTTP/mobile request reaches ingress, which routes it to a service. Explain the flow.** *(Source: p. 60.)*  
    **Answer:** The client reaches the ingress controller/load balancer; the controller applies host/path/TLS rules and forwards to a Kubernetes Service. The Service selects ready pods, and cluster networking routes to an endpoint. Configure probes, timeouts, TLS, authentication, and observability; ingress is not itself the application service.

12. **How does a pod connect to Azure Storage or Key Vault?** *(Source: p. 60.)*  
    **Answer:** Prefer workload identity/federated identity (AKS Workload Identity) to obtain short-lived tokens for a least-privilege managed identity. Use private endpoints/private DNS and network policies/firewalls where required. Avoid embedding account keys or client secrets in images or plain environment variables.

#### Engineering practices and containers

13. **Do SOLID principles apply to microservices?** *(Source: p. 61.)*  
    **Answer:** Yes, as design heuristics for code and service boundaries, but apply them pragmatically. Cohesion, substitutable contracts, and dependency inversion help; SOLID does not imply every class should be a service or that distributed boundaries should mirror class boundaries.

14. **What is the Sidecar design pattern?** *(Source: p. 61.)*  
    **Answer:** A companion process/container deployed alongside an application to provide supporting capabilities such as proxying, telemetry, or certificate handling. It shares the pod's network and may share volumes. It can simplify application code but adds resource, configuration, and operational overhead.

15. **Do you know Docker containers, and how do you create one?** *(Source: p. 61.)*  
    **Answer:** A container packages an application and its user-space dependencies while sharing the host kernel. Write a Dockerfile, build with `docker build`, run with `docker run`, and publish to a registry. Use a minimal trusted base image, non-root user, pinned dependencies, health/termination behavior, and scan/sign the resulting image.

16. **How do you choose a Docker image for a microservice? Is tagging the criterion?** *(Source: p. 61.)*  
    **Answer:** Choose a compatible, supported base/runtime image based on the service's runtime, architecture, security updates, size, and operational requirements; prefer trusted minimal images and pin an immutable digest for reproducibility. Tags are labels/version references, not the decision criterion and can be mutable. Build each service image from its actual runtime needs.

17. **You integrated Azure OpenAI—what was the use case?** *(Source: p. 61.)*  
    **Answer:** Give a truthful, specific example (e.g., grounded internal document Q&A). Explain the data flow, access controls, prompt/content filtering, evaluation, citation/abstention, latency/cost limits, and human fallback; if no direct project use, say so and describe a considered design instead.

18. **What is spec-driven development?** *(Source: p. 61.)*  
    **Answer:** Define and review an executable or machine-readable specification (such as API schema, acceptance criteria, or contract) before/during implementation, then generate/check tests and code against it. It improves shared understanding but specifications still need versioning, review, and validation against actual behavior.

---

## Complete extracted text: RealTimeInterviewQuestions.pdf

All PDF pages are retained below in page order so no interview prompt or context is omitted. The text is extracted with layout preservation; the original PDF remains the authoritative source for visual formatting.

### PDF page index

- [Pages 1–10](#pdf-page-1) · [Pages 11–20](#pdf-page-11) · [Pages 21–30](#pdf-page-21) · [Pages 31–40](#pdf-page-31) · [Pages 41–50](#pdf-page-41) · [Pages 51–60](#pdf-page-51) · [Pages 61–70](#pdf-page-61)

### PDF page 1

                                                          www.questpod.com


        C#,ASP.NET ,SQL , Azure , Microservices ,
        Architecture , AI ML interview Questions
                                                                    Compiled by

                                                                 www.questpod.com

                            Contributed by India’s Largest .NET whatsapp community
                                                       Do join and let’s hunt jobs together.

                                                               http://whatsapp.questpond.com



Contents
What is this Document ? ......................................................................................................................... 2
Structure of Document ........................................................................................................................... 3
Some common problems ........................................................................................................................ 3
How to prepare ? .................................................................................................................................... 4
Most asked Questions Across Companies .............................................................................................. 4
   OOP / SOLID ........................................................................................................................................ 4
   C# ........................................................................................................................................................ 5
   JAVASCRIPT ......................................................................................................................................... 7
   ANGULAR ............................................................................................................................................ 8
   .NET CORE WEB API ............................................................................................................................ 9
   ORM .................................................................................................................................................. 10
   Azure ................................................................................................................................................. 11
   SQL .................................................................................................................................................... 12
   Design Pattern and Architecture ...................................................................................................... 13
   SCENARIO BASED .............................................................................................................................. 14
   API Testing ........................................................................................................................................ 15
   SCENARIO BASED .............................................................................................................................. 16
   One of the Big multinational company ............................................................................................ 21
   CODING QUESTIONS ......................................................................................................................... 22
Company specific Interview question and experiences........................................................................ 24

### PDF page 2

                                                        www.questpod.com
  GlobalLogic Interview Questions : .................................................................................................... 24
  LTI Mindtree ...................................................................................................................................... 28
  EPAM ................................................................................................................................................. 31
  Carelon .............................................................................................................................................. 31
  Accenture (1st Round) ...................................................................................................................... 32
  Cognizant .......................................................................................................................................... 33
  Deloitte ............................................................................................................................................. 34
  GlobalData ........................................................................................................................................ 35
  Persistent - Architect interview questions asked to me ................................................................... 36
  CapGemini ......................................................................................................................................... 37
     5 years CapGemini exp.................................................................................................................. 37
     Capgemini Azure Lead .................................................................................................................. 38
  Persistent Azure Dotnet Role ............................................................................................................ 38
  Mhpasis ............................................................................................................................................. 39
     1st round ....................................................................................................................................... 39
     2nd round...................................................................................................................................... 40
  Virtusa ............................................................................................................................................... 47
     Virtusa Interview Process (Lead Software Engineer, YOE 10+) .................................................... 47
     Virtusa 11/June/2026 ( Microservices and Docker Focussed) ..................................................... 54
  TCS Consultancy 20 +LPA Package (.NET Full Stack + Al) .................................................................. 55
  TCS Experience 2 Special Notes from a developer. .......................................................................... 58
  Wolters interview ............................................................................................................................. 58
  Infosys ............................................................................................................................................... 62
  WIPRO ............................................................................................................................................... 62
     Wipro 1st round held on 30th April 2026 ..................................................................................... 62
     Wipro 2nd round held on 8th May 2026 ...................................................................................... 65
  FN(LastLetterAlphabet) 4 years Questions ....................................................................................... 68



What is this Document ?

This document contains some of the most frequently asked interview questions for C#, ASP.NET, SQL
Server, Angular, Azure, Microservices, React, Coding, and other technologies commonly expected
from a Microsoft Full Stack Developer.

### PDF page 3

                                     www.questpod.com
The questions in this document are not randomly collected from the internet. They are based on real
interview experiences shared by members of the Questpond WhatsApp community. You can join our
community using ( http://whatsapp.questpond.com )

Over the years, hundreds of developers from different companies have shared the questions they
faced during technical interviews. We have consolidated those experiences, identified the commonly
repeated patterns, and compiled the most frequently asked questions into this document.

The objective of this document is to help C# and .NET developers understand current market
expectations, prepare for interviews more effectively, and focus on topics that are repeatedly asked
by recruiters and technical panels.

Whether you are preparing for interviews in C#, ASP.NET Core, SQL Server, Angular, Azure,
Microservices, React, or Coding Rounds, this document serves as a practical reference based on real-
world interview feedback from the Questpond community.



Structure of Document

This document is divided into two sections.

Section 1: Most Asked Questions

This section contains the most frequently asked interview questions across multiple companies. We
have analyzed interview experiences shared by developers in the Questpond WhatsApp community
and consolidated the common question patterns that repeatedly appear in technical interviews. If
your goal is general interview preparation, this is the section you should focus on first.

Section 2: Company-Specific Questions

This section contains questions reported by candidates who attended interviews at specific
companies. For example, if you have an interview scheduled with Infosys, TCS, Accenture, Cognizant,
Capgemini, or any other company listed in this document, you can directly refer to that company's
section and review the questions shared by candidates who have already appeared for interviews
there.

In short, if you want to prepare broadly for the market, focus on the Most Asked Questions section.
If you are targeting a specific company, review the corresponding Company-Specific Questions
section for a more focused preparation.



Some common problems


15 days’ Notice period ISSUES.

### PDF page 4

                                     www.questpod.com
Interview Ghosting



How to prepare ?

If you are preparing for these questions by simply reading one-line answers, you may face difficulties
during interviews. In most interviews, the interviewer will ask follow-up and cross-questions to test
your understanding of the fundamentals.

The best way to build a strong foundation is by watching the interview question videos provided
below. My recommendation is to first watch the videos and then go through the questions. Once
you see the concepts explained with examples and demos, it becomes much easier to answer cross-
questions confidently.

I want to emphasize this point: once you watch a DEMO, handling follow-up questions becomes
significantly easier.



30 Important C# Interview Questions : https://youtu.be/BKynEBPqiIM
Software Architecture and Design Patterns Interview Questions : https://youtu.be/AtTgcbLOqMM
OOPS Interview Questions and Answers :- https://youtu.be/u99wAoBjDvQ
35 Important JavaScript Interview Questions : https://youtu.be/Zb4dPi7CANU
25 Important ASP.NET Interview Questions : https://youtu.be/pXmMdmJUC0g
25 Angular Interview Questions : https://youtu.be/-jeoyDJDsSM
20+ SQL Server Interview Questions : https://youtu.be/SEdAF8mSKS4




Most asked Questions Across Companies

OOP / SOLID

    1) What is method signature?
    2) What is the Order of constructor execution?
    3) What is destructor and how many can be there in a class( only 1)?
    4) Why do you need a destructor when GC is there ?
    5) What happens when constructor is private?
    6) If an Abstract class has concrete method so can its child class have same method name and
       can it modify it?
    7) Abstract vs Static class
    8) Abstract vs Interface
    9) Why abstract class has constructors

### PDF page 5

                                     www.questpod.com
     10) When to use abtract and when to use interface give real time example in c#
     11) Implicit and explicit interface
     12) What is default interface and why do we need it ?
     13) Polmorphism in c#
     14) Method overriding with example
     15) Difference b/w Virtual, Override and new keywords in C# ?
     16) Method hiding with example
     17) Is return type considered in method overloading?
     18) Why and when to use method hiding? give real time scenario
     19) How to extend a sealed class?
     20) What is extension method? Write an example
     21) What is solid principle and why to implement?
     22) State all the principles with example
     23) What is Liskovs Substitute Principle, explain with example
     24) Difference in Open close and Liskovs principle
     25) Diff between Liskov vs ISP ?
     26) Calculate salary of employee based on grades (strategy pattern)




C#
THEORY QUESTIONS -

     1) Value type vs Ref type.
     2) Ref vs Out, give real time example and usage.
     3) Can ref keyword be used in Async programming?
     4) Const vs Readonly.
     5) What is garbage collector?
     6) How to remove unmanaged code?( How we remove garbage in unmanaged code?)
     7) Throw vs Throw ex.
     8) What are access modifiers?
     9) What is Partial class give real time scenario.
     10) What are generics? Have you implemented any in your project?
     1) What are collections and its type.
     2) What is Dictionary, List, HashTable.
     3) What is Stack and Queue?

### PDF page 6

                                 www.questpod.com
4) Difference between Generics and non generics?
5) What is Concurrent collection?
6) What are delegates and built in delegates.
7) What is LINQ?
8) Single vs SingleOrDefault - First vs FirstOrDefault - Select vs SelectMany
9) LINQ Queries practice , left join , right , inner . get 3rd highest
10) What is the effect of combining AsParallel with LINQ queries?
11) IEnumerable vs toList.
12) IQueryable vs toList.
13) IQueryable vs IEnumerable.
14) What is yeild keyword?
15) break vs continue vs return.
16) Reflections in c#.
17) What is TPL?
18) TPL vs Thread
19) Async vs Thread
20) How to start a Task?
21) Does Async create threads ?
22) what is async and await?
23) difference between Async/Await and TPL.
24) how async await work under the hood?
25) difference between ValueTask vs Task
26) what is ThreadPool Starvation?
27) ISyncStateMachine
28) What is structure of query pipeline in EF ?
29) what latest .net version you have used and whats is special feature you have seen?
30) When we create a new .NET 6 or .NET 7 Console Application, there is no explicit Main
    method in the Program.cs file. Why is that, and how does the application know where to
    start execution?"
31) What is a concurrency issue in an application, and how do you handle it? Is this related to
    performance?
32) How to do memory optimization ?
33) Suppose the database layer is already optimized, but performance issues still exist on the
    application side. What steps would you take to improve performance at the application
    level?
34) If an application has heavy database interactions, how can we optimize performance—
    whether it's a legacy application, a modern Web API, or a front-end app like React?
35) Imagine there’s a production issue with very limited information available. How would you
    approach diagnosing and resolving the root cause of the issue?
36) Which code will perform better ?

### PDF page 7

                                   www.questpod.com




JAVASCRIPT

THEORY QUESTIONS

   1) difference in reassign and redeclaration
   2) var vs let vs const
   3) types of object creation in js
   4) what is hoisting concept in js?
   5) what do you mean by temporal dead zone?
   6) what is closure in js?
   7) what is a prototype?
   8) what is spread operator?
   9) what is rest operator?
   10) Difference between spread and rest operator
   11) splice vs slice
   12) What is arrow function?
   13) what is anonymous function?
   14) What is array and its types?
   15) How to flatten multi-dimentional array?
   16) = vs == vs === state the name of operators and explain the use

### PDF page 8

                                   www.questpod.com

ANGULAR
THEORY QUESTIONS -



   1) Why Angular?
   2) Which is the latest version going on?
   3) Difference JS and Angular?
   4) How to install angular?
   5) What is AOT?
   6) What are decorators?
   7) How do you create a component?
   8) How does angular app starts?
   9) Package.json vs package.lock.json file?
   10) How to download or install other packages?
   11) What is angular.json file?
   12) What are modules?
   13) What is ngModel?
   14) What is routing and how it is implemented?
   15) Lazy loading in routing?
   16) What are directives and its type. Explain with real examples
   17) How to create custom Directives?
   18) Types of data binding in angular with examples
   19) What is string interpollation?
   20) Difference between string interpollation and Property binding
   21) What is Http Interceptor? How to implement it?
   22) What are services in Angular?
   23) Dependency injection in Angular
   24) Ways to pass data from one component to another?
   25) List down all Life cycle hooks in sequence with examples
   26) Constructor vs ngOnInit
   27) ngOnChanges vs ngDoCheck
   28) What is ViewChild?
   29) How to create api calls in Angular?
   30) What are HTTP filters in Angular?
   31) How to add http headers or authorization in post call?
   32) What are Pipes and its type?
   33) How to create Pipes?
   34) Promise vs Observables
   35) Observables vs Subject
   36) Different Types of Subject
   37) Can we do asynchronous programing in Angular? How?
   38) How to implement asynchronous programing in Angular?
   39) Template driven forms and Reactive forms with their difference
   40) What is auth guard in angular?
   41) Types of auth guards in angular.

### PDF page 9

                                   www.questpod.com
  42) What is local storage?
  43) Have you used RXJS? why it is used?
  44) Give some operators of RXJS
  45) What is map operator?
  46) What is Template Reference Variable?
  47) How to do testing in Angular?
  48) State management in Angular
  49) Angular new features




.NET CORE WEB API

  1) Framework vs Core
  2) Advantages/features of .net core
  3) Disadvantages of web api
  4) How does .net core starts?
  5) What is middleware in dot net core
  6) Minimal API
  7) How did you create a customer middle ware ?
  8) what is request delegate?
  9) app.Use vs app.Run
  10) What are filters in dot net core?
  11) What is Authorization filter?
  12) What is Exception filter?
  13) What is Action filter?
  14) Middleware vs filter?
  15) Configure vs configure Services?
  16) Dot net core 5.0 vs 6.0
  17) Can we implement startup.cs file in 6.0?
  18) What is dependency injection and types
  19) What is service life cycle? (addTransient, addSingleton,addScoped), state thier difference
  20) What is Routing and its types
  21) What is an Attribute?
  22) Serialization and Deserialization
  23) How to implement Global exception?
  24) What is session management?
  25) What is caching and type of caching and how to implement it in dot net core?
  26) What is MVC pattern?
  27) What is MVVP pattern?
  28) What is CORS policy?
  29) Why CORS no issue in postman but in browser.

### PDF page 10

                                   www.questpod.com
   30) What is Restful API?
   31) SOAP vs REST
   32) What is JSON?
   33) JSON vs XML
   34) POST vs PUT ( Do not use records , idempotet)
   35) If We can modify resource via Post call then why we need PUT call?
   36) Can GET call have request body?
   37) Can we add resource using GET call?
   38) What is Content negotiations in Web API ?
   39) If REST API is slow what will you do , where will you hunt , how to optimize ?
   40) What is Api Versioning and its type? How to implement it
   41) Return types in .net core?
   42) Can we have multiple return types in net core?
   43) How to implement sorting and paging in .net core?
   44) How to implement Authentication/Authorization in dot net core?
   45) What is JWT? State its parts. How it is implemented?
   46) JWT logout.
   47) When should we use Openid , Oauth as compared to Custom JWT ?
   48) How to save sensitive data in database?
   49) What are some common encryption techniques?
   50) What data encryption techniques you have used in project?
   51) What is API gateway?
   52) What is Rate limiter?
   53) Suppose There is a requirement to upload a file with lakhs of records in database from mvc.
   54) How to do it so transaction will be faster
   55) Can you explain the MVC lifecycle and describe the flow of a request from the browser to
       the server and back?
   56) What is cache management? How can it be implemented in a React front-end application?
       How do we manage caching in an ASP.NET MVC web application? What about in an ASP.NET
       Core Web API?



ORM
THEORY QUESTIONS

   1) What is ORM?
   2) Have you used any ORM in your project ? Which one?
   3) What is Dapper?
   4) What is EF core?
   5) EF core vs framework?
   6) What are navigation property in EF core?
   7) How did you implement EF core with relational database?
   8) Which approach did you follow in your project?
   9) Code first vs Database first?
   10) Commands used for code first

### PDF page 11

                                    www.questpod.com
  11) What are fluent APIs?
  12) How do you call a Stored Procedure from EF core?
  13) What is lazy loading
  14) Eager loading in EF core
  15) 3.Tracking and Non Tracking in EF core
  16) 4.Can we handle transactions in EF core?
  17) 5.What is explicit transaction?



Azure

  1. What is Azure App Service and how do you deploy a .NET application to it?

  2. What are options to deploy a Web API on Azure? (App Service / AKS / Azure Container
     Service)

  3. Why did you choose App Service to deploy WebAPI?

  4. What is the difference between Azure App Service and Azure VM?

  5. What is Azure Functions? How is it used with .NET?

  6. How will you secure Azure Function ?

  7. I want to create storage account and function app. I don't want to enable storage account
     but Azure Function should read data otherwise hacker can tamper storage account.

  8. What are durable function apps ?

  9. Check function app is healthy ?

  10. 14. I want to receive an email if the function app is failing how do you configure this?

  11. What is Azure Key Vault and why is it important?

  12. What is Managed Identity in Azure?

  13. what is key vault and how to connect kayvault

  14. How do you implement CI/CD for a .NET project in Azure?

  15. What is Azure Service Bus and how does it work with .NET?

  16. What is azure Service bus and how we decide we required queue or topic inside.

  17. what is event grid in azure, when we use and why?

  18. what is Eventhub xplain with example?

  19. What is Data mart ?

  20. How do you implement Logging and Monitoring in Azure for .NET applications?

### PDF page 12

                                    www.questpod.com
   21. Azure APIM Questions

   22. How do you secure an ASP.NET Core API in Azure?

   23. Explain some AI services in Azure ?

   24. Have you used AKS ?

   25. How is app insights configured in your ecosystem? Can you see the app insight logs of all of
       your applications in single place or do I have to go to different URL/Tabs for each of the
       applications? How do you configure the same?

   26. How do you configure the deployment parameters (number of pods/max pods/min pods) for
       your Web App/API to be deployed to App Service? Do I need to have separate configuration
       for each of the WebApp/API or can I configure all the apps from single configuration?



SQL
THEORY QUESTIONS



   1) What is Normalization and state its form?
   2) What is BOYCE CODD normal form?
   3) Indexes in SQL
   4) Cluster vs Non cluster
   5) What is execution plan?
   6) Primary vs Unique
   7) Union vs Union ALL
   8) ISNULL vs Coalesce
   9) Truncate vs Delete
   10) Table vs Views
   11) Stored Procedure vs Function
   12) What is CTE?
   13) Temporary table vs CTE
   14) Temporary table vs Temporary Table variable
   15) Sub Queries vs CTE
   16) SQL Injection and how will you prevent it?
   17) Triggers in sql, purpose?
   18) What is Rownumber?
   19) Rank vs DenseRank
   20) What is partition
   21) Ties
   22) What are Magic tables?
   23) What is Deadlock?
   24) What is WITH (NOLOCK)?
   25) What is isolation level?
   26) What is transaction and How to implement it?

### PDF page 13

                                   www.questpod.com
   27) Merged Statement
   28) Send unlimited parameters through stored procedures
   29) in sql How we optimize queries returning 100 millions of records?




Design Pattern and Architecture

   1)  Sometimes they tell to write code also , Singleton pattern code so try to attempt
   2)  What is Design Pattern and why do we need?
   3)  Types of Design Pattern - Creational, Structural, Behavioral
   4)  Which Design patterns you have used in in your project?
   5)  Which Design patterns you know?
   6)  What is Singleton DP? Type down the singleton pattern
   7)  Connection pooling many answer in singleton but that can be problemtic. Especially from DB
       perspective.
   8) Singleton pattern disadvantage
   9) Breaking the Singleton using direct references. Iterator , IEnumerable
   10) What is Strategy DP?
   11) What is Adapter DP?
   12) What is Abstract Factory DP?
   13) What is Repository DP? list its advantages
   14) What is CQRS DP?
   15) What are 3 importanent elements
   16) Why is mediatr in CQRS
   17) Why CQRS is important in MS.
   18) What is event sourcing in CQRS
   19) Is SQL server Event source ?
   20) Have you worked on Clean Architecture?
   1) What are Microservices?
   2) Monolithic vs Microservices
   3) Layer vs Tiered
   4) Tier and N tier difference
   5) How does one microservice call other microservice internally ( Queues , WebAPI)
   6) Scenario in we have microservice in 2 services, we want to use Service 1 data in service 2
       how we can use in between? if we use web Api then how we authenticate in microservices?
   7) Strategic DDD vs Tactical DDD.
   8) Whats a domain and sub domain ?
   9) Types of sub domain
   10) Bounded context , context map and UL.
   11) What is a SAGA pattern?
   12) What is DDD?
   13) What are different classes , Entity , Value objects and Services
   14) Aggregate root.

### PDF page 14

                                   www.questpod.com
  15) CQRS Pattern
  16) Repository
  17) Queues.
  18) Draw infrastructure diagram
  19) How do you decide between a GraphQL and the Backend for Frontend (BFF) pattern?
  20) How do you optimize costs in an Azure Cloud microservices architecture?
  21) How do you implement the Rate Limiting pattern to isolate resource failures?
  22) What is the role of Consumer-Driven Contracts (CDC) in preventing breaking changes?
  23) How do you handle "Reference Data" (e.g., countries, currencies): replicate or query?
  24) When is the Anti-Corruption Layer (ACL) pattern essential?
  25) When do you need a Distributed Lock vs. Optimistic Concurrency?
  26) What specific challenges does Event Sourcing introduce in a microservices environment?
  27) How do you manage schema versioning for events in an Event-Driven Architecture?
  28) When is CQRS (Command Query Responsibility Segregation) necessary versus over-
      engineering?
  29) How do you securely inject secrets into services at runtime?
  30) What strategies do you use for Distributed Rate Limiting?
  31) How do you approach Multi-Tenancy: Silo model vs. Pool model?
  32) What are the pros and cons of Client-side vs. Server-side Service Discovery?
  33) How do you handle time synchronization and clock skew across distributed nodes?
  34) How do you manage long-running processes without holding open HTTP connections?
  35) What is your strategy for Canary Deployments in a dependent chain of services?
  36) How do you design a robust strategy for distributed cache invalidation?
  37) How do you implement Leader Election for singleton background tasks?
  38) How does the CAP theorem practically influence your choice of data stores per service?
  39) How does Conway’s Law influence your decisions on team structure and service boundaries?
  40) How do you manage a "Shared Kernel" without creating tight coupling?
  41) How do you apply Chaos Engineering to validate system recovery?
  42) How do you detect and solve the "N+1 Problem" in distributed service calls?




SCENARIO BASED

  1)   How will you increase performance in angular application?
  2)   If there is case of bulk upload photos/videos, what will be your approch?
  3)   If you find slowness in angular application how will you optimize it?
  4)   If an abstract Employee class has abstract method GetManager() and one concrete method
       GetSalary().
       And Developer class and QA class inherits it.

       Now your Client said there is a requirement to add a CEO class which does not have a
       manager but has salary.

### PDF page 15

                                     www.questpod.com
 How you will implement this scenario?

 Option - You can extend it as it is or you can change the existing code. But which one would be
optimal?



    5)    There are 2 different Interfaces A, B and both are having same method name.
         Can a Class e.g. class C, implement both of them?
         If yes, then how will it use the method from Class A.



    6) There is a sealed class and I want to add new methods to it without changing the existing
       logic.



    7) Can a class have multiple constructor with different access modifiers also with static
       constructor, if yes then in what order it will execute?

    8) If you need to fetch data but parameter/query string is complex then you will use GET or
        POST call and why?
    9) How to increase performance in dot net core?
    10) How to do bulk insert, update in dot net core?
    11) How to limit the API hits in dot net core?
    12) How to handle huge payloads that expires in 30 secs?
    13) In case of complex logic, will you use linq (dot net core) or SQL (database)?
    14) How to optimize a Stored procedure?
    15) If we told you to create a new project then by which approach you will choose Code first or
        Database first and why?
    16) How to do bulk insert, update in EF core?



API Testing

    1)   Why we need testing?
    2)   What is Unit testing?
    3)   What is Integration testing?
    4)   Difference Unit and Integration?
    5)   Which unit testing framework you used in your project?
    6)   What is the code coverage percentage?
    7)   What is Moq. Have you used it in project?
    8)   How to handle exception in unit testing?

### PDF page 16

                                          www.questpod.com




SCENARIO BASED

int[] arr = {2,2,5,6,1,3,3,6,6,5,4,5,5}



Sort the array in descending order of frequency of occurrence of each number. One with highest
frequency should come first. And if frequency is same highest number should come first.



For example here 5 is occuring 4 times then 5 should come first

Then 6 is 3 times 6 will be next

2 and 3 both are 2 times. Same fre

quency so 3 should come first then 2

Similar is 4 and 1



So out put will be

{5,5,5,5,6,6,6,3,3,2,2,4,1}



Given a array {3,7,2,4,5,1}

Sort the array in ascending order

Output {1,2,3,4,5,7}



{3,7,3,2,1,4,5,5,2,2}

Eliminate duplicate character from array

Output {3,7,2,1,4,5}



Count the occurrence of each character in string

"hello world"

### PDF page 17

                                    www.questpod.com
Output

h1

e1

l3

o2

w1

r1

d1



Print the maximum occuring character in string



for example here it will be

l3



Encode a String

Input AACBAEFEEH

Output should be

A2C1B1A1E1F1E2H1



Decode a string

Input A2C3B1A1

Output should be

AACCCBA



These encode and Decode string where asked in capgemini



Write a program to print following pattern



2

### PDF page 18

                                                 www.questpod.com
14

365

8 7 10 9

12 11 14 13 16



*

**

***

**

*



     *

 *       *

* *          *

 *       *

     *

------------------------------------------------------------------------------------------------------------------------------------

In Hpcl

These were sql queries asked

EmpName MobileNo

A                8888888888

B                7777777777

C                8999999999

D                8888888888

E                7777777777

F                7777777777

G                9888888888

H                8888888888

Fetch the names of Employee who have same mobile Number

### PDF page 19

                                   www.questpod.com
Here Output will be

A

B

C

D

E

F

H

As they have same mobile number

tblConsumer



Id Name

1 A

2 B

3 C

4 D

5 E

6 F



tblCustomer

OrderId ConsumerId DeliverDate

1      1              2023-04-01

2      1              2023-04-30

3      1              2023-06-30

4      1              2023-08-31

5      2              2023-01-04

6      2              2023-04-30

7      2              2023-08-31

8      2              2023-08-31

### PDF page 20

                                     www.questpod.com
9        4          2023-08-01

10       4          2023-08-31

11       3          2023-06-30

12       3          2023-01-04

13       3          2023-01-04

14       3          2023-04-01

15       3          2023-04-01

16       6          2023-04-01

17       6          2023-04-30

18       6          2023-08-31

19       6          2023-08-31



Output

RefillCount ConsumerCount

0            2

2            2

3            1

5            1

Write a sql query to generate above output that includes count of consumers based on number of
refills(orders) they have received between 1 april 2023 and 30 June 2023. Include those with zero
refills.

Example here consumer ID 4 and 5 has no orders place between 1 april and 30 June 2023. So for 0
refills there is 2 consumer count.

Consumer ID 2 and 6 have two orders each between 1 april and 30 June 2023. One is on 1 april and
other on 30 april for both. So for 2 refills consumer count is 2

Consumer ID 1 has 3 orders between 1 april and 30 June 2023. So for 3 refills there is only 1
consumer count.

There are no consumers who have place 4 orders. So there is no refill count 4 here

Consumer ID 3 has 5 orders between 1 april 2023 and 30 June 2023. So for 5 refills there is 1
consumer count.

### PDF page 21

                                      www.questpod.com
This sql query was also asked in hpcl written test.

Hint they give as use temp table to store consumerId and refillcount(order count) per consumer
between 1 april 2023 and 30 June 2023

Then join consumer table with temp table to get refillcount and consumercount



Suppose there is WebForm on that there is textbox to enter pan numver , in textbox while entering
PAN Number.

Example XYGPN1122P. How will you do masking and umasking?



After XYG it should be *** and in Database it should be saved as XYGPN1122P but in text box

When user enters pan number it should be displayed as

XYG***

In one company this was program was to check if two strings are anagrams. Strings are anagram if
one string is obtained by rearranging the letters of another

abac , baac are anagrams

rat , tar are anagrams

India , diani are anagrams

But abc and cbcaa are not anagrams




One of the Big multinational company

1) How do you convert your current project to Microservices ?

2) If you are asked to design a heavy traffic service with concurrent user requests and data should
not be corrupt, how would you do that?

3) How do you write middleware components in .Net Core?

4) If you have to migrate a project from .Net 4.5 to .Net 8.0 then how would you do it?

5) Where do you write AddScoped and AddTransient

6) What design patterns have you used?

7) What is SSL certificate?

8) How is deployment carried out?

### PDF page 22

                                     www.questpod.com
9) In entity framework how do you connect to database?

10) If you are architect, before project getting into development phase, what all questions you will
ask customer?

11) What logging mechanism have you used?

12) Have you worked with queues or events before?



Azure interview questions



Azure Bus



CODING QUESTIONS

14 Points to remember:-

    1.  To not use in-built functions and should have good performance.
    2.  Write on Notepad → Browser → VSCode / Visual Studio.
    3.  Use Fiddlenet to practice.
    4.  Camel Case and Pascal Case
    5.  Basic NameSpaces
            a. System.Text
            b. System.Collections.Generic ( Collections Plural , Singular).
    6. AI Software used for interviewing
            a. Berris MasterMind
            b. VeriKlick
            c. TestDom
            d. Mercer Mettl test
            e. Hacker Rank.
            f. AI TalentQuest (WebMobi 360)
            g. Zinterview.ai
    7. Performance is always measured by BIG O notation.
    8. Remember only 1 way which is the performance way and not using inbuilt function.
    9. String is an Array in C# with 0 Index.
    10. Strings are immutable.
    11. ASCII understanding.
    12. L shape LESS THAN. Keep one in Mind.
    13. Writing the Forloop back and forth.
    14. Understanding of collections very important.

    Questions asked.

### PDF page 23

                                 www.questpod.com
1) Reverse character in a string:-
        a. Prepend Approach.
        b. String Buffer.
        c. Swap:- 2 pointer technique.
                  i. Many times interviewer say to Reverse Array Same technique.
        d. Reverse words in a string ( Home Work)
2) Find occurrence of each character :- Dictionary methodology
3) Find the first non-repeating character :- Dictionary Methodology.
4) Find 2 numbers in an array that add up to a target :- Complement
5) Is String is a Palindrome: - Two pointer technique
6) Anagram : - ASCII + Tug of War technique.
7) Max Sum of a Contiguous Subarray of Size k :- Fixed Window Sliding.
8) Sort array which has 3 unique values -:- Dutch Nation Flag
        a. Bubble Sort
        b. Inserted Sort.
9) Valid Parentheses: - LIFO
10) Basic Algo : Prime , Factorial and Fibonnaci.
        a. Should know how to check for remainders. %
        b. Should know ODD and Even ( Divide by 2).
        c. Whole numbers :- All positive integers including zero, with no fractions or
            decimals.
        d. Prime numbers , greater than 1 , Divisble by 1 and himself , 2,3,5,7,11
        e. Fibonacci :- Next number by adding up the two numbers 0, 1, 1, 2, 3, 5, 8, 13
        f. 4 factorial is (4 X 3 X 2 X 1 = 24)
11) What is the next number approach ---- (Diff, Diff of Diff, Alternates , Cubes / Sqquares ,
    Gap, ,VIP)
        a. 4, 7, 10, 13, 16, ? :- Increasing
        b. 1, 4, 9, 16, ? :- Big difference Multiplication.
        c. 1,8,27,64,125 ( think about 124) :- More Big difference.
        d. 64, 58, 52, 46, ? :- Decreasing (-6)
        e. 2, 4, 8, 10, 14, ? :- Alternating
        f. 1,4,10,19,31 ? ( Difference of Difference)
        g. 1, 3, 7, 13, 21, 31, ? ( GAPs)
        h. 10, 21, 35, 56, 90, 145, 231, 360, ? :- Difference and GAPS
        i. Special series like Prime , Fibonacci.
12) Move 0’s to end of the array :- Single-Pass swap reader/placer
13) Left/Right rotate an array by K places :- Reversal technique(Mirro,Front(K),Back(K-N)/Back ,
    Front and Mirror).
14) Longest Substring Without Repeating Characters : Sliding window technique.
15) Reverse a number without converting to string :- Modulus division technique
16) Remove Duplicates from a String :- Dictionary method
17) Basic Star pattern examples
18) Find Nth highest Salary
19) Swapping two number
20) Extract the surname

### PDF page 24

                                      www.questpod.com
     21) Remove All White Spaces from a String
     22) Find the Longest Palindromic Substring
     23) . int[] GivenArray = { 1, 2, 3, 4, 5, 6 }; //Print Output-> //1 2 3 4//5 6 1 2
     24) Find max number in Array.
     25) Find Duplicate records in table
     26) Given an array of positive numbers, calculate the number of possible contiguous subarrays
         having product lesser than a given number K.
          ///________________________________________
         ///Example Input/Output:
          ///Input : arr[] = [1, 2, 3, 4]
          /// K = 10///Output : 7
         ///The subarrays are {1}, {2}, { 3}, { 4}, { 1, 2}, { 1, 2, 3} and {2, 3}

     27. given a target number of ingredients you must buy (for example, 3 items: Item 0, Item 1,
         and Item 2).You are handed a coupon sheet with mixed-up items and prices.Go through the
         coupon sheet, find the lowest price for each required item, and add them together.
     28. Min and Max numbers from Array.




Company specific Interview question and experiences.

GlobalLogic Interview Questions :
     1. What are .Net components - CLR, CTS

CLR - converts IL to Machine Code, Garbage collection, Exception Handling, Thread Mgmt, Security

BCL(Base Class Library) - collection of reusable classes - File Handling, DB Access(ADO.Net),
Collections and LINQ, Networking, IO

CTS(Common Type System) - how datatypes are declared and used, cross language compatibility(int
in c# to int32 in Int32)

CLS(Common Language Specifications) - Rules that all .Net languages must follow

JIT(Just in time compiler) -

Garbage Collector -

2.      What is Type safety?

Type safety means:A variable can only hold values of its declared type, and operations are allowed
only on compatible types. n .NET, it is enforced by the CTS and CLR, preventing invalid casts, memory
corruption, and runtime errors

### PDF page 25

                                       www.questpod.com
3.      Features of .Net 6.0 used

C# 10 version - global using directive file scoped namespace,

Hot Reload in .Net 6 - modify source code while it is running

AZ functions supports running serverless functions in .Net 6.0

4.      What is Garbage Collection and How does it works?

Garbage Collection is an automatic memory management system in .NET.

It, Allocates memory for objects, Tracks object usage, Frees memory when objects are no longer
needed, All of this is handled by the CLR

5.      Difference between abstract class and Interface?

An abstract class is used for sharing common implementation among related classes, while an
interface defines a contract that multiple unrelated classes can implement. Interfaces support
multiple inheritance, whereas abstract classes do not.

Abstract classes can have constructors, interfaces cannot

A class can implement multiple interfaces but inherit only one abstract class

6.      Why Nuget packages used in .net application?

NuGet packages are pre-built libraries that you can add to your .NET projects.

Instead of writing everything from scratch, you use existing libraries

Saves time by using tested solutions

NuGet packages are used in .NET for managing reusable libraries and dependencies. They help speed
up development, ensure consistency, and simplify version management by automatically resolving
dependencies.

7.      What is difference between .Net core and .Net framework?

.NET Framework is a Windows-only, legacy platform, while modern .NET (formerly .NET Core) is a
cross-platform, high-performance, and open-source framework designed for cloud and modern
application development.”

8.      Explain SOLID principles

9.      What is Tuple in c#

A tuple in C# is a lightweight data structure used to store multiple values without defining a class.
Modern C# uses ValueTuple with named fields, making it useful for returning multiple values from
methods.

10.     What are Generic Delegates and its types

### PDF page 26

                                        www.questpod.com
Generic delegates are predefined delegates in C# that use generic type parameters, so you don’t
need to define custom delegate types.

      1. Func<T> - Used when: Method returns a value

      2. Action<T> - Used when: Method does NOT return anything (void)

      3. Predicate<T> - Used when: Method returns bool (true/false)

All are widely used in LINQ and lambda expressions

11.       Why we use Async and Await with Task-

They are used to write asynchronous (non-blocking) code in a clean and readable way.

Instead of blocking a thread while waiting (e.g., API call, DB call), your program can do other work.

It improves application responsiveness and scalability by freeing threads while waiting for I/O
operations, especially in web and UI applications.

12.       How you create custom middleware?

In ASP.NET Core, middleware is a component that handles HTTP requests and responses in a
pipeline.

In ASP.NET Core, middleware is a component that handles HTTP requests and responses in a
pipeline.

13.       How microservices and different from Monolithic applications

Monolithic architecture is a single unified application, whereas microservices architecture consists of
independently deployable services. Microservices provide better scalability and fault isolation but
introduce complexity compared to monolithic systems.

14.       What are the different microservice architecture design patterns

Microservices architectures rely on several design patterns to handle communication, scalability,
resilience, and data management.

      1. API Gateway Pattern - A single entry point for all clients which Handles: Authentication,
         Routing, Rate limiting

      2. Service Discovery Pattern - Services dynamically find each other, Types:Client-side discovery,
         Server-side discovery

      3. Circuit Breaker Pattern - Prevents cascading failures, Stops calls to failing service, Returns
         fallback response

      4. Saga Pattern - Manages distributed transactions, Two types:Choreography (event-based),
         Choreography (event-based)

      5. Database per Service Pattern - Each service has its own database

### PDF page 27

                                       www.questpod.com
      6. Each service has its own database - Event-Driven Architecture, RabbitMQ, Kafka

      7. CQRS (Command Query Responsibility Segregation) - Separate: Read operations, Write
         operations

      8. Strangler Fig Pattern - Gradually replace monolith with microservices

      9. Retry Pattern - Automatically retry failed requests

      10. Sidecar Pattern - Attach helper service to main service - Logging, Monitoring, Security




15.       How do you implement fault tolerance in microservice based system

Fault tolerance means a system continues to function even when some services fail.

In microservices, failures are inevitable (network issues, service downtime), so we design systems to
handle and recover from them gracefully.

Use - Circuit Breaker Pattern, Retry Pattern,

16.       What is circuit breaker pattern

The Circuit Breaker pattern is used to prevent repeated calls to a failing service, avoiding cascading
failures in a distributed system.

Think of it like an electrical circuit breaker: If something goes wrong → it “trips”, Stops further
damage

The Circuit Breaker pattern prevents repeated calls to a failing service by opening the circuit after a
failure threshold. It has three states—closed, open, and half-open—and helps avoid cascading
failures in microservices.

17.       What is CQRS

CQRS (Command Query Responsibility Segregation) is an architectural pattern that separates read
and write operations into different models.

In simple terms: Command → modifies data (Create, Update, Delete)

Query → reads data

CQRS separates read and write operations into different models to improve performance, scalability,
and flexibility. The command side handles updates, while the query side is optimized for reads, often
using separate data stores.

18.       What is service discovery in microservice architecture

CQRS separates read and write operations into different models to improve performance, scalability,
and flexibility. The command side handles updates, while the query side is optimized for reads, often
using separate data stores.

### PDF page 28

                                       www.questpod.com
19.      What are the different ways of communication between microservices

Microservices communicate in two main ways:

      1. Synchronous Communication: Request → Response (real-time) - HTTP / REST APIs , gRPC
         (High Performance)

      2. Asynchronous Communication - Message/Event-based (decoupled) - Message Queue ,
         Event-Driven (Pub/Sub)

20.      What is a role of API gateway in microservice

An API Gateway acts as a single entry point in microservices, handling routing, authentication, rate
limiting, and response aggregation. It simplifies client interaction and centralizes cross-cutting
concerns.

21.      What is content negotiation

Content Negotiation is the process where the server selects the best response format (like JSON or
XML) based on the client’s request.

In ASP.NET Core, this is handled by formatters like JSON and XML automatically.

22.      ACID properties in RDBMS

23.      How we measure database servability

24.      What is replication

25.      How you identify slow running queries and optimize them

26.      What is indexing and how does it improve performance

27.      What is triggers and types of triggers

28.      What is difference between commit, rollback and savepoints

29.      What is difference between @@Identity and @ScopeIdentity

30.      What are materialized view

31.      What is difference between RANK and Dense Rank and Entile

32.      What are different types of NOSQL

33.      Why do we choose NOSQL over SQL




LTI Mindtree

### PDF page 29

                                         www.questpod.com
.NET / C#

      •   Stack vs Heap

      •   What is Garbage Collector? How does it work?

      •   If I don’t want to create instance of a class, how to do it (without static/abstract)?

      •   How to create Dependency Injection?

      •   Where does injected dependency reside?

      •   Constructor chaining

      •   Partial class – what & use

      •   Can partial class exist in different projects?

      •   Types of NullReferenceException

      •   Task vs Thread / Multithreading

      •   IEnumerable vs IQueryable (inheritance difference)

      •   Hashtable vs Dictionary / HashSet vs Dictionary

      •   Ref vs Out

      •   IDisposable & where to use

      •   .NET Framework vs .NET Core (developer perspective)

      •   Async & Await

Entity Framework / ADO.NET

      •   What is ADO.NET?

      •   Can we use Stored Procedure with ADO.NET?

      •   Difference between SQL query & Stored Procedure in ADO.NET

      •   EF Migrations commands

      •   AsNoTracking

      •   EF Code First vs Database First

      •   Eager loading

SQL

      •   Views vs Indexes

      •   Indexes (types)

      •   Triggers

### PDF page 30

                                      www.questpod.com
   •    SP vs Function

   •    What is SP?

   •    Where vs Having

   •    CTE

   •    Functions – can functions have multiple return types?

   •    Types of joins

   •    Query: If commission is null return 0

   •    Get 5th highest salary

   •    Department with 0 employees

   •    How to identify missing index?

   •    SQL optimization techniques

   •    Keyword to get last inserted record

ASP.NET Core / MVC / API

   •    Authentication & Authorization

   •    JWT (working, expiry, content)

   •    Logging

   •    Exception handling

   •    Middlewares

   •    REST principles

   •    Routing

   •    Model validation

   •    HTTP methods

   •    200 vs 201 status code

   •    Can we use [Post] on Get action?

   •    Clean code principles (DRY)

   •    Return error message to client

   •    Multiple views / Same view for different controllers

   •    DropDown performance handling

Azure

### PDF page 31

                                       www.questpod.com
   •     Azure Functions (data extraction from request)

   •     Azure Key Vault integration with .NET API

   •     Handling API issue from Cloud (analysis steps)

Architecture / Design

   •     Design patterns (Factory)

   •     SOLID principles

   •     Open/Closed principle violation example

   •     Caching (why, memory vs Redis)

   •     Microservices architecture



EPAM
Coding

   •     Count repeated characters → Alphabet + Number format

   •     LINQ group & sum

   •     Letter count in string

   •     LINQ join 2 tables

   •     Parallel programming

Concepts

   •     DI lifetimes (Scoped explanation specifically)

   •     Authentication & Authorization

   •     Logging

   •     Middlewares

   •     REST

   •     Routing

   •     Async / Await



Carelon

.NET / Project

   •     Current .NET version used

### PDF page 32

                                     www.questpod.com
     •   Latest .NET features

     •   Performance difference (.NET vs Framework)

     •   Ref vs Value types

     •   JWT working & expiry

MVC / API

     •   Multiple views same data

     •   AJAX / jQuery

     •   Error handling

     •   Display dropdown data efficiently

Coding

     •   Letter count program

     •   LINQ join



Accenture (1st Round)

OOPS

     •   OOPS concepts

     •   Inheritance & Multiple inheritance

     •   Types of Polymorphism

     •   Calling same method from multiple interfaces

C#

     •   Swap without 3rd variable

     •   Delegates

     •   Async programming

     •   Multithreading

MVC / API

     •   DTO

     •   ViewBag

     •   MVC lifecycle

### PDF page 33

                                          www.questpod.com
      •   Action return types

      •   Security in API

SQL

      •   Design DB

      •   Constraints

      •   SP

      •   Index & Clustered index

Other

      •   Microservices

      •   Hosting

      •   Attributes



Cognizant

Project & Process

      •   Roles & responsibilities

      •   Task planning

      •   Agile

      •   Estimation

      •   Client interaction

      •   Helping juniors

.NET / API

      •   DI & lifetimes

      •   Async/Await

      •   Authentication & Authorization

      •   API versioning

      •   Rate limiting

      •   Validations before controller

      •   Exception handling

### PDF page 34

                                       www.questpod.com
SQL / EF

   •      Prevent SQL injection

   •      Stored Procedures

   •      CTE

   •      Functions

   •      Types of joins

   •      EF parent-child loading

   •      Handling scoped DB connection expiry

Azure / DevOps

   •      Azure services used

   •      CI/CD

   •      OAuth



Deloitte

Concepts

   •      SOLID principles

   •      Dependency Injection

   •      JWT mechanism

   •      Authentication & Authorization

   •      Async programming

   •      Exception handling

   •      CQRS

Coding

   •      Singleton implementation

   •      Delegates (Func vs Action)

   •      Anagram program

   •      3rd largest number

Testing

### PDF page 35

                                         www.questpod.com
      •   Unit testing framework

      •   NUnit

      •   Mocking vs Stubbing



GlobalData

Project

      •   Module worked on

      •   ORM used

      •   Querying DB

ASP.NET

      •   Constructor in API

      •   app.Use vs app.Run

      •   Fix slow MVC page

      •   Validation before controller

SQL

      •   5th highest salary

      •   Department with 0 employees

      •   Joins

      •   Optimization

Caching

      •   Types of caching

      •   Redis storage location

      •   In-memory cache configuration


Verisk Interview

Project / Basics

      •   Tech stack

      •   SQL version

      •   SQL connection with application

### PDF page 36

                                       www.questpod.com
SQL

      •   Optimize query

      •   Group patients by doctor (today)

Coding

      •   3rd largest element using 1 loop

API

      •   REST API skeleton

      •   What is REST API

      •   Parallel request execution



Persistent - Architect interview questions asked to me



1) If a client comes to you for their application modernization, what would be your approach?

2) Explain flow of authentication in your Web application / Web API

3) Explain flow of authentication in general (JWT / OAuth2)

4) What are options to deploy a Web API on Azure? (App Service / AKS / Azure Container Service)

5) Why did you choose App Service to deploy WebAPI?

6) How do you do logging/tracing in your Web Application / WebAPI? How did you design
observability/traceability in your Application ecosystem?

7) How do you access data / database in your Web API? Dapper / EF Core / ADO.Net? Which one
would you choose under what circumstances?

8) How is app insights configured in your ecosystem? Can you see the app insight logs of all of your
applications in single place or do I have to go to different URL/Tabs for each of the applications?
How do you configure the same?

9) How do you configure the deployment parameters (number of pods/max pods/min pods) for your
Web App/API to be deployed to App Service? Do I need to have separate configuration for each of
the WebApp/API or can I configure all the apps from single configuration?

10) Your application is built in a legacy version of Angular. How would you estimate/approach for
upgrading it to the latest framework version?

11) How do you conduct TDD of DB objects (Stored procedures)?

12) Design patterns used in Event Driven Architecture

### PDF page 37

                                       www.questpod.com
13) What's use of Key Vault in your project?



CapGemini

5 years CapGemini exp


Was asked Mostly Fundamentals was asked



1. Explain D from SOLID with written example

2. Any design pattern which you have implemented. I explained Singleton.

3. How to pass data between controllers

4. Automapper

5. Bundles.config

6. How to implement caching

7. Difference between list and dictionary

8. How to implement security JWT token

10. Return type in API

11. How to handle exception on application level

12. SQL query for 2nd highest salary

13. SQL query for deletion of one of the duplicate records

14. How can we optimize query performance

15. How to create directory in C#

16. Angular interpolation binding

17. Pipes, impure and pure pipes

18. Give duplicates from a list using LINQ

19. SQL Joins

20. select * from Employees where 1=0. What is the output?

21. Abstract class vs Interface

### PDF page 38

                                    www.questpod.com

Capgemini Azure Lead


How do you handle State Management in Azure Functions?

How do you deploy APIM in Multi-Region with Budget Constraints?

How do you Handle Deduplication in Service Bus?

How do you Ensure Message Reliability in Service Bus?

My backend service is deployed inside a VNet, private endpoint but my APIs in APIM are public, how
do I ensure connectivity between them? What should I do?

How do I ensure that while using App Service Plan for Azure Functions, I always get pre-warmed
instances? Is that even possible?

How do we use partition keys in Cosmos DB?

What is meant by replay tools?



Persistent Azure Dotnet Role

 Middleware

 Exception handling

 Performance optimization technique in .NET API

 Explain your project structure

 How to make your application more scalable and more available

 Did you take downtime while deployment? How much? If yes then why?

 How to achieve 0 downtime (not even 0.00001%)

 What is Azure Service Bus

 Azure Service Bus vs Event Hub

 What is Azure Function App

 How do you decide to use Function App

 How to implement Azure Entra ID, what configuration do you have to do?

 How authentication works with Entra ID?

### PDF page 39

                                       www.questpod.com


Mhpasis

1st round


Intro, roles & responsibilities

Project & tech stack

Polymorphism

How to call base class constructor

DI & life cycles of DI

Throw vs throw ex

Garbage collector

Cache & types of cache

Middle ware

How to create a custom middleware

App.use( ) vs App.Run( )

Query parameters

How do you extract data from http request

How do you do Model validation ( DataAnnotation , Regex)

What is Model binding

Asynchronous programming

Task vs thread

Multi threading

What is the use of Idisposable

Microservices architecture

Design patterns used in project

CQRS

API versioning

If a service is failing continuously, how will you handle this situation

How do you make communication between services

### PDF page 40

                                      www.questpod.com
How do you call one service from another



Code :

1. given sentence and asked to reverse letters in a word without changing the order of words in the
sentence

2. Count the occurence of letters in a string



SQL

Indexes & types

How many cluster indexes will be there for table

CTE vs temp table

Window functions

Rank, dense rank



2nd round


Intro, roles responsibilities



Tell about current project, tech stack, what modules you worked on



Open note pad & Write a controller & show how you implemented repository pattern using DI and
entity framework



What DI life cycles used in your project



How do you write unit testing for this ?

What frameworks do you use for unit testing



Middle wares & custom middle wares, how you used middle wares in your project?

### PDF page 41

                                     www.questpod.com
Exception handling and how you return user friendly messages to front end



How you implemented Security in your project



Tell how jwt tokens work



Have you used cors and how does it work ?

What kind of requests it’ll block?



What deployments used in your project

EPAM Interview
Epam interview questions



Demonstrates good and well-rounded understanding of .NET Core and .NET Framework differences,
including runtime, hosting model, and performance implications.

Proficient in Dependency Injection concepts and practical implementation patterns.

Experienced with application logging using the ILogger interface and follows recommended logging
practices.



ASP.NET Core

Shows good understanding of ASP.NET middleware: especially pipeline behavior.

One gap noted: did not correctly articulate the short-circuiting behavior in middleware and instead
referenced the circuit breaker pattern.

Good understanding of JWT authentication, token validation, and middleware configuration.

Proficient in model binding and model validation, with clear real-world application examples.



C#

Good understanding of the IDisposable pattern and resource management.

proficiency with async/await and comprehension of Task vs Thread concepts.

### PDF page 42

                                       www.questpod.com
Experienced in using Parallel.ForEach and demonstrates very good foundational knowledge of
multithreading scenarios.

Clear understanding of ref vs out semantics and use cases.

Competent in exception handling, including structured use of try-catch-finally.

Successfully implemented a LINQ-based grouping and projection solution to aggregate sales by
ProductId, showcasing confidence with collections and object transformations.

Attempted a balanced brackets function with good structure and correct syntax, but was unable to
achieve the expected output



LINQ

Demonstrates good proficiency with LINQ.

Skilled in filtering, grouping, aggregation, and projection.

Shows good ability to apply LINQ in real-world business scenarios.



Software Design

Good understanding of DRY, clean coding guidelines, and maintainable design practices.

Capable of creating technical documentation, including coding standards and diagrams.



Design Patterns

Knowledgeable about the Decorator pattern with practical examples.

Understands Singleton and Factory patterns at a conceptual level, though examples were more high-
level than deeply implementation-focused.



Entity Framework

Good working knowledge of Code First approach and Code First Migrations.

Understands EF's tracking vs no-tracking queries and how they impact performance and memory
usage.

Comfortable discussing EF architecture and real-time usage scenarios.



SQL

Strong understanding of WHERE vs HAVING and SQL query execution order.

### PDF page 43

                                      www.questpod.com
Knows how SQL handles NULL values, including comparisons and function behavior.

Demonstrated logical reasoning by structuring an UPDATE query using a CASE expression, even
without recalling exact syntax showing strong conceptual SQL thinking.



Unit & Integration Testing

Demonstrates clear understanding of the AAA (Arrange, Act, Assert) pattern.

Good knowledge of mocking frameworks and how to isolate dependencies.

Can clearly differentiate between unit tests and integration tests, with examples of when each is
appropriate.



Microsoft Azure

Shows good knowledge of Azure Functions, serverless execution, triggers, and bindings.

Familiar with Azure Key Vault, including secrets management.

Possesses working knowledge of Application Insights for observability.

Understands fundamentals of Azure Application Gateway

Function apps vs App Services

Authentication Approach - AD

What is JWT

OAuth 2.0 vs Open ID



While doing the API Authentication, Client Credential Flow



Type of Identities, System based Identify, Managed Identify and User Identify



I have a .NET Web API I want to consume SQL Server and I want to Consume Blob Storage, In Azure
how the integration looks like



I have a background job which is recurring, what are the different ways to Do, for daily notifications



Web Jobs

### PDF page 44

                                      www.questpod.com


Can I do 2 webjgbs in same app Service Plan?



How will you decide I need a separate App Service Plan, looking at scalability or availability



How will you track the issues when it raised in Functions, app Services or etc.. - About Application
Insights



Have you worked on Azure Monitor - to track the issues which are attached to Application Insights



Publisher/Subscriber Pattern, How will you identify or track of where is my application, what is my
end point and how will you approach.



Did you work on Queues like Service Bus or Storage account queues - Explain me the scenario



I am using the single Topic in SB and multiple subscriber, I want to trigger based on a Field as the
parameters in Topics are same how would I achieve that



If you read the queue, how should we process at code level, how ServiceBusQueue class works,
DeadLetter Queue or some other concepts in SB works at Code level, explain?



On how many ways you can trigger the Service Bus explain



From .NET I wanted to access the KeyVault - How do I implement it. Explain Class Level



AKS: Have you deploy yourself or DevOps Team, How did you do, Helm Files, Ingress Configuration,
Configuration Service Files, CI/CD, etc..



ACR - what is use of it and explain - Mostly used for storing the doctor image



Container and Pod - Difference

### PDF page 45

                                      www.questpod.com


Helm Charts - What do Helm Chart do. Why do we need



What CLI you use to test HelmChart in Local machine - Is it with Doctor or something else



Any Experience on Creating VNET or any Networking Concepts




Dependency Injection - Types.. Scoped, Single and Transient



How will you handle the CORS Issues



API End point is consumed by Multiple Applications, How do you manage (Question is related to
APIM)



I don’t want to write code but I want to validate each request - how can I do in APIM



Did you work on Middleware - Explain me the scenario



How Middleware Executes in a scenario of API Execution to check the Authentication and
Authorisation before hitting the Web API



Difference between Sync and Async - give example scenario



Can we use Async without await ?

### PDF page 46

                                      www.questpod.com
I have my Web API I wanted to call an external API - wait once the things done and I wanted to put in
a queue base don the response for the further process, what should I Do?



What is the concept of concurrent list and normal list



Different between .Result vs ConfigureAwait




If want to connect to SQL what are all the possible approaches are there. ADO.NET, ORM
Frameworks etc..



How do we scale the database when the application is scaling



How will you ensure Connection Pooling



While doing the load testing we simulate the Pooling what type of issues we will face mostly



Have you use LINQ



IEnumerable or IQueryable - what is the difference between and when to use




SQL Server - what did you work and explain.



Scaler Functions - explain the scenario

### PDF page 47

                                      www.questpod.com
How will you check the query taking time - Execution Plan or SQL Profiler



What is Common Table Expression - explain the scenario

For Unit Testing what Tools you used for API’s - NUnit, XUnit, and MSTest



GitHubCopilot - any Specific things to define



CQRS and Mediator Design patterns - Explain

Web Page is Unavailable



Virtusa

Virtusa Interview Process (Lead Software Engineer, YOE 10+)

in each L1, L2 and client round They gave multiple problems to solve (provided IDE for both SQL and
C#)

In L1, they expect the exact output, so you need to write a complete and correct program.

In L2 and the client round, pseudocode and clear logic explanation are sufficient

1. AI Interview (MCQ Round)



 * Online assessment consisting of multiple-choice questions

 * Covers programming basics, and core concepts



2. L1 – Technical Interview



 * Focus on coding, problem-solving, and fundamental concepts

 * Includes SQL queries, C# programming, and output-based questions



3. L2 – Techno-Managerial Interview

### PDF page 48

                                       www.questpod.com
 * Combination of technical knowledge and practical experience

 * Questions on system design, real-time scenarios, and project discussions

 * May include decision-making and problem-handling approaches



4. Client Round (Technical Test – HackerRank)



 * Hands-on coding test conducted on HackerRank

 * Focus on Communication, problem-solving, coding efficiency, and logic building



5. HR Discussion



 * Final round covering salary, availability, and cultural fit

 * General discussion about background, career goals, and expectations




SQL QUESTIONS



L1



1. Given the tables:



 * Employees

 * Projects

 * EmployeeProjects



 Write a query to:

### PDF page 49

                                      www.questpod.com
 * Find employees who have worked on at least 2 projects

 * Identify the longest duration project

 * Return employee names along with the longest project they worked on

 * If multiple longest projects exist, return any one



L2

2. Tables:



* tblCustomer (CustId, CustName)

* tbleOrderdtls (OrderId, CustId, OrderNumber)



Write a query to find:



* CustId

* CustName

* OrderCount



For customers having more than 3 orders



3. Laptop Sales:



 Tables:



 * Laptop (L_Id, Brand)

 * Sales (Sale_Id, L_Id, Quantity)



 Write a query to display:



 * Brand

### PDF page 50

                                     www.questpod.com
    * TotalQuantity sold



    Include brands with zero sales



---



C# QUESTIONS



L1



1. Given a string "ABC", generate all combinations and permutations

    Example: ABC, BCA, CAB, AB, BC, AC



2. What will be the output?



class A

{

public void Display ()

{

Console.WriteLine("A");

}

}

class B:A

{

public void Display ()

{

Console.WriteLine("B");

}

}

### PDF page 51

                                      www.questpod.com


B b = new A();

b.Display();



A a = new B();

a.Display();



Also consider variations using virtual and override



3. Sort the array

 Input: {4,0,7,5,2,8,0,0,0}

 Output: {0,0,0,0,4,7,5,2,8}



4. Write a program to find duplicate characters in a string



5. What is Concurrent Programming?



6. What is GraphQL and why is it used?



7. Explain CI/CD



8. Differences:



* .NET 6 vs .NET 8

* var vs dynamic

* Task vs Thread

* async vs await



9. Explain:

### PDF page 52

                                         www.questpod.com


* static class

* middleware (ASP.NET)

* temp table



Can temp tables be used in:



* functions

* triggers



10. How to handle exceptions in:



* C#

* React

* SQL



11. REST API vs traditional API



12. How to implement API versioning



13. ASP.NET Filters



14. Can we create custom middleware? If yes, how?



15. Alternative for useEffect in React



---



JavaScript / React Output

### PDF page 53

                                       www.questpod.com


16. What will be the output?



console.log(true + 1)

console.log(1 + '2')

console.log('2' + 1)



17. What will be the output?



let c;



console.log(a);

console.log(b);

console.log(c);



var a;

let b;



---



C# L2



18. Write a method:



public static string MergeStrings(string s1, string s2)



Requirements:



* If same length → merge alternately

### PDF page 54

                                      www.questpod.com
 Example: ABC + 123 → A1B2C3



* If S1 longer → append remaining at end

 Example: ABCDEFG + 123 → A1B2C3DEFG



* If S2 longer → append remaining at end

 Example: ABC + 123456 → A1B2C3456



Virtusa 11/June/2026 ( Microservices and Docker Focussed)


### General & Background

* Can you introduce yourself?

### Microservices Architecture

* Have you done work with microservices?

* Have you done any monolith to microservices migration?

* Do you know how to identify what microservices to create from a monolith application?



* What is the advantage of using microservices over a monolithic application?



 * What was your biggest challenge when developing the microservices project, and how did you
solve it?



### C# & .NET Core



 * Write a C# code snippet to remove duplicate characters from an input string (e.g., if the input is
"Anand", the output should be "AND").



* Why does .NET Core run faster than the older .NET Framework?



### Cloud & DevOps (Azure & Docker)

### PDF page 55

                                        www.questpod.com


* What are the various Azure services you have used?



* Can you give an example of a task you have done using Azure Functions?



* How did you deploy the Azure function?



* Have you worked with Docker files?



* How did you deploy the microservice?



TCS Consultancy 20 +LPA Package (.NET Full Stack + Al)

Today, I attended an interview with TCS for a .NET Full Stack role, and I wanted to share some of the
key     questions that were asked. It was a great experience with a strong focus      on real-time
scenarios, problem-solving, and practical knowledge.



    .NET Core Questions



1) Explain Singleton lifetime in Dependency Injection with a real-time example.

2) Explain SOLID principles.

3) Difference between Hashtable and Dictionary.

4) Primary vs Secondary Constructor in C#.

5) Convert.ToString() vs ToString()

6) First(), FirstOrDefault(), Single(), SingleOrDefault() differences

7) Middleware + write custom middleware

8) GET vs POST - which is better and why



9) API performance optimization

### PDF page 56

                                        www.questpod.com
10) Pagination in .NET Core



11) Filters in MVC



12) IActionResult vs ActionResult



13) Why API is slow and how to fix it



    Al-Related Questions



1) Explain Al integration in your project



2) How to call an Al API in .NET



3) What is Prompt Engineering



4) Design Al-powered chatbot in .NET



5) Chatbot giving wrong answers - how to fix



   Scenario-Based Questions



1) You already have authorization applied to an API, but inside the API method, you need to bypass
authorization for a specific condition. How will you implement it in real-time?



2) You have int a = 1. At runtime, you want to assign dynamically like var e = a. Which data type
would you use and why?



3) Your MVC API returns JSON like { "id": 1, "name": null, "age": 25). How will you ignore null values
in the response?

### PDF page 57

                                      www.questpod.com


    SQL Scenarios



Query Behavior:



1) SELECT * INTO newtable FROM oldtable WHERE 1 = 0;



2) SELECT * INTO newtable FROM oldtable WHERE 1 = 1;



What will happen and where are these used in real-time?



3) Can you write pagination in SQL?



4) Insert values like Male = 'M', Female = 'F', Other = 'O'. Write SQL query.



5) While creating a UserDetails table, set default age = 18. Write SQL query.



4) Find the 3rd highest tanker level of water and also find duplicate tanker names.



   Overall, the interview focused heavily on:



   Real-time scenarios

   API performance

   SQL problem-solving

   Basic Al understanding



It was a great learning experience!



    Tip:

### PDF page 58

                                      www.questpod.com


Focus on real-time scenarios, API performance, SQL optimization, and system design thinking.



TCS Experience 2 Special Notes from a developer.

1st technical round is very important. Most of the time they start with very basic but practical
questions.

For me, they asked:

• One simple coding program related to string splitting and finding the 3rd highest integer
• .NET Core basics
• Authentication vs OAuth
• How to expire a JWT token
• Custom Middleware
• Uses of async / await
• DB query optimization
• CTE
• Demerits of Cursor in SQL
• LINQ — when to use AsNoTracking in Entity Framework
• SQL query to get department-wise max salary

The 2nd round was managerial and was more straightforward around project responsibilities and
tech roles.

This is why fundamentals + practical understanding matter a lot in interviews




Wolters interview

What's your approach towards designing a microservice pattern basically? Like, how do you
approach

to break down your project and the designing part of it? Like, how you design the solutions?



So, how you define the bounded context, like, if there are overlaps, then how will you take care of
this? Okay.

So, if there are overlaps, so, we will have a database, okay, a common database, okay, to take care
of that, right, like,

that we call the event sourcing, correct, eventual consistency. So, we will capture the sequence of
events, like, user has done

### PDF page 59

                                         www.questpod.com
button click, user has done this, this event is created, order is created. .




Can you explain, like, from practical standpoint, how you implement saga pattern.

So, basically, saga pattern is to take care of distributed transaction, right. We have one transaction,
like, payment, invoice,

whatever it is. And then, it says, basically, commit all or commit nothing, right.

So, if something is broken, we will reverse everything, okay. Yeah, I mean, that's the theory part of it.

From practical standpoint,

can you explain how many services you will create? Will saga be a different service, or it will be

 embedded into your, any of the microservice, then how will you compensate it from practical
standpoint? Okay.



So, where saga sits, basically, first of all?

Yeah. So, where that logic will be implemented?



Let's say, one service is inter-service communication. We are calling one service to another service,

then another service to, from the same service,

we are calling multiple service. So, there, we will implement saga pattern.



So, at every places where the communication happens, you will implement saga? So, who is
responsible for

implementing the saga, the consumer or the producer,

or who is responsible? Okay. And can you explain event sourcing? You just momentarily talked about
event sourcing. Yeah.




Okay. So, how do you connect your .NET Core Web API to any Azure resource which is protected?
Azure resource

### PDF page 60

                                       www.questpod.com
 which is protected? Protected in the sense? It's private, basically. It's private, not available over the
internet.



let's say, I want to manage my SQL connections efficiently, I mean, your .NET Core Web API app is
creating SQL connections whenever there is a request

 and there are high volume of traffic that is coming in and due to that, it is instantiating so many SQL
connections. Okay.



So, but you know, SQL has its own limitation to open number of, on the number of connections to be
open at a time, right? So, how you will optimize

or design this situation? Okay. You understood the problem statement, right? Yeah, I understood.
Basically,

we are getting so many traffics on SQL connection, right, SQL server side.



Right. So, basically we have to manage that. Connection pool, we can manage, right? We can set a
limit in the connection pool, like this 20, 30, 40,

whatever it is and then whatever the extra request will come, we will reject it.



No, we can't reject, we have to serve every request. Okay. So, there is a, okay, there is a pattern,
CQRS pattern, okay.




what's your design strategy to design K8 which is deployed into your VNet, like,

how many node pools you have and how many microservices you have and how you design

that to be deployed into the K8.?



One pod can have multiple containers. Can a pod have different containers of different types?



whenever the request comes from outside the world, right, from HTTP, from mobile, whatever it is,

it comes to ingress and ingress routes the traffic to the appropriate service.

Do you know how a pod connects to a storage account, Azure storage account, or maybe a key
vault?

### PDF page 61

                                       www.questpod.com
How it connects?



for microservices, do you think solid principles are applicable?



Do you know sidecar design pattern? Sidecar? Yeah.




Do you know Docker containers ? how to create a container and all?




So, how do you select an image? We, we pull the images, no? Docker pull images, right? No, no, I
mean, I mean, what's,

what's the criteria to select an image? What is the criteria to select the image? I mean, how do you
decide for this microservice

this should be the image, for that microservice this should be the image, how do you decide? So,
that we will, anyway we will pull,

we will, you know, define in our YAML file, right? We have a YAML file, correct? So, we know this...
Yeah, but the base, base image

you need to select anyway, right? Yeah, yeah, yeah. So, the base image, how do you decide this base
image should be of this type,

this base image should be of that type for this microservice? Because base image is not common for
every microservice, right?

So, we have something called tagging, with the help of tagging we can define, we can identify,

I'm just guessing, I'm not sure. We have tagging in the image, okay

We give a name, with the help of tagging we can give a name, okay, in the Docker. Whenever we
create a Docker image, Docker image

tagging hyphen T and then we give a name. With the help, obviously we will give a distinct name

for all the services, right? There we can, yeah, tagging is there.

you have integrated Azure OpenAI into your project, so, what was the use case?



Do you know spec-driven development?

### PDF page 62

                                      www.questpod.com

Infosys

I have attended the AI interview but not in Infosys so mostly it starts with the basic question with
.Net.

And you will have two options you need to tell the answer oral like an interview or you can type it
and enter, it will go through the answer and will give you a response. You can expect. Net core and
net framework and async await kind of questions it will also ask questions mostly from your resume
and the answer you are giving from previous questions mostly.

Better try to prepare string problems and small programs using C#.

I attended 2 AI interviews this was the pattern.



WIPRO

This is for 9 years experience.

Wipro 1st round held on 30th April 2026


---



## 1. General & Introduction



* Can you please tell me about yourself?

* How many years of experience do you have?

* What technologies do you have experience in? (e.g., .NET Core, Angular, Entity Framework, SQL
Server)

* Tell me about your last or current project. What is your role and responsibility in that project?

* Which version of .NET and Angular have you worked with?

* How many projects have you completed using .NET Core and Angular?

* On a scale of 1 to 5, how would you rate yourself in:

* .NET Core

* Angular

* Web API

* Entity Framework

### PDF page 63

                                      www.questpod.com




* What was the architecture and data flow followed in your last project?



---



## 2. .NET Core & C# (Server-Side)



* What is the difference between an abstract class and an interface? Give a real-time scenario where
you used an abstract class in your project.

* Have you used delegates? Can you provide a real-time scenario where you used them?

* What is the difference between `First`, `FirstOrDefault`, `Single`, and `SingleOrDefault` in LINQ?

* What is the difference between `readonly`, `const`, and `static`?

* What is the difference between String and StringBuilder?

* What is middleware? How do you create custom middleware in .NET Core?

* What is the Kestrel server in .NET Core?

* How do you handle sessions in .NET Core?

* How do you implement API versioning?

* What is content negotiation in .NET Core Web API?



### SOLID Principles & Architecture



* What is the Single Responsibility Principle (SRP)?

* What is the Liskov Substitution Principle (LSP)?

* What is Dependency Injection (DI) and why do we use it?

* What is the difference between `Transient`, `Scoped`, and `Singleton` lifetimes? Where and why
did you use them in your project?

* Can we inject a `Scoped` service into a `Singleton` service?

* **Scenario-based design question:** Suppose you go to a mall, purchase items, and go to the
billing counter. There are multiple payment options available (Credit Card, Debit Card, Voucher). In
the future, new options like Cash or Coupons might be introduced. How would you design a loosely

### PDF page 64

                                     www.questpod.com
coupled system for this? How many classes and interfaces would you create so that adding a new
system requires minimal changes?



### Security & Authentication



* What mechanism have you used for authentication and authorization in your applications?

* How do you generate a JWT (JSON Web Token) in Web API?



---



## 3. SQL Server & Databases



* How do you improve query performance on the SQL Server side?

* What is the difference between a clustered index and a non-clustered index?

* What is a covering index in SQL Server?

* What is a Common Table Expression (CTE)? Have you used a CTE inside a stored procedure, and
why?

* What is the difference between a Temporary Table (`#temp`) and a Table Variable (`@table`)?

* What is the difference between `ROW_NUMBER()` and `DENSE_RANK()` window functions?

* **Coding Query Challenge:** > Write a query to find the **second highest salary department-
wise**.

> * Table 1 (`Employee`): Columns are `EmployeeID`, `Salary`, and `DepartmentID`.

> * Table 2 (`Department`): Columns are `DepartmentID` and `DepartmentName`.

> * The final result set should display: `Employee Name`, `Salary`, and `Department Name`.

>

>




---

### PDF page 65

                                     www.questpod.com


## 4. Entity Framework (EF)



* What is the difference between Eager Loading, Explicit Loading, and Lazy Loading in Entity
Framework?

* What is the $N+1$ query problem? Does it happen while using eager loading or lazy loading?



---



## 5. Angular (Client-Side)



* If there are multiple components with no direct parent-child relation, how do you transfer data
between them?

* What is the difference between a `Subject`, `BehaviorSubject`, and `ReplaySubject`?

* What is an Interceptor, and why do we use them in Angular?

* How do you maintain/keep tokens (like JWT) in Angular?

* What is the `standalone` property in Angular components, and why do we use it?

* How do you improve the performance of an Angular application?

* What is the difference between an `Observable` and a `Promise`?



---



## 6. Cloud Services (Azure / AWS)



* Have you used Azure or AWS in your project? Which specific services did you use?

* *(Clarification from interviewer)* Have you worked with Azure Functions or AWS Lambda
functions?



Wipro 2nd round held on 8th May 2026


## 1. Architecture & Design Principles

### PDF page 66

                                      www.questpod.com


* Can you explain the application architecture of your recent project? (Beyond just listing the tech
stack).

* What is the difference between a Clean Architecture and a Traditional Three-Layer Architecture?

* Are you familiar with Layered Architecture? Can you explain how it works?



---



## 2. Angular (Client-Side)



* How do you refresh an expired authentication token on the front end using an HTTP Interceptor?

* *(Follow-up context)*: On receiving a `401 Unauthorized` error from the API while the user is
actively using the app, what RxJS operators (e.g., `switchMap`, `catchError`) and logic do you use to
seamlessly request a token refresh?




* How do you make an Angular service (like a `UserService`) a global **Singleton Service** so that it
maintains user entities/state consistently across multiple screens during CRUD operations?

* What is the purpose of the `@Injectable()` decorator, and what does the `providedIn: 'root'`
configuration indicate?



---



## 3. .NET Core & C# (Server-Side)



* What is Middleware in .NET Core?

* **Module Architecture Scenario:** If you are developing a brand new "Customer Module" within
an existing application, what specific files/layers will you create on the .NET Core API side (e.g.,
Controllers, Services, Repositories, DTOs)?

* Where and how do you configure connection strings and application settings in a .NET Core web
API? (e.g., `appsettings.json`).

### PDF page 67

                                      www.questpod.com
---



## 4. LINQ & Entity Framework Core (EF Core)



* **LINQ Coding Challenge:** How do you write a LINQ query to fetch the **third highest salaried
employee** from a list of customers/employees?

* How do you handle cases where multiple employees have the exact same salary when trying to
find the $N$-th highest salary using methods like `Skip()` and `Take()`? Why might simply using
`FirstOrDefault()` fail in this scenario?



---



## 5. SQL Server & Databases



* **Index Design Strategy:** If you have a massive table with around 30 columns and need to create
indexes on it, what specific factors or query patterns (e.g., `WHERE` clauses, selectivity, complexity)
do you analyze to choose the correct columns?

* **Performance Tuning Use-Case:** You have a transactional database table containing over **1
million records**. A stored procedure needs to fetch records from this table, perform complex
calculations, and return the output. Would you use a **Common Table Expression (CTE)** or a
**Global Temporary Table (`##temp`)**, and why?

* **SQL Coding Challenge:** Write a query to **remove duplicate records** from a `User` table
that has columns `Email`, `UserID`, and `UserName` but **no primary key**, utilizing a CTE and
`PARTITION BY`.

* How many types of indexes are available in SQL Server?



---



## 6. Azure Cloud Services



* Which specific Azure cloud services have you utilized for application deployment and hosting?

* What is the difference between an **HTTP Trigger** and a **Timer Trigger** in Azure Functions?

### PDF page 68

                                      www.questpod.com
* **Azure Coding Challenge:** Write the syntax/definition for a basic **Azure Function app** using
an HTTP trigger that handles adding an entity (like a customer).

FN(LastLetterAlphabet) 4 years Questions

Explain SOLID principles with real example.

Explain Open/Closed Principle using Strategy pattern.

Does Factory pattern violate OCP?

What is Dependency Inversion Principle and how DI implements it?

Why use constructor dependency injection?

How to design interest calculator using Strategy pattern?

When to use Factory vs Dependency Injection?

How to create concrete implementation of a strategy?

Design a simple banking system (Deposit & Withdraw).

Where should business logic be placed (Controller vs Service vs Domain)?

Why should controller not contain business logic?

How do you handle different account types without if-else?

How do you enforce rules like minimum balance?

What is resource leak?

Why should we use using keyword in C#?

What is SQL injection and how to prevent it?

What is race condition in banking transactions?

Why should we use database transactions?

Show total balance across accounts for each customer.

How to optimize query for latest transaction per account?



…



diff transiet and scope lifetime method on .net core

what are performance level use on EF core

diff eager loading and lazy loading

### PDF page 69

                                    www.questpod.com
how use or implent the oath 2.o security implent



Aync and await

angular - auth guard,interceptor




group by

having clause




id dept     country

1    IT    USA

2     HR    USA

3     Account     USA




select country from tblcountry group by country



string str="sarbeswarsahoo";

int count=0;

for(int i=0; i > str.length)

{

foreach(Listitem li in str[1]-1)

{

if(li )

### PDF page 70

                          www.questpod.com
{




}

char ch = li

count = li.count;

console.Write(count + )



}




}
