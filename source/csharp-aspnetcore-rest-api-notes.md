# C# and ASP.NET Core REST API Notes

A concise reference for C# range/index syntax, local functions, response formats, and controller return types.

## 1. C# Range and Index Syntax

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

## 2. Function Inside a Function in C#

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

## 3. `IActionResult`, `ActionResult`, and `ActionResult<T>`

These are ASP.NET Core MVC action return types.

### `IActionResult`

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

### `ActionResult`

The non-generic base class for MVC action results. Like `IActionResult`, it is useful when returning HTTP results without declaring a specific response model.

```csharp
[HttpDelete("{id}")]
public ActionResult DeleteProduct(int id)
{
    // Delete product...
    return NoContent();
}
```

### `ActionResult<T>`

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

### Quick comparison

| Return type | Best fit |
|---|---|
| `IActionResult` | Flexible HTTP results; no single declared success model |
| `ActionResult` | Non-generic MVC result |
| `ActionResult<T>` | A typed success model plus results such as `NotFound()` or `BadRequest()` |

**Practical default:** Prefer `ActionResult<T>` for typical CRUD endpoints that return a model on success. Use `IActionResult` when the action's result is not naturally represented by one model type. Both can return standard HTTP results.

## 4. Returning Different Formats from a REST API

If clients need JSON, XML, or CSV, choose the mechanism based on the format.

### JSON and XML with content negotiation

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

### CSV or another custom format

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

### Choosing the format-selection approach

| Approach | Use when |
|---|---|
| `Accept` header / content negotiation | Clients request a representation such as JSON or XML |
| Query parameter such as `?format=csv` | You intentionally expose a simple export option in the URL |
| Custom output formatter | Many endpoints need a consistent format such as CSV |

Keep the underlying response model and business logic independent of the representation where possible. Document supported media types and status codes in OpenAPI/Swagger, and test each format.
