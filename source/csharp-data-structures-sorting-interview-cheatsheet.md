# C# Data Structures, Heaps, Comparers, and Sorting — Interview Cheat Sheet

Target: modern C# / .NET (including `PriorityQueue<TElement, TPriority>` in .NET 6+).

## 1. Comparison rules: the foundation

A comparison function `Compare(a, b)` should return:

- A negative number when `a` should come **before** `b`.
- `0` when they are equal for sorting purposes.
- A positive number when `a` should come **after** `b`.

Prefer `CompareTo` or `Comparer<T>.Default.Compare` over subtracting values. `b - a` can overflow for integers.

```csharp
int[] values = { 5, 3, 8, 1, 2 };

Array.Sort(values);                                      // ascending
Array.Sort(values, (a, b) => b.CompareTo(a));            // descending

Array.Sort(values, Comparer<int>.Create(
    (a, b) => a.CompareTo(b)));                          // explicit comparer
```

## 2. Array and List sorting

```csharp
int[] array = { 5, 3, 8, 1, 2 };
Array.Sort(array);             // modifies array: 1, 2, 3, 5, 8
Array.Sort(array, (a, b) => b.CompareTo(a)); // descending
Array.Reverse(array);          // reverses current order; does not sort

var list = new List<int> { 5, 3, 8, 1, 2 };
list.Sort();                   // ascending, in place
list.Sort((a, b) => b.CompareTo(a)); // descending, in place
list.Reverse();                // reverses current order
```

### Sorting objects

```csharp
public sealed class Employee
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
    public decimal Salary { get; init; }
}

var employees = new List<Employee>
{
    new() { Name = "Mina", Age = 30, Salary = 90000m },
    new() { Name = "Amin", Age = 25, Salary = 80000m },
    new() { Name = "Zara", Age = 30, Salary = 95000m }
};

employees.Sort((x, y) => x.Age.CompareTo(y.Age)); // age ascending
employees.Sort((x, y) => y.Salary.CompareTo(x.Salary)); // salary descending

// Multiple criteria: age ascending, then name ascending
employees.Sort((x, y) =>
{
    int byAge = x.Age.CompareTo(y.Age);
    return byAge != 0
        ? byAge
        : StringComparer.OrdinalIgnoreCase.Compare(x.Name, y.Name);
});
```

### LINQ sorting (creates an ordered sequence)

```csharp
var ascending = employees.OrderBy(e => e.Age).ToList();
var descending = employees.OrderByDescending(e => e.Age).ToList();

var multiple = employees
    .OrderBy(e => e.Age)
    .ThenBy(e => e.Name, StringComparer.OrdinalIgnoreCase)
    .ToList();

var largestFirst = employees
    .OrderByDescending(e => e.Salary)
    .ThenBy(e => e.Name)
    .ToList();
```

**Choose:** `Array.Sort` / `List<T>.Sort` for in-place sorting; LINQ `OrderBy` when composing queries or keeping the source unchanged. LINQ sorting is deferred until enumerated, unless materialized with `ToList()`/`ToArray()`.

## 3. `Comparer<T>`, `IComparer<T>`, and `IComparable<T>`

Use a lambda for one-off rules; use a reusable comparer when the rule is shared.

```csharp
IComparer<string> byLength = Comparer<string>.Create(
    (x, y) => x.Length.CompareTo(y.Length));

var words = new List<string> { "pear", "fig", "banana" };
words.Sort(byLength);
```

Reusable comparer:

```csharp
public sealed class EmployeeAgeDescendingComparer : IComparer<Employee>
{
    public int Compare(Employee? x, Employee? y)
    {
        if (ReferenceEquals(x, y)) return 0;
        if (x is null) return 1;
        if (y is null) return -1;
        return y.Age.CompareTo(x.Age);
    }
}

// employees.Sort(new EmployeeAgeDescendingComparer());
```

- `IComparer<T>.Compare(x, y)`: external comparison strategy; ideal for alternate sort orders.
- `IComparable<T>.CompareTo(other)`: type's default natural ordering.
- `Comparer<T>.Default`: uses the type's default comparison when available.
- For strings, select the intended comparison explicitly when important, e.g. `StringComparer.OrdinalIgnoreCase`.

## 4. Priority queues and binary heaps

`PriorityQueue<TElement, TPriority>` is available in .NET 6+. It removes the element with the **lowest priority value first** by default. It is a min-priority queue, not a fully sorted collection.

### Min-heap: smallest value first

```csharp
var minHeap = new PriorityQueue<int, int>();
foreach (int n in new[] { 5, 1, 8, 2 })
    minHeap.Enqueue(n, n);

int smallest = minHeap.Peek();    // 1, does not remove it
int removed = minHeap.Dequeue();  // 1, removes it

while (minHeap.TryDequeue(out int element, out int priority))
    Console.WriteLine($"{element}, priority={priority}");
```

### Max-heap for integers

Option A: reverse the priority comparer. This avoids negating priorities (which can overflow for `int.MinValue`).

```csharp
var maxHeap = new PriorityQueue<int, int>(
    Comparer<int>.Create((a, b) => b.CompareTo(a)));

foreach (int n in new[] { 5, 1, 8, 2 })
    maxHeap.Enqueue(n, n);

while (maxHeap.TryDequeue(out int value, out _))
    Console.WriteLine(value); // 8, 5, 2, 1
```

Option B: store a reversed numeric priority (simple, but beware integer overflow):

```csharp
var maxHeap = new PriorityQueue<int, int>();
maxHeap.Enqueue(5, -5);
maxHeap.Enqueue(1, -1);
// Prefer a reversed comparer for values where negation could overflow.
```

### Custom priority with objects

```csharp
public sealed record Job(string Name, int Priority);

var jobs = new PriorityQueue<Job, int>(); // lower number runs first
jobs.Enqueue(new Job("Report", 3), 3);
jobs.Enqueue(new Job("Urgent fix", 1), 1);
jobs.Enqueue(new Job("Cleanup", 5), 5);

while (jobs.TryDequeue(out Job? job, out int priority))
    Console.WriteLine($"{priority}: {job?.Name}");
```

### Custom priority ordering with a comparer

```csharp
// Higher numeric priority first:
var jobsMaxFirst = new PriorityQueue<Job, int>(
    Comparer<int>.Create((a, b) => b.CompareTo(a)));

jobsMaxFirst.Enqueue(new Job("Normal", 2), 2);
jobsMaxFirst.Enqueue(new Job("Critical", 10), 10);
```

### Useful `PriorityQueue` members

- `Enqueue(element, priority)`: insert.
- `Dequeue()`: remove and return the next element; throws if empty.
- `TryDequeue(out element, out priority)`: safe removal; returns `false` if empty.
- `Peek()`: inspect next element; throws if empty.
- `TryPeek(out element, out priority)`: safe inspection.
- `EnqueueDequeue(element, priority)`: enqueue and then remove the minimum-priority item as one operation.
- `DequeueEnqueue(element, priority)`: remove the current minimum, then enqueue the supplied item.
- `Clear()`, `Count`, `EnsureCapacity(capacity)`, `TrimExcess()`, `UnorderedItems`.

The queue does **not** promise sorted enumeration. Equal priorities do not guarantee FIFO order. Add a sequence number to the priority if stable tie-breaking is required.

### Top-K pattern (keep the K largest numbers)

Keep a min-heap of at most `k` elements; the root is the smallest among the retained largest values.

```csharp
static int[] KLargest(int[] values, int k)
{
    if (k < 0 || k > values.Length)
        throw new ArgumentOutOfRangeException(nameof(k));
    if (k == 0) return Array.Empty<int>();

    var heap = new PriorityQueue<int, int>();
    foreach (int value in values)
    {
        heap.Enqueue(value, value);
        if (heap.Count > k)
            heap.Dequeue();
    }

    return heap.UnorderedItems
        .Select(item => item.Element)
        .OrderByDescending(x => x)
        .ToArray();
}
```

Complexity: `O(n log k)` time and `O(k)` extra space, excluding the final sort of the `k` results.

## 5. Common data structures and when to use them

| Type | Best for | Typical operation costs |
|---|---|---|
| `T[]` | Fixed-size indexed storage | Index `O(1)`; search `O(n)`; sort `O(n log n)` |
| `List<T>` | Resizable array, indexed access | Index `O(1)`; append amortized `O(1)`; middle insert/remove `O(n)` |
| `Dictionary<TKey,TValue>` | Key-to-value lookup | Average lookup/insert/remove `O(1)` |
| `HashSet<T>` | Uniqueness and membership tests | Average add/remove/contains `O(1)` |
| `SortedDictionary<TKey,TValue>` | Key/value data kept key-sorted | Lookup/insert/remove `O(log n)` |
| `SortedList<TKey,TValue>` | Sorted keys and index-based access | Lookup `O(log n)`; insert/remove `O(n)` |
| `SortedSet<T>` | Unique values kept sorted | Add/remove/contains `O(log n)` |
| `Queue<T>` | FIFO; BFS | Enqueue/dequeue/peek `O(1)` |
| `Stack<T>` | LIFO; DFS, parsing | Push/pop/peek `O(1)` |
| `LinkedList<T>` | Insert/remove when the node is already known | Node insert/remove `O(1)`; finding a value `O(n)` |
| `PriorityQueue<TElement,TPriority>` | Repeatedly get the lowest/highest-priority item | Enqueue/dequeue `O(log n)`; peek `O(1)` |
| `ConcurrentDictionary<TKey,TValue>` | Shared dictionary accessed by multiple threads | Thread-safe operations; typical average lookup `O(1)` |

Costs are typical, not hard real-time guarantees. Hash-based operations can degrade in pathological cases.

## 6. Everyday collection operations

### Arrays and lists

```csharp
int[] a = { 1, 2, 3 };
int first = a[0];
int length = a.Length;

var list = new List<int> { 1, 2, 3 };
list.Add(4);
list.AddRange(new[] { 5, 6 });
list.Insert(0, 99);
bool hasThree = list.Contains(3);
int index = list.IndexOf(3);
list.Remove(99);       // removes first matching value
list.RemoveAt(0);      // removes by index
list.Sort();
list.Clear();
```

### Dictionary

```csharp
var scores = new Dictionary<string, int>();
scores["Ada"] = 95; // add or replace
scores.Add("Lin", 88); // throws if key already exists

if (scores.TryGetValue("Ada", out int score))
    Console.WriteLine(score);

bool exists = scores.ContainsKey("Lin");
scores.Remove("Lin");

foreach (KeyValuePair<string, int> pair in scores)
    Console.WriteLine($"{pair.Key}: {pair.Value}");
```

Prefer `TryGetValue` when a missing key is normal. Use `Add` when duplicate keys should be treated as an error.

### HashSet operations

```csharp
var left = new HashSet<int> { 1, 2, 3 };
var right = new HashSet<int> { 3, 4, 5 };

left.UnionWith(right);        // {1,2,3,4,5}
left.IntersectWith(right);    // retain common values
left.ExceptWith(right);       // remove values present in right
bool isPresent = left.Contains(3);
```

Set operation methods mutate the receiver. Use `left.Union(right)` / `Intersect` / `Except` with LINQ when a new sequence is preferred.

### Queue and stack

```csharp
var queue = new Queue<string>();
queue.Enqueue("A");
queue.Enqueue("B");
if (queue.TryDequeue(out string? next)) { /* use next */ }

var stack = new Stack<string>();
stack.Push("A");
stack.Push("B");
if (stack.TryPop(out string? top)) { /* use top */ }
```

## 7. LINQ utilities commonly used in interviews

```csharp
int[] numbers = { 4, 1, 4, 2, 9, 2 };

var distinct = numbers.Distinct().ToArray();
var evens = numbers.Where(n => n % 2 == 0).ToArray();
var squares = numbers.Select(n => n * n).ToArray();
int count = numbers.Count(n => n > 3);
bool anyNegative = numbers.Any(n => n < 0);
bool allPositive = numbers.All(n => n > 0);
int sum = numbers.Sum();
int max = numbers.Max();
int min = numbers.Min();
var frequency = numbers
    .GroupBy(n => n)
    .ToDictionary(g => g.Key, g => g.Count());
```

For performance-sensitive hot paths, be aware that LINQ may allocate or enumerate multiple times. A simple loop can be clearer and faster when profiling shows it matters.

## 8. Binary search on sorted data

```csharp
int[] sorted = { 1, 3, 5, 7, 9 };
int index = Array.BinarySearch(sorted, 7); // index 3 if found
int missing = Array.BinarySearch(sorted, 6); // negative encoded insertion point
```

The input must already be sorted with the same ordering used for the search. For insertion point when not found: `int insertionIndex = ~missing;`.

## 9. Interview patterns to recognize

- **Fast membership / deduplication:** `HashSet<T>`.
- **Count frequencies / pair lookup:** `Dictionary<TKey,TValue>`.
- **BFS / shortest path in an unweighted graph:** `Queue<T>`.
- **DFS / matching brackets / monotonic stack:** `Stack<T>`.
- **Repeated minimum/maximum, Dijkstra, scheduling, Top-K:** `PriorityQueue<TElement,TPriority>`.
- **Sorted unique values:** `SortedSet<T>`.
- **Sort once, then binary search / two pointers:** array or list + sort.
- **Sliding window:** array/list plus two pointers; sometimes a dictionary or deque-like structure.
- **Custom ordering:** comparison lambda, `Comparer<T>.Create`, or `IComparer<T>`.

## 10. Pitfalls worth remembering

1. Avoid `(a, b) => a - b` and `(a, b) => b - a` for general integer comparison because subtraction can overflow. Use `CompareTo`.
2. `Array.Reverse` and `List<T>.Reverse` reverse the current order; they do not sort.
3. `List<T>.Sort` and `Array.Sort` are not stable: equal items are not guaranteed to keep their original order.
4. LINQ `OrderBy` is stable, but it creates an ordered sequence rather than sorting the source in place.
5. A `PriorityQueue` is not fully sorted; only the next priority is guaranteed to be at the top.
6. `Dictionary` and `HashSet` should not be relied on for sorted order.
7. `LinkedList<T>` is not automatically faster than `List<T>`; locating the node is `O(n)` and arrays usually have better cache locality.
8. For custom objects used as hash keys, ensure equality and hash-code behavior are consistent.
9. For a max-priority queue, a reversed comparer is safer than negating a priority that could be `int.MinValue`.
10. In production code, choose clarity first and optimize after measuring; Big-O does not capture allocations, cache behavior, or constant factors.

## 11. Quick templates

```csharp
// Ascending / descending
Array.Sort(a);
Array.Sort(a, (x, y) => y.CompareTo(x));

// List with custom comparison
list.Sort((x, y) => x.SomeProperty.CompareTo(y.SomeProperty));

// LINQ, multiple criteria
var result = items.OrderBy(x => x.Category)
                  .ThenByDescending(x => x.Score)
                  .ToList();

// Min-priority queue
var min = new PriorityQueue<string, int>();
min.Enqueue("low", 1);
min.TryDequeue(out string? item, out int priority);

// Max-priority queue
var max = new PriorityQueue<int, int>(
    Comparer<int>.Create((x, y) => y.CompareTo(x)));
max.Enqueue(10, 10);
max.Enqueue(4, 4);
max.TryDequeue(out int largest, out _); // 10
```
