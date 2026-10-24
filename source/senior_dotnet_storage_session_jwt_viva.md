# Web Storage, ASP.NET Core Session, and JWT Logout
## Senior .NET Developer Viva Preparation

This guide consolidates the key concepts, storage locations, retrieval methods, code examples, security considerations, and interview questions without repeating the same explanations.

---

## 1. Client-side storage vs. server-side session

### Local storage
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

### Session storage (browser API)
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

### Cookies
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

### Quick comparison

| Feature | Local storage | Browser session storage | Cookie |
|---|---|---|---|
| Main location | Browser | Browser, per tab/page session | Browser |
| Sent automatically with HTTP requests | No | No | Yes, when applicable |
| JavaScript access | Yes | Yes | Yes, unless `HttpOnly` |
| Lifetime | Until cleared | Normally until tab session ends | Session or configured expiry |
| Common use | Preferences | Temporary UI state | Authentication/session identifier |

**Security rule:** Do not store secrets or sensitive authentication tokens in local storage or session storage: JavaScript running in the origin can read them, including malicious script injected through an XSS vulnerability.

---

## 2. What “Session” means in ASP.NET Core

ASP.NET Core session is **server-side application state associated with a browser session**. The browser normally stores only a session identifier in a cookie, commonly named `.AspNetCore.Session`. The application uses that identifier to locate session data in an `IDistributedCache` implementation.

Conceptual flow:

1. The application accesses `HttpContext.Session`.
2. ASP.NET Core issues or uses a session ID cookie.
3. Session values are stored through the configured distributed cache.
4. The browser sends the session cookie on later applicable requests.
5. ASP.NET Core uses the ID to load the session values for that request.

The session cookie is not the session data itself. Do not confuse ASP.NET Core session with browser `sessionStorage`, which is a separate JavaScript API.

### Where ASP.NET Core session data is stored

It depends on the cache provider configured by the application:

- **Distributed memory cache:** In the current application process. Easy for development, but data is not shared across instances and is lost on process restart.
- **Redis:** Shared cache that can be used by multiple application instances.
- **SQL Server distributed cache:** Stores cache entries in SQL Server.
- **Other `IDistributedCache` providers:** Depending on the application's configuration.

`AddDistributedMemoryCache()` implements `IDistributedCache`, but despite the name, it is **not a true distributed store**. It stores data in the current process.

Session is ephemeral state, not a durable database. The cache may evict data, and a session can expire. Store business-critical or durable data in a database instead.

### Configure session in modern ASP.NET Core (`Program.cs`)

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

### Store, retrieve, remove, and clear session values

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

## 3. JWT authentication and logout

### Why logout is different with JWT

A signed JWT is normally validated using its signature and claims, including its expiry. In a purely stateless setup, the API does not keep a record of each access token. Consequently, deleting the token from the browser does **not** invalidate a copied token: that token can still work until it expires unless the API has a revocation mechanism.

### Recommended pattern: short-lived access token + rotating refresh token

1. Issue an access JWT with a short lifetime, for example 5–15 minutes, based on the application's risk profile.
2. Issue a refresh token and store its hash/server-side record with expiry and revocation status.
3. On refresh, validate and rotate the refresh token; reject reused or revoked tokens.
4. On logout, revoke the refresh token (and its token family where appropriate) and clear the client-side token/cookie.
5. Any already-issued access JWT can generally remain valid until its expiry unless you also implement access-token revocation or another server-side check.

If the access JWT is stored in an `HttpOnly` cookie, clear that cookie on logout using the same name and compatible path/domain options. If it is kept in memory by a client application, clear it there as well. Client-side deletion alone is not server-side revocation.

### JWT lifetime validation in ASP.NET Core

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

### Approaches for immediate access-token revocation

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

### Refresh-token security essentials

- Generate refresh tokens with a cryptographically secure random generator.
- Store only a hash of each refresh token when feasible.
- Rotate tokens after successful use and detect reuse.
- Revoke tokens on logout, password reset, account disablement, or suspected compromise as required by policy.
- Set expiry, and protect refresh tokens in transit and at rest.
- If refresh tokens are in cookies, account for CSRF protection and use `HttpOnly`, `Secure`, and an appropriate `SameSite` setting.

---

## 4. Session vs. JWT: senior-level comparison

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

## 5. Senior .NET viva questions and model answers

### Q1. Where is ASP.NET Core session data stored?
**Answer:** In the configured `IDistributedCache` provider. With `AddDistributedMemoryCache()`, it is held in the current process; Redis or SQL Server can provide shared storage for multiple instances.

### Q2. Is ASP.NET Core session the same as `sessionStorage`?
**Answer:** No. ASP.NET Core session is server-side state identified by a cookie. Browser `sessionStorage` is client-side JavaScript storage scoped to a tab/page session.

### Q3. What does the browser store for ASP.NET Core session?
**Answer:** Normally a session ID cookie, not the session values themselves. The server uses that identifier to locate data in the configured cache.

### Q4. Does `AddDistributedMemoryCache()` make session distributed?
**Answer:** No. The implementation of `IDistributedCache` is in-process memory. For multiple app instances, use a genuinely shared provider such as Redis or SQL Server.

### Q5. What does `IdleTimeout` mean?
**Answer:** It controls how long session data can remain idle before it expires from the session store. It is not necessarily the same as authentication-cookie expiry or a fixed absolute login lifetime.

### Q6. Why does deleting a JWT in the browser not invalidate it?
**Answer:** The API validates the token independently. A copied token remains valid until expiry unless the API checks revocation state or uses another server-side invalidation mechanism.

### Q7. What is the recommended JWT logout design?
**Answer:** Short-lived access JWTs plus refresh-token rotation and server-side refresh-token revocation. Add a JWT denylist or user token-version check when immediate access-token revocation is a requirement.

### Q8. Does revoking a refresh token invalidate an existing access JWT?
**Answer:** Not by itself. It prevents future refreshes, but an already-issued access JWT normally remains valid until expiry unless the API checks revocation state.

### Q9. Why is storing a JWT in local storage risky?
**Answer:** JavaScript can read local-storage values. An XSS vulnerability can expose the token. An `HttpOnly` cookie prevents JavaScript from reading the cookie, though cookie-based authentication requires appropriate CSRF defenses.

### Q10. Can JWTs be fully stateless and immediately revocable?
**Answer:** Immediate individual revocation requires the API to consult some state or use another mechanism that changes validation. That introduces a stateful dependency. Without it, short expiry limits the window of exposure.

### Q11. Should we store a user's full profile or permissions in session/JWT?
**Answer:** Store only what is needed. Large session objects increase cache load; JWT claims can become stale until expiry. For permissions that must change immediately, use server-side authorization data or a version/revocation check.

### Q12. What is the difference between authentication and authorization?
**Answer:** Authentication establishes who the caller is. Authorization determines what that caller is permitted to do. A valid session or JWT does not by itself guarantee access to every resource.

---

## 6. Key takeaways

- **Local storage:** browser-persistent, JavaScript-readable, not automatically sent with requests.
- **Browser session storage:** browser-side and tab-scoped; not ASP.NET Core session.
- **Cookies:** browser-held and automatically sent on matching requests; protect authentication cookies with suitable attributes.
- **ASP.NET Core session:** server-side data reached through a session ID cookie and a configured cache.
- **JWT logout:** client-side deletion is not token revocation. Use short access-token lifetimes, refresh-token rotation/revocation, and add server-side access-token revocation when immediate logout is required.
