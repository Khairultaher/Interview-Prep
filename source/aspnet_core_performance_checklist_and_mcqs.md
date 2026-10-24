# ASP.NET Core High-Performance API Tuning Checklist

Senior .NET developer and solution-architect reference. Use this
checklist to guide profiling, code reviews, and production
readiness---not as a substitute for measurement.

## 1. Establish a baseline first

-   [ ] Define latency and throughput targets, including p50, p95, and
    p99.
-   [ ] Load-test with realistic request sizes, concurrency,
    authentication, and data volumes.
-   [ ] Capture CPU, memory, allocation rate, garbage collection,
    thread-pool queueing, database time, and outbound dependency
    latency.
-   [ ] Compare changes against a repeatable baseline.
-   [ ] Profile before optimizing; do not assume the
    controller-versus-Minimal-API choice is the bottleneck.

## 2. HTTP pipeline and middleware

-   [ ] Keep middleware minimal and avoid unnecessary per-request
    allocations.
-   [ ] Preserve correct ordering for exception handling, routing, CORS,
    authentication, authorization, rate limiting, and endpoint
    execution.
-   [ ] Avoid synchronous blocking, expensive serialization, database
    calls, or repeated remote calls in generic middleware.
-   [ ] Short-circuit only when the request can safely be completed at
    that point.
-   [ ] Use `UseWhen` or `MapWhen` only when conditional pipeline
    branching is genuinely useful.
-   [ ] Ensure custom middleware correctly awaits `next(context)` and
    does not write to a response after it has started.

## 3. Minimal APIs versus controllers

-   [ ] Choose the endpoint model based on required features, team
    conventions, and maintainability.
-   [ ] Consider Minimal APIs for small services, focused endpoints, and
    simple HTTP surfaces.
-   [ ] Consider controllers when MVC features such as action filters,
    application conventions, and established controller organization are
    useful.
-   [ ] Remember that both use ASP.NET Core middleware, routing,
    dependency injection, authentication, authorization, and endpoint
    metadata.
-   [ ] Benchmark the actual application before claiming one style is
    materially faster. Database, network, serialization, and payload
    costs often dominate.

## 4. Request binding, validation, and response design

-   [ ] Keep request and response DTOs appropriately sized.
-   [ ] Validate input at the API boundary and enforce domain invariants
    in the application/domain layer.
-   [ ] Avoid returning EF Core entities directly.
-   [ ] Return only the fields clients need.
-   [ ] Avoid unnecessary buffering of request or response bodies.
-   [ ] Use pagination and sensible limits for collection endpoints.
-   [ ] Use `System.Text.Json` defaults unless a measured or
    compatibility requirement justifies customization.
-   [ ] Enable response compression where it benefits the workload and
    does not create unacceptable CPU overhead.

## 5. Asynchronous work and thread-pool health

-   [ ] Use asynchronous APIs for I/O-bound work, including database and
    HTTP calls.
-   [ ] Avoid `.Result`, `.Wait()`, and other sync-over-async patterns.
-   [ ] Do not wrap ordinary asynchronous I/O in `Task.Run`.
-   [ ] Use `IHttpClientFactory` or an appropriately managed
    `HttpClient`.
-   [ ] Apply timeouts and cancellation tokens to outbound calls and
    long-running operations.
-   [ ] Bound concurrency for expensive work; move durable, long-running
    jobs to background processing when appropriate.
-   [ ] Do not change thread-pool settings without evidence that the
    defaults are inadequate.

## 6. EF Core and database access

-   [ ] Inspect generated SQL and query plans for slow queries.
-   [ ] Use `AsNoTracking()` for read-only entity queries where tracking
    is not needed.
-   [ ] Project only required columns with `Select`.
-   [ ] Detect and eliminate N+1 query patterns.
-   [ ] Use `Include` only when related data is needed; avoid loading
    large graphs by default.
-   [ ] Apply pagination and suitable database indexes.
-   [ ] Avoid holding database transactions open while performing
    unrelated network calls.
-   [ ] Measure connection-pool pressure, query duration, and database
    CPU---not only API execution time.

## 7. Caching

-   [ ] Cache only data whose freshness and invalidation requirements
    are understood.
-   [ ] Use in-memory caching for process-local data and distributed
    caching when shared state across instances is required.
-   [ ] Consider output caching for suitable HTTP responses.
-   [ ] Set expiration and size limits; define invalidation behavior.
-   [ ] Do not cache user-specific or tenant-specific data under a
    shared key.
-   [ ] Ensure authorization and cache variation rules cannot leak one
    user's response to another.
-   [ ] Measure cache hit rate and backend load before and after
    changes.

## 8. Security and resilience

-   [ ] Validate JWT signature, issuer, audience, and lifetime using
    supported authentication middleware.
-   [ ] Apply endpoint authorization policies consistently.
-   [ ] Use rate limiting for abuse-sensitive endpoints and choose
    partition keys appropriate to the workload.
-   [ ] Apply timeouts and bounded retries to remote dependencies; avoid
    retry storms.
-   [ ] Use HTTPS and secure secret management.
-   [ ] Do not log access tokens, passwords, secrets, or sensitive
    request bodies.
-   [ ] Avoid disabling validation or security features merely to reduce
    latency.

## 9. Observability and production verification

-   [ ] Use structured logs, metrics, and distributed tracing.
-   [ ] Track request rate, error rate, duration, and saturation.
-   [ ] Monitor p95/p99 latency and dependency-specific timings.
-   [ ] Include correlation/trace identifiers without exposing sensitive
    data.
-   [ ] Verify performance under expected peak load and failure
    conditions.
-   [ ] Add regression tests or load-test thresholds for critical
    endpoints.

------------------------------------------------------------------------

# Interview MCQs: Minimal APIs versus Controllers

Each question has one best answer. The questions focus on distinctions
that commonly matter in senior .NET interviews.

## Q1. Which statement best describes Minimal APIs and controller-based APIs?

A. Minimal APIs bypass the ASP.NET Core middleware pipeline.\
B. Controllers cannot use dependency injection.\
C. Both can use the ASP.NET Core pipeline; they differ in endpoint
programming model and available framework features.\
D. Minimal APIs cannot require authorization.

**Answer: C.** Both use middleware, routing, DI, and authorization. The
programming model and feature set differ.

## Q2. What is a sound reason to choose Minimal APIs?

A. They automatically make every database query faster.\
B. They provide a concise endpoint model for focused HTTP services.\
C. They eliminate the need for input validation.\
D. They bypass authentication middleware.

**Answer: B.** Minimal APIs reduce ceremony, but they do not remove the
need for validation or security.

## Q3. What is a sound reason to choose controllers?

A. Controllers are always faster.\
B. Controllers do not support middleware.\
C. MVC features such as action filters, conventions, and controller
organization are valuable to the application.\
D. Controllers automatically prevent SQL injection regardless of
data-access code.

**Answer: C.** Select controllers when their MVC capabilities and
organizational model are useful.

## Q4. Can a single ASP.NET Core application use both Minimal APIs and controllers?

A. No.\
B. Yes, by mapping Minimal API endpoints and calling
`MapControllers()`.\
C. Only when authentication is disabled.\
D. Only in development.

**Answer: B.** Both endpoint styles can coexist in one application.

## Q5. Which statement about authorization is correct?

A. Only controllers can use authorization.\
B. Minimal APIs cannot use authorization policies.\
C. Controllers and Minimal APIs can both be protected using ASP.NET Core
authorization.\
D. Authorization should be implemented only in JavaScript.

**Answer: C.** Controllers can use attributes such as `[Authorize]`;
Minimal APIs can use endpoint conventions such as
`.RequireAuthorization()`.

## Q6. Which feature is specifically associated with the MVC controller filter pipeline?

A. Middleware\
B. Action filters\
C. Kestrel\
D. `HttpClient`

**Answer: B.** Action filters run around MVC action execution.
Middleware surrounds endpoint processing at the HTTP pipeline level.

## Q7. Do Minimal APIs support endpoint filters?

A. No, they support only middleware.\
B. Yes, endpoint filters can run around Minimal API handler execution.\
C. Only if the endpoint is a controller action.\
D. Only in .NET Framework.

**Answer: B.** Endpoint filters are distinct from MVC action filters and
are supported by Minimal APIs.

## Q8. Which statement about model validation is most accurate?

A. Minimal APIs never need validation.\
B. Controllers using `[ApiController]` provide automatic model-state
validation behavior for supported scenarios; Minimal APIs have their own
binding and validation options depending on framework version and
configuration.\
C. All validation must occur in middleware.\
D. Database constraints replace API validation.

**Answer: B.** Know the behavior of the target .NET version rather than
assuming the two models behave identically.

## Q9. An API's p99 latency is high because a database query takes 800 ms. What should be investigated first?

A. Rewrite every controller as a Minimal API.\
B. Inspect the query, execution plan, indexes, data access pattern, and
database load.\
C. Remove authentication.\
D. Increase JSON indentation.

**Answer: B.** Optimize the measured bottleneck; changing endpoint style
is unlikely to fix slow database execution.

## Q10. Which approach is best for comparing Minimal API and controller performance?

A. Assume Minimal APIs are always twice as fast.\
B. Compare unrelated applications from different machines.\
C. Benchmark equivalent endpoints under controlled, representative
conditions.\
D. Compare only source-code line counts.

**Answer: C.** Use equivalent workloads and measure latency, throughput,
CPU, allocations, and end-to-end dependencies.

## Q11. What is the main risk of caching an authenticated API response under a key that ignores the user or tenant?

A. The response always becomes slower.\
B. One user's or tenant's data may be returned to another.\
C. The JWT signature changes.\
D. Routing stops working.

**Answer: B.** Cache-key design and authorization boundaries are
security-critical.

## Q12. Which is the best general rule for asynchronous I/O in request handlers?

A. Use `.Result` to reduce allocations.\
B. Wrap every I/O call in `Task.Run`.\
C. Await asynchronous I/O and propagate cancellation where appropriate.\
D. Make all methods `async` even when no asynchronous work exists.

**Answer: C.** Avoid sync-over-async and unnecessary thread-pool work.

## Q13. Which statement about middleware order is correct?

A. Middleware order never matters.\
B. Authentication should run after authorization.\
C. Middleware order affects behavior; authentication normally precedes
authorization, and routing must be configured appropriately for
endpoint-aware middleware.\
D. Middleware executes only for controller endpoints.

**Answer: C.** Follow the documented pipeline requirements for the
middleware being used.

## Q14. What is the best use of `AsNoTracking()`?

A. Every query, including all updates.\
B. Read-only EF Core queries where change tracking is not required.\
C. To disable database indexes.\
D. To make network calls asynchronous.

**Answer: B.** It can reduce tracking overhead for read-only entity
queries; measure the impact.

## Q15. Which performance practice is most reliable?

A. Remove all filters regardless of purpose.\
B. Disable JWT validation.\
C. Profile representative workloads, identify the bottleneck, make a
targeted change, and measure again.\
D. Replace every controller with a Minimal API.

**Answer: C.** Performance engineering should be evidence-driven and
must preserve correctness and security.

------------------------------------------------------------------------

## Quick review: interview takeaways

-   **Minimal APIs vs controllers:** choose based on features,
    consistency, and maintainability; benchmark before making
    performance claims.
-   **Middleware:** operates across the HTTP request pipeline.
-   **MVC filters:** provide hooks around MVC/controller execution.
-   **Endpoint filters:** provide hooks around supported endpoint
    handlers, especially Minimal APIs.
-   **Performance:** profile end-to-end latency, data access,
    allocations, serialization, caching, and dependencies.
-   **Security:** never trade away authentication, authorization, tenant
    isolation, or token validation to gain speed.
