# Senior .NET Interview Guide: Async, Threading, Locking, and Distributed Systems

> A consolidated reference for senior .NET developer and solution architect interviews. Repeated explanations have been removed; related topics are grouped together.

## 1. Core mental models

- **`async`/`await` is not automatically multithreading.** It is a way to compose asynchronous operations without blocking a thread while an operation is incomplete.
- **I/O-bound work:** use genuinely asynchronous APIs, such as `HttpClient.GetAsync`, EF Core async methods, and `File.ReadAllTextAsync`.
- **CPU-bound work:** consumes CPU while it runs. `async` does not make the calculation faster or remove its CPU cost.
- **`Task` represents an operation**, not necessarily a dedicated thread.
- ASP.NET Core does not normally have a custom request `SynchronizationContext`. Blocking async work is still harmful because it occupies ThreadPool threads and can cause starvation.
- Prefer async end-to-end for I/O. Do not add `Task.Run` merely to make synchronous I/O look asynchronous.

## 2. `Task`, `async`/`await`, `Task.Run`, and `Task.FromResult`

### `async`/`await`: compose asynchronous work

```csharp
[HttpGet]
public async Task<IActionResult> Get(CancellationToken cancellationToken)
{
    var data = await _repository.GetAsync(cancellationToken);
    return Ok(data);
}
```

When the awaited task is incomplete, the method suspends and its current thread can do other work. When the operation completes, the method's continuation runs later, possibly on another ThreadPool thread.

### `Task.Run`: queue CPU work to the ThreadPool

```csharp
[HttpGet("calculation")]
public async Task<IActionResult> GetCalculation()
{
    var result = await Task.Run(() => HeavyCpuCalculation());
    return Ok(result);
}
```

This can be reasonable for a **short, bounded CPU calculation** when the result is needed in the same request and measurements justify moving the calculation off the current request-execution thread. It is not a default ASP.NET Core performance technique: `Task.Run` still consumes ThreadPool capacity and CPU. Under high concurrency, it can worsen contention and latency.

For heavy, lengthy, or high-volume calculations, prefer a bounded worker pool, durable queue, or separately scalable compute service. Return `202 Accepted` with a job ID when the result will be available later.

Do not use `Task.Run` for ordinary asynchronous I/O, and do not fire-and-forget it from a controller:

```csharp
Task.Run(() => DoWork()); // unsafe fire-and-forget in a web request
```

The work may be lost on process shutdown, exceptions may go unobserved, and the application has no durable retry or coordination guarantee.

### `Task.FromResult`: return an already-known value as a completed task

```csharp
public Task<bool> IsFeatureEnabledAsync()
{
    return Task.FromResult(true);
}
```

Use it when an API contract requires `Task<T>` but the result is already available synchronously, such as a cache hit, simple in-memory result, or test mock. It **does not** run work asynchronously.

Incorrect:

```csharp
return Task.FromResult(DoBlockingWork()); // DoBlockingWork runs before Task.FromResult
```

Use a genuinely asynchronous API for I/O instead.

### `Task.Run` vs `Task.FromResult`

| Need | Appropriate choice |
|---|---|
| Await database/network/file I/O | Async API + `await` |
| Return a value already available synchronously from a `Task<T>` method | `Task.FromResult(value)` |
| Run CPU work on a ThreadPool thread | `Task.Run`, only when justified |
| Durable, long-running background job | Queue + worker/job system |
| Make blocking I/O “async” | Neither |

## 3. Synchronization primitives: `lock`, `Monitor`, `Mutex`, `Semaphore`, `SemaphoreSlim`

| Primitive | Scope | Concurrent entrants | Async waiting | Main purpose |
|---|---|---:|---|---|
| `lock` | One process | 1 | No `await` inside the lock | Protect a short in-memory critical section |
| `Monitor` | One process | 1 per monitor at a time | No async wait API | Mutual exclusion plus condition signaling |
| `Mutex` | Can coordinate processes when named appropriately | 1 owner | `WaitOne` blocks a thread | Cross-process mutual exclusion |
| `Semaphore` | Can be named for cross-process use on supported platforms | Up to configured count | `WaitOne` blocks a thread | Limit concurrent access to N slots |
| `SemaphoreSlim` | One process | Up to configured count | `WaitAsync` | Async-compatible concurrency limiting or mutual exclusion when count is 1 |

### `lock`: one thread at a time in a critical section

```csharp
private readonly object _gate = new();
private int _counter;

public void Increment()
{
    lock (_gate)
    {
        _counter++;
    }
}
```

- If the gate is free, the thread enters.
- If another thread owns it, the contender blocks until it can enter.
- The lock is released automatically when control exits the block, including when an exception is thrown.
- Use a private dedicated object; do not lock on `this`, a string, or a public object.
- Keep the critical section short.
- `lock` is reentrant for the owning thread.
- Do not use `await` inside a `lock` block.

### `Monitor`: `lock` plus explicit coordination

`lock (gate)` is conceptually implemented using `Monitor.Enter` and `Monitor.Exit` in a `try/finally`. Use `Monitor` directly when you specifically need `TryEnter`, `Wait`, `Pulse`, or `PulseAll`.

```csharp
private readonly object _gate = new();
private bool _dataAvailable;

public void WaitForData()
{
    lock (_gate)
    {
        while (!_dataAvailable)
        {
            Monitor.Wait(_gate); // releases gate and waits
        }

        // The condition is true and this thread owns gate again.
        ConsumeData();
    }
}

public void PublishData()
{
    lock (_gate)
    {
        ProduceData();
        _dataAvailable = true;
        Monitor.Pulse(_gate); // wake one waiter
    }
}
```

Important details:

- `Monitor.Wait(gate)` requires the current thread to own the monitor. It releases the monitor and waits.
- `Pulse(gate)` moves one waiting thread toward being eligible to reacquire the monitor; it does **not** immediately transfer ownership.
- `PulseAll(gate)` makes all waiting threads eligible to compete to reacquire the monitor.
- A pulsed thread cannot continue past `Wait` until it reacquires the monitor.
- Always check the condition in a `while`, not an `if`: the condition may no longer be true when the thread reacquires the monitor.
- Signal after updating the shared state, while holding the same monitor.

### `Mutex`: one owner across processes

```csharp
using var mutex = new Mutex(false, @"Global\MyApplicationJob");

bool ownsMutex = false;
try
{
    try
    {
        ownsMutex = mutex.WaitOne(TimeSpan.FromSeconds(5));
    }
    catch (AbandonedMutexException)
    {
        // The previous owner died. The caller now owns the mutex,
        // but must validate/recover the protected resource.
        ownsMutex = true;
    }

    if (!ownsMutex)
        throw new TimeoutException("Could not acquire the mutex.");

    DoProtectedWork();
}
finally
{
    if (ownsMutex)
        mutex.ReleaseMutex();
}
```

A named mutex can coordinate processes on supported operating systems. `WaitOne` blocks a thread, and ownership is thread-affine: the owning thread must release it. An abandoned mutex indicates that the previous owner terminated without releasing it; protected data may be inconsistent and needs recovery. For web applications, do not use a mutex as a substitute for durable distributed job coordination.

### `SemaphoreSlim`: allow up to N concurrent entrants

```csharp
private readonly SemaphoreSlim _slots = new(initialCount: 3, maxCount: 3);

public async Task ProcessAsync(CancellationToken cancellationToken)
{
    await _slots.WaitAsync(cancellationToken);
    try
    {
        await CallDownstreamServiceAsync(cancellationToken);
    }
    finally
    {
        _slots.Release();
    }
}
```

- At most three operations can hold a slot concurrently.
- When the count reaches zero, additional callers wait asynchronously.
- `Release` returns a slot and allows a waiter to proceed.
- Always release in `finally`.
- The count limits concurrency only within this `SemaphoreSlim` instance/process. It does not impose a cluster-wide limit.
- `SemaphoreSlim(1, 1)` can serialize async operations, but it is not owner-thread-affine in the way `Mutex` is.

### How do you allow more than one thread into a protected region?

Use a semaphore with a count greater than one, such as `SemaphoreSlim(5, 5)`, to permit up to five concurrent entrants. This is a **concurrency limit**, not exclusive mutual exclusion. Ensure the shared resource is safe for concurrent access; a semaphore does not make arbitrary shared state thread-safe.

### Quick selection guide

| Requirement | Choose |
|---|---|
| Protect a short in-memory update | `lock` |
| Wait for a condition and signal waiters | `Monitor.Wait` / `Pulse` / `PulseAll` |
| Coordinate a named mutex across processes on one machine | `Mutex` |
| Limit async operations in one process to N | `SemaphoreSlim(N, N)` |
| Coordinate ownership across multiple servers | Database/queue/distributed coordination pattern |
| Make a long-running job durable | Durable queue or job store, not a lock |

## 4. Deadlocks and ThreadPool starvation

### Lock-order deadlock

Thread A acquires lock A and waits for B; thread B acquires B and waits for A. Prevent this by acquiring multiple locks in a consistent global order, keeping lock scopes short, and avoiding unnecessary nested locks.

### Sync-over-async

```csharp
var result = GetDataAsync().Result; // blocks a ThreadPool thread
```

This may deadlock in environments with a synchronization context and can cause ThreadPool starvation in ASP.NET Core under load. Prefer `await GetDataAsync()` end-to-end. `ConfigureAwait(false)` is useful in many library contexts to avoid capturing a context, but it does not make blocking safe.

### ThreadPool starvation

Common causes include `.Result`, `.Wait()`, `GetAwaiter().GetResult()`, `Thread.Sleep`, synchronous blocking I/O, and excessive work queued through `Task.Run`.

Typical symptoms: high request latency, growing queues, low-to-moderate CPU despite requests hanging, and slow recovery. Investigate traces, thread-pool counters, blocked stacks, dependency latency, and load tests. Fix the blocking path rather than simply increasing thread counts.

## 5. Async in the ASP.NET Core request pipeline

Typical flow:

1. Kestrel accepts and parses the HTTP request.
2. Middleware executes in order.
3. Routing, authentication/authorization, model binding, filters, and the endpoint execute as applicable.
4. Async I/O is awaited without holding a ThreadPool thread for the entire wait.
5. The response is written to the client.

Example middleware:

```csharp
app.Use(async (context, next) =>
{
    // Before downstream middleware
    await next(context);
    // After downstream middleware
});
```

Best practices:

- Use async I/O APIs end-to-end.
- Pass `CancellationToken` (often `HttpContext.RequestAborted`) to cancellable operations.
- Avoid `.Result`, `.Wait()`, synchronous I/O, and unnecessary `Task.Run`.
- Do not start untracked fire-and-forget work in a controller.
- Use `BackgroundService` for in-process hosted work, but remember that in-process work alone is not durable across restarts and is not a cluster-wide singleton.
- Move durable or lengthy work to a queue and worker.

## 6. Distributed async system design

### Durable background processing

Recommended flow:

```text
HTTP request
   |
   v
Validate + persist/enqueue job
   |
   v
Return 202 Accepted + job ID
   |
   v
Durable queue (e.g., Azure Service Bus or RabbitMQ)
   |
   v
Worker(s) -> idempotent processing -> database/result store
   |
   v
Client polls job status or receives a notification
```

Design for at-least-once delivery: messages may be redelivered. Use idempotency keys, database unique constraints, bounded retries with exponential backoff and jitter, dead-letter queues, timeouts, and observability. Do not claim exactly-once execution unless the full end-to-end guarantees genuinely support it.

### Distributed locks and alternatives

`lock`, `Monitor`, and `SemaphoreSlim` coordinate only within a process. A named `Mutex` can coordinate processes on a supported single-machine scope, but it is not a general multi-server lock.

For cross-server coordination, consider:

- **Database claim/lease:** atomically claim a job row using a conditional update or transactional locking; use lease expiry and recovery.
- **Message queue:** competing consumers, acknowledgments, retries, and dead-lettering often avoid the need for a separate lock.
- **Distributed lease/lock:** use a well-understood implementation with expiry, ownership-checked release, and fencing tokens where stale owners could corrupt data.

A lock alone does not make a job durable or guarantee exactly-once effects. Use idempotency and transactional boundaries.

### Resilience patterns

- **Timeout:** bound how long a dependency call can take.
- **Retry:** retry only transient failures, with a limit and backoff.
- **Circuit breaker:** stop repeatedly calling a dependency that is failing.
- **Bulkhead/concurrency limit:** isolate resources and cap concurrent work.
- **Backpressure:** reject, defer, or slow new work when capacity is exhausted.
- **Saga:** coordinate multi-service business workflows with compensating actions when necessary.
- **Observability:** propagate correlation/trace IDs and capture structured logs, metrics, traces, queue depth, retries, and processing duration.

## 7. MCQ and scenario exam

### Multiple-choice questions

**Q1. What does `async` do by itself?**

A. Creates a new thread  
B. Enables asynchronous method composition and `await`  
C. Makes CPU work parallel  
D. Guarantees better performance  

**Answer: B.** It does not automatically create a thread or parallelize CPU work.

**Q2. What does `await` do when the task is incomplete?**

A. Always blocks the current thread  
B. Suspends the async method and arranges a continuation  
C. Always creates a new thread  
D. Cancels the operation  

**Answer: B.** The current thread can return to other work.

**Q3. What does `Task.FromResult(5)` do?**

A. Runs calculation 5 on another thread  
B. Returns an already-completed `Task<int>` containing 5  
C. Waits five seconds  
D. Starts a background job  

**Answer: B.**

**Q4. Which primitive can asynchronously wait for a concurrency slot?**

A. `lock`  
B. `Monitor.Wait`  
C. `SemaphoreSlim.WaitAsync`  
D. `Mutex.WaitOne`  

**Answer: C.**

**Q5. What does `Monitor.Pulse` guarantee?**

A. The waiter immediately owns the monitor  
B. One waiting thread becomes eligible to compete to reacquire the monitor  
C. All waiting threads enter the critical section  
D. The monitor is released permanently  

**Answer: B.**

**Q6. How can up to five operations enter a region concurrently within one process?**

A. `lock`  
B. `SemaphoreSlim(1, 1)`  
C. `SemaphoreSlim(5, 5)`  
D. `Monitor.PulseAll`  

**Answer: C**, provided the protected resource supports concurrent access.

**Q7. Why is `.Result` dangerous in a web request?**

A. It always causes a deadlock  
B. It blocks a thread and can cause starvation; deadlock is also possible in some contexts  
C. It always starts another process  
D. It cancels the task  

**Answer: B.**

**Q8. Which option is suitable for durable background work across several application instances?**

A. `Task.Run` in a controller  
B. A static boolean  
C. A durable queue with worker consumers  
D. A process-local `lock`  

**Answer: C.**

**Q9. What is the main reason for idempotent message processing?**

A. To make every message execute faster  
B. To safely handle retries and duplicate delivery  
C. To avoid using a database  
D. To guarantee messages are never redelivered  

**Answer: B.**

**Q10. Is `Task.Run(() => HeavyCpuCalculation())` always wrong in a controller?**

A. Yes, in every possible case  
B. No; it can be a measured, bounded compromise, but it consumes ThreadPool/CPU capacity and can hurt under load  
C. It makes the work durable  
D. It makes the work run on another server  

**Answer: B.**

### Scenario questions

**Scenario 1: An API becomes slow under load, with many `.Result` calls and relatively low CPU.**

Expected answer: suspect sync-over-async and ThreadPool starvation. Trace blocking stacks and dependency latency; use async APIs end-to-end and remove blocking calls.

**Scenario 2: Five downstream requests may run concurrently, but no more than five.**

Expected answer: use a shared `SemaphoreSlim(5, 5)` in that process, `WaitAsync` with cancellation, and `Release` in `finally`. Use a distributed limiter or gateway if the limit must apply across all instances.

**Scenario 3: Consumers wait until a producer publishes data.**

Expected answer: use a condition protected by the same monitor, check it in a `while` loop, call `Monitor.Wait` while false, and update the condition before `Pulse`/`PulseAll`.

**Scenario 4: A scheduled job runs on two application replicas.**

Expected answer: prefer queue-based competing consumers or atomic database job claiming. Add idempotency, lease recovery, and retries. A local `lock` cannot coordinate replicas.

**Scenario 5: A controller performs a CPU-intensive report calculation that takes several seconds.**

Expected answer: use a durable queue and worker or separately scalable compute service, return `202 Accepted` with a job ID, and expose status/results. Use `Task.Run` only for bounded, measured cases where immediate response is necessary.

## 8. Code review checklist

### Async correctness
- [ ] No `.Result`, `.Wait()`, or `GetAwaiter().GetResult()` in request paths without a compelling, reviewed reason.
- [ ] No `async void` except event handlers.
- [ ] I/O uses true asynchronous APIs.
- [ ] No unnecessary `Task.Run` around I/O or trivial synchronous code.
- [ ] CPU-heavy work is measured and bounded or offloaded.
- [ ] Cancellation tokens are propagated where supported.
- [ ] Exceptions from background work are observed and handled.

### Locking correctness
- [ ] Lock object is private and dedicated.
- [ ] Critical section is short; no `await` inside `lock`.
- [ ] Nested locks follow a consistent acquisition order.
- [ ] Shared state is actually protected by the same synchronization strategy.
- [ ] `Monitor.Wait` checks its condition in a `while` loop.
- [ ] `Pulse` versus `PulseAll` is chosen intentionally.
- [ ] `SemaphoreSlim.Release()` is in `finally`; configured capacity is correct.
- [ ] Mutex ownership, timeout, abandoned-mutex recovery, and release are handled correctly.

### Distributed reliability
- [ ] Background jobs are durable if they must survive restarts.
- [ ] Duplicate delivery/retries are safe through idempotency.
- [ ] Retry policy is bounded and distinguishes transient from permanent failures.
- [ ] Queue depth, failures, latency, and dead-letter messages are observable.
- [ ] Any cluster-wide limit/lock is implemented with a distributed-aware mechanism.
- [ ] Long-running jobs have status, cancellation/timeout strategy, and recovery behavior.

## 9. Interview summary

> “I use async/await to avoid blocking during I/O, not to disguise CPU work. I use `lock` for short in-process critical sections, `Monitor` for condition signaling, `Mutex` for suitable cross-process single-machine coordination, and `SemaphoreSlim` for async concurrency limits. For durable multi-instance background processing, I use queues or job stores, idempotent handlers, bounded retries, and observability.”
