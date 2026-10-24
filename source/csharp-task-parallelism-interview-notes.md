# Task Parallelism in C# — Interview Notes

A concise guide to `Task`, `Task.WhenAll`, `Parallel.ForEach`, exception handling, cancellation, and choosing the right approach.

## 1. The key distinction

- **Concurrency**: multiple operations make progress during overlapping periods.
- **Parallelism**: multiple operations execute at the same time, typically on multiple CPU cores.
- **`Task` / `Task<T>`**: represents an operation that may complete in the future.
- **`async` / `await`**: makes asynchronous code easier to compose; `await` waits without blocking the calling thread.
- **`Task.Run`**: queues work to the thread pool. Commonly useful for CPU-bound work, not usually needed to call an already-asynchronous I/O API.
- **`Task.WhenAll`**: creates a task that completes when all supplied tasks complete.
- **`Parallel.For` / `Parallel.ForEach`**: parallel loop APIs intended mainly for CPU-bound work.

> Important: tasks do not guarantee one dedicated thread per task, nor do they guarantee that work runs simultaneously.

## 2. Run one operation with `Task.Run`

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

## 3. Run independent asynchronous operations with `Task.WhenAll`

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

### Collect results directly

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

## 4. CPU-bound loops with `Parallel.ForEach`

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

### Avoid race conditions

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

## 5. `Parallel.For`

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

## 6. Exceptions

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

## 7. Cancellation with `CancellationToken`

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

## 8. Quick decision guide

| Situation | Prefer |
|---|---|
| Several independent async API/database/I/O calls | Start each operation, then `await Task.WhenAll(...)` |
| CPU-heavy calculation that can use multiple cores | `Parallel.For` / `Parallel.ForEach`, or `Task.Run` for one offloaded operation |
| Need one operation's future result | `Task<T>` |
| Need to wait for all operations | `Task.WhenAll` |
| Need cancellation | `CancellationToken` / `CancellationTokenSource` |
| Need to process a small collection of simple values | A normal loop or LINQ may be simpler and faster |

## 9. Interview points to remember

1. `await` does not itself create a new thread.
2. `Task.WhenAll` waits for multiple tasks; it does not start cold, unstarted tasks automatically. Calling async methods usually starts their operations.
3. `Task.Run` is generally for CPU-bound work or deliberately offloading work—not a requirement for asynchronous I/O.
4. `Parallel.ForEach` is synchronous and blocks the caller until the loop finishes.
5. Do not mutate shared state from parallel iterations without synchronization or a thread-safe approach.
6. Do not use `.Wait()` or `.Result` in ordinary async code when `await` is available; blocking can cause thread starvation and, in some contexts, deadlocks.
7. Parallel work has overhead. Measure before using it for small tasks.
8. For database work, use the database driver's asynchronous APIs. Do not run concurrent operations on the same EF Core `DbContext`; use separate contexts when truly running independent EF Core operations concurrently.
