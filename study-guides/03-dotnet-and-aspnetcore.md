# .NET and ASP.NET Core

This guide covers each topic once. Where several source notes explained the same thing, they have been merged into a single section.

## Contents

- [Middleware, filters, and attributes](#aspnetcore-middleware-filters-dotnet8-9-md)
- [Middleware and filters — extra senior notes](#middleware-senior-extras)
- [High-performance API tuning checklist](#aspnet-core-performance-checklist-and-mcqs-md)
- [Dependency injection in .NET](#dependency-injection)
- [IIS, application pools, and Kestrel](#iis-kestrel-senior-dotnet-viva-md)
- [SignalR with .NET 8](#signalr-dotnet8-viva-guide-md)
---

<a id="aspnetcore-middleware-filters-dotnet8-9-md"></a>

## ASP.NET Core Middleware, Filters, and Attributes in .NET 8/9

> A practical guide to how the request pipeline works, how middleware
> and MVC filters execute internally, when to use each, and how to test
> your understanding with HackerRank-style MCQs.

#### Contents

1.  [Learning goals and terminology](#1-learning-goals-and-terminology)
2.  [The ASP.NET Core request
    pipeline](#2-the-aspnet-core-request-pipeline)
3.  [Middleware: what it is and how it
    works](#3-middleware-what-it-is-and-how-it-works)
4.  [Use, Run, Map, and UseWhen](#4-use-run-map-and-usewhen)
5.  [Writing and registering
    middleware](#5-writing-and-registering-middleware)
6.  [Middleware DI, lifetimes, and
    factories](#6-middleware-di-lifetimes-and-factories)
7.  [Middleware ordering and common
    built-ins](#7-middleware-ordering-and-common-built-ins)
8.  [MVC filters and attributes](#8-mvc-filters-and-attributes)
9.  [How filters execute internally](#9-how-filters-execute-internally)
10. [Filter types and use cases](#10-filter-types-and-use-cases)
11. [Creating and registering custom
    filters](#11-creating-and-registering-custom-filters)
12. [Middleware vs filters vs endpoint filters vs
    attributes](#12-middleware-vs-filters-vs-endpoint-filters-vs-attributes)
13. [Decision guide and real-world
    scenarios](#13-decision-guide-and-real-world-scenarios)
14. [Common mistakes and interview
    notes](#14-common-mistakes-and-interview-notes)
15. [HackerRank-style MCQs](#15-hackerrank-style-mcqs)
16. [Answer key](#16-answer-key)

------------------------------------------------------------------------

#### 1. Learning goals and terminology

In ASP.NET Core, **middleware** and **filters** are extension points for
code that runs while an HTTP request is being handled.

-   **Middleware** belongs to the application-wide HTTP pipeline. It
    works with `HttpContext` and can run before and after downstream
    middleware.
-   **MVC filters** belong to the MVC controller/action execution
    pipeline. They can access action arguments, controller context,
    model state, and action results.
-   **Attributes** are metadata placed on classes, methods, parameters,
    or properties. Some ASP.NET Core attributes configure behavior; some
    implement or select filters. An attribute is not automatically a
    filter.
-   **Endpoint filters** are a separate feature primarily for Minimal
    APIs. They wrap endpoint-handler execution and can inspect
    arguments/results.
-   **Authorization** should normally use ASP.NET Core
    authentication/authorization services and policies, not a home-grown
    authorization filter or middleware.

#### 2. The ASP.NET Core request pipeline

A simplified controller API request often follows this shape:

``` text
HTTP request
    |
    v
Exception handling middleware
    |
HTTPS redirection / security / CORS (depending on configuration)
    |
Static files (if configured; may short-circuit)
    |
Routing middleware
    |
Authentication middleware
    |
Authorization middleware
    |
Endpoint execution
    |
    +-- MVC controller endpoint:
    |      MVC authorization filters
    |      Resource filters
    |      Model binding and validation
    |      Action filters
    |      Controller action
    |      Exception filters if an eligible exception occurs
    |      Result filters / result execution
    |
    +-- Minimal API endpoint:
           Endpoint filters (if configured)
           Handler
    |
Response travels back through middleware in reverse order
    |
HTTP response
```

This is a conceptual overview, not a universal exact sequence.
Middleware order is configured by the application, and endpoint types
differ. MVC's filter pipeline is entered only when an MVC endpoint
executes.

##### Why middleware has a "before" and "after" phase

A middleware component can execute code before it calls the next
delegate and after that delegate completes:

``` csharp
app.Use(async (context, next) =>
{
    // Before downstream components
    Console.WriteLine("Before");

    await next(context);

    // After downstream components return
    Console.WriteLine("After");
});
```

If multiple middleware are registered in order A, B, C, the request
phase is A → B → C and the response/unwinding phase is C → B → A.

A middleware can **short-circuit** by not calling `next`. Later
middleware and endpoint execution will not run.

#### 3. Middleware: what it is and how it works

A middleware component is code in the HTTP request pipeline. It
commonly:

1.  Reads or modifies `HttpContext.Request`.
2.  Calls the next middleware.
3.  Reads or modifies the response after downstream execution.
4.  Or returns a response immediately without calling the next
    middleware.

##### How the pipeline is built

At application startup, calls such as `app.Use(...)`, `app.Run(...)`,
and `app.UseMiddleware<T>()` register components. ASP.NET Core builds
them into a chain of request delegates. When a request arrives, the
first delegate runs and may invoke the next delegate.

Conceptually:

``` text
RequestDelegate A
  └── RequestDelegate B
        └── RequestDelegate C
              └── Endpoint
```

This is a conceptual model; the implementation uses delegates and
pipeline-building machinery rather than a literal linked-list object.

##### Minimal example

``` csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Use(async (context, next) =>
{
    Console.WriteLine($"Incoming: {context.Request.Method} {context.Request.Path}");

    await next(context);

    Console.WriteLine($"Outgoing: {context.Response.StatusCode}");
});

app.MapGet("/", () => "Hello from ASP.NET Core");

app.Run();
```

**Important:** In modern ASP.NET Core, the `next` delegate supplied to
`app.Use` is commonly called as `await next(context)`. The `Use`
overload that accepts `RequestDelegate` differs from older examples that
show a parameterless `next()`.

#### 4. Use, Run, Map, and UseWhen

##### `Use`

Adds middleware that can call the next component.

``` csharp
app.Use(async (context, next) =>
{
    // Before
    await next(context);
    // After
});
```

Use it for logging, timing, headers, correlation IDs, and other logic
that should wrap downstream execution.

##### `Run`

Adds a terminal middleware delegate. It does not receive a `next`
delegate, so it ends the pipeline if execution reaches it.

``` csharp
app.Run(async context =>
{
    context.Response.ContentType = "text/plain";
    await context.Response.WriteAsync("Pipeline ended here.");
});
```

A terminal `Run` registered early can prevent endpoints registered later
from being reached. `WebApplication.Run()` with no delegate, used at the
end of `Program.cs`, instead starts the web host; do not confuse the two
overloads.

##### `Map`

Branches the middleware pipeline based on a path prefix. The matched
prefix is removed from `Request.Path` inside the branch and placed in
`Request.PathBase`.

``` csharp
app.Map("/admin", admin =>
{
    admin.Run(async context =>
    {
        await context.Response.WriteAsync(
            $"Admin branch. Remaining path: {context.Request.Path}");
    });
});
```

A request to `/admin/users` enters this branch with a remaining path of
`/users`. `Map` is useful when a whole path subtree needs a different
pipeline. It is not the same as endpoint routing's `MapGet` or
`MapControllers`.

##### `UseWhen`

Creates a conditional branch based on a predicate. If the branch
pipeline completes normally, execution can rejoin the main pipeline.

``` csharp
app.UseWhen(
    context => context.Request.Path.StartsWithSegments("/reports"),
    branch =>
    {
        branch.Use(async (context, next) =>
        {
            Console.WriteLine("Reports-specific logic");
            await next(context);
        });
    });
```

Use it for conditional cross-cutting behavior, such as extra logging for
selected paths. It is not a substitute for endpoint authorization.

##### Quick comparison

  -----------------------------------------------------------------------
  API                     Main purpose            Can continue to later
                                                  middleware?
  ----------------------- ----------------------- -----------------------
  `Use`                   Add wrapping middleware Yes, if it calls `next`

  `Run` (delegate         Terminal middleware     No next delegate
  overload)                                       

  `Map`                   Path-prefix branch      The branch itself is
                                                  separate; a terminal
                                                  branch ends that branch

  `UseWhen`               Predicate-based branch  Normally rejoins the
                                                  main pipeline if branch
                                                  calls through
  -----------------------------------------------------------------------

#### 5. Writing and registering middleware

There are two common class-based approaches.

##### Conventional middleware

``` csharp
public sealed class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(
        RequestDelegate next,
        ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var started = System.Diagnostics.Stopwatch.StartNew();

        await _next(context);

        started.Stop();
        _logger.LogInformation(
            "{Method} {Path} returned {StatusCode} in {ElapsedMs} ms",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            started.ElapsedMilliseconds);
    }
}
```

Register it:

``` csharp
app.UseMiddleware<RequestTimingMiddleware>();
```

The middleware instance is generally created when the pipeline is built.
Constructor-injected dependencies must therefore be appropriate for that
lifetime. Do **not** inject a scoped service such as a typical EF Core
`DbContext` into a conventional middleware constructor. Put per-request
scoped dependencies in `InvokeAsync` parameters or use `IMiddleware`.

##### Extension method for clean registration

An extension method makes the registration expressive and reusable; it
does not change the middleware's execution semantics.

``` csharp
public static class RequestTimingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(
        this IApplicationBuilder app)
        => app.UseMiddleware<RequestTimingMiddleware>();
}
```

Use it in `Program.cs`:

``` csharp
app.UseRequestTiming();
```

##### Inline middleware

For small, one-off behavior, inline middleware is reasonable:

``` csharp
app.Use(async (context, next) =>
{
    context.Response.Headers["X-App-Version"] = "1";
    await next(context);
});
```

For substantial logic, prefer a named class so it can be tested and
maintained independently.

#### 6. Middleware DI, lifetimes, and factories

##### Conventional middleware and scoped dependencies

Conventional middleware is constructed while the pipeline is built. A
scoped service should not normally be constructor-injected into it.

You can request a scoped dependency in the `InvokeAsync` method:

``` csharp
public sealed class TenantMiddleware
{
    private readonly RequestDelegate _next;

    public TenantMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(
        HttpContext context,
        ITenantResolver tenantResolver)
    {
        var tenant = await tenantResolver.ResolveAsync(context);
        context.Items["Tenant"] = tenant;

        await _next(context);
    }
}
```

ASP.NET Core resolves additional `InvokeAsync` parameters from request
services. This pattern is useful when the middleware itself does not
need to be resolved as a DI service per request.

##### Factory-based middleware with `IMiddleware`

Use `IMiddleware` when you want the middleware instance itself to be
created through DI, especially when it needs scoped services in its
constructor.

``` csharp
public sealed class AuditMiddleware : IMiddleware
{
    private readonly IAuditService _auditService;
    private readonly ILogger<AuditMiddleware> _logger;

    public AuditMiddleware(
        IAuditService auditService,
        ILogger<AuditMiddleware> logger)
    {
        _auditService = auditService;
        _logger = logger;
    }

    public async Task InvokeAsync(
        HttpContext context,
        RequestDelegate next)
    {
        _logger.LogInformation("Request started");
        await _auditService.RecordStartAsync(context);

        await next(context);

        await _auditService.RecordEndAsync(
            context, context.Response.StatusCode);
    }
}
```

Register the middleware and its dependencies:

``` csharp
builder.Services.AddScoped<IAuditService, AuditService>();
builder.Services.AddScoped<AuditMiddleware>();

// After builder.Build():
app.UseMiddleware<AuditMiddleware>();
```

For `IMiddleware`, ASP.NET Core uses `IMiddlewareFactory` to resolve the
middleware from the request service provider. The registered lifetime
controls its lifetime; scoped is a common choice when it consumes scoped
services.

##### Which approach should you use?

-   **Conventional middleware:** good default for simple middleware;
    constructor dependencies should be safe for the pipeline lifetime.
-   **Conventional middleware with `InvokeAsync` parameters:** good when
    only a request-scoped dependency is needed for a single invocation.
-   **`IMiddleware`:** good when the middleware itself needs DI-managed,
    per-request construction or scoped constructor dependencies.

#### 7. Middleware ordering and common built-ins

Middleware order matters because each component can change what later
components see or stop execution.

A common controller API configuration looks like:

``` csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddAuthentication();
builder.Services.AddAuthorization();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseRouting();

app.UseCors("DefaultPolicy"); // If CORS is configured
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

Configure CORS services and policies if using `UseCors`. The exact
placement can vary with the application's endpoint and hosting
configuration; follow the ASP.NET Core guidance for the specific setup.

Common middleware:

  -----------------------------------------------------------------------
  Middleware                          Typical purpose
  ----------------------------------- -----------------------------------
  `UseExceptionHandler`               Production exception handling

  `UseDeveloperExceptionPage`         Detailed development errors; not
                                      for production

  `UseHttpsRedirection`               Redirect HTTP requests to HTTPS

  `UseHsts`                           Tell browsers to prefer HTTPS

  `UseStaticFiles`                    Serve static files (if
                                      used/configured)

  `UseRouting`                        Select an endpoint

  `UseCors`                           Apply CORS policy

  `UseAuthentication`                 Establish the current principal

  `UseAuthorization`                  Enforce endpoint authorization

  `UseRateLimiter`                    Apply configured rate-limiting
                                      policies

  `UseResponseCaching` /              Cache responses according to
  `UseOutputCache`                    configured policies

  `UseSession`                        Enable session state when
                                      configured
  -----------------------------------------------------------------------

**Ordering reminders:** - Exception-handling middleware should be early
enough to catch errors from components downstream of it. -
Authentication should normally run before authorization. - Routing must
select an endpoint before endpoint-specific authorization metadata can
be evaluated. - A middleware that short-circuits means later middleware
and the endpoint may never execute. - Middleware can observe static-file
requests only if it is registered before static-file middleware;
static-file middleware may end the pipeline when it serves a file.

#### 8. MVC filters and attributes

##### What is an attribute?

An attribute is metadata attached to code. Examples:

``` csharp
[Authorize]
[HttpGet("{id:int}")]
[ProducesResponseType(StatusCodes.Status200OK)]
public IActionResult GetById(int id) => Ok();
```

These attributes do not all work the same way: - `[HttpGet]` contributes
routing/HTTP-method metadata. - `[ProducesResponseType]` contributes
API-description metadata. - `[Authorize]` supplies authorization
metadata consumed by the ASP.NET Core authorization system. - A custom
filter attribute can implement a filter interface or select a filter
implementation.

Therefore, **"attribute" and "filter" are not synonyms**. An attribute
may configure framework behavior without being an MVC filter.

##### What is an MVC filter?

MVC filters run inside the MVC action-invocation pipeline after routing
has selected an MVC endpoint. They can access MVC-specific information
such as action arguments, model state, controller/action descriptors,
and action results.

A typical MVC filter can be: - Registered globally. - Applied to a
controller. - Applied to an action. - Created through an attribute,
`ServiceFilter`, `TypeFilter`, or a global DI registration.

Filters do not run for every HTTP request. For example, a request served
directly by static-file middleware does not execute MVC action filters.

#### 9. How filters execute internally

For a controller action, the simplified sequence is:

``` text
Endpoint routing selects an MVC endpoint
    |
MVC authorization filters
    |
Resource filters: before
    |
Model binding and model validation
    |
Action filters: before
    |
Controller action
    |
Action filters: after
    |
Result filters: before result execution
    |
Result executes
    |
Result filters: after
    |
Response
```

Exception filters are a separate exception-handling stage within MVC.
They can handle eligible exceptions from controller creation, model
binding, action filters, or action execution, but they do **not** handle
every exception from every part of the request pipeline. In particular,
they do not replace middleware-level exception handling.

##### Before/after nesting

Filters with before/after methods generally wrap later stages. If filter
A wraps filter B, the before methods run A → B and the after methods run
B → A.

Filter ordering can be affected by: - Filter type/stage. - Scope:
global, controller, or action. - `IOrderedFilter.Order`. -
Framework-defined behavior for a particular filter stage.

Do not assume that every filter stage has identical ordering rules.
Authorization filters run early; resource filters surround much of the
MVC processing; action filters wrap action invocation; result filters
wrap result execution.

#### 10. Filter types and use cases

  -----------------------------------------------------------------------------
  Filter stage            Common interface              What it is good for
  ----------------------- ----------------------------- -----------------------
  Authorization           `IAuthorizationFilter` /      MVC-specific
                          `IAsyncAuthorizationFilter`   authorization checks;
                                                        normally prefer
                                                        `[Authorize]` and
                                                        policies

  Resource                `IResourceFilter` /           Logic around most of
                          `IAsyncResourceFilter`        MVC processing;
                                                        sometimes
                                                        short-circuiting or
                                                        specialized caching

  Action                  `IActionFilter` /             Logic before/after
                          `IAsyncActionFilter`          controller action
                                                        execution; action
                                                        arguments and result
                                                        are available

  Exception               `IExceptionFilter` /          MVC-specific exception
                          `IAsyncExceptionFilter`       handling

  Result                  `IResultFilter` /             Logic before/after MVC
                          `IAsyncResultFilter`          result execution

  Always-run result       `IAlwaysRunResultFilter` /    Result handling for
                          async counterpart             certain short-circuit
                                                        results where ordinary
                                                        result filters might
                                                        not run
  -----------------------------------------------------------------------------

##### Authorization filters

Authorization filters run early in MVC. For most applications, use the
built-in authorization system:

``` csharp
[Authorize(Policy = "CanReadReports")]
[HttpGet("reports")]
public IActionResult GetReports() => Ok();
```

Define policies in service registration and use them consistently. Avoid
duplicating authentication and authorization logic in custom filters
unless there is a specific MVC-only need.

##### Resource filters

Resource filters can short-circuit before model binding and surround
much of the remaining MVC pipeline. They can be useful for specialized
MVC-level caching or work that should happen before model binding. They
are not a replacement for middleware that must apply to all HTTP
endpoints.

##### Action filters

Action filters are useful when logic specifically concerns controller
action invocation, such as auditing action names or inspecting action
arguments.

``` csharp
public sealed class ActionTimingFilter : IActionFilter
{
    private readonly ILogger<ActionTimingFilter> _logger;

    public ActionTimingFilter(ILogger<ActionTimingFilter> logger)
        => _logger = logger;

    public void OnActionExecuting(ActionExecutingContext context)
        => _logger.LogInformation(
            "Starting {Action}",
            context.ActionDescriptor.DisplayName);

    public void OnActionExecuted(ActionExecutedContext context)
        => _logger.LogInformation(
            "Finished {Action}",
            context.ActionDescriptor.DisplayName);
}
```

For asynchronous work, use `IAsyncActionFilter` and `await next()`
rather than blocking asynchronous operations.

##### Exception filters

Exception filters can translate eligible MVC exceptions into an MVC
result. For application-wide exception handling, including middleware
failures and non-MVC endpoints, prefer exception-handling middleware or
the built-in exception-handler infrastructure.

##### Result filters

Result filters wrap MVC result execution. Use them when you need to act
around MVC result generation. They do not universally intercept every
response produced by the application: middleware, Minimal APIs, static
files, and some short-circuit paths can bypass MVC result filters.

#### 11. Creating and registering custom filters

##### Attribute-based filter for simple, self-contained behavior

A custom attribute can implement `IActionFilter`:

``` csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public sealed class AddActionHeaderAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        context.HttpContext.Response.Headers["X-Action-Filter"] = "Applied";
    }
}
```

Apply it:

``` csharp
[AddActionHeader]
[HttpGet("example")]
public IActionResult Example() => Ok(new { Message = "Hello" });
```

This is suitable for simple behavior that does not need
constructor-injected application services.

##### `ServiceFilter`

Use `ServiceFilter` when the filter is registered in DI:

``` csharp
public sealed class AuditActionFilter : IAsyncActionFilter
{
    private readonly IAuditService _auditService;

    public AuditActionFilter(IAuditService auditService)
        => _auditService = auditService;

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        await _auditService.RecordActionStartAsync(
            context.ActionDescriptor.DisplayName);

        var executed = await next();

        await _auditService.RecordActionEndAsync(
            context.ActionDescriptor.DisplayName,
            executed.Exception is null);
    }
}
```

Registration and usage:

``` csharp
builder.Services.AddScoped<AuditActionFilter>();

// On a controller or action:
[ServiceFilter(typeof(AuditActionFilter))]
public IActionResult Update() => Ok();
```

`ServiceFilter` asks DI to resolve the filter. Register the filter and
its dependencies.

##### `TypeFilter`

`TypeFilter` creates the filter type and uses DI to supply constructor
dependencies. The filter type itself does not have to be registered as a
service in the same way as a `ServiceFilter`, although its dependencies
must be resolvable.

``` csharp
[TypeFilter(typeof(AuditActionFilter))]
public IActionResult Update() => Ok();
```

Prefer a clear, consistent registration approach. For filters applied
broadly, global registration may be simpler.

##### Global filter registration

``` csharp
builder.Services.AddControllers(options =>
{
    options.Filters.Add<AuditActionFilter>();
});
```

Ensure the filter is registered appropriately in DI, and choose a
lifetime compatible with its dependencies. Global registration applies
it across MVC actions unless its logic or configuration limits the
behavior.

#### 12. Middleware vs filters vs endpoint filters vs attributes

  -----------------------------------------------------------------------
  Question                            Best starting point
  ----------------------------------- -----------------------------------
  Must it run for nearly every HTTP   Middleware
  request, including non-MVC          
  endpoints?                          

  Must it wrap the entire downstream  Middleware
  pipeline?                           

  Does it need controller action      MVC filter
  arguments or MVC model state?       

  Does it need to wrap controller     Action filter
  action execution?                   

  Does it need to wrap MVC result     Result filter
  execution?                          

  Is it an authorization rule for an  `[Authorize]` and authorization
  endpoint/controller?                policies

  Does it wrap a Minimal API handler  Endpoint filter
  and inspect handler                 
  arguments/results?                  

  Is it only routing/API metadata,    Appropriate attribute/endpoint
  such as HTTP method or response     metadata
  documentation?                      
  -----------------------------------------------------------------------

##### Endpoint filters for Minimal APIs

Endpoint filters are distinct from MVC filters. They are designed for
Minimal API handlers:

``` csharp
app.MapGet("/hello", (string name) => $"Hello, {name}")
   .AddEndpointFilter(async (context, next) =>
   {
       // Inspect context.Arguments before the handler
       var result = await next(context);
       // Inspect or transform the result
       return result;
   });
```

Use endpoint filters when behavior is specific to Minimal API endpoint
invocation. Do not expect an MVC `IActionFilter` to run for a Minimal
API handler.

#### 13. Decision guide and real-world scenarios

##### Scenario A: Log every request, including static files

**Use middleware**, registered before static-file middleware if
static-file requests must be logged. An MVC action filter will miss
static files and non-MVC endpoints.

##### Scenario B: Audit only selected controller actions

**Use an action filter** if the audit needs MVC action metadata or
action arguments. If auditing is a security/compliance requirement,
define exactly what data is recorded, avoid logging secrets, and make
the audit write resilient.

##### Scenario C: Add correlation ID to all requests

**Use middleware.** Create or validate a correlation ID, store it in
`HttpContext`, and optionally return it in a response header.

##### Scenario D: Enforce a role/policy on a controller

**Use `[Authorize]` with a policy or role**, backed by
`AddAuthentication` and `AddAuthorization`. Do not roll your own access
control in a generic action filter.

##### Scenario E: Validate a request DTO

Use normal model validation and API conventions first. For controller
APIs, `[ApiController]` can automatically return a 400 response for
invalid model state. Use an action filter only when you need reusable
custom behavior beyond built-in validation. Middleware is usually too
early because it does not naturally have the bound DTO or `ModelState`.

##### Scenario F: Handle unhandled exceptions application-wide

**Use exception-handling middleware** (such as `UseExceptionHandler`) so
it can cover MVC, Minimal APIs, and downstream middleware. Use exception
filters only for MVC-specific exception-to-result behavior.

##### Scenario G: Apply behavior to Minimal API handler arguments

**Use an endpoint filter** or handler-level validation pattern. MVC
action filters are for MVC controllers.

##### Scenario H: Add a response header to all responses

**Use middleware**, with attention to when the response starts. Set the
header before calling `next` when possible; once the response has
started, headers generally cannot be changed.

##### Scenario I: Cache a complete response

Choose the built-in output caching or response caching features when
they meet the requirements. Use resource filters only for specialized
MVC-level caching, not as the default general-purpose HTTP caching
mechanism.

#### 14. Common mistakes and interview notes

1.  **Thinking every attribute is a filter.** Attributes can supply
    routing, authorization, or documentation metadata without
    implementing an MVC filter.
2.  **Using filters for all cross-cutting concerns.** Filters do not
    cover static files or every endpoint type.
3.  **Using middleware when action arguments are required.** Middleware
    runs before MVC action invocation and does not naturally receive
    bound action arguments.
4.  **Injecting scoped services into conventional middleware
    constructors.** Use `InvokeAsync` parameters or `IMiddleware`.
5.  **Assuming `UseWhen` is path-specific.** Its condition can be any
    predicate; use `Map` when branching on a path prefix.
6.  **Assuming `Map` and `MapGet` are the same.** `Map` branches the
    middleware pipeline; `MapGet` maps a Minimal API route.
7.  **Putting authentication after authorization.** Authentication
    normally must run first so authorization can use the established
    user principal.
8.  **Relying only on exception filters.** They do not catch all
    application exceptions.
9.  **Changing response headers after the response has started.**
    Headers must be set before they are committed.
10. **Blocking asynchronous work in filters or middleware.** Prefer
    async interfaces and `await`.
11. **Assuming middleware order is cosmetic.** Order changes what runs,
    what sees the request, and whether an endpoint is reached.
12. **Using a custom filter instead of built-in authorization or
    validation without a reason.** Prefer framework features where
    possible.

##### A useful mental model

-   **Middleware:** "Should this HTTP request continue through the
    application pipeline, and what should happen around that journey?"
-   **MVC filter:** "What should happen around this
    controller/action/result?"
-   **Endpoint filter:** "What should happen around this Minimal API
    handler?"
-   **Attribute:** "What metadata or declarative behavior should be
    attached to this code element?"

------------------------------------------------------------------------

#### 15. HackerRank-style MCQs

Choose one best answer for each question. Try answering before checking
the answer key.

##### Middleware questions

**Q1.** Middleware A, B, and C are registered in that order. If all call
the next delegate, what is the order of their "after" logs?

A. A → B → C\
B. C → B → A\
C. The order is random\
D. Only C logs after execution

**Q2.** What does the `Run` delegate overload do?

A. Always calls the next middleware\
B. Adds an MVC filter\
C. Adds terminal middleware with no next delegate\
D. Maps a controller route

**Q3.** Which API branches the middleware pipeline based on a path
prefix?

A. `UseWhen`\
B. `Map`\
C. `UseRouting`\
D. `AddControllers`

**Q4.** Which API is designed to branch on an arbitrary predicate and
normally rejoin the main pipeline if the branch calls through?

A. `Run`\
B. `MapControllers`\
C. `UseWhen`\
D. `UseAuthorization`

**Q5.** You need to log requests for static files as well as APIs. What
is the best choice?

A. MVC action filter\
B. Middleware registered before static-file middleware\
C. MVC result filter\
D. Controller attribute

**Q6.** A conventional middleware needs a scoped EF Core `DbContext`.
Which is a suitable approach?

A. Always inject it into the middleware constructor\
B. Use it as a static field\
C. Resolve it as an `InvokeAsync` parameter or use `IMiddleware` with
DI\
D. Replace `Use` with `Run`

**Q7.** What happens if middleware does not call the next delegate?

A. ASP.NET Core automatically calls it\
B. The request short-circuits at that point\
C. MVC filters execute anyway\
D. The server retries the request

**Q8.** Which statement about `UseMiddleware<T>()` is correct?

A. It registers an MVC filter\
B. It adds class-based middleware to the application pipeline\
C. It only works with Minimal APIs\
D. It always creates a new middleware instance for every request

##### Filters and attributes questions

**Q9.** Which component naturally has access to MVC action arguments?

A. General middleware\
B. `IActionFilter`\
C. Static-file middleware\
D. `UseHttpsRedirection`

**Q10.** Which statement is correct?

A. Every attribute is an MVC filter\
B. Every MVC filter must be declared as an attribute\
C. Attributes can supply metadata without being filters\
D. Attributes run before all middleware

**Q11.** Which is usually the best choice for a controller authorization
policy?

A. A custom logging middleware\
B. `[Authorize(Policy = "...")]` with configured authorization services\
C. An `IResultFilter`\
D. `UseWhen` alone

**Q12.** Which filter stage wraps controller action invocation?

A. Resource filter\
B. Action filter\
C. Result filter\
D. Static-file middleware

**Q13.** Which is the best default for application-wide exception
handling across MVC and Minimal APIs?

A. Only `IExceptionFilter`\
B. Exception-handling middleware\
C. `IResultFilter`\
D. `[HttpGet]`

**Q14.** Which filter type is designed to run around MVC result
execution?

A. `IResultFilter`\
B. `IAuthorizationFilter`\
C. `IActionFilter`\
D. `IResourceFilter`

**Q15.** Which feature is designed to wrap Minimal API handler
execution?

A. `IActionFilter`\
B. MVC result filter\
C. Endpoint filter\
D. `ServiceFilter` only

**Q16.** Where do MVC filters execute?

A. Before the HTTP request enters middleware\
B. Inside MVC endpoint execution after routing selects the MVC endpoint\
C. For every static-file request\
D. Only after the response has been sent

**Q17.** Which is the most accurate statement about exception filters?

A. They catch every exception from every middleware\
B. They replace all middleware exception handling\
C. They handle eligible exceptions within the MVC filter/action
pipeline, not every application exception\
D. They execute before authorization filters

**Q18.** You need reusable logic before model binding and may want to
short-circuit MVC processing. Which filter is the best match?

A. Resource filter\
B. Result filter\
C. Action filter\
D. Endpoint filter

##### Scenario questions

**Q19.** An API needs a correlation ID on all requests and responses,
including non-MVC endpoints. Choose:

A. Action filter\
B. Middleware\
C. Result filter\
D. `[Produces]`

**Q20.** A custom rule needs to inspect a bound DTO and `ModelState`
before the controller action. For controller APIs, what should you
consider first?

A. Request middleware only\
B. Built-in model validation and `[ApiController]`; use an action filter
only for additional custom behavior\
C. Static-file middleware\
D. `Map("/api")`

**Q21.** A response header must be set before downstream processing
begins, because the response might start later in the pipeline. Choose:

A. Set it in middleware before `await next(context)`\
B. Set it after the response has started\
C. Use only an authorization filter\
D. Use an attribute that does not execute code

**Q22.** A filter with constructor-injected application services must be
applied using `ServiceFilter`. What else is required?

A. Register the filter and its dependencies in DI\
B. Register it using `UseMiddleware`\
C. Add `UseWhen`\
D. Make all services static

**Q23.** What does a middleware extension method primarily provide?

A. A new execution stage with different semantics\
B. A convenient, reusable registration API\
C. Automatic authorization\
D. Automatic model binding

**Q24.** Which is the most accurate high-level comparison?

A. Middleware and filters are interchangeable\
B. Middleware is for the HTTP pipeline; MVC filters are for MVC
action/result stages\
C. Filters always run before routing\
D. Middleware can directly receive bound controller arguments

------------------------------------------------------------------------

#### 16. Answer key

  --------------------------------------------------------------------------
  Question                Answer                  Explanation
  ----------------------- ----------------------- --------------------------
  1                       B                       "Before" logs follow
                                                  registration order;
                                                  "after" logs unwind in
                                                  reverse order.

  2                       C                       The delegate overload of
                                                  `Run` is terminal and has
                                                  no next delegate.

  3                       B                       `Map` branches on a path
                                                  prefix.

  4                       C                       `UseWhen` branches based
                                                  on a predicate and
                                                  normally rejoins when the
                                                  branch continues.

  5                       B                       Middleware placed before
                                                  static-file middleware can
                                                  observe requests that
                                                  static files
                                                  short-circuit.

  6                       C                       Scoped dependencies can be
                                                  `InvokeAsync` parameters,
                                                  or the middleware can use
                                                  `IMiddleware` and DI.

  7                       B                       Not calling next prevents
                                                  downstream middleware and
                                                  endpoint execution.

  8                       B                       It adds class-based
                                                  middleware to the
                                                  pipeline; lifetime
                                                  behavior depends on
                                                  middleware style.

  9                       B                       MVC action filters receive
                                                  an
                                                  `ActionExecutingContext`
                                                  with action arguments.

  10                      C                       Attributes may provide
                                                  routing, authorization, or
                                                  documentation metadata
                                                  without being filters.

  11                      B                       Use the built-in
                                                  authorization system and
                                                  policies.

  12                      B                       Action filters wrap action
                                                  invocation.

  13                      B                       Exception middleware
                                                  covers a broader
                                                  application pipeline than
                                                  MVC exception filters.

  14                      A                       Result filters wrap MVC
                                                  result execution.

  15                      C                       Endpoint filters wrap
                                                  Minimal API handlers.

  16                      B                       MVC filters run when an
                                                  MVC endpoint executes
                                                  after routing.

  17                      C                       Exception filters have a
                                                  limited MVC scope and do
                                                  not catch every pipeline
                                                  exception.

  18                      A                       Resource filters can run
                                                  before model binding and
                                                  can short-circuit MVC
                                                  processing.

  19                      B                       Middleware can apply
                                                  across endpoint types and
                                                  the full HTTP pipeline.

  20                      B                       Start with built-in model
                                                  validation; add a filter
                                                  only for extra reusable
                                                  behavior.

  21                      A                       Set headers before
                                                  downstream execution may
                                                  start the response.

  22                      A                       `ServiceFilter` resolves
                                                  the filter from DI.

  23                      B                       An extension method makes
                                                  registration cleaner; it
                                                  does not change execution
                                                  semantics.

  24                      B                       Middleware wraps the HTTP
                                                  pipeline; MVC filters
                                                  operate within MVC
                                                  execution stages.
  --------------------------------------------------------------------------

------------------------------------------------------------------------

#### Final revision checklist

You should be able to explain these without looking at the notes:

-   [ ] How the middleware delegate chain is built and invoked.
-   [ ] The difference between `Use`, terminal `Run`, path-based `Map`,
    and predicate-based `UseWhen`.
-   [ ] Why middleware order matters and how the response unwinds.
-   [ ] Conventional middleware vs `IMiddleware`, including scoped
    dependency injection.
-   [ ] Why an extension method improves registration but does not
    change behavior.
-   [ ] Why an attribute is not necessarily a filter.
-   [ ] The MVC filter stages and what context each stage can access.
-   [ ] Middleware vs MVC filters vs endpoint filters.
-   [ ] When to use built-in authorization, model validation, output
    caching, and exception handling rather than custom code.
-   [ ] How to choose a suitable extension point for a real-world
    requirement.

---

<a id="middleware-senior-extras"></a>

## Middleware and Filters — Extra Senior Notes

These notes add what the guide above does not already cover: simple analogies, the "pipeline behavior" question, a plain-language view of "after routing", and production pitfalls. `Use`, `Run`, `Map`, ordering, and filter stages are explained once above and not repeated here.

### Easy analogy to remember

- **Middleware** = security guard at the building entrance. Checks *everyone* before they enter.
- **Routing** = reception desk. Works out *which department* you are visiting, but doesn't take you there yet.
- **Filters / attributes** = rules inside the department. They only apply once you reach that department (controller/action).

You can't apply department rules at the front gate, because nobody yet knows which department you're going to.

### Middleware vs pipeline behavior

- **Middleware** is *one* processing unit: it can act before `next`, call `next`, act after `next`, or short-circuit.
- **Pipeline behavior** is the *overall flow* you get when many middleware are chained: request goes down A → B → C → endpoint, response unwinds C → B → A.

```text
Request -->
   M1 (before)
      M2 (before)
         M3 (before)
         M3 (after)
      M2 (after)
   M1 (after)
<-- Response
```

Ordering rules are really *pipeline behavior*, not features of any single middleware:

- Authentication before Authorization: authorization needs the user identity.
- Routing before Authorization: authorization needs endpoint metadata such as `[Authorize]`.
- Static files early: they short-circuit and skip controllers entirely.

> **Interview answer:** "Middleware is a building block that can act before or after the next delegate. Pipeline behavior is the runtime execution model created when they are chained — request entry, forwarding, short-circuiting, and response unwind."

### What "attributes run only after routing" means

Attributes belong to a controller or action, so they can't run until the framework knows *which* controller and action will handle the request.

```text
┌──────────────────────────────┐
│ 1. Middleware                │  ← attributes do NOT run here
└──────────────────────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Routing                   │  ← finds controller + action, reads route values (id = 5)
└──────────────────────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Controller selected       │  ← controller created, model binding prepared
└──────────────────────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Filters / attributes run  │  ← [Authorize], [ServiceFilter], custom filters
└──────────────────────────────┘
               ↓
┌──────────────────────────────┐
│ 5. Action method executes    │
└──────────────────────────────┘
```

For example, `GET /api/users/5` is matched by routing to `[HttpGet("api/users/{id}")] GetUser(int id)`. Routing only *decides*; it does not execute the action. Routing works in two phases: `UseRouting()` selects the endpoint, and the endpoint middleware (`MapControllers`) executes it later. That gap is why authorization can sit between them.

### What each side can access

| Feature | Middleware | Filter / attribute |
|---|---|---|
| `HttpContext` | ✔ Full access | ✔ Via `context.HttpContext` |
| Route values | Only after `UseRouting` | ✔ |
| Action arguments | ❌ | ✔ |
| `ModelState` | ❌ | ✔ |
| Controller instance | ❌ | ✔ |
| Short-circuit | Don't call `next` | Set `context.Result` |
| Runs for static files / non-MVC | ✔ | ❌ |

**Can middleware replace action filters?** Only for pipeline-wide logic. If the logic depends on action arguments, model state, or the action result, use a filter.

**Can attributes replace middleware?** No. They don't run before routing, can't see requests that never reach MVC, and can't short-circuit before MVC.

They work well together: middleware for logging and exception handling, an action filter for model validation and timing a specific action.

### Middleware vs filters vs `DelegatingHandler`

| | Middleware | MVC filters | `DelegatingHandler` |
|---|---|---|---|
| Scope | Whole app, incoming requests | MVC actions only | Outgoing `HttpClient` calls |
| Stage | Around routing and endpoints | Inside MVC | Before an external HTTP request is sent |
| Best for | Cross-cutting concerns | Action-specific logic | Retry, auth headers, and logging on outbound calls |

### Production-grade example: API key middleware

```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly string _apiKey;

    public ApiKeyMiddleware(RequestDelegate next, IConfiguration config)
    {
        _next = next;
        _apiKey = config["ApiKey"];
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue("X-API-Key", out var key))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("API Key Missing");
            return; // short-circuit: later middleware never runs
        }

        if (key != _apiKey)
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsync("Invalid API Key");
            return;
        }

        await _next(context);
    }
}

// Program.cs
app.UseMiddleware<ApiKeyMiddleware>();
```

### Rate limiting (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", config =>
    {
        config.PermitLimit = 5;
        config.Window = TimeSpan.FromSeconds(10);
    });
});

app.UseRateLimiter();
```

### High-performance tips

- Always `async`/`await`; never block with `.Result` or `.Wait()`.
- Don't allocate large objects per request; cache static objects such as `JsonSerializerOptions` outside `InvokeAsync`.
- Avoid reading the whole request body unless you must, because it's expensive.
- `HttpContext.Features` gives low-level access with low overhead.

### Quick Q&A

- **`UseAuthorization` before `UseAuthentication`?** Authorization sees an empty user, so protected endpoints fail.
- **Can middleware catch exceptions from later middleware?** Yes, which is why exception handling goes first.
- **Can middleware read the request body twice?** Not by default, because the body stream is forward-only. Call `context.Request.EnableBuffering()` first, then reset `Body.Position = 0`.
- **How does short-circuiting work internally?** The middleware simply doesn't invoke `next`.
- **Where does `IHttpContextAccessor` get `HttpContext` from?** An `AsyncLocal<T>`, so it flows with the async call chain.

### Pitfalls interviewers ask about

- ❌ Synchronous or blocking code inside middleware.
- ❌ Wrong registration order (e.g. `UseCors` or `UseRouting` placed after `MapControllers`).
- ❌ Forgetting `await next(context)`, which silently ends the pipeline.
- ❌ Writing to the response and then calling `next`, which causes "headers already sent" errors.
- ❌ Not limiting request body size, which is a DoS risk.

> **One-paragraph summary:** Middleware in ASP.NET Core is a chain-of-responsibility pipeline. Each component receives `HttpContext`, may do work before calling `next`, and may do work on the way back. Registration order decides the whole request flow. Use middleware for global, cross-cutting concerns; use filters for controller/action-specific logic.

---

<a id="aspnet-core-performance-checklist-and-mcqs-md"></a>

## ASP.NET Core High-Performance API Tuning Checklist

Senior .NET developer and solution-architect reference. Use this
checklist to guide profiling, code reviews, and production
readiness---not as a substitute for measurement.

#### 1. Establish a baseline first

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

#### 2. HTTP pipeline and middleware

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

#### 3. Minimal APIs versus controllers

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

#### 4. Request binding, validation, and response design

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

#### 5. Asynchronous work and thread-pool health

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

#### 6. EF Core and database access

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

#### 7. Caching

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

#### 8. Security and resilience

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

#### 9. Observability and production verification

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

### Interview MCQs: Minimal APIs versus Controllers

Each question has one best answer. The questions focus on distinctions
that commonly matter in senior .NET interviews.

#### Q1. Which statement best describes Minimal APIs and controller-based APIs?

A. Minimal APIs bypass the ASP.NET Core middleware pipeline.\
B. Controllers cannot use dependency injection.\
C. Both can use the ASP.NET Core pipeline; they differ in endpoint
programming model and available framework features.\
D. Minimal APIs cannot require authorization.

**Answer: C.** Both use middleware, routing, DI, and authorization. The
programming model and feature set differ.

#### Q2. What is a sound reason to choose Minimal APIs?

A. They automatically make every database query faster.\
B. They provide a concise endpoint model for focused HTTP services.\
C. They eliminate the need for input validation.\
D. They bypass authentication middleware.

**Answer: B.** Minimal APIs reduce ceremony, but they do not remove the
need for validation or security.

#### Q3. What is a sound reason to choose controllers?

A. Controllers are always faster.\
B. Controllers do not support middleware.\
C. MVC features such as action filters, conventions, and controller
organization are valuable to the application.\
D. Controllers automatically prevent SQL injection regardless of
data-access code.

**Answer: C.** Select controllers when their MVC capabilities and
organizational model are useful.

#### Q4. Can a single ASP.NET Core application use both Minimal APIs and controllers?

A. No.\
B. Yes, by mapping Minimal API endpoints and calling
`MapControllers()`.\
C. Only when authentication is disabled.\
D. Only in development.

**Answer: B.** Both endpoint styles can coexist in one application.

#### Q5. Which statement about authorization is correct?

A. Only controllers can use authorization.\
B. Minimal APIs cannot use authorization policies.\
C. Controllers and Minimal APIs can both be protected using ASP.NET Core
authorization.\
D. Authorization should be implemented only in JavaScript.

**Answer: C.** Controllers can use attributes such as `[Authorize]`;
Minimal APIs can use endpoint conventions such as
`.RequireAuthorization()`.

#### Q6. Which feature is specifically associated with the MVC controller filter pipeline?

A. Middleware\
B. Action filters\
C. Kestrel\
D. `HttpClient`

**Answer: B.** Action filters run around MVC action execution.
Middleware surrounds endpoint processing at the HTTP pipeline level.

#### Q7. Do Minimal APIs support endpoint filters?

A. No, they support only middleware.\
B. Yes, endpoint filters can run around Minimal API handler execution.\
C. Only if the endpoint is a controller action.\
D. Only in .NET Framework.

**Answer: B.** Endpoint filters are distinct from MVC action filters and
are supported by Minimal APIs.

#### Q8. Which statement about model validation is most accurate?

A. Minimal APIs never need validation.\
B. Controllers using `[ApiController]` provide automatic model-state
validation behavior for supported scenarios; Minimal APIs have their own
binding and validation options depending on framework version and
configuration.\
C. All validation must occur in middleware.\
D. Database constraints replace API validation.

**Answer: B.** Know the behavior of the target .NET version rather than
assuming the two models behave identically.

#### Q9. An API's p99 latency is high because a database query takes 800 ms. What should be investigated first?

A. Rewrite every controller as a Minimal API.\
B. Inspect the query, execution plan, indexes, data access pattern, and
database load.\
C. Remove authentication.\
D. Increase JSON indentation.

**Answer: B.** Optimize the measured bottleneck; changing endpoint style
is unlikely to fix slow database execution.

#### Q10. Which approach is best for comparing Minimal API and controller performance?

A. Assume Minimal APIs are always twice as fast.\
B. Compare unrelated applications from different machines.\
C. Benchmark equivalent endpoints under controlled, representative
conditions.\
D. Compare only source-code line counts.

**Answer: C.** Use equivalent workloads and measure latency, throughput,
CPU, allocations, and end-to-end dependencies.

#### Q11. What is the main risk of caching an authenticated API response under a key that ignores the user or tenant?

A. The response always becomes slower.\
B. One user's or tenant's data may be returned to another.\
C. The JWT signature changes.\
D. Routing stops working.

**Answer: B.** Cache-key design and authorization boundaries are
security-critical.

#### Q12. Which is the best general rule for asynchronous I/O in request handlers?

A. Use `.Result` to reduce allocations.\
B. Wrap every I/O call in `Task.Run`.\
C. Await asynchronous I/O and propagate cancellation where appropriate.\
D. Make all methods `async` even when no asynchronous work exists.

**Answer: C.** Avoid sync-over-async and unnecessary thread-pool work.

#### Q13. Which statement about middleware order is correct?

A. Middleware order never matters.\
B. Authentication should run after authorization.\
C. Middleware order affects behavior; authentication normally precedes
authorization, and routing must be configured appropriately for
endpoint-aware middleware.\
D. Middleware executes only for controller endpoints.

**Answer: C.** Follow the documented pipeline requirements for the
middleware being used.

#### Q14. What is the best use of `AsNoTracking()`?

A. Every query, including all updates.\
B. Read-only EF Core queries where change tracking is not required.\
C. To disable database indexes.\
D. To make network calls asynchronous.

**Answer: B.** It can reduce tracking overhead for read-only entity
queries; measure the impact.

#### Q15. Which performance practice is most reliable?

A. Remove all filters regardless of purpose.\
B. Disable JWT validation.\
C. Profile representative workloads, identify the bottleneck, make a
targeted change, and measure again.\
D. Replace every controller with a Minimal API.

**Answer: C.** Performance engineering should be evidence-driven and
must preserve correctness and security.

------------------------------------------------------------------------

#### Quick review: interview takeaways

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

---

<a id="dependency-injection"></a>

## Dependency Injection in .NET

A practical guide to the built-in .NET DI container — lifetimes, scopes, runtime parameters, keyed services, and the patterns interviewers ask about — with examples for ASP.NET Core, background workers, and desktop apps.

#### What dependency injection solves

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


**DI vs IoC vs Service Locator**

-   **Inversion of Control (IoC):** a broad principle in which control
    over object creation or flow is moved away from the component.
-   **Dependency Injection:** one way to implement IoC.
-   **Service Locator:** a component asks a container/provider for
    dependencies itself. This hides dependencies and is usually less
    desirable than constructor injection.

DI is a technique, not a guarantee of good design. A class with too many constructor parameters may still have too many responsibilities.

#### How the built-in container works

##### Registration

When you write:

```csharp
services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

the service collection records a descriptor containing information such as:

- The service type (`IOrderRepository`)
- The implementation type (`SqlOrderRepository`)
- The lifetime (`Scoped`)

Registrations are configuration; this line normally does **not** immediately create `SqlOrderRepository`.

##### Building the provider

In a typical application:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<OrderService>();

var app = builder.Build();
```

`Build()` creates the service provider used to resolve registered services. The built-in provider prepares resolution machinery and caches information needed to construct service call sites. It does not eagerly instantiate every registered service.

Avoid building a second provider manually with `services.BuildServiceProvider()` during registration. Doing so can create duplicate singleton instances and separate disposal tracking. Register services first, then let the application host build the provider.

##### Resolving a service

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

##### Constructor selection

Prefer one public constructor per DI-managed class. If a class has multiple public constructors, the container attempts to choose a suitable constructor based on the services it can resolve; ambiguous choices can cause an exception. A single, explicit constructor is easier to understand and maintain.

##### What happens when a service is disposable?

If the container creates a disposable service, it tracks it for disposal at the end of the applicable lifetime:

- Transient disposable: typically disposed when the scope that resolved it is disposed.
- Scoped disposable: disposed when its scope is disposed.
- Singleton disposable: disposed when the root provider is disposed.

Do not manually dispose a service that was injected and is owned by the container. If you create an object yourself, you generally own its disposal unless ownership is explicitly transferred.

#### Service lifetimes

.NET's built-in container supports three standard lifetimes.

##### Transient

```csharp
services.AddTransient<IFormatter, JsonFormatter>();
```

A new instance is created each time the service is resolved. If a transient is a dependency of a singleton, however, it is created when the singleton is constructed and then held by that singleton. So “transient” does not mean that every method call creates a new instance.

**Good fits:** small, stateless helpers and lightweight services.

##### Scoped

```csharp
services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

One instance is reused within a particular DI scope. A new scope gets a different instance.

In a typical ASP.NET Core request, the framework creates a request scope, so a scoped service is usually shared within that request and disposed at its end. A scope is not inherently an HTTP request: in desktop apps, console apps, and background workers, the application defines scopes explicitly.

**Good fits:** EF Core `DbContext`, unit-of-work services, and services that share state during one operation.

##### Singleton

```csharp
services.AddSingleton<IClock, SystemClock>();
```

One instance is used for the lifetime of the root service provider. It may be resolved from many requests or scopes.

**Good fits:** thread-safe shared services, immutable configuration-derived helpers, and caches designed for concurrent access.

Singleton services must be designed for concurrency. Avoid storing request- or operation-specific state in a singleton, and do not directly inject scoped services into one.

##### Comparison

| Lifetime | Instance reuse | Typical disposal boundary | Common use |
|---|---|---|---|
| Transient | New instance per resolution, subject to dependency graph behavior | Scope/provider that tracks it | Lightweight stateless helper |
| Scoped | One per scope | End of scope | `DbContext`, unit of work |
| Singleton | One per root provider | Provider/host shutdown | Thread-safe shared service |

##### Example: observing instance reuse

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

#### What is a logical scope?

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

#### Lifetime mismatches and captive dependencies

A *captive dependency* occurs when a longer-lived service holds a reference to a shorter-lived service beyond the shorter-lived service's intended lifetime.

##### The classic error: singleton consumes scoped service

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

##### Can a scoped service depend on a singleton?

Yes, provided the singleton is safe to share and does not retain operation-specific state.

```csharp
services.AddSingleton<IClock, SystemClock>();
services.AddScoped<OrderService>();
```

The scoped `OrderService` can use the singleton clock.

##### Can a singleton depend on a transient?

Technically, yes. But the transient is resolved when the singleton is created and is then retained as a field for the singleton's lifetime. It will not be recreated for every operation. If that transient is stateful or not thread-safe, this may cause bugs.

##### Can a transient depend on a scoped service?

This can work when the transient is resolved inside a scope. But do not resolve a scoped dependency from the root provider or let a longer-lived object retain the transient that captured it. Keep the whole dependency graph within the intended scope.

##### Lifetime compatibility guide

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

##### Scope validation

For a generic host, enable validation in development where applicable. `Host.CreateApplicationBuilder` and ASP.NET Core's development hosting configuration typically enable useful validation defaults. You can also configure provider validation explicitly in custom setups. Validation can detect many captive dependencies and missing or invalid registrations early, but it cannot prove the whole application's concurrency or lifecycle design is correct.

#### Circular dependencies

A circular dependency exists when the dependency graph loops back to a service already being constructed.

```csharp
public sealed class ServiceA(ServiceB serviceB);

public sealed class ServiceB(ServiceA serviceA);
```

The container cannot construct either service first, so resolution fails with a circular-dependency exception.

##### Preferred solution: change the design

Often the two classes have responsibilities that should be separated. Extract the shared behavior into a third service:

```text
ServiceA ──► SharedService
ServiceB ──► SharedService
```

Or let one higher-level coordinator call both services instead of having them call each other.

##### Event/callback alternative

If one component only needs to react to something another component does, an event, callback, mediator, or domain event may remove the direct cycle. Choose the simplest mechanism that fits the architecture.

##### Avoid using `IServiceProvider` to hide the cycle

Resolving one service manually from `IServiceProvider` may defer the failure or obscure dependencies rather than fix the design. The built-in container also does not automatically support `Lazy<T>` injection as a general circular-dependency solution. A factory can be appropriate for a genuine runtime creation requirement, but should not merely conceal a dependency cycle.

#### Parameterized injection and factories

Constructor injection is ideal when dependencies are known when the object is created. Sometimes an object also needs a value that is known only at runtime, such as a file name, customer ID, or export format.

##### Values from configuration: Options pattern

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

Settings class and registration, with validation so bad configuration fails at startup instead of at first use:

```csharp
public sealed class EmailOptions
{
    public const string SectionName = "Email";
    public string Sender { get; set; } = "";
    public int RetryCount { get; set; }
}

// Program.cs

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

Options are for configuration values, not arbitrary per-call values such as a user-selected file path.

##### Factory registration with a fixed value

```csharp
services.AddTransient<IFileExporter>(sp =>
    new FileExporter("csv"));
```

This supplies a fixed `"csv"` value every time the factory creates the service. It is not a runtime parameter that changes per request.

##### Runtime parameters: define a factory abstraction

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

##### `ActivatorUtilities`

`ActivatorUtilities.CreateInstance<T>(provider, explicitArgument)` can combine explicit construction arguments with dependencies resolved from DI:

```csharp
var exporter = ActivatorUtilities.CreateInstance<CsvReportExporter>(
    serviceProvider,
    outputPath);
```

For example, the constructor could accept `string outputPath` plus `ILogger<CsvReportExporter>`. The explicit argument supplies the runtime path and DI supplies the logger. Ensure argument types and constructor selection are unambiguous.

> The built-in container does not automatically provide arbitrary `Func<T>` factories for every service. Register a delegate yourself or use a dedicated factory type if that is the intended design.

##### Method injection

Useful when a dependency is needed only for one ASP.NET Core endpoint:

``` csharp
app.MapGet("/health/details", (IDiagnosticService diagnostics) =>
    diagnostics.GetDetails());
```

MVC actions can also use `[FromServices]` for endpoint-specific
dependencies.

#### Keyed services (.NET 8+)

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

#### What changed through .NET 10?

##### .NET 8

- **Keyed DI services**: register and resolve implementations by key.
- The built-in DI container continues to support constructor injection, service factories, scopes, and the options pattern.

##### .NET 9

- No new general-purpose built-in DI source generator replaced the standard container. Avoid treating community proposals or third-party container features as built-in .NET features.

##### .NET 10

- The current DI documentation includes `KeyedService.AnyKey` behavior for keyed registrations. In particular, calling singular `GetKeyedService()` with `KeyedService.AnyKey` throws `InvalidOperationException`; use `AnyKey` with plural keyed-service enumeration where appropriate.
- .NET 10 also contains broader runtime, library, and ASP.NET Core improvements, but do not assume these represent a new DI lifetime model.
- The core lifetimes remain **Singleton, Scoped, and Transient**. There is no general built-in feature that makes injecting a scoped dependency directly into a singleton safe.

For version-specific details, consult the official documentation linked below.

#### Scoped services in background work

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

#### Dependency injection in desktop apps

Unlike ASP.NET Core, WPF and Windows Forms do not automatically create a DI scope for each request. The app should decide whether a scope represents a button operation, a dialog, a window, or another unit of work.

##### Configure the host

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

##### Scope per operation

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

##### Scope per window or dialog

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

##### Do not resolve scoped services from the root provider

This is a common desktop-app mistake:

```csharp
// Avoid for scoped services:
var service = appHost.Services.GetRequiredService<IOrderService>();
```

If `IOrderService` is scoped, resolving it from the root provider can either fail under validation or effectively retain it for too long when validation is disabled. Create a scope and resolve it from `scope.ServiceProvider`.

#### `HttpClient` and typed clients

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

#### DI in Clean Architecture and CQRS

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

#### Validation, disposal, and common mistakes

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
- **Be intentional about duplicate registrations.** For a single service type the last registration wins; `IEnumerable<T>` returns all of them in registration order.
- **Don't register the same implementation twice with different lifetimes** if you expect one shared instance — separate registrations create separate instances.
- **Don't add an interface for every class.** Add abstractions where they create a useful boundary or enable substitution.
- **Don't run parallel operations on one `DbContext`.** Use separate scopes/contexts for concurrent work.
- **Too many constructor parameters** usually means the class has too many responsibilities.

#### Testing DI and application behavior

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

#### Quick reference

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

#### Extra interview questions

##### What happens if a service is registered more than once?

For a single service resolution, the last registration normally wins.
Resolving `IEnumerable<T>` returns all registrations. Registration order
can therefore matter.

##### Is the built-in container always sufficient?

It supports common registrations, lifetimes, factories, open generics,
and enumerable registrations. A third-party container may be justified
for specific advanced features, but adds integration and maintenance
costs.

##### How would you diagnose a DI error in production?

Read the complete exception and service type chain; check registrations,
constructor parameters, lifetime mismatches, environment-specific
configuration, and scope boundaries. Add a small integration test that
builds the host/provider with validation enabled.

#### Quick revision checklist

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

#### Interview answer framework

For architecture questions, structure your answer in this order:

1.  **Definition** --- explain the concept precisely.
2.  **Mechanics** --- describe what .NET does.
3.  **Trade-offs** --- explain benefits and costs.
4.  **Failure mode** --- give a real pitfall.
5.  **Production example** --- connect it to an API, EF Core, a worker,
    or a CQRS handler.

#### Final takeaway

Think of the DI container as an object-construction and lifetime-management system. **Registration** tells it what to create, **resolution** builds the dependency graph, **lifetimes** determine reuse, and **scopes** define where scoped objects are shared and disposed. In desktop applications especially, deliberately choose the boundary for each unit of work and keep scoped services—particularly EF Core contexts—inside that boundary.

#### Official references

- [Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Dependency injection guidelines](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines)
- [Dependency injection in ASP.NET Core (.NET 10)](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-10.0)
- [What's new in .NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview)

---

<a id="iis-kestrel-senior-dotnet-viva-md"></a>

## IIS, Application Pools, Kestrel, and High-Traffic ASP.NET Core

#### Senior .NET Developer Viva Preparation

> Goal: explain the hosting architecture accurately, reason about concurrency and bottlenecks, and describe how you would scale a real production API.

#### 1. The big picture

For a typical ASP.NET Core application hosted on Windows with IIS:

```text
Browser / API client
        |
        | HTTPS
        v
HTTP.sys (Windows kernel-mode HTTP listener)
        |
        v
IIS site + URL/host bindings
        |
        v
Application Pool
        |
        v
w3wp.exe (IIS worker process)
        |
        v
ASP.NET Core Module (ANCM)
        |
        | Usually reverse-proxy forwarding in out-of-process hosting
        v
Kestrel inside the ASP.NET Core application process
        |
        v
ASP.NET Core middleware -> routing -> authentication/authorization
        |
        v
Endpoint / controller / application services
        |
        v
SQL Server / Redis / external services
```

The exact path depends on the ASP.NET Core hosting model. In **out-of-process hosting**, IIS/ANCM forwards requests to the application running on Kestrel. In **in-process hosting**, the ASP.NET Core app runs inside `w3wp.exe`; requests are handled through the IIS server integration rather than being proxied to a separate Kestrel process. Be ready to explain this distinction.

#### 2. What an IIS Application Pool actually is

An **application pool is an IIS configuration and isolation boundary**, not a request-processing thread and not a load balancer. It groups one or more IIS applications under a worker-process configuration.

Its main responsibilities are:

- **Process isolation:** applications assigned to different pools can run in separate worker processes.
- **Identity and permissions:** configure the Windows identity used by the worker process, commonly `ApplicationPoolIdentity`.
- **Lifecycle management:** start, stop, and recycle worker processes.
- **Reliability controls:** configure rapid-fail protection and other process-model settings.
- **Resource and queue configuration:** configure such settings as queue length and recycling behavior.

##### Application pool vs worker process

- **Application pool:** configuration boundary in IIS.
- **Worker process (`w3wp.exe`):** operating-system process that executes IIS-hosted workloads.
- **Worker-process count:** an application pool normally has one worker process, but can be configured as a *web garden* with multiple worker processes.
- **Application:** a website or application mapped to an application pool.

Multiple applications can share a pool, but then they also share its process boundary and lifecycle. For stronger isolation, separate pools are often preferable.

##### A useful analogy

Think of the application pool as the **rules and supervision for a workspace**. The worker process is the actual running program inside that workspace. The pool configures how the process starts, under which identity it runs, and when it is recycled.

#### 3. How IIS receives and routes a request

1. The client opens an HTTP/HTTPS connection.
2. On Windows, **HTTP.sys** receives HTTP traffic in kernel mode and can queue requests for the relevant URL group.
3. IIS site bindings and request configuration determine which site/application should handle the URL.
4. The request is associated with the relevant application pool.
5. If necessary, WAS (Windows Process Activation Service) activates the pool and worker process.
6. IIS modules and the applicable hosting integration process or forward the request.
7. The application generates a response; it returns through the hosting path to IIS/HTTP.sys and then to the client.

Do not say that IIS creates a new thread for every request. That is not how modern asynchronous ASP.NET Core request handling should be understood.

#### 4. How multiple requests are handled

A single worker process can handle many concurrent requests. Concurrency is not the same as executing every request simultaneously on a separate CPU core.

ASP.NET Core uses asynchronous I/O and the .NET ThreadPool to use resources efficiently:

- When code awaits non-blocking I/O, such as an asynchronous database or HTTP call, the request can wait without holding a dedicated thread for the whole wait.
- When the operation completes, continuation work can run on a ThreadPool thread.
- CPU-bound code still consumes CPU time while it executes.
- Blocking calls (`.Result`, `.Wait()`, synchronous I/O, long locks) can occupy threads and reduce throughput.
- Every request still consumes resources: memory, sockets, database connections, CPU, and downstream capacity.

##### Example: 1,000 concurrent requests

Suppose 1,000 requests arrive and most spend time awaiting SQL Server:

- The application does not need 1,000 dedicated threads sitting idle while SQL responds.
- Asynchronous database calls allow threads to return to the pool while I/O is pending.
- When SQL responses arrive, continuations run and finish the requests.
- If SQL Server can only sustain 200 operations at once, adding more web threads will not make the database faster. It may make the queue and timeouts worse.

**Interview line:** “I optimize for bounded resource usage and end-to-end throughput, not simply for the largest possible thread or connection count.”

#### 5. Queueing, backpressure, and overload

Requests may wait at multiple layers: network/load balancer, HTTP.sys/IIS, the application, a rate limiter, a connection pool, or a downstream service.

If incoming traffic exceeds the system's sustainable processing rate, queues grow. Longer queues increase latency and can lead to timeouts, retries, and even more load.

A robust design uses **backpressure** rather than accepting unlimited work:

- Set appropriate request-body and request-time limits.
- Use rate limiting and concurrency limits where appropriate.
- Bound queues; reject or defer work when capacity is exhausted.
- Return meaningful `429 Too Many Requests` or `503 Service Unavailable` responses when appropriate.
- Use timeouts and cancellation tokens for downstream calls.
- Avoid retry storms; use bounded retries with exponential backoff and jitter.
- Move long-running work to a durable queue and process it in background workers.

The IIS application-pool queue length is not a magic throughput setting. Increasing it may allow more requests to wait, but does not increase the app's actual processing capacity. Tune it based on measurements and the desired overload behavior.

#### 6. IIS with Kestrel: the important hosting models

##### Out-of-process hosting

- IIS receives the external request.
- ANCM forwards it to the ASP.NET Core application running in a separate process.
- Kestrel listens on the configured local endpoint.
- The response travels back through ANCM/IIS.

ANCM participates in process startup and management. The exact forwarding transport and configuration depend on the installed hosting bundle and hosting setup; do not overstate that it always uses a particular port or transport.

##### In-process hosting

- The ASP.NET Core application runs inside `w3wp.exe`.
- IIS server integration handles requests within that process.
- There is no separate Kestrel process receiving proxied requests in this mode.

##### Why use IIS?

IIS provides Windows hosting integration, site bindings, TLS configuration, request filtering, logging, process activation, and other hosting controls. Kestrel is the cross-platform web server used by ASP.NET Core and can also be hosted directly behind another reverse proxy or exposed directly when the deployment architecture and security controls support it.

**Avoid saying:** “Kestrel must always sit behind IIS/Nginx” or “Kestrel has no security.” Kestrel can be used directly; whether a reverse proxy is appropriate depends on the environment and required edge features.

#### 7. What happens when an application pool recycles?

Recycling replaces a worker process according to configured rules, such as scheduled, memory-based, or manual recycling.

- IIS can use **overlapped recycling**: a replacement worker process starts before the old one exits, where applicable.
- Overlapped recycling reduces interruption, but does not guarantee every in-flight request or every kind of application state survives.
- In-memory state is process-local. It is lost when that process exits.
- In-process ASP.NET session state is not shared across workers or servers.
- A warm-up/preload strategy can reduce cold-start latency, but must be configured and tested.

For reliability:
- Keep application state external where possible (database or distributed cache).
- Make startup safe and fast.
- Use readiness/health checks at the load-balancer or orchestrator layer.
- Ensure shutdown and cancellation are handled gracefully.
- Do not rely on recycling as the primary fix for a memory leak; diagnose the leak.

**Rapid-Fail Protection:** IIS can disable a pool when its worker process fails repeatedly within a configured interval. This protects the server from an endless crash/restart loop; it is not a general-purpose health check for all application failures.

#### 8. Web garden vs web farm

| Concept | Meaning | Main concern |
|---|---|---|
| Web garden | Multiple worker processes for one application pool on one server | More process-local state and complexity; not automatically faster |
| Web farm | Multiple application instances across servers/VMs/containers | Load balancing, health, shared state, deployment, and resilience |

A web garden is not the default solution for high traffic. It can help specific workloads after testing, but may hurt cache locality, complicate session state, and increase memory usage. For horizontal scale and fault isolation, a web farm or multiple app instances behind a load balancer is more common.

#### 9. How would you handle a million requests?

First clarify what “a million requests” means:

- One million requests per day is about **11.6 requests/second on average**.
- One million requests per hour is about **278 requests/second on average**.
- One million requests per minute is about **16,667 requests/second on average**.

These are averages. Real systems have peaks, uneven traffic, and expensive versus cheap endpoints. Capacity planning must use peak rate, latency targets, payload sizes, and workload mix.

##### A senior-level approach

1. **Define SLOs:** target throughput, p95/p99 latency, availability, and error rate.
2. **Load-test realistically:** use production-like data and measure at expected peaks and beyond.
3. **Find the bottleneck:** CPU, allocation/GC, locks, thread-pool starvation, SQL, Redis, network, or external APIs.
4. **Optimize the endpoint:** avoid unnecessary work, reduce payloads, paginate, use async I/O, and remove blocking calls.
5. **Cache appropriate data:** in-memory cache for local ephemeral data; distributed cache such as Redis when instances need shared cache state. Define expiration and invalidation deliberately.
6. **Protect the database:** indexes, efficient query plans, bounded connection usage, appropriate transactions, and fewer round trips. Do not blindly raise connection-pool limits.
7. **Scale horizontally:** multiple healthy app instances behind a load balancer; autoscale based on useful signals and account for startup time.
8. **Apply backpressure:** rate limits, bounded concurrency, timeouts, queueing, and graceful overload responses.
9. **Move slow work off the request path:** use a durable queue and background workers for reports, email, file processing, and other long-running tasks.
10. **Observe continuously:** correlate traces, logs, and metrics; track request rate, failure rate, latency, CPU, memory, GC, queue length, SQL waits, connection-pool saturation, and dependency latency.
11. **Test failure modes:** instance loss, slow SQL, dependency timeouts, traffic spikes, and deployments.

A single IIS application pool cannot guarantee a million requests. Capacity comes from the entire system and must be demonstrated through measurement.

#### 10. Database connection pooling: a frequent trap

ADO.NET providers such as `Microsoft.Data.SqlClient` pool physical connections. Opening and disposing a logical connection per operation is normal when using `using`/`await using`; it usually returns the connection to the pool rather than closing the physical connection each time.

- Keep database operations short and asynchronous.
- Dispose connections reliably.
- Avoid holding a connection while doing unrelated work.
- Tune pool limits only after checking SQL Server capacity and pool exhaustion.
- Remember that adding application instances multiplies potential database connection demand.

If 20 app instances each can open 100 SQL connections, the database may face up to roughly 2,000 connections across those pools. Scaling the API without planning database capacity can move the bottleneck downstream.

#### 11. Performance and configuration: measure before tuning

Potentially useful controls include:
- IIS application-pool queue and recycling settings.
- Always-running/preload options where supported and appropriate.
- ASP.NET Core request limits, rate limiting, and timeouts.
- Response compression and caching for suitable responses.
- .NET GC and runtime configuration appropriate to the workload.
- Connection-pool and database settings.

Do not blindly copy settings such as “maximum requests per CPU,” increase queue lengths, or enable a web garden because a blog recommends them. Defaults and available settings differ by IIS/ASP.NET version and hosting model. Establish a baseline, change one thing at a time, and validate under load.

#### 12. Useful diagnostic signals

| Symptom | Possible causes to investigate |
|---|---|
| High CPU | CPU-heavy code, serialization, excessive logging, inefficient queries, lock contention |
| High latency but low CPU | Slow database/API, network waits, queueing, connection-pool exhaustion |
| Thread-pool starvation | Blocking waits, sync-over-async, long-running work on ThreadPool threads |
| Rising memory / frequent GC | Allocation pressure, retained objects, cache growth, leak |
| IIS queue growing | App cannot drain requests quickly enough, worker unavailable, downstream bottleneck |
| SQL timeouts | Slow queries, blocking/deadlocks, overloaded DB, pool exhaustion |
| Failures after recycle | Cold-start issues, process-local state, startup dependencies, shutdown handling |

Use IIS logs and Windows performance counters/Event Viewer alongside ASP.NET Core metrics, distributed tracing, and Application Insights or another APM tool.

#### 13. Senior interview questions and answer outlines

##### Q1. What is the responsibility of an IIS application pool?
It defines the process-isolation and lifecycle configuration for IIS-hosted applications: worker-process identity, startup/shutdown/recycling behavior, failure protection, and relevant queue/process settings. It does not itself execute each request; the worker process and hosting pipeline do that.

##### Q2. Does IIS create one thread per request?
No. Modern ASP.NET Core uses asynchronous I/O and the .NET ThreadPool. Requests may await I/O without holding a thread for the entire wait. CPU-bound work still requires CPU execution time.

##### Q3. If we increase the IIS queue length, will throughput increase?
Not necessarily. A larger queue permits more waiting and may increase latency and memory pressure. It does not increase the app or database's sustainable processing rate.

##### Q4. Should I enable a web garden to handle more requests?
Not automatically. Benchmark first. Multiple worker processes can help some workloads but add memory and process-local state complexity. Horizontal scaling behind a load balancer is often the more flexible option.

##### Q5. Why might adding API instances make the database slower?
Each instance can create additional concurrent database work and its own connection pools. The aggregate load may exceed SQL Server capacity, increasing waits, blocking, and timeouts.

##### Q6. How would you diagnose slow API responses?
Start with p95/p99 latency and distributed traces, separate application time from dependency time, inspect CPU/memory/GC and queueing, then investigate SQL query plans, waits, and connection-pool saturation. Change settings only after identifying the bottleneck.

##### Q7. What is the difference between in-process and out-of-process IIS hosting?
In-process runs the ASP.NET Core app inside `w3wp.exe` using IIS integration. Out-of-process runs the app in a separate process with Kestrel and ANCM forwarding requests.

##### Q8. How do you avoid losing user state during recycling or scaling?
Keep durable/shared state outside a single worker process, use distributed storage/cache where appropriate, make instances replaceable, and avoid relying on in-memory session state for correctness.

#### 14. A concise answer to memorize

“An IIS application pool is an isolation and lifecycle boundary, while `w3wp.exe` is the worker process. HTTP.sys receives HTTP traffic, IIS maps it to a site and application pool, and the configured hosting model invokes the ASP.NET Core app. With out-of-process hosting, ANCM forwards requests to the Kestrel process; with in-process hosting, the app runs inside `w3wp.exe`. ASP.NET Core handles concurrency efficiently through asynchronous I/O and the ThreadPool, but throughput is limited by the slowest constrained resource, often the database or a downstream API. To scale, I measure realistic peak traffic, identify bottlenecks, use caching and efficient queries, bound concurrency and queues, apply timeouts and rate limits, and scale healthy instances behind a load balancer. I validate every change with load tests and production telemetry rather than blindly increasing IIS limits.”

#### 15. Recommended practice exercise

Build a small ASP.NET Core API with:
- One lightweight endpoint and one SQL-backed endpoint.
- Asynchronous database calls.
- Structured logging and tracing.
- A bounded rate limit.
- A load test with gradually increasing concurrency.

Record throughput, p95/p99 latency, error rate, CPU, memory, ThreadPool behavior, and SQL latency. Then introduce an intentionally slow dependency and observe how queues and timeouts behave. This gives you concrete examples to discuss in the viva.

---

#### Accuracy reminders

- An application pool is not a load balancer.
- A request queue is not extra processing capacity.
- Async improves resource utilization during I/O waits; it does not make CPU work free.
- A million-request target is meaningless without a time window, peak profile, and latency/error objectives.
- Kestrel can run directly; a reverse proxy is an architectural choice, not an absolute requirement.
- Always distinguish IIS in-process and out-of-process ASP.NET Core hosting.

---

<a id="signalr-dotnet8-viva-guide-md"></a>

## SignalR with .NET 8 REST API — Viva Preparation Guide

A concise, step-by-step guide to building a .NET 8 REST API that sends real-time notifications to everyone, an individual user, or a group based on roles stored in a database. Includes common viva questions.

#### 1. What is SignalR?

**ASP.NET Core SignalR** enables real-time, two-way communication between a server and connected clients. Instead of repeatedly polling a REST endpoint, the server can push updates to clients as events happen.

Typical use cases:
- Notifications and alerts
- Chat and messaging
- Live dashboards and progress updates
- Updates targeted at users, teams, or roles

SignalR negotiates a suitable transport, generally preferring WebSockets when available and falling back to supported alternatives when needed.

#### 2. REST API vs SignalR

| REST API | SignalR |
|---|---|
| Usually request/response | Persistent real-time connection |
| Client asks for data | Server can push events to clients |
| Good for CRUD and business operations | Good for live updates and notifications |

They complement each other. A REST API can perform a business operation, save it to the database, and use SignalR to notify connected clients.

#### 3. Create the project

```bash
dotnet new webapi -n SignalRNotifications
cd SignalRNotifications
```

In ASP.NET Core 8, the server-side SignalR APIs are included in the shared ASP.NET Core framework. A separate `Microsoft.AspNetCore.SignalR` package is generally not needed for a normal Web API project.

#### 4. Create a hub

Create `Hubs/NotificationHub.cs`:

```csharp
using Microsoft.AspNetCore.SignalR;

namespace SignalRNotifications.Hubs;

public sealed class NotificationHub : Hub
{
    // A client can join a group after the server authorizes the request.
    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }

    public async Task LeaveGroup(string groupName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
    }
}
```

A **Hub** is SignalR's high-level API for handling client-server communication. Do not let clients join arbitrary sensitive groups without server-side authorization. In a production application, derive group membership from authenticated identity and trusted database data rather than accepting a client-supplied role or group name.

#### 5. Register SignalR and map the hub

Example `Program.cs`:

```csharp
using SignalRNotifications.Hubs;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddSignalR();

var app = builder.Build();

app.MapControllers();
app.MapHub<NotificationHub>("/hubs/notifications");

app.Run();
```

If a browser-based frontend is hosted on another origin, configure a specific CORS policy for that frontend. For credentialed requests, do not combine `AllowAnyOrigin()` with `AllowCredentials()`.

#### 6. Send notifications from a REST controller

Create `Models/NotificationRequest.cs`:

```csharp
namespace SignalRNotifications.Models;

public sealed record NotificationRequest(string Message);
```

Create `Controllers/NotificationsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.SignalR;
using SignalRNotifications.Hubs;
using SignalRNotifications.Models;

namespace SignalRNotifications.Controllers;

[ApiController]
[Route("api/notifications")]
public sealed class NotificationsController : ControllerBase
{
    private readonly IHubContext<NotificationHub> _hub;

    public NotificationsController(IHubContext<NotificationHub> hub)
    {
        _hub = hub;
    }

    [HttpPost("all")]
    public async Task<IActionResult> NotifyAll(
        [FromBody] NotificationRequest request)
    {
        await _hub.Clients.All.SendAsync(
            "ReceiveNotification", request.Message);

        return Accepted();
    }

    [HttpPost("user/{userId}")]
    public async Task<IActionResult> NotifyUser(
        string userId,
        [FromBody] NotificationRequest request)
    {
        await _hub.Clients.User(userId).SendAsync(
            "ReceiveNotification", request.Message);

        return Accepted();
    }

    [HttpPost("group/{groupName}")]
    public async Task<IActionResult> NotifyGroup(
        string groupName,
        [FromBody] NotificationRequest request)
    {
        await _hub.Clients.Group(groupName).SendAsync(
            "ReceiveNotification", request.Message);

        return Accepted();
    }
}
```

`IHubContext<T>` lets services and controllers send messages without being inside a Hub instance.

##### Main targeting APIs

```csharp
await _hub.Clients.All.SendAsync("ReceiveNotification", message);
await _hub.Clients.User(userId).SendAsync("ReceiveNotification", message);
await _hub.Clients.Group(groupName).SendAsync("ReceiveNotification", message);
```

- `Clients.All`: every connected client.
- `Clients.User(userId)`: all active connections associated with that SignalR user identifier.
- `Clients.Group(groupName)`: connections currently in that group.

These calls do not guarantee delivery to disconnected users. Persist important notifications in a database if they must be available later.

#### 7. Important: how `Clients.User(userId)` identifies a user

By default, SignalR uses the authenticated user's `ClaimTypes.NameIdentifier` claim as the user identifier. Ensure it matches the ID used by your application.

For JWT authentication, configure authentication and authorization in the normal ASP.NET Core pipeline and provide the access token when establishing the SignalR connection. Browser WebSocket/SSE connections may send the token in the `access_token` query parameter; restrict token extraction to the hub path, use HTTPS, and avoid logging tokens.

Do **not** maintain your own static `Dictionary<userId, connectionId>` for normal targeting. One user may have multiple browser tabs/devices, connections can reconnect, and static state does not work reliably across multiple servers. Prefer `Clients.User(userId)` with authenticated identity.

#### 8. Add users to groups based on roles in the database

A safe approach is to authenticate the connection, resolve the user's ID from trusted claims, load current roles from your database, and add the connection to server-selected groups.

Illustrative hub pattern (adapt the database query to your application's entities and EF Core context):

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

namespace SignalRNotifications.Hubs;

[Authorize]
public sealed class NotificationHub : Hub
{
    private readonly ApplicationDbContext _db;

    public NotificationHub(ApplicationDbContext db)
    {
        _db = db;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (string.IsNullOrWhiteSpace(userId))
        {
            Context.Abort();
            return;
        }

        // Example only: adapt Users / UserRoles / Roles to your schema.
        var roles = await _db.UserRoles
            .Where(x => x.UserId == userId)
            .Select(x => x.Role.Name)
            .ToListAsync();

        foreach (var role in roles)
        {
            await Groups.AddToGroupAsync(
                Context.ConnectionId, $"role:{role}");
        }

        await base.OnConnectedAsync();
    }
}
```

This sample assumes EF Core navigation properties such as `UserRoles`, `UserId`, and `Role.Name`; change them to match your schema and add the relevant EF Core and entity namespaces. If the role data is already present in trusted, validated claims, those claims may be used instead, but database changes and claim refresh policies must be considered.

Then send to a role group:

```csharp
await _hub.Clients.Group($"role:{roleName}")
    .SendAsync("ReceiveNotification", message);
```

**Security checklist:**
- Require authentication for protected hubs and notification endpoints.
- Authorize who may trigger notifications; never allow any caller to broadcast to everyone or arbitrary roles.
- Never trust a role name sent by the browser as proof of authorization.
- Decide how role changes take effect for existing connections. Group membership is not automatically refreshed just because a database role changes.
- Use tenant-qualified group names in multi-tenant applications, for example `tenant:{tenantId}:role:{roleName}`, and authorize tenant access on the server.
- Group names are routing labels, not security boundaries by themselves.

#### 9. JavaScript client example

For a quick browser test, load the SignalR JavaScript client from a trusted source compatible with your app, then:

```javascript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("https://localhost:7000/hubs/notifications")
    .withAutomaticReconnect()
    .build();

connection.on("ReceiveNotification", message => {
    console.log("Notification:", message);
});

async function start() {
    try {
        await connection.start();
        console.log("Connected to SignalR");
    } catch (error) {
        console.error(error);
        setTimeout(start, 5000);
    }
}

start();
```

Replace the URL with the actual HTTPS URL and port shown by your application. Automatic reconnect does not retry the initial `start()` failure by itself, which is why the example retries startup.

A REST request to test the broadcast endpoint:

```http
POST /api/notifications/all
Content-Type: application/json

{
  "message": "System maintenance starts at 8 PM."
}
```

Use a valid JWT or other configured authentication if the endpoint requires it.

#### 10. Connection and delivery concepts

- `Context.ConnectionId`: identifies one connection, not necessarily one person.
- One user can have several simultaneous connections.
- SignalR group membership is connection-based and must be re-established when a connection reconnects.
- `OnConnectedAsync()` and `OnDisconnectedAsync()` are lifecycle hooks; avoid relying on in-memory connection tracking in a multi-server deployment.
- For scale-out, consider Azure SignalR Service or a supported Redis backplane, depending on the hosting environment.
- SignalR is not a durable message queue. Store notifications that must survive disconnects and let clients retrieve unread notifications from a REST endpoint.

#### 11. Viva questions and short answers

##### Q1. What is SignalR?
SignalR is an ASP.NET Core library for real-time, two-way communication. It lets the server push events to connected clients.

##### Q2. How is SignalR different from REST?
REST is commonly request/response; SignalR maintains a real-time connection so the server can push updates without waiting for another client request.

##### Q3. What is a Hub?
A Hub is a high-level abstraction for receiving client calls and sending messages to clients, users, or groups.

##### Q4. What is the difference between `Clients.All`, `Clients.User`, and `Clients.Group`?
`All` targets all connected clients, `User` targets all connections for one user identifier, and `Group` targets connections that belong to a group.

##### Q5. How does SignalR know the user ID?
By default, it uses the authenticated principal's `NameIdentifier` claim. Authentication and claim mapping must be configured consistently.

##### Q6. Can groups be based on database roles?
Yes. After authenticating the connection, the server can load roles from the database and add that connection to server-controlled role groups.

##### Q7. Is a SignalR group a security feature?
No. A group is a message-routing mechanism. The server must authorize group membership and notification-sending operations.

##### Q8. Why use `IHubContext` in a controller?
It allows code outside the Hub—such as a REST controller or background service—to send messages to connected clients.

##### Q9. What happens if the user is offline?
A live SignalR message is not delivered to a disconnected client. Store important notifications persistently and provide an API to retrieve them later.

##### Q10. What happens when a user has multiple tabs open?
Each tab usually has its own connection. `Clients.User(userId)` targets all active connections associated with that user identifier.

##### Q11. How do you scale SignalR?
Use a scale-out option such as Azure SignalR Service or a Redis backplane where appropriate, and avoid relying on static in-memory connection maps.

##### Q12. How would you implement a production notification flow?
Authenticate and authorize the request, perform and persist the business operation, commit the transaction, then publish a SignalR event. For reliable delivery, use persisted notifications and consider an outbox/background-worker pattern.

#### 12. Suggested viva demo sequence

1. Run the .NET 8 API and connect two browser clients.
2. Call `POST /api/notifications/all` and show both clients receive the event.
3. Authenticate clients with different user IDs and test `Clients.User(...)`.
4. Add users to a server-authorized group and test `Clients.Group(...)`.
5. Explain why database-backed roles, authorization, and persistent storage matter in production.

#### Quick revision

Remember these five points:

1. **Hub** = real-time communication endpoint.
2. **`IHubContext`** = send SignalR messages from controllers/services.
3. **`Clients.All/User/Group`** = choose recipients.
4. **Authentication + authorization** = trustworthy user identity and permissions.
5. **Database persistence** = important notifications remain available when clients are offline.
