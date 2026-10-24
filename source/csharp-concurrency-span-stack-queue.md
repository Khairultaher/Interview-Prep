# C# Interview Notes: Synchronization, Span, Stack, and Queue

## 1. `lock`, `Monitor`, and `Interlocked`

These APIs help make code thread-safe, but they solve different problems.

### `lock` — protect a critical section

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

### `Monitor` — more control over locking

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

### `Interlocked` — atomic operations on individual values

Use it for simple atomic updates such as incrementing a counter. It does not protect a larger multi-step operation.

```csharp
using System.Threading;

private int _requestCount;

public void RecordRequest()
{
    Interlocked.Increment(ref _requestCount);
}
```

### Quick comparison

| Need | Best fit |
|---|---|
| Protect a block of related operations | `lock` |
| Lock timeout or `Wait`/`Pulse` signaling | `Monitor` |
| Atomic increment, exchange, or compare-and-swap | `Interlocked` |

**Rule of thumb:** start with `lock` for shared mutable state; use `Interlocked` for a genuinely simple atomic operation; use `Monitor` when its additional control is needed.

---

## 2. `Span<T>` and `Slice()`

`Span<T>` is a lightweight view over contiguous memory. It lets you access a portion of an array or other supported memory without copying the elements.

### Slice an array

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

### Slice a string without immediately creating a substring

Strings are immutable, so use `ReadOnlySpan<char>` to view their characters.

```csharp
string text = "Hello, World!";
ReadOnlySpan<char> word = text.AsSpan(7, 5);

Console.WriteLine(word.ToString()); // World
```

Calling `ToString()` creates a string. If you can process the span directly, you can avoid that allocation.

### Important limitations

- A slice must stay within the source span's bounds.
- `Span<T>` and `ReadOnlySpan<T>` are `ref struct` types; they cannot be stored in ordinary class fields or used across `await`/`yield` boundaries in the usual way.
- `ReadOnlySpan<T>` prevents writes through that view; it does not necessarily make the underlying data immutable through other references.

---

## 3. Stack and Queue in C#

A **stack** is LIFO (last in, first out); a **queue** is FIFO (first in, first out). Use the built-in generic collections unless you specifically need to implement the data structures yourself.

### Built-in stack

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

### Built-in queue

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

### Custom implementations with a linked list

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

## 4. Complexity summary

| Structure / operation | Typical time |
|---|---:|
| Stack `Push`, `Pop`, `Peek` | O(1) |
| Queue `Enqueue`, `Dequeue`, `Peek` | O(1) |
| `Span<T>` indexing | O(1) |
| `Span<T>.Slice()` | O(1) |

`Span<T>.Slice()` creates a view, not a copy. Creating a stack or queue item, by contrast, stores an element in a collection.

## 5. Practical guidance

- Prefer `Stack<T>` and `Queue<T>` for normal application code.
- Build custom collections when learning data structures or when requirements call for specialized behavior.
- Use `Span<T>` for efficient processing of buffers, arrays, and character data.
- Use `lock` to guard multi-step changes to shared state and `Interlocked` for simple atomic updates.
