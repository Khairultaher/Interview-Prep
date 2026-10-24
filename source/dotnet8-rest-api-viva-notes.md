# REST API and ASP.NET Core 8 — Viva Preparation Notes

## 1. What is a REST API?

REST (Representational State Transfer) is an architectural style for designing networked APIs around **resources**. ASP.NET Core Web API commonly exposes resources over HTTP and exchanges JSON.

- `GET /api/employees` — retrieve employees
- `GET /api/employees/1` — retrieve one employee
- `POST /api/employees` — create an employee
- `PUT /api/employees/1` — replace/update an employee
- `PATCH /api/employees/1` — partially update an employee
- `DELETE /api/employees/1` — delete an employee

REST is not a protocol. HTTP is the protocol commonly used to implement REST-style APIs.

## 2. How an HTTP request travels through .NET 8

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

### What happens at each stage?

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

## 3. Middleware: the key concept

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

### Common middleware

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

## 4. Minimal API vs. Controller-based API

Both use the same ASP.NET Core HTTP server and middleware pipeline. The difference is mainly how endpoints are declared and which framework features are available.

### Minimal API

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/employees/{id:int}", (int id) =>
    Results.Ok(new { Id = id, Name = "Ayesha" }));

app.Run();
```

### Controller-based API

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

## 5. Dependency injection and business logic

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

## 6. Authorization in Minimal APIs

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

## 7. Swagger / OpenAPI in .NET 8 Minimal APIs

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

### Add example request/response data

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

## 8. REST API vs. gRPC — quick comparison

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

## 9. HTTP methods and status codes to remember

### Methods

- `GET`: read a resource; should not change server state.
- `POST`: create a resource or trigger an operation.
- `PUT`: replace a resource representation; intended to be idempotent.
- `PATCH`: partially modify a resource.
- `DELETE`: remove a resource; intended to be idempotent in its effect.
- `HEAD`: like GET but without a response body.
- `OPTIONS`: discover communication options; commonly used in CORS preflight.

**Idempotent** means repeating the same request has the same intended effect as making it once. PUT or DELETE may return different status codes on later calls and still be idempotent.

### Status codes

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

## 10. Common viva questions and short answers

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

## Final mental model

**Kestrel receives → middleware processes → routing selects → authentication identifies → authorization permits/denies → endpoint binds inputs and runs → services/data layer work → HTTP result is serialized → response returns through middleware to the client.**
