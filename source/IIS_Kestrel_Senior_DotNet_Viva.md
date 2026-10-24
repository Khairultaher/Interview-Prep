# IIS, Application Pools, Kestrel, and High-Traffic ASP.NET Core
## Senior .NET Developer Viva Preparation

> Goal: explain the hosting architecture accurately, reason about concurrency and bottlenecks, and describe how you would scale a real production API.

## 1. The big picture

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

## 2. What an IIS Application Pool actually is

An **application pool is an IIS configuration and isolation boundary**, not a request-processing thread and not a load balancer. It groups one or more IIS applications under a worker-process configuration.

Its main responsibilities are:

- **Process isolation:** applications assigned to different pools can run in separate worker processes.
- **Identity and permissions:** configure the Windows identity used by the worker process, commonly `ApplicationPoolIdentity`.
- **Lifecycle management:** start, stop, and recycle worker processes.
- **Reliability controls:** configure rapid-fail protection and other process-model settings.
- **Resource and queue configuration:** configure such settings as queue length and recycling behavior.

### Application pool vs worker process

- **Application pool:** configuration boundary in IIS.
- **Worker process (`w3wp.exe`):** operating-system process that executes IIS-hosted workloads.
- **Worker-process count:** an application pool normally has one worker process, but can be configured as a *web garden* with multiple worker processes.
- **Application:** a website or application mapped to an application pool.

Multiple applications can share a pool, but then they also share its process boundary and lifecycle. For stronger isolation, separate pools are often preferable.

### A useful analogy

Think of the application pool as the **rules and supervision for a workspace**. The worker process is the actual running program inside that workspace. The pool configures how the process starts, under which identity it runs, and when it is recycled.

## 3. How IIS receives and routes a request

1. The client opens an HTTP/HTTPS connection.
2. On Windows, **HTTP.sys** receives HTTP traffic in kernel mode and can queue requests for the relevant URL group.
3. IIS site bindings and request configuration determine which site/application should handle the URL.
4. The request is associated with the relevant application pool.
5. If necessary, WAS (Windows Process Activation Service) activates the pool and worker process.
6. IIS modules and the applicable hosting integration process or forward the request.
7. The application generates a response; it returns through the hosting path to IIS/HTTP.sys and then to the client.

Do not say that IIS creates a new thread for every request. That is not how modern asynchronous ASP.NET Core request handling should be understood.

## 4. How multiple requests are handled

A single worker process can handle many concurrent requests. Concurrency is not the same as executing every request simultaneously on a separate CPU core.

ASP.NET Core uses asynchronous I/O and the .NET ThreadPool to use resources efficiently:

- When code awaits non-blocking I/O, such as an asynchronous database or HTTP call, the request can wait without holding a dedicated thread for the whole wait.
- When the operation completes, continuation work can run on a ThreadPool thread.
- CPU-bound code still consumes CPU time while it executes.
- Blocking calls (`.Result`, `.Wait()`, synchronous I/O, long locks) can occupy threads and reduce throughput.
- Every request still consumes resources: memory, sockets, database connections, CPU, and downstream capacity.

### Example: 1,000 concurrent requests

Suppose 1,000 requests arrive and most spend time awaiting SQL Server:

- The application does not need 1,000 dedicated threads sitting idle while SQL responds.
- Asynchronous database calls allow threads to return to the pool while I/O is pending.
- When SQL responses arrive, continuations run and finish the requests.
- If SQL Server can only sustain 200 operations at once, adding more web threads will not make the database faster. It may make the queue and timeouts worse.

**Interview line:** “I optimize for bounded resource usage and end-to-end throughput, not simply for the largest possible thread or connection count.”

## 5. Queueing, backpressure, and overload

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

## 6. IIS with Kestrel: the important hosting models

### Out-of-process hosting

- IIS receives the external request.
- ANCM forwards it to the ASP.NET Core application running in a separate process.
- Kestrel listens on the configured local endpoint.
- The response travels back through ANCM/IIS.

ANCM participates in process startup and management. The exact forwarding transport and configuration depend on the installed hosting bundle and hosting setup; do not overstate that it always uses a particular port or transport.

### In-process hosting

- The ASP.NET Core application runs inside `w3wp.exe`.
- IIS server integration handles requests within that process.
- There is no separate Kestrel process receiving proxied requests in this mode.

### Why use IIS?

IIS provides Windows hosting integration, site bindings, TLS configuration, request filtering, logging, process activation, and other hosting controls. Kestrel is the cross-platform web server used by ASP.NET Core and can also be hosted directly behind another reverse proxy or exposed directly when the deployment architecture and security controls support it.

**Avoid saying:** “Kestrel must always sit behind IIS/Nginx” or “Kestrel has no security.” Kestrel can be used directly; whether a reverse proxy is appropriate depends on the environment and required edge features.

## 7. What happens when an application pool recycles?

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

## 8. Web garden vs web farm

| Concept | Meaning | Main concern |
|---|---|---|
| Web garden | Multiple worker processes for one application pool on one server | More process-local state and complexity; not automatically faster |
| Web farm | Multiple application instances across servers/VMs/containers | Load balancing, health, shared state, deployment, and resilience |

A web garden is not the default solution for high traffic. It can help specific workloads after testing, but may hurt cache locality, complicate session state, and increase memory usage. For horizontal scale and fault isolation, a web farm or multiple app instances behind a load balancer is more common.

## 9. How would you handle a million requests?

First clarify what “a million requests” means:

- One million requests per day is about **11.6 requests/second on average**.
- One million requests per hour is about **278 requests/second on average**.
- One million requests per minute is about **16,667 requests/second on average**.

These are averages. Real systems have peaks, uneven traffic, and expensive versus cheap endpoints. Capacity planning must use peak rate, latency targets, payload sizes, and workload mix.

### A senior-level approach

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

## 10. Database connection pooling: a frequent trap

ADO.NET providers such as `Microsoft.Data.SqlClient` pool physical connections. Opening and disposing a logical connection per operation is normal when using `using`/`await using`; it usually returns the connection to the pool rather than closing the physical connection each time.

- Keep database operations short and asynchronous.
- Dispose connections reliably.
- Avoid holding a connection while doing unrelated work.
- Tune pool limits only after checking SQL Server capacity and pool exhaustion.
- Remember that adding application instances multiplies potential database connection demand.

If 20 app instances each can open 100 SQL connections, the database may face up to roughly 2,000 connections across those pools. Scaling the API without planning database capacity can move the bottleneck downstream.

## 11. Performance and configuration: measure before tuning

Potentially useful controls include:
- IIS application-pool queue and recycling settings.
- Always-running/preload options where supported and appropriate.
- ASP.NET Core request limits, rate limiting, and timeouts.
- Response compression and caching for suitable responses.
- .NET GC and runtime configuration appropriate to the workload.
- Connection-pool and database settings.

Do not blindly copy settings such as “maximum requests per CPU,” increase queue lengths, or enable a web garden because a blog recommends them. Defaults and available settings differ by IIS/ASP.NET version and hosting model. Establish a baseline, change one thing at a time, and validate under load.

## 12. Useful diagnostic signals

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

## 13. Senior interview questions and answer outlines

### Q1. What is the responsibility of an IIS application pool?
It defines the process-isolation and lifecycle configuration for IIS-hosted applications: worker-process identity, startup/shutdown/recycling behavior, failure protection, and relevant queue/process settings. It does not itself execute each request; the worker process and hosting pipeline do that.

### Q2. Does IIS create one thread per request?
No. Modern ASP.NET Core uses asynchronous I/O and the .NET ThreadPool. Requests may await I/O without holding a thread for the entire wait. CPU-bound work still requires CPU execution time.

### Q3. If we increase the IIS queue length, will throughput increase?
Not necessarily. A larger queue permits more waiting and may increase latency and memory pressure. It does not increase the app or database's sustainable processing rate.

### Q4. Should I enable a web garden to handle more requests?
Not automatically. Benchmark first. Multiple worker processes can help some workloads but add memory and process-local state complexity. Horizontal scaling behind a load balancer is often the more flexible option.

### Q5. Why might adding API instances make the database slower?
Each instance can create additional concurrent database work and its own connection pools. The aggregate load may exceed SQL Server capacity, increasing waits, blocking, and timeouts.

### Q6. How would you diagnose slow API responses?
Start with p95/p99 latency and distributed traces, separate application time from dependency time, inspect CPU/memory/GC and queueing, then investigate SQL query plans, waits, and connection-pool saturation. Change settings only after identifying the bottleneck.

### Q7. What is the difference between in-process and out-of-process IIS hosting?
In-process runs the ASP.NET Core app inside `w3wp.exe` using IIS integration. Out-of-process runs the app in a separate process with Kestrel and ANCM forwarding requests.

### Q8. How do you avoid losing user state during recycling or scaling?
Keep durable/shared state outside a single worker process, use distributed storage/cache where appropriate, make instances replaceable, and avoid relying on in-memory session state for correctness.

## 14. A concise answer to memorize

“An IIS application pool is an isolation and lifecycle boundary, while `w3wp.exe` is the worker process. HTTP.sys receives HTTP traffic, IIS maps it to a site and application pool, and the configured hosting model invokes the ASP.NET Core app. With out-of-process hosting, ANCM forwards requests to the Kestrel process; with in-process hosting, the app runs inside `w3wp.exe`. ASP.NET Core handles concurrency efficiently through asynchronous I/O and the ThreadPool, but throughput is limited by the slowest constrained resource, often the database or a downstream API. To scale, I measure realistic peak traffic, identify bottlenecks, use caching and efficient queries, bound concurrency and queues, apply timeouts and rate limits, and scale healthy instances behind a load balancer. I validate every change with load tests and production telemetry rather than blindly increasing IIS limits.”

## 15. Recommended practice exercise

Build a small ASP.NET Core API with:
- One lightweight endpoint and one SQL-backed endpoint.
- Asynchronous database calls.
- Structured logging and tracing.
- A bounded rate limit.
- A load test with gradually increasing concurrency.

Record throughput, p95/p99 latency, error rate, CPU, memory, ThreadPool behavior, and SQL latency. Then introduce an intentionally slow dependency and observe how queues and timeouts behave. This gives you concrete examples to discuss in the viva.

---

## Accuracy reminders

- An application pool is not a load balancer.
- A request queue is not extra processing capacity.
- Async improves resource utilization during I/O waits; it does not make CPU work free.
- A million-request target is meaningless without a time window, peak profile, and latency/error objectives.
- Kestrel can run directly; a reverse proxy is an architectural choice, not an absolute requirement.
- Always distinguish IIS in-process and out-of-process ASP.NET Core hosting.
