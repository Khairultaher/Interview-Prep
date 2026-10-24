# HTTP Headers in .NET REST APIs — Viva Preparation

A concise reference to common HTTP request/response headers, caching, ETags, and security headers, with ASP.NET Core examples.

## 1. What is an HTTP header?

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

## 2. Important request headers

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

### Read a request header in ASP.NET Core

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

## 3. Important response headers

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

### Add a custom response header

```csharp
[HttpGet]
public IActionResult Get()
{
    Response.Headers["X-Correlation-ID"] = HttpContext.TraceIdentifier;
    return Ok(new { message = "Success" });
}
```

For headers applying across many endpoints, middleware is usually more maintainable.

## 4. ETag: caching and concurrency

An **ETag** (entity tag) is an opaque validator identifying a particular version of a representation. It can save bandwidth and help prevent one client from overwriting another client's changes.

### Conditional GET

1. The server returns a body and `ETag: "product-42-v3"`.
2. The client stores the ETag.
3. The client later sends `If-None-Match: "product-42-v3"`.
4. If unchanged, the server can return `304 Not Modified` with no response body. Otherwise, it returns the updated representation and a new ETag.

### Conditional update

A client can send `If-Match: "product-42-v3"` with an update. The server updates only if the current version matches; otherwise it returns `412 Precondition Failed`. The check and database update must be atomic to avoid race conditions.

### ASP.NET Core example: conditional GET

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

## 5. Cache-Control

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

## 6. Security headers

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

### Example middleware

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

## 7. CORS

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

## 8. File downloads

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

## 9. Viva quick questions

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

## 10. Practical checklist

- Use HTTPS; configure authentication and authorization separately from headers.
- Validate inputs and apply correct authorization on every protected endpoint.
- Use `Cache-Control: no-store` for sensitive responses where appropriate.
- Implement ETags only when validators accurately reflect the resource version.
- Configure CORS for known frontend origins.
- Add browser security headers where applicable; test CSP and framing restrictions.
- Avoid leaking tokens, cookies, internal server details, or sensitive data in headers/logs.
- Let ASP.NET Core manage standard protocol headers unless there is a clear reason to override them.
