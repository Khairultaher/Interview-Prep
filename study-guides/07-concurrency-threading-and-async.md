# Concurrency, Threading, and Async

This guide combines the distinct source materials below. Original explanations, examples, and interview questions are retained; exact duplicate source files are excluded and documented in the library index.

## Included source materials

- [csharp-concurrency-span-stack-queue.md](#csharp-concurrency-span-stack-queue-md)
- [csharp-task-parallelism-interview-notes.md](#csharp-task-parallelism-interview-notes-md)
- [dotnet-threading-async-await-senior-viva.md](#dotnet-threading-async-await-senior-viva-md)
---

<a id="csharp-concurrency-span-stack-queue-md"></a>

## Source: C# Interview Notes: Synchronization, Span, Stack, and Queue

*Imported from `csharp-concurrency-span-stack-queue.md`.*

### C# Interview Notes: Synchronization, Span, Stack, and Queue

#### 1. `lock`, `Monitor`, and `Interlocked`

These APIs help make code thread-safe, but they solve different problems.

##### `lock` — protect a critical section

Use `lock` when multiple operations must run as one protected unit.

```csharp
private readonly object _gate = new();
private int _balance;

public void Deposit(int amount)
{
    lock (_gate)
    {
        _balance += amount;
    }
}
```

- Only one thread at a time can enter a `lock` using the same lock object.
- The lock is released automatically, including when an exception occurs.
- Lock a private object, not `this`, a `Type`, or a publicly accessible object.

##### `Monitor` — more control over locking

`lock` is the preferred syntax for ordinary locking. `Monitor` is useful when you need features such as a timeout or condition signaling with `Wait`/`Pulse`.

```csharp
using System;
using System.Threading;

private readonly object _gate = new();

public void TryUpdate()
{
    if (!Monitor.TryEnter(_gate, TimeSpan.FromSeconds(2)))
    {
        Console.WriteLine("Could not acquire the lock in time.");
        return;
    }

    try
    {
        Console.WriteLine("Updating safely.");
    }
    finally
    {
        Monitor.Exit(_gate);
    }
}
```

Always call `Monitor.Exit` only after successfully entering, and place it in `finally`.

##### `Interlocked` — atomic operations on individual values

Use it for simple atomic updates such as incrementing a counter. It does not protect a larger multi-step operation.

```csharp
using System.Threading;

private int _requestCount;

public void RecordRequest()
{
    Interlocked.Increment(ref _requestCount);
}
```

##### Quick comparison

| Need | Best fit |
|---|---|
| Protect a block of related operations | `lock` |
| Lock timeout or `Wait`/`Pulse` signaling | `Monitor` |
| Atomic increment, exchange, or compare-and-swap | `Interlocked` |

**Rule of thumb:** start with `lock` for shared mutable state; use `Interlocked` for a genuinely simple atomic operation; use `Monitor` when its additional control is needed.

---

#### 2. `Span<T>` and `Slice()`

`Span<T>` is a lightweight view over contiguous memory. It lets you access a portion of an array or other supported memory without copying the elements.

##### Slice an array

```csharp
int[] numbers = { 10, 20, 30, 40, 50 };

Span<int> span = numbers;
Span<int> middle = span.Slice(1, 3); // 20, 30, 40

middle[0] = 99;

Console.WriteLine(string.Join(", ", numbers));
// 10, 99, 30, 40, 50
```

`Slice(start, length)` uses a **zero-based start index** and a **number of elements**, not an ending index. The slice refers to the original memory, so changes to its elements affect the array.

You can also use range syntax:

```csharp
Span<int> middle = span[1..4]; // indices 1, 2, and 3
```

##### Slice a string without immediately creating a substring

Strings are immutable, so use `ReadOnlySpan<char>` to view their characters.

```csharp
string text = "Hello, World!";
ReadOnlySpan<char> word = text.AsSpan(7, 5);

Console.WriteLine(word.ToString()); // World
```

Calling `ToString()` creates a string. If you can process the span directly, you can avoid that allocation.

##### Important limitations

- A slice must stay within the source span's bounds.
- `Span<T>` and `ReadOnlySpan<T>` are `ref struct` types; they cannot be stored in ordinary class fields or used across `await`/`yield` boundaries in the usual way.
- `ReadOnlySpan<T>` prevents writes through that view; it does not necessarily make the underlying data immutable through other references.

---

#### 3. Stack and Queue in C#

A **stack** is LIFO (last in, first out); a **queue** is FIFO (first in, first out). Use the built-in generic collections unless you specifically need to implement the data structures yourself.

##### Built-in stack

```csharp
using System;
using System.Collections.Generic;

var stack = new Stack<int>();

stack.Push(10);
stack.Push(20);
stack.Push(30);

Console.WriteLine(stack.Pop());  // 30
Console.WriteLine(stack.Peek()); // 20
Console.WriteLine(stack.Count);  // 2
```

- `Push`: add to the top.
- `Pop`: remove and return the top item.
- `Peek`: read the top item without removing it.

##### Built-in queue

```csharp
using System;
using System.Collections.Generic;

var queue = new Queue<int>();

queue.Enqueue(10);
queue.Enqueue(20);
queue.Enqueue(30);

Console.WriteLine(queue.Dequeue()); // 10
Console.WriteLine(queue.Peek());    // 20
Console.WriteLine(queue.Count);     // 2
```

- `Enqueue`: add to the back.
- `Dequeue`: remove and return the front item.
- `Peek`: read the front item without removing it.

`Pop()` on an empty stack and `Dequeue()` on an empty queue throw `InvalidOperationException`. Check `Count`, or use `TryPop` / `TryDequeue` where available in your target .NET version.

##### Custom implementations with a linked list

The following implementations demonstrate the underlying logic. Each operation that adds or removes an item takes **O(1)** time.

```csharp
using System;

public sealed class CustomStack<T>
{
    private sealed class Node
    {
        public T Value { get; }
        public Node? Next { get; set; }

        public Node(T value) => Value = value;
    }

    private Node? _top;
    public int Count { get; private set; }

    public void Push(T item)
    {
        _top = new Node(item) { Next = _top };
        Count++;
    }

    public T Pop()
    {
        if (_top is null)
            throw new InvalidOperationException("Stack is empty.");

        T value = _top.Value;
        _top = _top.Next;
        Count--;
        return value;
    }

    public T Peek() =>
        _top is null
            ? throw new InvalidOperationException("Stack is empty.")
            : _top.Value;
}

public sealed class CustomQueue<T>
{
    private sealed class Node
    {
        public T Value { get; }
        public Node? Next { get; set; }

        public Node(T value) => Value = value;
    }

    private Node? _front;
    private Node? _rear;
    public int Count { get; private set; }

    public void Enqueue(T item)
    {
        var node = new Node(item);

        if (_rear is null)
            _front = _rear = node;
        else
        {
            _rear.Next = node;
            _rear = node;
        }

        Count++;
    }

    public T Dequeue()
    {
        if (_front is null)
            throw new InvalidOperationException("Queue is empty.");

        T value = _front.Value;
        _front = _front.Next;

        if (_front is null)
            _rear = null;

        Count--;
        return value;
    }

    public T Peek() =>
        _front is null
            ? throw new InvalidOperationException("Queue is empty.")
            : _front.Value;
}
```

The custom classes use nullable reference annotations (`Node?`), so enable nullable reference types in the project (for example, `<Nullable>enable</Nullable>` in the project file). They are educational examples and are not thread-safe.

---

#### 4. Complexity summary

| Structure / operation | Typical time |
|---|---:|
| Stack `Push`, `Pop`, `Peek` | O(1) |
| Queue `Enqueue`, `Dequeue`, `Peek` | O(1) |
| `Span<T>` indexing | O(1) |
| `Span<T>.Slice()` | O(1) |

`Span<T>.Slice()` creates a view, not a copy. Creating a stack or queue item, by contrast, stores an element in a collection.

#### 5. Practical guidance

- Prefer `Stack<T>` and `Queue<T>` for normal application code.
- Build custom collections when learning data structures or when requirements call for specialized behavior.
- Use `Span<T>` for efficient processing of buffers, arrays, and character data.
- Use `lock` to guard multi-step changes to shared state and `Interlocked` for simple atomic updates.

---

<a id="csharp-task-parallelism-interview-notes-md"></a>

## Source: Task Parallelism in C# — Interview Notes

*Imported from `csharp-task-parallelism-interview-notes.md`.*

### Task Parallelism in C# — Interview Notes

A concise guide to `Task`, `Task.WhenAll`, `Parallel.ForEach`, exception handling, cancellation, and choosing the right approach.

#### 1. The key distinction

- **Concurrency**: multiple operations make progress during overlapping periods.
- **Parallelism**: multiple operations execute at the same time, typically on multiple CPU cores.
- **`Task` / `Task<T>`**: represents an operation that may complete in the future.
- **`async` / `await`**: makes asynchronous code easier to compose; `await` waits without blocking the calling thread.
- **`Task.Run`**: queues work to the thread pool. Commonly useful for CPU-bound work, not usually needed to call an already-asynchronous I/O API.
- **`Task.WhenAll`**: creates a task that completes when all supplied tasks complete.
- **`Parallel.For` / `Parallel.ForEach`**: parallel loop APIs intended mainly for CPU-bound work.

> Important: tasks do not guarantee one dedicated thread per task, nor do they guarantee that work runs simultaneously.

#### 2. Run one operation with `Task.Run`

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        Console.WriteLine("Start");

        int result = await Task.Run(() => CalculateSquare(12));

        Console.WriteLine($"Result: {result}");
        Console.WriteLine("End");
    }

    static int CalculateSquare(int number) => number * number;
}
```

Use `Task.Run` when you need to move CPU-bound work off a responsive thread, such as a UI thread. In ASP.NET Core, avoid wrapping ordinary synchronous work in `Task.Run` just to make a request handler appear asynchronous.

#### 3. Run independent asynchronous operations with `Task.WhenAll`

This is a common pattern when several I/O operations can happen independently, such as API calls or database queries.

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        Task<string> userTask = FetchUserAsync();
        Task<string> ordersTask = FetchOrdersAsync();
        Task<string> alertsTask = FetchAlertsAsync();

        // All three operations have been started before we await them.
        await Task.WhenAll(userTask, ordersTask, alertsTask);

        Console.WriteLine(await userTask);
        Console.WriteLine(await ordersTask);
        Console.WriteLine(await alertsTask);
    }

    static async Task<string> FetchUserAsync()
    {
        await Task.Delay(500); // Simulates asynchronous I/O
        return "User loaded";
    }

    static async Task<string> FetchOrdersAsync()
    {
        await Task.Delay(700);
        return "Orders loaded";
    }

    static async Task<string> FetchAlertsAsync()
    {
        await Task.Delay(300);
        return "Alerts loaded";
    }
}
```

The simulated operations overlap, so total elapsed time is roughly the duration of the slowest operation, plus overhead—not the sum of all delays. `Task.Delay` is used here instead of blocking with `.Wait()`.

##### Collect results directly

```csharp
Task<int> first = GetNumberAsync(10);
Task<int> second = GetNumberAsync(20);

int[] results = await Task.WhenAll(first, second);
// results: [20, 40]

static async Task<int> GetNumberAsync(int value)
{
    await Task.Delay(100);
    return value * 2;
}
```

`Task.WhenAll` preserves the input task order in its result array; completion order may differ.

#### 4. CPU-bound loops with `Parallel.ForEach`

Use this for independent, CPU-intensive work on collection items—not as a default replacement for asynchronous I/O.

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        int[] numbers = Enumerable.Range(1, 5).ToArray();
        int[] squares = new int[numbers.Length];

        Parallel.ForEach(numbers, (number, state, index) =>
        {
            squares[index] = number * number;
        });

        Console.WriteLine(string.Join(", ", squares));
        // 1, 4, 9, 16, 25
    }
}
```

Each iteration writes to a distinct array index, avoiding shared-write conflicts. The execution order is not guaranteed.

##### Avoid race conditions

This is unsafe because multiple iterations can update `total` at the same time:

```csharp
int total = 0;

Parallel.ForEach(numbers, number =>
{
    total += number; // Race condition
});
```

Use the loop's thread-local accumulation overload instead:

```csharp
int total = 0;

Parallel.ForEach(
    numbers,
    () => 0,                         // Initial local total per worker
    (number, state, localTotal) => localTotal + number,
    localTotal => Interlocked.Add(ref total, localTotal));

Console.WriteLine(total);
```

For simple sums, LINQ's `numbers.Sum()` is usually clearer; parallelism is not automatically faster for small workloads.

#### 5. `Parallel.For`

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        Parallel.For(1, 6, i =>
        {
            Console.WriteLine($"{i} squared = {i * i}");
        });
    }
}
```

The upper bound is **exclusive**, so this loop processes `1` through `5`. Output order can vary.

#### 6. Exceptions

Exceptions from a task are observed when it is awaited. With `Task.WhenAll`, all supplied tasks are allowed to complete; the task returned by `WhenAll` records the failures.

```csharp
Task allTask = Task.WhenAll(
    Task.Run(() => throw new InvalidOperationException("First failed")),
    Task.Run(() => throw new ArgumentException("Second failed")));

try
{
    await allTask;
}
catch
{
    if (allTask.Exception is { } aggregate)
    {
        foreach (Exception error in aggregate.Flatten().InnerExceptions)
            Console.WriteLine(error.Message);
    }
}
```

Place this code inside an `async` method. `await` typically throws one underlying exception; inspect `allTask.Exception` when you need all failures.

#### 7. Cancellation with `CancellationToken`

Cancellation is cooperative: the operation must observe the token and stop. Pass the token to the operation and use APIs that support cancellation.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        using var cts = new CancellationTokenSource();
        cts.CancelAfter(TimeSpan.FromSeconds(2));

        try
        {
            await ProcessAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Operation canceled.");
        }
    }

    static async Task ProcessAsync(CancellationToken token)
    {
        for (int i = 1; i <= 10; i++)
        {
            token.ThrowIfCancellationRequested();

            Console.WriteLine($"Processing item {i}");
            await Task.Delay(500, token);
        }
    }
}
```

Cancellation does not forcibly kill arbitrary code. CPU-bound loops should periodically check the token; asynchronous APIs should receive it when supported.

#### 8. Quick decision guide

| Situation | Prefer |
|---|---|
| Several independent async API/database/I/O calls | Start each operation, then `await Task.WhenAll(...)` |
| CPU-heavy calculation that can use multiple cores | `Parallel.For` / `Parallel.ForEach`, or `Task.Run` for one offloaded operation |
| Need one operation's future result | `Task<T>` |
| Need to wait for all operations | `Task.WhenAll` |
| Need cancellation | `CancellationToken` / `CancellationTokenSource` |
| Need to process a small collection of simple values | A normal loop or LINQ may be simpler and faster |

#### 9. Interview points to remember

1. `await` does not itself create a new thread.
2. `Task.WhenAll` waits for multiple tasks; it does not start cold, unstarted tasks automatically. Calling async methods usually starts their operations.
3. `Task.Run` is generally for CPU-bound work or deliberately offloading work—not a requirement for asynchronous I/O.
4. `Parallel.ForEach` is synchronous and blocks the caller until the loop finishes.
5. Do not mutate shared state from parallel iterations without synchronization or a thread-safe approach.
6. Do not use `.Wait()` or `.Result` in ordinary async code when `await` is available; blocking can cause thread starvation and, in some contexts, deadlocks.
7. Parallel work has overhead. Measure before using it for small tasks.
8. For database work, use the database driver's asynchronous APIs. Do not run concurrent operations on the same EF Core `DbContext`; use separate contexts when truly running independent EF Core operations concurrently.

---

<a id="dotnet-threading-async-await-senior-viva-md"></a>

## Source: .NET Threading and Async/Await — Senior Developer Viva Preparation

*Imported from `dotnet-threading-async-await-senior-viva.md`.*

### .NET Threading and Async/Await — Senior Developer Viva Preparation

#### 1. The Big Picture

In .NET, **threads**, **tasks**, **concurrency**, **parallelism**, and **asynchronous programming** are related, but they are not the same thing.

- **Thread:** An execution path managed by the operating system.
- **Task:** A .NET representation of an operation that may complete in the future. A task is not itself a thread.
- **Concurrency:** Multiple operations make progress during overlapping periods; they do not have to run at the exact same instant.
- **Parallelism:** Multiple operations execute at the same instant, typically on different CPU cores.
- **Asynchronous I/O:** An operation such as an HTTP request or database call can wait for I/O without occupying a thread for the entire wait.

**Senior-level rule of thumb:** Use `async`/`await` for naturally asynchronous I/O, `Task.Run` for CPU-bound work that should run away from the current thread, and direct `Thread` management only when you genuinely need a dedicated thread or low-level thread controls.

---

#### 2. What Are the Ways to Start Work on Another Thread?

There is no single official count of ways to “create a thread,” because .NET offers several levels of abstraction. These are the common approaches:

##### A. Create a `Thread` directly

```csharp
using System;
using System.Threading;

var thread = new Thread(() =>
{
    Console.WriteLine($"Running on thread {Environment.CurrentManagedThreadId}");
});

thread.Start();
thread.Join();
```

This creates a dedicated operating-system thread. Use it when you need low-level control, such as setting thread priority or managing a dedicated thread's lifetime. Creating threads has more overhead than reusing thread-pool workers.

##### B. Queue work to the `ThreadPool`

```csharp
using System;
using System.Threading;

ThreadPool.QueueUserWorkItem(_ =>
{
    Console.WriteLine($"Running on thread {Environment.CurrentManagedThreadId}");
});
```

The .NET thread pool reuses worker threads, avoiding the cost of creating a new thread for every short operation. It is useful for short-running work, but it does not provide the same direct lifecycle control as `Thread`.

##### C. Use `Task.Run`

```csharp
Task task = Task.Run(() =>
{
    DoCpuBoundWork();
});

await task;
```

`Task.Run` normally queues CPU-bound work to the thread pool. It gives you task composition, exception propagation, cancellation-token support, and APIs such as `Task.WhenAll`. It does **not** mean that every `Task` creates a new thread.

##### D. Use `async`/`await` with an asynchronous API

```csharp
async Task<string> GetDataAsync(HttpClient httpClient, string url)
{
    return await httpClient.GetStringAsync(url);
}
```

This is asynchronous I/O. While the HTTP request is waiting for a response, the application generally does not need to keep a worker thread blocked for that wait. `async`/`await` does not automatically create a new thread.

> **Important distinction:** `new Thread(...)` creates a thread. `Task.Run(...)` schedules work, usually on a thread-pool thread. Calling an asynchronous I/O API starts or represents an asynchronous operation; it does not necessarily create a thread.

---

#### 3. What Do `Start()` and `Join()` Do?

##### `Start()`

`Thread.Start()` asks the runtime to begin executing the thread's entry-point method.

```csharp
var thread = new Thread(() => Console.WriteLine("Worker started"));
thread.Start();
```

Creating a `Thread` object alone does not run its delegate. You must call `Start()`.

##### `Join()`

`Thread.Join()` blocks the calling thread until the target thread finishes.

```csharp
var thread = new Thread(() =>
{
    Thread.Sleep(500);
    Console.WriteLine("Worker finished");
});

thread.Start();
thread.Join(); // The calling thread waits here.
Console.WriteLine("Continue after worker completes");
```

Use `Join()` when synchronous waiting for a particular thread is required. In asynchronous code, prefer task-based coordination such as `await task` rather than blocking with `.Wait()` or `.Result`.

---

#### 4. How Can We Get a Return Value from Work on a Thread?

A `Thread` does not have a built-in return-value mechanism. You can communicate through a shared variable, but you must coordinate access and ensure the worker has completed before reading the result.

```csharp
int result = 0;

var thread = new Thread(() =>
{
    result = 42;
});

thread.Start();
thread.Join();

Console.WriteLine(result); // 42
```

For ordinary application code, a generic task is usually clearer:

```csharp
Task<int> task = Task.Run(() => CalculateValue());

int result = await task;
Console.WriteLine(result);

static int CalculateValue() => 42;
```

`Task<T>` represents an operation that produces a value of type `T`. Awaiting it also propagates exceptions from the operation. Avoid `.Result` in asynchronous code when you can use `await`, because `.Result` blocks the calling thread and can contribute to deadlocks in some synchronization contexts.

---

#### 5. `Thread` vs. `Task`

| Topic | `Thread` | `Task` |
|---|---|---|
| Meaning | An execution thread | A representation of an operation |
| Thread creation | `new Thread(...)` creates a dedicated thread | A task may use a thread pool, an existing thread, or no thread while awaiting I/O |
| Return value | No built-in return value | `Task<T>` supports a result |
| Waiting | `Join()` | Prefer `await`; `.Wait()` and `.Result` block |
| Exceptions | Exceptions escaping the thread entry point are not exposed through a task object | Exceptions are stored in the task and observed when awaited or otherwise inspected |
| Composition | Manual coordination is often needed | Supports `WhenAll`, `WhenAny`, continuations, and cancellation tokens |
| Low-level control | Supports controls such as priority and thread lifetime | Does not provide direct control over a specific worker thread |
| Typical use | Dedicated thread or special low-level requirements | Most application-level asynchronous and concurrent operations |

**Interview answer:** A thread is a mechanism for executing code. A task represents work and its completion. A task may use a thread, but the two are not interchangeable.

---

#### 6. What Is the Role of the Thread Pool?

The .NET thread pool is a runtime-managed collection of reusable worker threads. It helps run short background operations without repeatedly creating and destroying threads.

Common users include `Task.Run`, many task-scheduling operations, and `ThreadPool.QueueUserWorkItem`.

##### Benefits

- Reuses threads and reduces thread-creation overhead.
- Helps the runtime manage worker-thread availability.
- Integrates with the Task Parallel Library.

##### Limitations

- Blocking thread-pool workers for long periods can reduce throughput and delay other work.
- It does not provide a dedicated thread for each task.
- Thread-pool scheduling does not guarantee which thread will run a particular task.

For long-running services in ASP.NET Core, use a hosted service such as `BackgroundService` for lifecycle management rather than creating an unmanaged infinite loop in a request handler. A hosted service still needs an appropriate strategy for CPU-bound work, blocking work, cancellation, and resource management.

---

#### 7. Is a Task Multithreaded?

**No—not necessarily.** A task represents an operation, not a promise that a new or separate thread will be used.

##### Example 1: Asynchronous I/O

```csharp
async Task<string> LoadAsync(HttpClient client, string url)
{
    return await client.GetStringAsync(url);
}
```

The HTTP operation can wait without occupying a thread for the whole wait. The method resumes when the awaited operation completes. Depending on the environment and continuation context, the continuation may run on a different thread.

##### Example 2: CPU-bound work using `Task.Run`

```csharp
Task<int> first = Task.Run(() => Calculate(10));
Task<int> second = Task.Run(() => Calculate(20));

int[] results = await Task.WhenAll(first, second);
```

Both calculations can run concurrently on thread-pool workers, and may run in parallel if resources and scheduling allow. Parallel execution is not guaranteed merely because two tasks were created.

##### Example 3: A task that runs synchronously

```csharp
Task<int> task = Task.FromResult(42);
int result = await task;
```

This task is already completed; it does not need to schedule work on another thread.

---

#### 8. Concurrency vs. Multithreading vs. Parallelism

These terms describe different aspects of execution:

- **Concurrency:** Several operations are in progress over an overlapping period. A single thread can coordinate concurrent asynchronous I/O by starting operations and awaiting their completion.
- **Multithreading:** An application uses multiple threads. Those threads may take turns on one CPU core or run on multiple cores.
- **Parallelism:** Multiple pieces of work execute at the same time, commonly on separate CPU cores.
- **Asynchrony:** The caller can continue without synchronously waiting for an operation to finish. Asynchrony does not automatically mean multithreading or parallelism.

##### Concurrent I/O example

```csharp
async Task<string[]> LoadBothAsync(HttpClient client)
{
    Task<string> first = client.GetStringAsync("https://example.com");
    Task<string> second = client.GetStringAsync("https://example.org");

    return await Task.WhenAll(first, second);
}
```

Both requests are started before the method awaits their combined completion. Their I/O waits overlap; this does not require one dedicated thread per request.

##### Parallel CPU-work example

```csharp
Parallel.For(0, 100, i =>
{
    ProcessItem(i);
});
```

`Parallel.For` is designed for parallel CPU work. It may execute iterations on multiple threads, subject to runtime scheduling and available resources. It is not the default choice for I/O-bound operations.

##### Quick comparison

| Concept | Main idea | Example |
|---|---|---|
| Concurrency | Multiple operations make progress over overlapping periods | Start multiple HTTP requests and await them |
| Multithreading | Multiple threads are used | A worker thread plus the main thread |
| Parallelism | Work executes simultaneously | CPU calculations across multiple cores |
| Asynchrony | Avoid blocking while waiting | `await ReadAsync(...)` |

---

#### 9. When Should a Senior .NET Developer Use Each Approach?

| Scenario | Recommended approach |
|---|---|
| HTTP, database, or file I/O with async APIs | Use the API's async method and `await` |
| CPU-bound work that should not run on the current thread | Consider `Task.Run` where appropriate |
| Parallel CPU calculations over independent items | Consider `Parallel.For`, `Parallel.ForEach`, or suitable task-based parallelism |
| Need explicit control over a dedicated thread | Use `Thread` only when the requirement justifies it |
| Background processing in an ASP.NET Core service | Use `IHostedService` / `BackgroundService` with cancellation and proper lifecycle handling |
| Run several asynchronous operations and await all | Use `Task.WhenAll` |
| Need a result from asynchronous work | Use `Task<T>` and `await` |

##### Two common mistakes to avoid

1. **Wrapping asynchronous I/O in `Task.Run` without a reason.** For example, prefer `await httpClient.GetStringAsync(url)` over `await Task.Run(() => httpClient.GetStringAsync(url))`.
2. **Blocking asynchronous work unnecessarily.** In async call chains, prefer `await task` over `task.Wait()` or `task.Result`.

---

#### 10. Senior-Level Viva Questions and Short Answers

**Q: Does `async` create a thread?**  
A: No. `async` enables a method to use `await`. Whether threads are involved depends on the operation being awaited.

**Q: Does `await` block a thread?**  
A: Normally, `await` suspends the async method when the awaited operation is incomplete and returns control to its caller. It does not synchronously block the current thread while waiting.

**Q: Does every task run on the thread pool?**  
A: No. `Task.Run` normally uses the thread pool, but tasks can represent I/O operations, already-completed results, or work scheduled through other mechanisms.

**Q: Is multithreading always faster?**  
A: No. Thread scheduling, synchronization, contention, and context switching have costs. Parallelism helps mainly when work can be divided effectively and resources are available.

**Q: Can two tasks run on the same thread?**  
A: Yes. Tasks are not permanently bound to distinct threads. A task may execute synchronously, and asynchronous continuations can run on a thread selected by the scheduler or synchronization context.

**Q: What should I use instead of `Thread.Sleep` in async code?**  
A: Use `await Task.Delay(duration, cancellationToken)` when an asynchronous delay is intended. `Thread.Sleep` blocks the current thread.

**Q: What is the difference between `Task.WhenAll` and `Task.WaitAll`?**  
A: `Task.WhenAll` returns a task that completes when all supplied tasks complete and can be awaited without blocking the calling thread. `Task.WaitAll` blocks the calling thread until all tasks complete.

**Q: If a web API needs to call three independent services, should I use three threads?**  
A: Usually no. Start the three asynchronous I/O operations and await them with `Task.WhenAll`, assuming they are independent and the downstream services can handle the concurrent requests.

---

#### Final Takeaway

For most modern .NET application code, **prefer the Task-based model**. Use `async`/`await` for asynchronous I/O, use `Task.Run` or parallel APIs for suitable CPU-bound work, and use `Thread` when you specifically need low-level thread control. Always distinguish the operation (`Task`), the execution resource (`Thread`), overlapping progress (concurrency), and simultaneous execution (parallelism).
