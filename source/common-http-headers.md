# Common HTTP Headers

HTTP headers are key-value pairs exchanged between clients and servers to provide metadata about a request or response. The same header can appear in both directions; each header is listed once below to avoid duplication.

## 1. Request and Representation Headers

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

## 2. Response, Caching, and Redirect Headers

| Header | Purpose | Example |
|---|---|---|
| `Cache-Control` | Controls caching behavior. | `Cache-Control: no-store` |
| `ETag` | Provides a validator for a particular representation. | `ETag: "v3"` |
| `Expires` | Gives a response expiration date; `Cache-Control` generally takes precedence. | `Expires: Wed, 21 Oct 2026 07:28:00 GMT` |
| `Location` | Indicates a redirect target or the location of a newly created resource. | `Location: /api/orders/123` |
| `Retry-After` | Tells a client when it may retry after certain responses, such as `429` or `503`. | `Retry-After: 60` |
| `Set-Cookie` | Asks the client to store a cookie. | `Set-Cookie: sessionId=<value>; HttpOnly; Secure; SameSite=Lax` |
| `Vary` | Identifies request headers that affect which cached representation is selected. | `Vary: Accept-Encoding` |

## 3. CORS Headers

Cross-Origin Resource Sharing (CORS) headers let a server state which cross-origin browser requests are permitted.

| Header | Purpose | Example |
|---|---|---|
| `Access-Control-Allow-Origin` | Specifies the allowed origin. | `Access-Control-Allow-Origin: https://app.example.com` |
| `Access-Control-Allow-Credentials` | Allows credentialed CORS requests when set to `true`; wildcard origin cannot be used for credentialed responses. | `Access-Control-Allow-Credentials: true` |
| `Access-Control-Allow-Methods` | Lists methods permitted by the CORS policy. | `Access-Control-Allow-Methods: GET, POST, PUT` |
| `Access-Control-Allow-Headers` | Lists request headers permitted by the CORS policy. | `Access-Control-Allow-Headers: Content-Type, Authorization` |
| `Access-Control-Expose-Headers` | Lists response headers browser JavaScript may read beyond the CORS-safelisted response headers. | `Access-Control-Expose-Headers: ETag, X-Correlation-ID` |
| `Access-Control-Max-Age` | Specifies how long a preflight result may be cached. | `Access-Control-Max-Age: 600` |

## 4. Security Headers

| Header | Purpose | Example |
|---|---|---|
| `Content-Security-Policy` | Restricts permitted sources for scripts, styles, images, and other resources. | `Content-Security-Policy: default-src 'self'` |
| `Permissions-Policy` | Controls access to selected browser features. | `Permissions-Policy: camera=(), microphone=()` |
| `Referrer-Policy` | Controls how much referrer information the browser sends. | `Referrer-Policy: strict-origin-when-cross-origin` |
| `Strict-Transport-Security` | Tells browsers to use HTTPS for the specified period. Send only over HTTPS. | `Strict-Transport-Security: max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | Prevents MIME-type sniffing. | `X-Content-Type-Options: nosniff` |
| `X-Frame-Options` | Controls whether a page may be embedded in a frame; CSP `frame-ancestors` is the more flexible modern control. | `X-Frame-Options: DENY` |

**Note:** `X-XSS-Protection` is obsolete and should not be relied on for security. Prefer a well-configured Content Security Policy and context-appropriate output encoding.

## 5. API and Application-Specific Headers

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

## 6. Practical Notes

- Header names are case-insensitive.
- Do not manually set `Host` in ordinary browser or HTTP-client code; the client generally manages it.
- `Content-Type` describes the body being sent; `Accept` describes the response formats the client prefers.
- `Authorization` and `Cookie` carry credentials or session data. Do not log their values.
- Use `Cache-Control: no-store` for sensitive responses that should not be stored by caches.
- CORS is enforced by browsers; it is not a replacement for server-side authentication or authorization.
- `Server` may disclose implementation details and is often omitted or minimized.
- `X-Requested-With: XMLHttpRequest` is a legacy convention, not a reliable security control.
- HTTP/2 and HTTP/3 have protocol-specific restrictions on connection-specific headers such as `Connection`, `Keep-Alive`, `Transfer-Encoding`, and `Upgrade`.

## 7. Example API Exchange

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
