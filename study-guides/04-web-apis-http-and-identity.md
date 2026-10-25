# Web APIs, HTTP, and Identity

Each concept is explained once. Where several notes covered the same topic, they have been merged into a single section, and related topics in other guides are linked instead of repeated.

## Contents

- [C# and ASP.NET Core REST API Notes](#csharp-aspnetcore-rest-api-notes-md)
- [REST API and ASP.NET Core 8 — Viva Preparation Notes](#dotnet8-rest-api-viva-notes-md)
- [HTTP Headers in .NET REST APIs — Viva Preparation](#http-headers-dotnet-viva-md)
- [Web Storage, ASP.NET Core Session, and JWT Logout](#senior-dotnet-storage-session-jwt-viva-md)
- [HTTP and API Interview Question Bank](#http-and-api-interview-question-bank)
- [Additional Questpond question sets](#additional-questpond-question-sets)

---

<a id="csharp-aspnetcore-rest-api-notes-md"></a>

## C# and ASP.NET Core REST API Notes

A concise reference for C# range/index syntax, local functions, response formats, and controller return types.

#### 1. C# Range and Index Syntax

These features were introduced in C# 8.0.

Given:

```csharp
int[] nums = { 10, 20, 30, 40, 50 };
```

| Expression | Meaning | Result |
|---|---|---|
| `nums[..1]` | From the start up to, but excluding, index `1` | `{ 10 }` |
| `nums[1..]` | From index `1` to the end | `{ 20, 30, 40, 50 }` |
| `nums[..^1]` | From the start up to, but excluding, the last element | `{ 10, 20, 30, 40 }` |
| `nums[^1]` | The last element | `50` |

- `..` defines a range (a slice).
- `^1` means one position from the end.
- The range's end is exclusive.
- For arrays, slicing with a range returns a new array.

Equivalent examples:

```csharp
nums[..1]  // nums[0..1]
nums[1..]  // nums[1..nums.Length]
nums[..^1] // nums[0..(nums.Length - 1)]
nums[^1]   // nums[nums.Length - 1]
```

#### 2. Function Inside a Function in C#

C# supports **local functions**: named helper methods declared inside another method. They are available only within the enclosing method.

```csharp
static void Main()
{
    int Add(int a, int b)
    {
        return a + b;
    }

    Console.WriteLine(Add(5, 10)); // 15
}
```

Use a **local function** for a named helper with multiple steps or reusable logic inside one method. Use a **lambda** for a short function assigned to a delegate or passed to another method:

```csharp
Func<int, int, int> add = (a, b) => a + b;
Console.WriteLine(add(5, 10)); // 15
```

#### 3. `IActionResult`, `ActionResult`, and `ActionResult<T>`

These are ASP.NET Core MVC action return types.

##### `IActionResult`

An interface for an HTTP action result. Use it when the action may return different kinds of results and you do not need a declared response model type.

```csharp
[HttpGet("{id}")]
public IActionResult GetProduct(int id)
{
    if (id <= 0)
        return BadRequest("Invalid product ID");

    var product = new Product(id, "Laptop", 1500m);
    return Ok(product);
}
```

##### `ActionResult`

The non-generic base class for MVC action results. Like `IActionResult`, it is useful when returning HTTP results without declaring a specific response model.

```csharp
[HttpDelete("{id}")]
public ActionResult DeleteProduct(int id)
{
    // Delete product...
    return NoContent();
}
```

##### `ActionResult<T>`

Use this when the normal success response is a specific model, while the action may also return HTTP error results.

```csharp
[HttpGet("{id}")]
[ProducesResponseType(typeof(Product), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public ActionResult<Product> GetProduct(int id)
{
    var product = FindProduct(id);

    if (product is null)
        return NotFound();

    return product; // Implicitly produces a 200 OK result with the product
}
```

Example model and placeholder lookup:

```csharp
public record Product(int Id, string Name, decimal Price);

private static Product? FindProduct(int id) =>
    id == 1 ? new Product(1, "Laptop", 1500m) : null;
```

##### Quick comparison

| Return type | Best fit |
|---|---|
| `IActionResult` | Flexible HTTP results; no single declared success model |
| `ActionResult` | Non-generic MVC result |
| `ActionResult<T>` | A typed success model plus results such as `NotFound()` or `BadRequest()` |

**Practical default:** Prefer `ActionResult<T>` for typical CRUD endpoints that return a model on success. Use `IActionResult` when the action's result is not naturally represented by one model type. Both can return standard HTTP results.

#### 4. Returning Different Formats from a REST API

If clients need JSON, XML, or CSV, choose the mechanism based on the format.

##### JSON and XML with content negotiation

ASP.NET Core Web APIs return JSON by default. To enable XML output for controller APIs:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddControllers()
    .AddXmlSerializerFormatters();

var app = builder.Build();

app.MapControllers();
app.Run();
```

A controller can return a model and let the client's `Accept` header select a supported representation:

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    [Produces("application/json", "application/xml")]
    public ActionResult<Product> Get(int id)
    {
        var product = id == 1
            ? new Product(1, "Laptop", 1500m)
            : null;

        if (product is null)
            return NotFound();

        return product;
    }
}
```

Example request headers:

```http
Accept: application/json
```

or

```http
Accept: application/xml
```

The requested format must be supported by a registered output formatter. If no acceptable formatter is available, configure and test the API's behavior for unsupported `Accept` values.

##### CSV or another custom format

CSV is not built in as a standard object output formatter. For a simple download, generate CSV and return it as a file:

```csharp
[HttpGet("{id}/export")]
public IActionResult ExportCsv(int id)
{
    var product = new Product(id, "Laptop", 1500m);

    // Escape fields properly in production; this simple example has no commas or quotes.
    var csv = $"Id,Name,Price\r\n{product.Id},{product.Name},{product.Price}";

    return File(
        System.Text.Encoding.UTF8.GetBytes(csv),
        "text/csv",
        "product.csv");
}
```

For a larger API where many endpoints must support CSV through `Accept: text/csv`, consider implementing and registering a reusable custom output formatter rather than duplicating CSV generation in controllers.

##### Choosing the format-selection approach

| Approach | Use when |
|---|---|
| `Accept` header / content negotiation | Clients request a representation such as JSON or XML |
| Query parameter such as `?format=csv` | You intentionally expose a simple export option in the URL |
| Custom output formatter | Many endpoints need a consistent format such as CSV |

Keep the underlying response model and business logic independent of the representation where possible. Document supported media types and status codes in OpenAPI/Swagger, and test each format.

---

<a id="dotnet8-rest-api-viva-notes-md"></a>

## REST API and ASP.NET Core 8 — Viva Preparation Notes

#### 1. What is a REST API?

REST (Representational State Transfer) is an architectural style for designing networked APIs around **resources**. ASP.NET Core Web API commonly exposes resources over HTTP and exchanges JSON.

- `GET /api/employees` — retrieve employees
- `GET /api/employees/1` — retrieve one employee
- `POST /api/employees` — create an employee
- `PUT /api/employees/1` — replace/update an employee
- `PATCH /api/employees/1` — partially update an employee
- `DELETE /api/employees/1` — delete an employee

REST is not a protocol. HTTP is the protocol commonly used to implement REST-style APIs.

#### 2. How an HTTP request travels through .NET 8

```text
Client (browser/Postman)
        |
        v
Kestrel web server
        |
        v
ASP.NET Core middleware pipeline
        |
        +--> Exception handling / logging / HTTPS / CORS (as configured)
        |
        v
Routing selects an endpoint
        |
        v
Authentication identifies the caller
        |
        v
Authorization checks permissions
        |
        v
Endpoint execution (Minimal API handler or Controller action)
        |
        v
Application/service layer
        |
        v
Repository / EF Core / database (if needed)
        |
        v
Result becomes an HTTP response
        |
        v
Middleware unwinds and response returns to client
```

The exact pipeline depends on `Program.cs`. Not every application uses every middleware component.

##### What happens at each stage?

1. **Client sends an HTTP request.** It includes a method, URL, headers and, when needed, a body.
2. **Kestrel receives the request.** Kestrel is ASP.NET Core's cross-platform web server. Behind a reverse proxy, the proxy may receive the network connection first.
3. **Middleware runs in registration order.** Each component can inspect or change the request, call the next component, and inspect or change the response on the way back. Middleware can short-circuit the pipeline.
4. **Routing matches an endpoint.** For example, `GET /api/employees/1` can match a route containing `{id}`.
5. **Authentication runs.** When configured, it validates credentials such as a bearer token and establishes `HttpContext.User`.
6. **Authorization runs.** It checks endpoint policies, roles or claims. Authentication asks “Who are you?”; authorization asks “Are you allowed?”
7. **The endpoint executes.** A Minimal API handler or controller action receives bound inputs and calls application logic.
8. **Input binding and validation happen.** Route/query values are converted to parameters; JSON request bodies are deserialized. Controller APIs using `[ApiController]` automatically return 400 for model-validation errors. Minimal API validation depends on the validation setup and framework/package version.
9. **The response is created.** Common results include `200 OK`, `201 Created`, `204 No Content`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden` and `404 Not Found`.
10. **The response travels back through middleware** that can add headers, compress content or log the status, then Kestrel sends it to the client.

#### 3. Middleware: the key concept

Each middleware can run code before and after `await next(context)`, or short-circuit by not calling `next`. `Use`/`Run`/`Map`, ordering, and middleware vs filters are explained once in [.NET and ASP.NET Core → Middleware](03-dotnet-and-aspnetcore.md#aspnetcore-middleware-filters-dotnet8-9-md).

Typical order for an API:

```csharp
app.UseExceptionHandler();
app.UseHttpsRedirection();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers(); // Or map Minimal API endpoints
```

#### 4. Minimal API vs. Controller-based API

Both use the same ASP.NET Core HTTP server and middleware pipeline. The difference is mainly how endpoints are declared and which framework features are available.

##### Minimal API

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/employees/{id:int}", (int id) =>
    Results.Ok(new { Id = id, Name = "Ayesha" }));

app.Run();
```

##### Controller-based API

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
app.Run();
```

`Controllers/EmployeesController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    [HttpGet("{id:int}")]
    public IActionResult GetById(int id)
    {
        return Ok(new { Id = id, Name = "Ayesha" });
    }
}
```

| Area | Minimal API | Controller-based API |
|---|---|---|
| Endpoint definition | `MapGet`, `MapPost`, etc. | Controller actions and attributes |
| Structure | Often fewer files and less ceremony | Familiar separation into controllers/actions |
| Filters and MVC features | Supports endpoint filters; some MVC features differ | Rich MVC filters and controller conventions |
| Model validation | Configure validation explicitly as needed | `[ApiController]` provides automatic 400 responses for invalid models |
| Good fit | Focused endpoints, small services, microservices | APIs needing MVC conventions, filters and established controller structure |

Neither approach is automatically best for every application. Maintainability depends on design, organization, testing and team conventions—not only endpoint syntax.

#### 5. Dependency injection and business logic

Avoid putting complex business logic directly in endpoint lambdas or controller actions. Register application services in DI and call them from endpoints.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IEmployeeService, EmployeeService>();

var app = builder.Build();

app.MapGet("/api/employees/{id:int}",
    async (int id, IEmployeeService service) =>
    {
        var employee = await service.GetByIdAsync(id);
        return employee is null
            ? Results.NotFound()
            : Results.Ok(employee);
    });

app.Run();
```

`IEmployeeService` and `EmployeeService` are application-defined types. Implement them and inject repositories or an EF Core `DbContext` as appropriate. Use asynchronous database operations for I/O-bound work.

#### 6. Authorization in Minimal APIs

Add authentication and authorization services before building the app, then add middleware and protect endpoints.

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // Configure a real trusted issuer and audience, or your
        // identity provider's supported settings.
        options.Authority = builder.Configuration["Authentication:Authority"];
        options.Audience = builder.Configuration["Authentication:Audience"];
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ItDepartment", policy =>
        policy.RequireClaim("department", "IT"));
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/public", () => Results.Ok("No authentication required"));
app.MapGet("/private", () => Results.Ok("Authenticated"))
   .RequireAuthorization();
app.MapGet("/it-only", () => Results.Ok("IT department"))
   .RequireAuthorization("ItDepartment");

app.Run();
```

Configure valid issuer/audience values and signing-key validation for the actual identity provider. Do not copy placeholder values into production. `.RequireAuthorization()` is commonly used to protect Minimal API endpoints; `[Authorize]` metadata can also be used when configured correctly.

- **401 Unauthorized:** the request is not authenticated or its credentials are invalid.
- **403 Forbidden:** the caller is authenticated but does not have permission.

#### 7. Swagger / OpenAPI in .NET 8 Minimal APIs

OpenAPI is a machine-readable API description; Swagger UI is a web interface that displays that description and lets developers try endpoints. In .NET 8, a common setup uses Swashbuckle.

Install:

```bash
dotnet add package Swashbuckle.AspNetCore
```

Configure before `builder.Build()`:

```csharp
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Employee API",
        Version = "v1"
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapGet("/api/employees/{id:int}", (int id) =>
    Results.Ok(new EmployeeExample(id, "Ayesha", "Developer")))
   .WithSummary("Get an employee by ID")
   .WithDescription("Returns an example employee.");

app.Run();

public record EmployeeExample(int Id, string Name, string Position);
```

With the default Swashbuckle configuration, browse to `/swagger` in Development. The exact URL and port come from the app's launch settings.

##### Add example request/response data

There is an important distinction:

- A C# model/schema tells OpenAPI the shape of the data.
- `WithSummary` and `WithDescription` add endpoint documentation.
- To show a **specific example payload** in Swagger UI, use OpenAPI example support appropriate to your library/version. Swashbuckle does not provide a universal `WithExample(...)` method for every version.

A common Swashbuckle approach is to use an operation filter or schema filter, or an example package such as `Swashbuckle.AspNetCore.Filters`. For example, install:

```bash
dotnet add package Swashbuckle.AspNetCore.Filters
```

Register example support (exact setup can vary by package version):

```csharp
builder.Services.AddSwaggerGen(options =>
{
    options.ExampleFilters();
});
builder.Services.AddSwaggerExamplesFromAssemblyOf<CreateEmployeeExample>();
```

Define a request model and example provider:

```csharp
using Swashbuckle.AspNetCore.Filters;

public record CreateEmployeeRequest(string Name, string Position);

public sealed class CreateEmployeeExample
    : IExamplesProvider<CreateEmployeeRequest>
{
    public CreateEmployeeRequest GetExamples() =>
        new("Ayesha", "Software Engineer");
}
```

Attach the example to the endpoint using the package's supported request-example metadata/filter mechanism. If the installed version does not apply examples automatically to Minimal APIs, configure an operation filter or use a supported OpenAPI transformer. Verify the rendered `/swagger` page; returning sample data from an endpoint does not guarantee Swagger displays it as a request-body example.

#### 8. REST API vs. gRPC — quick comparison

| Feature | REST-style HTTP API | gRPC |
|---|---|---|
| Interface | Usually resource URLs and HTTP verbs | Service methods defined in `.proto` files |
| Common payload | JSON | Protocol Buffers (binary by default) |
| Transport | Commonly HTTP/1.1 or HTTP/2 | HTTP/2 |
| Browser integration | Straightforward with `fetch` and standard HTTP tooling | Browser use typically needs gRPC-Web and supporting infrastructure |
| Streaming | Possible, but requires suitable HTTP/API design | Unary, server streaming, client streaming and bidirectional streaming |
| Developer experience | Easy to inspect manually; broad tooling support | Generated clients/servers provide strongly typed contracts |
| Common fit | Public APIs, web/mobile clients, broad interoperability | Efficient service-to-service communication, typed contracts, streaming |

gRPC is not always faster in every workload; performance depends on payloads, network, serialization, implementation and call patterns. Choose based on client compatibility, contract needs, streaming, operational requirements and measured performance.

#### 9. HTTP methods and status codes to remember

##### Methods

- `GET`: read a resource; should not change server state.
- `POST`: create a resource or trigger an operation.
- `PUT`: replace a resource representation; intended to be idempotent.
- `PATCH`: partially modify a resource.
- `DELETE`: remove a resource; intended to be idempotent in its effect.
- `HEAD`: like GET but without a response body.
- `OPTIONS`: discover communication options; commonly used in CORS preflight.

**Idempotent** means repeating the same request has the same intended effect as making it once. PUT or DELETE may return different status codes on later calls and still be idempotent.

##### Status codes

| Code | Meaning |
|---|---|
| `200 OK` | Request succeeded |
| `201 Created` | Resource created; often includes a `Location` header |
| `204 No Content` | Success with no response body |
| `400 Bad Request` | Invalid request or input |
| `401 Unauthorized` | Missing/invalid authentication |
| `403 Forbidden` | Authenticated but not allowed |
| `404 Not Found` | Resource or endpoint not found |
| `409 Conflict` | Conflict with current resource state |
| `500 Internal Server Error` | Unexpected server failure |

#### 10. Common viva questions and short answers

**Q: Is REST a protocol?**  
No. REST is an architectural style; HTTP is the protocol commonly used to implement REST APIs.

**Q: What is statelessness?**  
Each request carries the information needed to process it. The server does not rely on stored conversational client state from a previous request. Statelessness does not mean the application cannot use a database or store domain data.

**Q: What is the difference between authentication and authorization?**  
Authentication establishes identity; authorization determines what that identity is permitted to do.

**Q: What is model binding?**  
The framework maps request data—such as route values, query parameters, headers or JSON bodies—to handler parameters or .NET objects.

**Q: What does Swagger do?**  
OpenAPI describes the API contract; Swagger UI renders an interactive page for exploring and testing endpoints.

**Q: Minimal API or controllers—which should I choose?**  
Choose based on conventions, organization, validation and filter needs, team familiarity, and testing—not a blanket assumption that one is always faster or better.

**Q: What is the difference between 401 and 403?**  
401 means the request has not established valid authentication; 403 means the authenticated caller is not allowed to perform the action.

**Q: What is the difference between REST and gRPC?**  
REST-style APIs commonly use resource URLs and JSON; gRPC uses service contracts, generated clients and Protocol Buffers, with strong support for streaming.

#### Final mental model

**Kestrel receives → middleware processes → routing selects → authentication identifies → authorization permits/denies → endpoint binds inputs and runs → services/data layer work → HTTP result is serialized → response returns through middleware to the client.**

---

<a id="http-headers-dotnet-viva-md"></a>

## HTTP Headers in .NET REST APIs — Viva Preparation

A concise reference to common HTTP request/response headers, caching, ETags, and security headers, with ASP.NET Core examples.

#### 1. What is an HTTP header?

An HTTP header is metadata sent with an HTTP request or response. Headers influence content negotiation, authentication, caching, browser security, and how clients handle responses.

```http
GET /api/products/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer <access-token>
```

Example response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{"id":42,"name":"Keyboard"}
```

#### 2. Important request headers

| Header | Purpose | Example / Viva note |
|---|---|---|
| `Accept` | Media types the client can receive | `Accept: application/json` |
| `Content-Type` | Format of the request body | `Content-Type: application/json` |
| `Authorization` | Credentials or access token | `Authorization: Bearer <token>` |
| `User-Agent` | Identifies client software | Browser, `curl`, or API client |
| `Accept-Language` | Preferred response language | `Accept-Language: en-US` |
| `Accept-Encoding` | Compression formats accepted | `Accept-Encoding: gzip, br` |
| `Cache-Control` | Client caching instructions | `Cache-Control: no-cache` |
| `If-None-Match` | Conditional GET using an ETag | `304` if unchanged |
| `If-Match` | Conditional update using an ETag | Prevents lost updates; mismatch normally returns `412` |
| `Origin` | Origin initiating a browser cross-origin request | Used in CORS processing |
| `Cookie` | Sends cookies stored by the client | `Cookie: sessionId=...` |
| `Referer` | Referring page, when permitted by policy | May be omitted or reduced for privacy |

**Remember:** `Accept` describes the response format the client can handle; `Content-Type` describes the format of the body being sent.

##### Read a request header in ASP.NET Core

```csharp
[HttpGet("client-info")]
public IActionResult GetClientInfo()
{
    var userAgent = Request.Headers.UserAgent.ToString();
    var correlationId = Request.Headers["X-Correlation-ID"].ToString();

    return Ok(new { userAgent, correlationId });
}
```

Do not return sensitive values such as `Authorization` in API responses or logs.

#### 3. Important response headers

| Header | Purpose | Example / Viva note |
|---|---|---|
| `Content-Type` | Describes response body format | `application/json; charset=utf-8` |
| `Content-Length` | Body size in bytes, when applicable | Usually managed by the server |
| `Cache-Control` | Defines caching rules | `private, max-age=60` |
| `ETag` | Validator for a representation | `ETag: "v17"` |
| `Last-Modified` | Time representation last changed | Used for date-based conditional requests |
| `Location` | Target URI after redirect or resource creation | Common with `201 Created` |
| `Set-Cookie` | Asks the client to store a cookie | Use secure attributes for sensitive cookies |
| `WWW-Authenticate` | Describes an authentication challenge | Common with `401 Unauthorized` |
| `Access-Control-Allow-Origin` | Allows a browser origin under CORS rules | Prefer explicit origins where appropriate |
| `Access-Control-Allow-Methods` | Methods permitted by CORS | `GET, POST, PUT` |
| `Access-Control-Allow-Headers` | Request headers permitted by CORS | `Authorization, Content-Type` |
| `Retry-After` | Suggests when to retry | Useful with `429` or some `503` responses |
| `Content-Disposition` | Suggests inline display or download | `attachment; filename="report.pdf"` |
| `Vary` | Identifies request headers affecting a cached response | e.g. `Vary: Accept-Encoding` |

ASP.NET Core generally sets standard headers such as `Content-Type` automatically. Add or configure only what an endpoint needs.

##### Add a custom response header

```csharp
[HttpGet]
public IActionResult Get()
{
    Response.Headers["X-Correlation-ID"] = HttpContext.TraceIdentifier;
    return Ok(new { message = "Success" });
}
```

For headers applying across many endpoints, middleware is usually more maintainable.

#### 4. ETag: caching and concurrency

An **ETag** (entity tag) is an opaque validator identifying a particular version of a representation. It can save bandwidth and help prevent one client from overwriting another client's changes.

##### Conditional GET

1. The server returns a body and `ETag: "product-42-v3"`.
2. The client stores the ETag.
3. The client later sends `If-None-Match: "product-42-v3"`.
4. If unchanged, the server can return `304 Not Modified` with no response body. Otherwise, it returns the updated representation and a new ETag.

##### Conditional update

A client can send `If-Match: "product-42-v3"` with an update. The server updates only if the current version matches; otherwise it returns `412 Precondition Failed`. The check and database update must be atomic to avoid race conditions.

##### ASP.NET Core example: conditional GET

This simplified example uses a stable version value. In a real API, obtain the version from the resource/database and ensure it changes whenever the representation changes.

```csharp
[HttpGet("{id:int}")]
public IActionResult GetProduct(int id)
{
    // Example only: retrieve the product and its current version.
    var product = new { Id = id, Name = "Keyboard" };
    var etag = "\"product-42-v3\"";

    if (Request.Headers.IfNoneMatch.Any(value =>
        value == etag || value == "*"))
    {
        return StatusCode(StatusCodes.Status304NotModified);
    }

    Response.Headers.ETag = etag;
    return Ok(product);
}
```

For production use, parse conditional headers according to HTTP semantics (including lists and weak ETags), generate validators from actual resource versions/content, and account for representation changes. Do not use a base64-encoded copy of JSON as though it were automatically a secure or correct ETag.

**Strong vs weak ETag**
- Strong: `"v3"` — indicates byte-for-byte equivalent representation.
- Weak: `W/"v3"` — indicates semantic equivalence; it is not suitable for `If-Match` strong comparison.

#### 5. Cache-Control

| Directive | Meaning |
|---|---|
| `no-store` | Do not store the response in a cache |
| `no-cache` | A stored response must be revalidated before reuse; it does **not** mean “never store” |
| `private` | Shared caches must not store the response; a private client cache may |
| `public` | Shared caches may store the response, subject to other rules |
| `max-age=60` | Fresh for 60 seconds |

For sensitive/user-specific data:

```csharp
Response.Headers.CacheControl = "no-store";
```

For a response safe for short-lived private caching:

```csharp
Response.Headers.CacheControl = "private, max-age=60";
```

Do not mark personalized or confidential API responses `public` without assessing cache risks.

#### 6. Security headers

Security headers are primarily enforced by browsers. They are useful for browser-facing APIs and especially for web applications serving HTML.

| Header | Purpose | Important note |
|---|---|---|
| `Strict-Transport-Security` (HSTS) | Tells browsers to use HTTPS for future requests | Send only over HTTPS; understand `includeSubDomains` before enabling it |
| `X-Content-Type-Options: nosniff` | Stops MIME-type sniffing | Useful general hardening |
| `Content-Security-Policy` (CSP) | Restricts sources for scripts, styles, images, and other content | Most important for HTML; API-only JSON responses usually gain less from it |
| `X-Frame-Options: DENY` | Prevents framing in browsers | Helps against clickjacking; CSP `frame-ancestors` is the modern alternative |
| `Referrer-Policy` | Limits referrer information sent by browsers | e.g. `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Restricts browser features such as camera or microphone | Mainly relevant to web pages |
| `Cross-Origin-Opener-Policy` (COOP) | Controls relationships between browsing contexts | Mainly web-application hardening |
| `Cross-Origin-Resource-Policy` (CORP) | Controls cross-origin loading of a resource | Configure carefully for resources intended for cross-origin use |

`X-XSS-Protection` is obsolete in modern browsers and should not be relied on; use CSP and secure coding practices instead. Security headers do not replace input validation, authorization, output encoding, or dependency patching.

##### Example middleware

```csharp
public sealed class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        context.Response.OnStarting(() =>
        {
            var headers = context.Response.Headers;
            headers["X-Content-Type-Options"] = "nosniff";
            headers["X-Frame-Options"] = "DENY";
            headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
            headers["Permissions-Policy"] =
                "camera=(), microphone=(), geolocation=()";

            // Only use HSTS when the site is served over HTTPS.
            if (context.Request.IsHttps)
                headers["Strict-Transport-Security"] = "max-age=31536000";

            // Add CSP only after choosing directives that fit the app.
            return Task.CompletedTask;
        });

        await _next(context);
    }
}
```

Register in `Program.cs`:

```csharp
app.UseHttpsRedirection();
app.UseMiddleware<SecurityHeadersMiddleware>();
```

In production, confirm HTTPS termination/proxy configuration and avoid blindly adding `includeSubDomains` or `preload`. A CSP must be tailored to the app; `default-src 'self'` can break legitimate frontend resources if applied without testing.

#### 7. CORS

**CORS is a browser-enforced policy**, not an authentication or authorization mechanism. Configure it with ASP.NET Core rather than manually writing CORS headers.

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Frontend", policy =>
        policy.WithOrigins("https://app.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Authorization", "Content-Type"));
});

var app = builder.Build();
app.UseRouting();
app.UseCors("Frontend");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

If cookies or other credentials are required, configure `.AllowCredentials()` with explicit trusted origins; do not combine credentials with `AllowAnyOrigin()`.

#### 8. File downloads

Use the framework's file result to set the content type and download filename:

```csharp
[HttpGet("report")]
public IActionResult DownloadReport()
{
    byte[] fileBytes = System.IO.File.ReadAllBytes("report.pdf");
    return File(fileBytes, "application/pdf", "report.pdf");
}
```

The filename causes ASP.NET Core to generate an appropriate `Content-Disposition` header.

#### 9. Viva quick questions

**Q: What is the difference between a header and the body?**  
A: Headers carry metadata; the body carries the representation or payload.

**Q: `304` vs `412`?**  
A: `304 Not Modified` is used for conditional retrieval such as `If-None-Match`; `412 Precondition Failed` means a request precondition such as `If-Match` was not satisfied.

**Q: `no-cache` vs `no-store`?**  
A: `no-cache` allows storage but requires revalidation before reuse; `no-store` tells caches not to store the response.

**Q: Is CORS security for the API?**  
A: No. It controls which browser origins can read responses. It does not stop direct requests from tools or servers and does not replace authentication/authorization.

**Q: Is an ETag an authorization token?**  
A: No. It is a representation validator, not a secret and not proof of identity.

**Q: Should every API add every security header?**  
A: No. Choose headers based on whether the endpoint serves browser-rendered content, JSON, files, or other resources, and test for compatibility.

#### 10. Practical checklist

- Use HTTPS; configure authentication and authorization separately from headers.
- Validate inputs and apply correct authorization on every protected endpoint.
- Use `Cache-Control: no-store` for sensitive responses where appropriate.
- Implement ETags only when validators accurately reflect the resource version.
- Configure CORS for known frontend origins.
- Add browser security headers where applicable; test CSP and framing restrictions.
- Avoid leaking tokens, cookies, internal server details, or sensitive data in headers/logs.
- Let ASP.NET Core manage standard protocol headers unless there is a clear reason to override them.

#### 11. More headers worth knowing

**CORS response headers.** These are set by ASP.NET Core's CORS policy, so don't write them by hand.

| Header | Purpose | Example |
|---|---|---|
| `Access-Control-Allow-Credentials` | Allows credentialed requests; can't be combined with a `*` origin | `true` |
| `Access-Control-Expose-Headers` | Response headers that browser JavaScript is allowed to read | `ETag, X-Correlation-ID` |
| `Access-Control-Max-Age` | How long the browser may cache the preflight result | `600` |

**API and application headers.** These are conventions, not all of them standardized. Document them in your API contract.

| Header | Purpose | Example |
|---|---|---|
| `Idempotency-Key` | Lets the API recognise a retried POST and avoid doing the work twice | `Idempotency-Key: 7f2d...` |
| `X-Correlation-ID` / `X-Request-ID` | Trace one request across services and logs | `X-Correlation-ID: 8f14e45f` |
| `RateLimit-Limit` / `-Remaining` / `-Reset` | Tell clients their quota | `RateLimit-Remaining: 20` |
| `X-API-Key` | Simple API-key auth; never put keys in URLs | `X-API-Key: <key>` |
| `Expires` | Old-style expiry date; `Cache-Control` wins if both are present | `Expires: Wed, 21 Oct 2026 07:28:00 GMT` |

**Good to remember**

- Header names are case-insensitive.
- Never log `Authorization` or `Cookie` values.
- Don't set `Host` yourself; the HTTP client manages it.
- Avoid the `X-` prefix for **new** custom headers unless an existing contract requires it.
- `X-XSS-Protection` and `X-Requested-With` are legacy; don't rely on them for security.
- `Server` can leak implementation details, so it's often removed.
- HTTP/2 and HTTP/3 forbid connection-specific headers such as `Connection`, `Keep-Alive`, and `Transfer-Encoding`.

---

<a id="senior-dotnet-storage-session-jwt-viva-md"></a>

## Web Storage, ASP.NET Core Session, and JWT Logout

#### Senior .NET Developer Viva Preparation

This guide consolidates the key concepts, storage locations, retrieval methods, code examples, security considerations, and interview questions without repeating the same explanations.

---

#### 1. Client-side storage vs. server-side session

##### Local storage
- **Where stored:** In the browser, scoped to the origin (scheme + host + port).
- **Lifetime:** Persists across tabs and browser restarts until cleared by the user or application.
- **Access:** JavaScript via `localStorage`.
- **Sent automatically to the server?** No.
- **Typical use:** Non-sensitive preferences, such as theme or language.

```javascript
localStorage.setItem("theme", "dark");
const theme = localStorage.getItem("theme");
localStorage.removeItem("theme");
localStorage.clear(); // Clears all local-storage keys for this origin
```

##### Session storage (browser API)
- **Where stored:** In the browser, scoped to an origin and a particular top-level browsing context (normally a tab).
- **Lifetime:** Usually ends when that tab's page session ends.
- **Access:** JavaScript via `sessionStorage`.
- **Sent automatically to the server?** No.
- **Typical use:** Temporary, non-sensitive UI state, such as a multi-step form's progress.

```javascript
sessionStorage.setItem("step", "2");
const step = sessionStorage.getItem("step");
sessionStorage.removeItem("step");
sessionStorage.clear();
```

##### Cookies
- **Where stored:** In the browser, associated with a domain/path and cookie attributes.
- **Lifetime:** A session cookie generally lasts for the browser session; a persistent cookie has an expiry or `Max-Age`.
- **Access:** Browser JavaScript can read ordinary cookies through `document.cookie`; cookies marked `HttpOnly` cannot be read by JavaScript.
- **Sent automatically to the server?** Matching cookies are sent with applicable HTTP requests.
- **Typical use:** Authentication cookies, session identifiers, and small pieces of state that the server needs to receive.

```javascript
// Demonstration only: do not put secrets in a JavaScript-readable cookie.
document.cookie = "theme=dark; Path=/; Max-Age=604800; Secure; SameSite=Lax";
console.log(document.cookie);
```

For authentication cookies, prefer server-set cookies with `HttpOnly`, `Secure`, and an appropriate `SameSite` policy. `SameSite` helps mitigate some CSRF scenarios but does not replace a full CSRF strategy where one is needed.

##### Quick comparison

| Feature | Local storage | Browser session storage | Cookie |
|---|---|---|---|
| Main location | Browser | Browser, per tab/page session | Browser |
| Sent automatically with HTTP requests | No | No | Yes, when applicable |
| JavaScript access | Yes | Yes | Yes, unless `HttpOnly` |
| Lifetime | Until cleared | Normally until tab session ends | Session or configured expiry |
| Common use | Preferences | Temporary UI state | Authentication/session identifier |

**Security rule:** Do not store secrets or sensitive authentication tokens in local storage or session storage: JavaScript running in the origin can read them, including malicious script injected through an XSS vulnerability.

---

#### 2. What “Session” means in ASP.NET Core

ASP.NET Core session is **server-side application state associated with a browser session**. The browser normally stores only a session identifier in a cookie, commonly named `.AspNetCore.Session`. The application uses that identifier to locate session data in an `IDistributedCache` implementation.

Conceptual flow:

1. The application accesses `HttpContext.Session`.
2. ASP.NET Core issues or uses a session ID cookie.
3. Session values are stored through the configured distributed cache.
4. The browser sends the session cookie on later applicable requests.
5. ASP.NET Core uses the ID to load the session values for that request.

The session cookie is not the session data itself. Do not confuse ASP.NET Core session with browser `sessionStorage`, which is a separate JavaScript API.

##### Where ASP.NET Core session data is stored

It depends on the cache provider configured by the application:

- **Distributed memory cache:** In the current application process. Easy for development, but data is not shared across instances and is lost on process restart.
- **Redis:** Shared cache that can be used by multiple application instances.
- **SQL Server distributed cache:** Stores cache entries in SQL Server.
- **Other `IDistributedCache` providers:** Depending on the application's configuration.

`AddDistributedMemoryCache()` implements `IDistributedCache`, but despite the name, it is **not a true distributed store**. It stores data in the current process.

Session is ephemeral state, not a durable database. The cache may evict data, and a session can expire. Store business-critical or durable data in a database instead.

##### Configure session in modern ASP.NET Core (`Program.cs`)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

// Required cache implementation for session.
// Suitable for local development/single-instance examples only.
builder.Services.AddDistributedMemoryCache();

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(20);
    options.Cookie.Name = ".MyApp.Session";
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true; // Use only if justified by consent requirements.
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Lax;
});

var app = builder.Build();

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseSession(); // Before endpoints that access session
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.Run();
```

For production with multiple instances, configure a shared provider such as Redis or SQL Server instead of relying on in-process memory. Follow the provider's documentation and secure its connection.

##### Store, retrieve, remove, and clear session values

ASP.NET Core session exposes basic byte storage, with convenience extensions for strings and integers.

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/session")]
public class SessionController : ControllerBase
{
    [HttpPost]
    public IActionResult Set()
    {
        HttpContext.Session.SetString("DisplayName", "Example User");
        HttpContext.Session.SetInt32("CartCount", 3);

        return Ok(new { message = "Session values saved." });
    }

    [HttpGet]
    public IActionResult Get()
    {
        var displayName =
            HttpContext.Session.GetString("DisplayName");

        int? cartCount =
            HttpContext.Session.GetInt32("CartCount");

        return Ok(new { displayName, cartCount });
    }

    [HttpDelete("display-name")]
    public IActionResult RemoveDisplayName()
    {
        HttpContext.Session.Remove("DisplayName");
        return NoContent();
    }

    [HttpDelete]
    public IActionResult Clear()
    {
        HttpContext.Session.Clear();
        return NoContent();
    }
}
```

Notes:
- `GetInt32()` returns `int?`, because the key may not exist.
- Session values are not automatically persisted to a database.
- For complex objects, serialize to JSON and store a string/byte array, or store an identifier and retrieve the object from a database. Avoid putting large objects in session.
- `IdleTimeout` is the time the session can remain idle; activity can renew the timeout. It is not necessarily a fixed maximum lifetime from sign-in.
- `HttpContext.Session` is not available until the session middleware has loaded/established session state.

---

#### 3. JWT authentication and logout

##### How does JWT authentication work, step by step?

```text
Step 1 — User logs in
         Browser submits username and password over HTTPS
            ↓
Step 2 — API validates credentials
         Checks the user record and verifies the password hash
            ↓
Step 3 — API creates and signs the JWT
         Adds claims and an expiration time, then signs the token
            ↓
Step 4 — Browser receives the JWT
         For example, in an HttpOnly, Secure cookie
            ↓
Step 5 — Browser requests a protected API
         Browser sends the cookie automatically
            ↓
Step 6 — API validates the JWT
         Checks signature, issuer, audience, expiration, and required claims
            ↓
Step 7 — Authorization
         Checks whether the authenticated user is allowed to perform the operation
```

Notice that at **Step 6** the API does not normally need to query the database to verify the password again. The valid signature proves the token was issued by the trusted server and hasn't been modified. That's what makes JWTs fast and stateless, and it's also why logout is tricky, as the next part explains.

##### Why logout is different with JWT

A signed JWT is normally validated using its signature and claims, including its expiry. In a purely stateless setup, the API does not keep a record of each access token. Consequently, deleting the token from the browser does **not** invalidate a copied token: that token can still work until it expires unless the API has a revocation mechanism.

##### Recommended pattern: short-lived access token + rotating refresh token

1. Issue an access JWT with a short lifetime, for example 5–15 minutes, based on the application's risk profile.
2. Issue a refresh token and store its hash/server-side record with expiry and revocation status.
3. On refresh, validate and rotate the refresh token; reject reused or revoked tokens.
4. On logout, revoke the refresh token (and its token family where appropriate) and clear the client-side token/cookie.
5. Any already-issued access JWT can generally remain valid until its expiry unless you also implement access-token revocation or another server-side check.

If the access JWT is stored in an `HttpOnly` cookie, clear that cookie on logout using the same name and compatible path/domain options. If it is kept in memory by a client application, clear it there as well. Client-side deletion alone is not server-side revocation.

##### JWT lifetime validation in ASP.NET Core

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var signingKey = builder.Configuration["Jwt:SigningKey"]
    ?? throw new InvalidOperationException("JWT signing key is missing.");

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],

            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],

            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(signingKey)),

            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });
```

`ClockSkew = TimeSpan.Zero` removes the default clock-skew allowance. In production, use a strong secret from a secure configuration source such as a secret manager; do not hard-code it or commit it to source control.

##### Approaches for immediate access-token revocation

**A. Denylist / revoked-token store**
- Give each JWT a unique `jti` claim.
- On logout, record that identifier in Redis or a database until the token's expiry.
- On each authenticated request, check whether its `jti` is revoked.
- Reject revoked tokens.

This enables prompt revocation but adds storage and a lookup to requests. Avoid using an unbounded in-memory `HashSet` in a multi-instance or production deployment. A token hash can also be used as a denylist key, but a unique identifier such as `jti` is usually easier to manage.

**B. User-level security stamp or token version**
- Keep a token version/security stamp for the user.
- Include the issued version in the token.
- Compare it against current server-side state when validating requests.
- Increment it to revoke all relevant tokens for that user.

This is useful for “sign out everywhere” or account-compromise scenarios, but it makes validation stateful and requires careful cache invalidation.

**C. Change the signing key**
Changing a shared signing key invalidates every token signed with the old key, not just the logging-out user's token. It is generally a key-rotation or emergency response mechanism, not a normal per-user logout strategy.

##### Refresh-token security essentials

- Generate refresh tokens with a cryptographically secure random generator.
- Store only a hash of each refresh token when feasible.
- Rotate tokens after successful use and detect reuse.
- Revoke tokens on logout, password reset, account disablement, or suspected compromise as required by policy.
- Set expiry, and protect refresh tokens in transit and at rest.
- If refresh tokens are in cookies, account for CSRF protection and use `HttpOnly`, `Secure`, and an appropriate `SameSite` setting.

---

#### 4. Session vs. JWT: senior-level comparison

| Topic | ASP.NET Core session | JWT access token |
|---|---|---|
| State model | Server-side state referenced by a cookie | Usually self-contained token validated by API |
| Client typically holds | Session ID cookie | Token, often in memory or a secure cookie |
| Server storage | Session data in configured cache | No per-access-token storage required unless revocation is implemented |
| Logout | Clear session data and expire/clear session cookie | Clear client token and revoke refresh token; revoke access JWT separately if immediate invalidation is required |
| Multi-instance deployment | Use a shared cache/provider | APIs need compatible validation configuration and signing keys; shared revocation state if used |
| Main trade-off | Easy server-side state, with cache dependency | Convenient distributed validation, but immediate revocation requires extra design |

Neither approach is automatically “more secure” in every architecture. Security depends on token/cookie handling, XSS/CSRF defenses, TLS, expiry, authorization, key management, and operational controls.

---

#### 5. Senior .NET viva questions and model answers

##### Q1. Where is ASP.NET Core session data stored?
**Answer:** In the configured `IDistributedCache` provider. With `AddDistributedMemoryCache()`, it is held in the current process; Redis or SQL Server can provide shared storage for multiple instances.

##### Q2. Is ASP.NET Core session the same as `sessionStorage`?
**Answer:** No. ASP.NET Core session is server-side state identified by a cookie. Browser `sessionStorage` is client-side JavaScript storage scoped to a tab/page session.

##### Q3. What does the browser store for ASP.NET Core session?
**Answer:** Normally a session ID cookie, not the session values themselves. The server uses that identifier to locate data in the configured cache.

##### Q4. Does `AddDistributedMemoryCache()` make session distributed?
**Answer:** No. The implementation of `IDistributedCache` is in-process memory. For multiple app instances, use a genuinely shared provider such as Redis or SQL Server.

##### Q5. What does `IdleTimeout` mean?
**Answer:** It controls how long session data can remain idle before it expires from the session store. It is not necessarily the same as authentication-cookie expiry or a fixed absolute login lifetime.

##### Q6. Why does deleting a JWT in the browser not invalidate it?
**Answer:** The API validates the token independently. A copied token remains valid until expiry unless the API checks revocation state or uses another server-side invalidation mechanism.

##### Q7. What is the recommended JWT logout design?
**Answer:** Short-lived access JWTs plus refresh-token rotation and server-side refresh-token revocation. Add a JWT denylist or user token-version check when immediate access-token revocation is a requirement.

##### Q8. Does revoking a refresh token invalidate an existing access JWT?
**Answer:** Not by itself. It prevents future refreshes, but an already-issued access JWT normally remains valid until expiry unless the API checks revocation state.

##### Q9. Why is storing a JWT in local storage risky?
**Answer:** JavaScript can read local-storage values. An XSS vulnerability can expose the token. An `HttpOnly` cookie prevents JavaScript from reading the cookie, though cookie-based authentication requires appropriate CSRF defenses.

##### Q10. Can JWTs be fully stateless and immediately revocable?
**Answer:** Immediate individual revocation requires the API to consult some state or use another mechanism that changes validation. That introduces a stateful dependency. Without it, short expiry limits the window of exposure.

##### Q11. Should we store a user's full profile or permissions in session/JWT?
**Answer:** Store only what is needed. Large session objects increase cache load; JWT claims can become stale until expiry. For permissions that must change immediately, use server-side authorization data or a version/revocation check.

#### 6. Key takeaways

- **Local storage:** browser-persistent, JavaScript-readable, not automatically sent with requests.
- **Browser session storage:** browser-side and tab-scoped; not ASP.NET Core session.
- **Cookies:** browser-held and automatically sent on matching requests; protect authentication cookies with suitable attributes.
- **ASP.NET Core session:** server-side data reached through a session ID cookie and a configured cache.
- **JWT logout:** client-side deletion is not token revocation. Use short access-token lifetimes, refresh-token rotation/revocation, and add server-side access-token revocation when immediate logout is required.

---

## HTTP and API Interview Question Bank

The following question bank, API standards checklist, testing guidance, and senior scenarios are consolidated here from `senior_dotnet_interview_guide.md`.


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

The request flow (Kestrel → middleware → routing → auth → endpoint) is shown in the REST notes above, section 2. This part zooms in on how the **response** is produced.

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

---

## Additional Questpond question sets

### .NET Core Web API

###### 1. Framework vs. Core (PDF p. 9)
**Answer:** .NET Framework is the original Windows-focused implementation, commonly used for existing ASP.NET applications. Modern .NET (called .NET Core through 3.1, then .NET 5+) is cross-platform, actively developed, and the recommended platform for new applications; “Core” is not a separate current product name.

###### 2. Advantages/features of .NET Core (PDF p. 9)
**Answer:** Modern .NET is cross-platform, open source, performant, modular, and supports side-by-side runtimes, dependency injection, modern hosting, and cloud/container deployment. Choose features and hosting models appropriate to the target .NET version.

###### 3. Disadvantages of Web API (PDF p. 9)
**Answer:** A Web API introduces network latency, operational/deployment complexity, versioning and compatibility obligations, and security/availability concerns. These are trade-offs rather than inherent defects; a simple in-process application may be preferable when remote access or service separation is unnecessary.

###### 4. How does .NET Core start? (PDF p. 9)
**Answer:** The process starts at the compiled entry point (`Main`, often expressed as top-level statements), creates and configures the host, registers services, builds the application, and begins serving requests through the configured server (typically Kestrel, often behind a proxy or platform host).

###### 5. What is middleware in .NET Core? (PDF p. 9)
**Answer:** Middleware is a component in the HTTP request pipeline that can inspect/modify a request or response and either invoke the next component or short-circuit. Ordering matters; common examples include exception handling, authentication, authorization, routing, and endpoint execution.

###### 6. What is a Minimal API? (PDF p. 9)
**Answer:** Minimal APIs define HTTP endpoints with a lightweight route-handler model and less ceremony than controllers. They still support dependency injection, filters, validation patterns, OpenAPI, and middleware, and can be organized into route groups for larger services.

###### 7. How did you create custom middleware? (PDF p. 9)
**Answer:** Create a middleware class that accepts `RequestDelegate` and other dependencies, implement `InvokeAsync(HttpContext)`, and call `await _next(context)` when continuing. Register it in the correct order with `app.UseMiddleware<CustomMiddleware>()`; use `Use`/`Run` extensions where appropriate.

###### 8. What is a request delegate? (PDF p. 9)
**Answer:** `RequestDelegate` is a function-like delegate that accepts an `HttpContext` and returns a `Task`. Middleware uses it to represent the next component in the HTTP pipeline.

###### 9. `app.Use` vs. `app.Run` (PDF p. 9)
**Answer:** `Use` adds middleware that can invoke the next delegate and then run post-processing. `Run` adds terminal middleware with no next delegate, so it short-circuits the remainder of the pipeline.

###### 10. What are filters in .NET Core? (PDF p. 9)
**Answer:** MVC filters run at defined stages around MVC action execution and include authorization, resource, action, exception, and result filters. They are scoped to MVC endpoint processing, unlike general middleware, which participates in the broader HTTP pipeline.

###### 11. What is an authorization filter? (PDF p. 9)
**Answer:** It enforces authorization for an MVC action/controller before normal action execution. Prefer ASP.NET Core's authorization policies and attributes such as `[Authorize]` rather than writing a custom authorization filter for standard cases.

###### 12. What is an exception filter? (PDF p. 9)
**Answer:** An MVC exception filter handles unhandled exceptions during MVC action execution and can translate them into an MVC result. Global exception-handling middleware is usually the better place for consistent handling across all endpoints and middleware.

###### 13. What is an action filter? (PDF p. 9)
**Answer:** An action filter runs before and/or after an MVC action method. It can inspect action arguments/results or apply cross-cutting MVC behavior; use middleware for concerns that should apply outside MVC too.

###### 14. Middleware vs. filter (PDF p. 9)
**Answer:** Middleware runs in the application-wide HTTP pipeline and can affect any request. Filters run within the MVC action pipeline and have access to MVC-specific context such as action arguments/results. Use middleware for broad concerns and filters for MVC-specific behavior.

###### 15. `Configure` vs. `ConfigureServices` (PDF p. 9)
**Answer:** In the older Startup pattern, `ConfigureServices` registers dependencies and framework services; `Configure` builds the HTTP middleware pipeline. Modern hosting often performs these steps in `Program.cs` with `builder.Services` and `app`.

###### 16. .NET Core 5.0 vs. 6.0 (PDF p. 9)
**Answer:** .NET 6 is an LTS release and introduced major hosting simplifications such as minimal hosting and top-level `Program.cs`, plus performance and API improvements. .NET 5 was a non-LTS release; compare specific capabilities against the application's target and support lifecycle.

###### 17. Can we implement `Startup.cs` in .NET 6? (PDF p. 9)
**Answer:** Yes. The minimal hosting model is the default, but the application can still use a `Startup` class by calling `builder.Host.ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>())`, or by organizing equivalent service/pipeline setup in a class.

###### 18. What is dependency injection and what are its types? (PDF p. 9)
**Answer:** Dependency injection supplies a class's dependencies from outside rather than having the class construct concrete dependencies itself. Common forms are constructor, property, and method injection; .NET's built-in container primarily encourages constructor injection and supports service registrations.

###### 19. What is service lifetime? Compare `AddTransient`, `AddSingleton`, and `AddScoped` (PDF p. 9)
**Answer:** Transient creates a new instance each resolution; scoped creates one instance per scope (normally per web request); singleton creates one instance for the application's lifetime. Singleton services must be thread-safe and must not capture shorter-lived scoped services directly.

###### 20. What is routing and what are its types? (PDF p. 9)
**Answer:** Routing maps an incoming URL and HTTP method to an endpoint. ASP.NET Core supports attribute routing (templates on controllers/actions) and conventional routing (route patterns configured centrally); endpoint routing underlies both.

###### 21. What is an attribute? (PDF p. 9)
**Answer:** A C# attribute is metadata attached to a program element and read at compile time or runtime. ASP.NET Core uses attributes such as `[HttpGet]`, `[Route]`, and `[Authorize]` to describe endpoint behavior.

###### 22. Serialization and deserialization (PDF p. 9)
**Answer:** Serialization converts an object/value into a transport or storage representation (such as JSON); deserialization reconstructs a value from that representation. Validate input and configure serialization options deliberately, especially for public API contracts.

###### 23. How do you implement global exception handling? (PDF p. 9)
**Answer:** Configure exception-handling middleware early in the pipeline (`UseExceptionHandler`) and return a consistent, non-sensitive problem response (for example, `ProblemDetails`) while logging the exception with a correlation ID. Do not expose stack traces or secrets to clients in production.

###### 24. What is session management? (PDF p. 9)
**Answer:** Session management maintains user-specific state across HTTP requests, commonly through an opaque session identifier in a cookie and server-side storage. Web APIs are often stateless; if sessions are used, configure secure cookies, expiration, distributed storage, and appropriate privacy/security controls.

###### 25. What is caching, what types are there, and how do you implement it in .NET Core? (PDF p. 9)
**Answer:** Caching stores reusable data to avoid repeated computation or I/O. Common types include in-process memory cache, distributed cache, HTTP response/output cache, and CDN/browser cache. Choose keys, TTLs, invalidation, and consistency rules carefully; use `IMemoryCache`, `IDistributedCache`, or output-caching middleware as appropriate.

###### 26. What is the MVC pattern? (PDF p. 9)
**Answer:** Model-View-Controller separates domain/data concerns (model), presentation (view), and request/input orchestration (controller). It helps separate responsibilities, though a Web API often uses controllers without server-rendered views.

###### 27. What is the MVVP pattern? (PDF p. 9)
**Answer:** The prompt likely means MVVM (“Model–View–ViewModel”); “MVVP” is not a commonly standardized pattern name. In MVVM, a ViewModel exposes presentation state and commands for a View and coordinates with model/domain services, reducing view logic and supporting testability.

###### 28. What is a CORS policy? (PDF p. 9)
**Answer:** Cross-Origin Resource Sharing is a browser-enforced mechanism in which a server declares which origins, methods, and headers may access its responses. Configure a narrow allowlist and required credentials/headers; CORS is not authentication and does not restrict non-browser clients.

###### 29. Why does CORS cause no issue in Postman but does in a browser? (PDF p. 9)
**Answer:** Browsers enforce the same-origin policy and CORS response rules; Postman is not a browser and does not enforce them. Configure the API to allow the intended frontend origin and handle preflight `OPTIONS` requests; never treat CORS as access control.

###### 30. What is a RESTful API? (PDF p. 10)
**Answer:** A REST-style API models resources and uses HTTP semantics and representations, commonly with stateless requests, resource URLs, and methods such as GET, POST, PUT, PATCH, and DELETE. A REST API should respect HTTP status, caching, and idempotency semantics.

###### 31. SOAP vs. REST (PDF p. 10)
**Answer:** SOAP is a formal messaging protocol with XML envelopes, service contracts, and extensible WS-* standards. REST is an architectural style commonly implemented over HTTP with resource-oriented URLs and representations such as JSON; choose based on interoperability, contract, and operational requirements.

###### 32. What is JSON? (PDF p. 10)
**Answer:** JSON is a text-based data interchange format built from objects, arrays, strings, numbers, booleans, and `null`. It is commonly used for web APIs because it is broadly supported and relatively compact.

###### 33. JSON vs. XML (PDF p. 10)
**Answer:** JSON has a compact object/array syntax and maps naturally to common programming data structures. XML supports namespaces, attributes, mixed content, and mature schema/transform tooling. Select the format required by consumers and contracts rather than assuming one is universally superior.

###### 34. POST vs. PUT (do not use records; idempotent) (PDF p. 10)
**Answer:** POST commonly submits data for server-defined processing or creates a subordinate resource; repeating it may create multiple effects. PUT replaces or creates the resource at a known URI and is intended to be idempotent. A record/value being immutable does not define the HTTP method's semantics.

###### 35. If we can modify a resource via POST, why do we need PUT? (PDF p. 10)
**Answer:** HTTP methods communicate semantics to clients, caches, and intermediaries. POST is flexible processing and is not generally idempotent; PUT expresses replacement at a known URI and is idempotent, enabling clients to retry safely when implemented correctly.

###### 36. Can a GET call have a request body? (PDF p. 10)
**Answer:** HTTP does not define generally useful semantics for GET request content, and some clients/proxies may ignore or reject it. Put retrieval criteria in the URI/query string when reasonable; use a purpose-designed POST search endpoint for complex criteria that cannot safely fit a URI.

###### 37. Can you add a resource using GET? (PDF p. 10)
**Answer:** No. GET is defined as a safe retrieval operation and should not cause a state change. Use POST to create a server-selected resource URI or PUT to create/replace a resource at a known URI.

###### 38. What is content negotiation in Web API? (PDF p. 10)
**Answer:** Content negotiation selects a representation based on request headers such as `Accept` and the formats supported by the server. The server can return an appropriate content type or `406 Not Acceptable`; request-body formats are indicated by `Content-Type`.

###### 39. If a REST API is slow, where will you investigate and how will you optimize it? (PDF p. 10)
**Answer:** Start with latency metrics and distributed traces, then isolate time spent in application code, database, network, serialization, and dependencies. Address the measured bottleneck—such as query count, payload size, blocking, or missing caching—then load-test and verify correctness and tail latency.

###### 40. What is API versioning, what are its types, and how is it implemented? (PDF p. 10)
**Answer:** Versioning manages incompatible contract changes. Common approaches put the version in the URL path, query, header, or media type. Define a support/deprecation policy, document versions in OpenAPI, and use a maintained ASP.NET API-versioning package or explicit endpoint conventions.

###### 41. Return types in .NET Core (PDF p. 10)
**Answer:** Controller actions commonly return `IActionResult` for varied HTTP responses, `ActionResult<T>` for a typed body plus status results, or a concrete DTO for a simple success response. Minimal APIs can return typed results; use async variants when work is asynchronous.

###### 42. Can we have multiple return types in .NET Core? (PDF p. 10)
**Answer:** Yes. An action can return different status/body outcomes through `IActionResult` or `ActionResult<T>` (for example, `Ok(dto)`, `NotFound()`, and `BadRequest()`), while OpenAPI metadata should describe the possible responses.

###### 43. How do you implement sorting and paging in .NET Core? (PDF p. 10)
**Answer:** Accept validated sort keys/direction and page size/continuation parameters, apply them to an ordered database query before materialization, and cap page sizes. Prefer keyset/continuation paging for large, changing datasets; return metadata or a continuation token as part of the API contract.

###### 44. How do you implement authentication/authorization in .NET Core? (PDF p. 10)
**Answer:** Configure an authentication scheme (for example, OpenID Connect/JWT bearer), call `UseAuthentication()` before `UseAuthorization()`, and protect endpoints with policies/roles/claims. Authentication identifies the caller; authorization decides what that caller may do.

###### 45. What is JWT? State its parts and implementation (PDF p. 10)
**Answer:** A JSON Web Token commonly has three Base64URL-encoded parts: header, payload/claims, and signature (JWS). Validate signature, issuer, audience, and expiry on the API using trusted keys; avoid sensitive secrets in claims because the payload is encoded, not encrypted.

###### 46. JWT logout (PDF p. 10)
**Answer:** A self-contained access token remains valid until expiry unless the server adds revocation. Use short-lived access tokens with refresh-token revocation/rotation, or a denylist/session version when immediate revocation is required; clear client-held credentials and use secure storage.

###### 47. When should OpenID Connect/OAuth be used instead of custom JWT? (PDF p. 10)
**Answer:** Use OpenID Connect for standardized user authentication and OAuth 2.0 for delegated authorization/API access, typically through a trusted identity provider. Custom token issuance is rarely justified because key management, validation, rotation, and protocol security are easy to get wrong.

###### 48. How do you save sensitive data in a database? (PDF p. 10)
**Answer:** Minimize and classify data, restrict access, use parameterized queries, encrypt transport and storage, and manage keys separately through a vault/KMS with rotation and auditing. Hash passwords with a dedicated slow password-hashing algorithm and unique salts; encryption is not a substitute for authorization.

###### 49. What are common encryption techniques? (PDF p. 10)
**Answer:** Symmetric authenticated encryption (such as AES-GCM) protects data with a shared key; public-key cryptography (such as RSA-OAEP or elliptic-curve schemes) supports key exchange/signatures; TLS protects transport. Hashing is one-way, not encryption. Prefer vetted libraries and managed key services.

###### 50. What data encryption techniques have you used in a project? (PDF p. 10)
**Answer:** Answer truthfully and describe the actual scope: for example, TLS for transport, database/platform encryption at rest, and application-level authenticated encryption for selected sensitive fields using keys from a managed vault. Explain access controls, rotation, and how keys are kept separate from data.

###### 51. What is an API gateway? (PDF p. 10)
**Answer:** An API gateway is an entry point that can route requests to services and centralize concerns such as authentication, rate limits, TLS termination, and observability. Keep business logic in services and avoid making the gateway a single overloaded bottleneck.

###### 52. What is a rate limiter? (PDF p. 10)
**Answer:** A rate limiter constrains request volume over a time window or concurrency limit, typically per client, user, or route, to protect capacity and provide fair use. Choose an algorithm such as token bucket, return `429` and retry guidance, and consider distributed coordination when running multiple instances.

###### 53–54. Uploading lakhs of records from MVC: how do you make the transaction faster? (PDF p. 10)
**Answer:** Validate the file and stream or parse it in bounded batches, then use bulk-copy/bulk-insert facilities and set-based updates rather than one ORM insert per row. Avoid one enormous transaction: use appropriately sized transactions, idempotent job tracking, staging tables, and background processing with progress/error reporting. Tune batch size and indexes with measurements.

###### 55. Explain the MVC lifecycle and flow of a request from browser to server and back (PDF p. 10)
**Answer:** The browser sends an HTTP request; the server middleware handles cross-cutting concerns, routing selects an endpoint, model binding/validation prepares inputs, and filters/action logic invoke application services. The result is serialized or rendered, passes back through middleware, and is returned as an HTTP response for the browser to process.

###### 56. What is cache management? How can it be implemented in React, ASP.NET MVC, and ASP.NET Core Web API? (PDF p. 10)
**Answer:** Cache management defines what is cached, for how long, and how stale entries are invalidated. In React, use HTTP caching or a server-state cache with freshness/invalidation rules; in ASP.NET MVC, use output/data caching as appropriate; in ASP.NET Core Web API, use response/output caching or `IMemoryCache`/`IDistributedCache`. Ensure authorization-sensitive responses are not shared across users.

---

### API Testing

###### 1. Why do we need testing? (PDF p. 15)
**Answer:** Testing checks that software meets expected behavior and catches regressions before release. A balanced suite provides fast feedback at unit level and confidence in integration, security, and end-to-end behavior.

###### 2. What is unit testing? (PDF p. 15)
**Answer:** A unit test verifies a small unit of behavior in isolation, usually with controlled dependencies and inputs. It should be deterministic, fast, and focused on observable outcomes rather than implementation details.

###### 3. What is integration testing? (PDF p. 15)
**Answer:** An integration test verifies that multiple components work together, such as an API endpoint with routing, serialization, dependency injection, and a test database. It exercises real boundaries that unit tests intentionally isolate.

###### 4. Difference between unit and integration testing (PDF p. 15)
**Answer:** Unit tests isolate a small component and are usually faster, pinpointing logic errors. Integration tests validate interactions across components or infrastructure and offer broader confidence, but take longer and require more setup.

###### 5. Which unit-testing framework have you used in your project? (PDF p. 15)
**Answer:** Name the framework you actually used—such as xUnit, NUnit, or MSTest—and describe a representative test fixture, assertions, setup, and CI execution. Do not claim a framework or project experience you have not had.

###### 6. What is the code coverage percentage? (PDF p. 15)
**Answer:** Code coverage is the proportion of measured code executed by tests (line, branch, or other coverage criteria). State the project's actual report and metric; high coverage alone does not prove tests are meaningful, so focus on critical behavior and uncovered risk.

###### 7. What is Moq? Have you used it in a project? (PDF p. 15)
**Answer:** Moq is a .NET mocking library used to create test doubles for interfaces/dependencies and verify interactions. Describe actual use—for example, configuring a repository mock and asserting a service result—or be transparent if you have not used it; prefer real implementations for integration tests.

###### 8. How do you handle exceptions in unit testing? (PDF p. 15)
**Answer:** Assert the expected exception type and, when relevant, its message or properties using the framework's exception assertion (for example, xUnit's `Assert.ThrowsAsync<T>`). Also test that expected failures are translated/handled correctly; avoid broad catches that make a test pass for the wrong exception.
