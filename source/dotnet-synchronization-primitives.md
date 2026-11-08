# .NET Synchronization: `lock`, `Monitor`, `Mutex`, and `Semaphore`

When multiple threads access shared resources, synchronization helps prevent race conditions and coordinate access. The right mechanism depends on whether you need exclusive access, signaling, or a limit on concurrent access.

## 1. At a glance

| Mechanism | Typical scope | What it does | Common use |
|---|---|---|---|
| `lock` | Threads within one process | Provides mutual exclusion around a critical section | Protecting shared in-memory state |
| `Monitor` | Threads within one process | Provides mutual exclusion plus timed acquisition and thread signaling | Advanced in-process coordination |
| `Mutex` | Within one process or across processes when named | Provides exclusive ownership | Ensuring only one app/process performs an operation |
| `Semaphore` | Within one process or across processes when named | Allows up to a configured number of concurrent entrants | Limiting access to a resource pool |
| `SemaphoreSlim` | Within one process | Lightweight semaphore; supports async waiting | Limiting concurrent async operations |

**Important:** `lock` is built on `Monitor` semantics in C#. A `Mutex` is not simply a “low-level lock,” and `Semaphore` is not necessarily process-level: named OS-backed mutexes and semaphores can coordinate across processes, while unnamed instances are generally used within one process.

## 2. `lock` — the simple choice

Use `lock` when only one thread at a time should execute a critical section in the same process.

```csharp
private readonly object _sync = new();

public void Update()
{
    lock (_sync)
    {
        // Only one thread at a time can execute this section
        // while locking the same _sync object.
    }
}
```

**Remember**
- Keep the locked section short.
- Lock on a private object, not `this`, a string, or a publicly accessible object.
- All threads must lock on the same object for the lock to coordinate them.
- Do not `await` inside a traditional `lock` block.

## 3. `Monitor` — more control than `lock`

`Monitor` provides the locking behavior behind `lock`, plus methods such as `TryEnter`, `Wait`, `Pulse`, and `PulseAll`.

```csharp
using System;
using System.Threading;

private readonly object _sync = new();

public void UpdateWithTimeout()
{
    bool lockTaken = false;

    try
    {
        Monitor.TryEnter(_sync, TimeSpan.FromSeconds(2), ref lockTaken);

        if (!lockTaken)
        {
            Console.WriteLine("Could not acquire the lock in time.");
            return;
        }

        // Critical section
    }
    finally
    {
        if (lockTaken)
            Monitor.Exit(_sync);
    }
}
```

Use `Monitor` when you need a timeout or thread signaling. Always pair a successful `Monitor.Enter`/`TryEnter` with `Monitor.Exit`, typically in a `finally` block.

## 4. `Mutex` — exclusive access, potentially across processes

A mutex allows only one owning thread at a time. A **named mutex** can coordinate separate processes on the same operating system.

```csharp
using System;
using System.Threading;

using var mutex = new Mutex(false, @"Global\MyApp_ExclusiveJob");

bool acquired = false;

try
{
    try
    {
        acquired = mutex.WaitOne(TimeSpan.FromSeconds(5));
    }
    catch (AbandonedMutexException)
    {
        // The previous owner ended without releasing the mutex.
        // The current thread has acquired it; check shared state if needed.
        acquired = true;
    }

    if (!acquired)
    {
        Console.WriteLine("Another process is using the resource.");
        return;
    }

    // Exclusive operation
}
finally
{
    if (acquired)
        mutex.ReleaseMutex();
}
```

**Notes**
- The `Global\` namespace and cross-session behavior depend on the operating system and permissions; test the intended deployment environment.
- The thread that acquires a mutex must release it.
- A mutex has more overhead than an in-process `lock`.

## 5. `Semaphore` — limit concurrent access

A semaphore allows a fixed number of threads to enter at the same time. For example, an initial count of `3` allows up to three concurrent entrants.

```csharp
using System;
using System.Threading;

using var semaphore = new Semaphore(initialCount: 3, maximumCount: 3);

if (semaphore.WaitOne(TimeSpan.FromSeconds(5)))
{
    try
    {
        // Up to three threads can be in this section concurrently.
    }
    finally
    {
        semaphore.Release();
    }
}
else
{
    Console.WriteLine("No slot became available in time.");
}
```

A **named** `Semaphore` can coordinate processes. Be careful to release it exactly once for each successful acquisition; unlike a mutex, a semaphore is not owned by the acquiring thread in the same way.

## 6. `SemaphoreSlim` — often best for async code

For limiting concurrent work inside a .NET process, `SemaphoreSlim` is often a good fit because it supports `WaitAsync`.

```csharp
private readonly SemaphoreSlim _gate = new(initialCount: 3, maxCount: 3);

public async Task ProcessAsync(CancellationToken cancellationToken)
{
    await _gate.WaitAsync(cancellationToken);

    try
    {
        // At most three calls can run this section concurrently.
        await DoWorkAsync(cancellationToken);
    }
    finally
    {
        _gate.Release();
    }
}
```

`SemaphoreSlim` is not a cross-process synchronization primitive. It is especially useful for throttling async operations without blocking a thread while waiting.

## 7. Main differences

| Feature | `lock` | `Monitor` | `Mutex` | `Semaphore` | `SemaphoreSlim` |
|---|---|---|---|---|---|
| Exclusive access | Yes | Yes | Yes | No, permits multiple | No, permits multiple |
| Limit concurrent entrants | One | One | One | Yes | Yes |
| Timeout support | Not directly | Yes | Yes | Yes | Yes |
| Thread signaling (`Wait`/`Pulse`) | No direct API | Yes | No | No | No |
| Cross-process support | No | No | Yes, with a named mutex | Yes, with a named semaphore | No |
| Async waiting | No | No | No | No | Yes, via `WaitAsync` |
| Typical overhead | Low | Low | Higher | Higher than in-process lock | Lightweight in-process |

## 8. Which one should I use?

- **Use `lock`** for straightforward, synchronous protection of shared in-memory state.
- **Use `Monitor`** when you need timed lock acquisition or `Wait`/`Pulse` signaling.
- **Use `Mutex`** when one thread or process must have exclusive ownership of an operation.
- **Use `Semaphore`** when access must be limited to a fixed number of concurrent entrants, including across processes when named.
- **Use `SemaphoreSlim`** to limit concurrent operations in one process, especially when the code uses `async`/`await`.

## Conclusion

These synchronization tools solve related but different problems. `lock` and `Monitor` are generally for coordinating threads inside one process. A named `Mutex` can provide exclusive access across processes, while a semaphore controls how many entrants may access a resource at once. For async code, `SemaphoreSlim` is often the most practical option for limiting concurrency. Choose based on the coordination requirement rather than treating one primitive as universally better than the others.
