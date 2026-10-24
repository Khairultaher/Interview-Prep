# SignalR with .NET 8 REST API — Viva Preparation Guide

A concise, step-by-step guide to building a .NET 8 REST API that sends real-time notifications to everyone, an individual user, or a group based on roles stored in a database. Includes common viva questions.

## 1. What is SignalR?

**ASP.NET Core SignalR** enables real-time, two-way communication between a server and connected clients. Instead of repeatedly polling a REST endpoint, the server can push updates to clients as events happen.

Typical use cases:
- Notifications and alerts
- Chat and messaging
- Live dashboards and progress updates
- Updates targeted at users, teams, or roles

SignalR negotiates a suitable transport, generally preferring WebSockets when available and falling back to supported alternatives when needed.

## 2. REST API vs SignalR

| REST API | SignalR |
|---|---|
| Usually request/response | Persistent real-time connection |
| Client asks for data | Server can push events to clients |
| Good for CRUD and business operations | Good for live updates and notifications |

They complement each other. A REST API can perform a business operation, save it to the database, and use SignalR to notify connected clients.

## 3. Create the project

```bash
dotnet new webapi -n SignalRNotifications
cd SignalRNotifications
```

In ASP.NET Core 8, the server-side SignalR APIs are included in the shared ASP.NET Core framework. A separate `Microsoft.AspNetCore.SignalR` package is generally not needed for a normal Web API project.

## 4. Create a hub

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

## 5. Register SignalR and map the hub

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

## 6. Send notifications from a REST controller

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

### Main targeting APIs

```csharp
await _hub.Clients.All.SendAsync("ReceiveNotification", message);
await _hub.Clients.User(userId).SendAsync("ReceiveNotification", message);
await _hub.Clients.Group(groupName).SendAsync("ReceiveNotification", message);
```

- `Clients.All`: every connected client.
- `Clients.User(userId)`: all active connections associated with that SignalR user identifier.
- `Clients.Group(groupName)`: connections currently in that group.

These calls do not guarantee delivery to disconnected users. Persist important notifications in a database if they must be available later.

## 7. Important: how `Clients.User(userId)` identifies a user

By default, SignalR uses the authenticated user's `ClaimTypes.NameIdentifier` claim as the user identifier. Ensure it matches the ID used by your application.

For JWT authentication, configure authentication and authorization in the normal ASP.NET Core pipeline and provide the access token when establishing the SignalR connection. Browser WebSocket/SSE connections may send the token in the `access_token` query parameter; restrict token extraction to the hub path, use HTTPS, and avoid logging tokens.

Do **not** maintain your own static `Dictionary<userId, connectionId>` for normal targeting. One user may have multiple browser tabs/devices, connections can reconnect, and static state does not work reliably across multiple servers. Prefer `Clients.User(userId)` with authenticated identity.

## 8. Add users to groups based on roles in the database

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

## 9. JavaScript client example

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

## 10. Connection and delivery concepts

- `Context.ConnectionId`: identifies one connection, not necessarily one person.
- One user can have several simultaneous connections.
- SignalR group membership is connection-based and must be re-established when a connection reconnects.
- `OnConnectedAsync()` and `OnDisconnectedAsync()` are lifecycle hooks; avoid relying on in-memory connection tracking in a multi-server deployment.
- For scale-out, consider Azure SignalR Service or a supported Redis backplane, depending on the hosting environment.
- SignalR is not a durable message queue. Store notifications that must survive disconnects and let clients retrieve unread notifications from a REST endpoint.

## 11. Viva questions and short answers

### Q1. What is SignalR?
SignalR is an ASP.NET Core library for real-time, two-way communication. It lets the server push events to connected clients.

### Q2. How is SignalR different from REST?
REST is commonly request/response; SignalR maintains a real-time connection so the server can push updates without waiting for another client request.

### Q3. What is a Hub?
A Hub is a high-level abstraction for receiving client calls and sending messages to clients, users, or groups.

### Q4. What is the difference between `Clients.All`, `Clients.User`, and `Clients.Group`?
`All` targets all connected clients, `User` targets all connections for one user identifier, and `Group` targets connections that belong to a group.

### Q5. How does SignalR know the user ID?
By default, it uses the authenticated principal's `NameIdentifier` claim. Authentication and claim mapping must be configured consistently.

### Q6. Can groups be based on database roles?
Yes. After authenticating the connection, the server can load roles from the database and add that connection to server-controlled role groups.

### Q7. Is a SignalR group a security feature?
No. A group is a message-routing mechanism. The server must authorize group membership and notification-sending operations.

### Q8. Why use `IHubContext` in a controller?
It allows code outside the Hub—such as a REST controller or background service—to send messages to connected clients.

### Q9. What happens if the user is offline?
A live SignalR message is not delivered to a disconnected client. Store important notifications persistently and provide an API to retrieve them later.

### Q10. What happens when a user has multiple tabs open?
Each tab usually has its own connection. `Clients.User(userId)` targets all active connections associated with that user identifier.

### Q11. How do you scale SignalR?
Use a scale-out option such as Azure SignalR Service or a Redis backplane where appropriate, and avoid relying on static in-memory connection maps.

### Q12. How would you implement a production notification flow?
Authenticate and authorize the request, perform and persist the business operation, commit the transaction, then publish a SignalR event. For reliable delivery, use persisted notifications and consider an outbox/background-worker pattern.

## 12. Suggested viva demo sequence

1. Run the .NET 8 API and connect two browser clients.
2. Call `POST /api/notifications/all` and show both clients receive the event.
3. Authenticate clients with different user IDs and test `Clients.User(...)`.
4. Add users to a server-authorized group and test `Clients.Group(...)`.
5. Explain why database-backed roles, authorization, and persistent storage matter in production.

## Quick revision

Remember these five points:

1. **Hub** = real-time communication endpoint.
2. **`IHubContext`** = send SignalR messages from controllers/services.
3. **`Clients.All/User/Group`** = choose recipients.
4. **Authentication + authorization** = trustworthy user identity and permissions.
5. **Database persistence** = important notifications remain available when clients are offline.
