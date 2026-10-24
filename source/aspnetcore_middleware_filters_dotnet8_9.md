# ASP.NET Core Middleware, Filters, and Attributes in .NET 8/9

> A practical guide to how the request pipeline works, how middleware
> and MVC filters execute internally, when to use each, and how to test
> your understanding with HackerRank-style MCQs.

## Contents

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

## 1. Learning goals and terminology

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

## 2. The ASP.NET Core request pipeline

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

### Why middleware has a "before" and "after" phase

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

## 3. Middleware: what it is and how it works

A middleware component is code in the HTTP request pipeline. It
commonly:

1.  Reads or modifies `HttpContext.Request`.
2.  Calls the next middleware.
3.  Reads or modifies the response after downstream execution.
4.  Or returns a response immediately without calling the next
    middleware.

### How the pipeline is built

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

### Minimal example

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

## 4. Use, Run, Map, and UseWhen

### `Use`

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

### `Run`

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

### `Map`

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

### `UseWhen`

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

### Quick comparison

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

## 5. Writing and registering middleware

There are two common class-based approaches.

### Conventional middleware

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

### Extension method for clean registration

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

### Inline middleware

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

## 6. Middleware DI, lifetimes, and factories

### Conventional middleware and scoped dependencies

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

### Factory-based middleware with `IMiddleware`

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

### Which approach should you use?

-   **Conventional middleware:** good default for simple middleware;
    constructor dependencies should be safe for the pipeline lifetime.
-   **Conventional middleware with `InvokeAsync` parameters:** good when
    only a request-scoped dependency is needed for a single invocation.
-   **`IMiddleware`:** good when the middleware itself needs DI-managed,
    per-request construction or scoped constructor dependencies.

## 7. Middleware ordering and common built-ins

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

## 8. MVC filters and attributes

### What is an attribute?

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

### What is an MVC filter?

MVC filters run inside the MVC action-invocation pipeline after routing
has selected an MVC endpoint. They can access MVC-specific information
such as action arguments, model state, controller/action descriptors,
and action results.

A typical MVC filter can be: - Registered globally. - Applied to a
controller. - Applied to an action. - Created through an attribute,
`ServiceFilter`, `TypeFilter`, or a global DI registration.

Filters do not run for every HTTP request. For example, a request served
directly by static-file middleware does not execute MVC action filters.

## 9. How filters execute internally

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

### Before/after nesting

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

## 10. Filter types and use cases

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

### Authorization filters

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

### Resource filters

Resource filters can short-circuit before model binding and surround
much of the remaining MVC pipeline. They can be useful for specialized
MVC-level caching or work that should happen before model binding. They
are not a replacement for middleware that must apply to all HTTP
endpoints.

### Action filters

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

### Exception filters

Exception filters can translate eligible MVC exceptions into an MVC
result. For application-wide exception handling, including middleware
failures and non-MVC endpoints, prefer exception-handling middleware or
the built-in exception-handler infrastructure.

### Result filters

Result filters wrap MVC result execution. Use them when you need to act
around MVC result generation. They do not universally intercept every
response produced by the application: middleware, Minimal APIs, static
files, and some short-circuit paths can bypass MVC result filters.

## 11. Creating and registering custom filters

### Attribute-based filter for simple, self-contained behavior

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

### `ServiceFilter`

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

### `TypeFilter`

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

### Global filter registration

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

## 12. Middleware vs filters vs endpoint filters vs attributes

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

### Endpoint filters for Minimal APIs

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

## 13. Decision guide and real-world scenarios

### Scenario A: Log every request, including static files

**Use middleware**, registered before static-file middleware if
static-file requests must be logged. An MVC action filter will miss
static files and non-MVC endpoints.

### Scenario B: Audit only selected controller actions

**Use an action filter** if the audit needs MVC action metadata or
action arguments. If auditing is a security/compliance requirement,
define exactly what data is recorded, avoid logging secrets, and make
the audit write resilient.

### Scenario C: Add correlation ID to all requests

**Use middleware.** Create or validate a correlation ID, store it in
`HttpContext`, and optionally return it in a response header.

### Scenario D: Enforce a role/policy on a controller

**Use `[Authorize]` with a policy or role**, backed by
`AddAuthentication` and `AddAuthorization`. Do not roll your own access
control in a generic action filter.

### Scenario E: Validate a request DTO

Use normal model validation and API conventions first. For controller
APIs, `[ApiController]` can automatically return a 400 response for
invalid model state. Use an action filter only when you need reusable
custom behavior beyond built-in validation. Middleware is usually too
early because it does not naturally have the bound DTO or `ModelState`.

### Scenario F: Handle unhandled exceptions application-wide

**Use exception-handling middleware** (such as `UseExceptionHandler`) so
it can cover MVC, Minimal APIs, and downstream middleware. Use exception
filters only for MVC-specific exception-to-result behavior.

### Scenario G: Apply behavior to Minimal API handler arguments

**Use an endpoint filter** or handler-level validation pattern. MVC
action filters are for MVC controllers.

### Scenario H: Add a response header to all responses

**Use middleware**, with attention to when the response starts. Set the
header before calling `next` when possible; once the response has
started, headers generally cannot be changed.

### Scenario I: Cache a complete response

Choose the built-in output caching or response caching features when
they meet the requirements. Use resource filters only for specialized
MVC-level caching, not as the default general-purpose HTTP caching
mechanism.

## 14. Common mistakes and interview notes

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

### A useful mental model

-   **Middleware:** "Should this HTTP request continue through the
    application pipeline, and what should happen around that journey?"
-   **MVC filter:** "What should happen around this
    controller/action/result?"
-   **Endpoint filter:** "What should happen around this Minimal API
    handler?"
-   **Attribute:** "What metadata or declarative behavior should be
    attached to this code element?"

------------------------------------------------------------------------

## 15. HackerRank-style MCQs

Choose one best answer for each question. Try answering before checking
the answer key.

### Middleware questions

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

### Filters and attributes questions

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

### Scenario questions

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

## 16. Answer key

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

## Final revision checklist

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
