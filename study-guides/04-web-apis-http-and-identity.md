# Web APIs, HTTP, and Identity

This guide combines the distinct source materials below. Original explanations, examples, and interview questions are retained; exact duplicate source files are excluded and documented in the library index.

## Included source materials

- [csharp-aspnetcore-rest-api-notes.md](#csharp-aspnetcore-rest-api-notes-md)
- [dotnet8-rest-api-viva-notes.md](#dotnet8-rest-api-viva-notes-md)
- [common-http-headers.md](#common-http-headers-md)
- [http-headers-dotnet-viva.md](#http-headers-dotnet-viva-md)
- [senior_dotnet_storage_session_jwt_viva.md](#senior-dotnet-storage-session-jwt-viva-md)
---

<a id="csharp-aspnetcore-rest-api-notes-md"></a>

## Source: C# and ASP.NET Core REST API Notes

*Imported from `csharp-aspnetcore-rest-api-notes.md`.*

### C# and ASP.NET Core REST API Notes

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

## Source: REST API and ASP.NET Core 8 — Viva Preparation Notes

*Imported from `dotnet8-rest-api-viva-notes.md`.*

### REST API and ASP.NET Core 8 — Viva Preparation Notes

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

Middleware components are chained. A component can run code before and after `await next(context)`.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Use(async (context, next) =>
{
    Console.WriteLine($"Request: {context.Request.Method} {context.Request.Path}");

    await next(context); // Continue to the next component

    Console.WriteLine($"Response: {context.Response.StatusCode}");
});

app.MapGet("/hello", () => Results.Ok(new { message = "Hello" }));
app.Run();
```

The first log runs on the way in; the second runs after downstream processing completes. If middleware does not call `next`, it short-circuits the pipeline.

##### Common middleware

| Component | Purpose |
|---|---|
| Exception handling | Converts unhandled exceptions into controlled error responses |
| HTTPS redirection | Redirects HTTP requests to HTTPS |
| CORS | Controls which browser origins can call the API |
| Authentication | Establishes the caller's identity |
| Authorization | Enforces access policies |
| Response compression | Compresses eligible response bodies |
| Custom logging | Records request/response information |

A typical configuration order is:

```csharp
app.UseExceptionHandler();
app.UseHttpsRedirection();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers(); // Or map Minimal API endpoints
```

Register services before `builder.Build()`. Configure middleware before mapping/running endpoints as appropriate. Routing is handled by endpoint routing; explicit `UseRouting()` is often unnecessary in simple modern applications, but can be used when precise ordering is needed.

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

**Q: What is middleware?**  
A component in the HTTP pipeline that can process a request, call the next component, and process the response.

**Q: What is the difference between authentication and authorization?**  
Authentication establishes identity; authorization determines what that identity is permitted to do.

**Q: What is model binding?**  
The framework maps request data—such as route values, query parameters, headers or JSON bodies—to handler parameters or .NET objects.

**Q: What is dependency injection?**  
A technique where a class receives dependencies from a container instead of creating them itself. ASP.NET Core includes a built-in DI container.

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

<a id="common-http-headers-md"></a>

## Source: Common HTTP Headers

*Imported from `common-http-headers.md`.*

### Common HTTP Headers

HTTP headers are key-value pairs exchanged between clients and servers to provide metadata about a request or response. The same header can appear in both directions; each header is listed once below to avoid duplication.

#### 1. Request and Representation Headers

| Header | Purpose | Example |
|---|---|---|
| `Accept` | Media types the client can process. | `Accept: application/json` |
| `Accept-Encoding` | Compression formats the client supports. | `Accept-Encoding: gzip, br` |
| `Accept-Language` | Preferred response language(s). | `Accept-Language: en-US,en;q=0.9` |
| `Authorization` | Credentials used to authenticate a request. | `Authorization: Bearer <token>` |
| `Content-Encoding` | Encoding applied to the message body. | `Content-Encoding: gzip` |
| `Content-Length` | Message-body size in bytes, when applicable. | `Content-Length: 1500` |
| `Content-Type` | Media type of the message body. | `Content-Type: application/json` |
| `Cookie` | Cookies sent by the client to the server. | `Cookie: sessionId=<value>` |
| `Host` | Target host and optional port in HTTP/1.1. | `Host: api.example.com` |
| `If-Match` | Makes a request conditional on a matching entity tag; often used to prevent lost updates. | `If-Match: "v3"` |
| `If-None-Match` | Supports cache validation or conditional requests when an entity tag does not match. | `If-None-Match: "v3"` |
| `Origin` | Identifies the origin that initiated a request; used by CORS. | `Origin: https://app.example.com` |
| `Referer` | Identifies the referring page, subject to browser privacy policies. | `Referer: https://example.com/page` |
| `User-Agent` | Identifies the client software; format and reliability vary. | `User-Agent: ExampleClient/1.0` |

#### 2. Response, Caching, and Redirect Headers

| Header | Purpose | Example |
|---|---|---|
| `Cache-Control` | Controls caching behavior. | `Cache-Control: no-store` |
| `ETag` | Provides a validator for a particular representation. | `ETag: "v3"` |
| `Expires` | Gives a response expiration date; `Cache-Control` generally takes precedence. | `Expires: Wed, 21 Oct 2026 07:28:00 GMT` |
| `Location` | Indicates a redirect target or the location of a newly created resource. | `Location: /api/orders/123` |
| `Retry-After` | Tells a client when it may retry after certain responses, such as `429` or `503`. | `Retry-After: 60` |
| `Set-Cookie` | Asks the client to store a cookie. | `Set-Cookie: sessionId=<value>; HttpOnly; Secure; SameSite=Lax` |
| `Vary` | Identifies request headers that affect which cached representation is selected. | `Vary: Accept-Encoding` |

#### 3. CORS Headers

Cross-Origin Resource Sharing (CORS) headers let a server state which cross-origin browser requests are permitted.

| Header | Purpose | Example |
|---|---|---|
| `Access-Control-Allow-Origin` | Specifies the allowed origin. | `Access-Control-Allow-Origin: https://app.example.com` |
| `Access-Control-Allow-Credentials` | Allows credentialed CORS requests when set to `true`; wildcard origin cannot be used for credentialed responses. | `Access-Control-Allow-Credentials: true` |
| `Access-Control-Allow-Methods` | Lists methods permitted by the CORS policy. | `Access-Control-Allow-Methods: GET, POST, PUT` |
| `Access-Control-Allow-Headers` | Lists request headers permitted by the CORS policy. | `Access-Control-Allow-Headers: Content-Type, Authorization` |
| `Access-Control-Expose-Headers` | Lists response headers browser JavaScript may read beyond the CORS-safelisted response headers. | `Access-Control-Expose-Headers: ETag, X-Correlation-ID` |
| `Access-Control-Max-Age` | Specifies how long a preflight result may be cached. | `Access-Control-Max-Age: 600` |

#### 4. Security Headers

| Header | Purpose | Example |
|---|---|---|
| `Content-Security-Policy` | Restricts permitted sources for scripts, styles, images, and other resources. | `Content-Security-Policy: default-src 'self'` |
| `Permissions-Policy` | Controls access to selected browser features. | `Permissions-Policy: camera=(), microphone=()` |
| `Referrer-Policy` | Controls how much referrer information the browser sends. | `Referrer-Policy: strict-origin-when-cross-origin` |
| `Strict-Transport-Security` | Tells browsers to use HTTPS for the specified period. Send only over HTTPS. | `Strict-Transport-Security: max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | Prevents MIME-type sniffing. | `X-Content-Type-Options: nosniff` |
| `X-Frame-Options` | Controls whether a page may be embedded in a frame; CSP `frame-ancestors` is the more flexible modern control. | `X-Frame-Options: DENY` |

**Note:** `X-XSS-Protection` is obsolete and should not be relied on for security. Prefer a well-configured Content Security Policy and context-appropriate output encoding.

#### 5. API and Application-Specific Headers

These are common conventions, not all standardized HTTP headers. Prefer a documented naming scheme and avoid adding the `X-` prefix to new custom names unless required by an existing API contract.

| Header | Purpose | Example |
|---|---|---|
| `Idempotency-Key` | Lets an API recognize retries of the same operation when the API explicitly supports it. | `Idempotency-Key: 7f2d...` |
| `X-Correlation-ID` | Custom identifier used to trace a request across services. | `X-Correlation-ID: 8f14e45f...` |
| `X-Request-ID` | Custom identifier for a request, often used in logs and support diagnostics. | `X-Request-ID: req-12345` |
| `RateLimit-Limit` | Can communicate the applicable request quota when supported by the API. | `RateLimit-Limit: 100` |
| `RateLimit-Remaining` | Can communicate the remaining quota when supported by the API. | `RateLimit-Remaining: 20` |
| `RateLimit-Reset` | Can communicate when a quota resets, according to the API's documented format. | `RateLimit-Reset: 60` |

`X-API-Key` is another custom convention sometimes used for API-key authentication. For example: `X-API-Key: <key>`. Prefer secure transport, avoid exposing keys in URLs, and use the authentication mechanism documented by the API.

#### 6. Practical Notes

- Header names are case-insensitive.
- Do not manually set `Host` in ordinary browser or HTTP-client code; the client generally manages it.
- `Content-Type` describes the body being sent; `Accept` describes the response formats the client prefers.
- `Authorization` and `Cookie` carry credentials or session data. Do not log their values.
- Use `Cache-Control: no-store` for sensitive responses that should not be stored by caches.
- CORS is enforced by browsers; it is not a replacement for server-side authentication or authorization.
- `Server` may disclose implementation details and is often omitted or minimized.
- `X-Requested-With: XMLHttpRequest` is a legacy convention, not a reliable security control.
- HTTP/2 and HTTP/3 have protocol-specific restrictions on connection-specific headers such as `Connection`, `Keep-Alive`, `Transfer-Encoding`, and `Upgrade`.

#### 7. Example API Exchange

Request:

```http
GET /api/products HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer <token>
X-Correlation-ID: 8f14e45f
```

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
X-Correlation-ID: 8f14e45f

{"items":[]}
```

*Examples use placeholder domains and credentials. Never use real secrets in documentation or logs.*

---

<a id="http-headers-dotnet-viva-md"></a>

## Source: HTTP Headers in .NET REST APIs — Viva Preparation

*Imported from `http-headers-dotnet-viva.md`.*

### HTTP Headers in .NET REST APIs — Viva Preparation

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

**Q: `401` vs `403`?**  
A: `401` means valid authentication credentials are missing or unacceptable; `403` means the server refuses access, commonly because the authenticated caller lacks permission.

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

---

<a id="senior-dotnet-storage-session-jwt-viva-md"></a>

## Source: Web Storage, ASP.NET Core Session, and JWT Logout

*Imported from `senior_dotnet_storage_session_jwt_viva.md`.*

### Web Storage, ASP.NET Core Session, and JWT Logout
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

##### Q12. What is the difference between authentication and authorization?
**Answer:** Authentication establishes who the caller is. Authorization determines what that caller is permitted to do. A valid session or JWT does not by itself guarantee access to every resource.

---

#### 6. Key takeaways

- **Local storage:** browser-persistent, JavaScript-readable, not automatically sent with requests.
- **Browser session storage:** browser-side and tab-scoped; not ASP.NET Core session.
- **Cookies:** browser-held and automatically sent on matching requests; protect authentication cookies with suitable attributes.
- **ASP.NET Core session:** server-side data reached through a session ID cookie and a configured cache.
- **JWT logout:** client-side deletion is not token revocation. Use short access-token lifetimes, refresh-token rotation/revocation, and add server-side access-token revocation when immediate logout is required.
