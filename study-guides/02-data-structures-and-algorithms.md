# Data Structures and Algorithms

This guide combines the distinct source materials below. Original explanations, examples, and interview questions are retained; exact duplicate source files are excluded and documented in the library index.

## Included source materials

- [csharp-data-structures-sorting-interview-cheatsheet.md](#csharp-data-structures-sorting-interview-cheatsheet-md)
- [algorithmic_techniques_guide.md](#algorithmic-techniques-guide-md)
- [C# Data Structures Guide.pdf](#c-data-structures-guide-pdf)
---

<a id="csharp-data-structures-sorting-interview-cheatsheet-md"></a>

## Source: C# Data Structures, Heaps, Comparers, and Sorting — Interview Cheat Sheet

*Imported from `csharp-data-structures-sorting-interview-cheatsheet.md`.*

### C# Data Structures, Heaps, Comparers, and Sorting — Interview Cheat Sheet

Target: modern C# / .NET (including `PriorityQueue<TElement, TPriority>` in .NET 6+).

#### 1. Comparison rules: the foundation

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

#### 2. Array and List sorting

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

##### Sorting objects

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

##### LINQ sorting (creates an ordered sequence)

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

#### 3. `Comparer<T>`, `IComparer<T>`, and `IComparable<T>`

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

#### 4. Priority queues and binary heaps

`PriorityQueue<TElement, TPriority>` is available in .NET 6+. It removes the element with the **lowest priority value first** by default. It is a min-priority queue, not a fully sorted collection.

##### Min-heap: smallest value first

```csharp
var minHeap = new PriorityQueue<int, int>();
foreach (int n in new[] { 5, 1, 8, 2 })
    minHeap.Enqueue(n, n);

int smallest = minHeap.Peek();    // 1, does not remove it
int removed = minHeap.Dequeue();  // 1, removes it

while (minHeap.TryDequeue(out int element, out int priority))
    Console.WriteLine($"{element}, priority={priority}");
```

##### Max-heap for integers

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

##### Custom priority with objects

```csharp
public sealed record Job(string Name, int Priority);

var jobs = new PriorityQueue<Job, int>(); // lower number runs first
jobs.Enqueue(new Job("Report", 3), 3);
jobs.Enqueue(new Job("Urgent fix", 1), 1);
jobs.Enqueue(new Job("Cleanup", 5), 5);

while (jobs.TryDequeue(out Job? job, out int priority))
    Console.WriteLine($"{priority}: {job?.Name}");
```

##### Custom priority ordering with a comparer

```csharp
// Higher numeric priority first:
var jobsMaxFirst = new PriorityQueue<Job, int>(
    Comparer<int>.Create((a, b) => b.CompareTo(a)));

jobsMaxFirst.Enqueue(new Job("Normal", 2), 2);
jobsMaxFirst.Enqueue(new Job("Critical", 10), 10);
```

##### Useful `PriorityQueue` members

- `Enqueue(element, priority)`: insert.
- `Dequeue()`: remove and return the next element; throws if empty.
- `TryDequeue(out element, out priority)`: safe removal; returns `false` if empty.
- `Peek()`: inspect next element; throws if empty.
- `TryPeek(out element, out priority)`: safe inspection.
- `EnqueueDequeue(element, priority)`: enqueue and then remove the minimum-priority item as one operation.
- `DequeueEnqueue(element, priority)`: remove the current minimum, then enqueue the supplied item.
- `Clear()`, `Count`, `EnsureCapacity(capacity)`, `TrimExcess()`, `UnorderedItems`.

The queue does **not** promise sorted enumeration. Equal priorities do not guarantee FIFO order. Add a sequence number to the priority if stable tie-breaking is required.

##### Top-K pattern (keep the K largest numbers)

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

#### 5. Common data structures and when to use them

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

#### 6. Everyday collection operations

##### Arrays and lists

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

##### Dictionary

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

##### HashSet operations

```csharp
var left = new HashSet<int> { 1, 2, 3 };
var right = new HashSet<int> { 3, 4, 5 };

left.UnionWith(right);        // {1,2,3,4,5}
left.IntersectWith(right);    // retain common values
left.ExceptWith(right);       // remove values present in right
bool isPresent = left.Contains(3);
```

Set operation methods mutate the receiver. Use `left.Union(right)` / `Intersect` / `Except` with LINQ when a new sequence is preferred.

##### Queue and stack

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

#### 7. LINQ utilities commonly used in interviews

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

#### 8. Binary search on sorted data

```csharp
int[] sorted = { 1, 3, 5, 7, 9 };
int index = Array.BinarySearch(sorted, 7); // index 3 if found
int missing = Array.BinarySearch(sorted, 6); // negative encoded insertion point
```

The input must already be sorted with the same ordering used for the search. For insertion point when not found: `int insertionIndex = ~missing;`.

#### 9. Interview patterns to recognize

- **Fast membership / deduplication:** `HashSet<T>`.
- **Count frequencies / pair lookup:** `Dictionary<TKey,TValue>`.
- **BFS / shortest path in an unweighted graph:** `Queue<T>`.
- **DFS / matching brackets / monotonic stack:** `Stack<T>`.
- **Repeated minimum/maximum, Dijkstra, scheduling, Top-K:** `PriorityQueue<TElement,TPriority>`.
- **Sorted unique values:** `SortedSet<T>`.
- **Sort once, then binary search / two pointers:** array or list + sort.
- **Sliding window:** array/list plus two pointers; sometimes a dictionary or deque-like structure.
- **Custom ordering:** comparison lambda, `Comparer<T>.Create`, or `IComparer<T>`.

#### 10. Pitfalls worth remembering

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

#### 11. Quick templates

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

---

<a id="algorithmic-techniques-guide-md"></a>

## Source: Core Algorithmic Techniques: A Practical Guide

*Imported from `algorithmic_techniques_guide.md`.*

### Core Algorithmic Techniques: A Practical Guide

This guide explains **recursion, divide and conquer, backtracking, greedy algorithms, and dynamic programming** with self-contained text diagrams, examples, C# code, and a comparison table.

#### 1. The big picture

These terms describe related but different things:

- **Recursion** is a programming technique: a function calls itself.
- **Divide and conquer** is a strategy: split a problem into independent smaller problems, solve them, then combine their answers.
- **Backtracking** is a search strategy: build a candidate solution, abandon it when it cannot work, and try another choice.
- **Greedy** is a decision strategy: make the best-looking choice at each step, without revisiting earlier choices.
- **Dynamic programming (DP)** is an optimization strategy: solve overlapping subproblems and save their answers so they are not repeatedly recomputed.

A technique is not necessarily an alternative to all the others. For example, divide and conquer and backtracking often use recursion. Dynamic programming can also be implemented recursively with memoization.

#### 2. Recursion

##### What it means

A recursive function solves a problem by calling itself on a smaller input. It needs:

1. A **base case** that stops the calls.
2. A **recursive case** that makes progress toward the base case.

Without a reachable base case, the calls may continue until a stack overflow occurs.

##### Example: factorial

For a positive integer `n`:

`n! = n × (n - 1)!`, and `0! = 1`.

```csharp
static int Factorial(int n)
{
    if (n < 0)
        throw new ArgumentOutOfRangeException(nameof(n));

    if (n <= 1)             // Base case
        return 1;

    return n * Factorial(n - 1); // Recursive case
}
```

##### Call-and-return diagram for `Factorial(4)`

```text
Calls go down
Factorial(4)
  = 4 × Factorial(3)
        = 3 × Factorial(2)
              = 2 × Factorial(1)
                    = 1             ← base case

Returns go back up
Factorial(1) = 1
Factorial(2) = 2 × 1 = 2
Factorial(3) = 3 × 2 = 6
Factorial(4) = 4 × 6 = 24
```

Each call waits for the smaller call to return. This uses the call stack.

**Use recursion when:** the problem naturally has nested or smaller versions of itself, such as traversing a tree, walking directories, or processing nested structures.

**Important:** recursion is not automatically an efficient algorithm. The plain recursive Fibonacci implementation, for example, repeats work.

#### 3. Divide and conquer

##### What it means

Divide and conquer breaks a problem into smaller, usually independent subproblems, solves them, and combines their results.

The usual pattern is:

1. **Divide** the input.
2. **Conquer** each part, often recursively.
3. **Combine** the partial answers.

##### Example: merge sort

Merge sort sorts an array by splitting it into halves, sorting each half, and merging the sorted halves.

```text
Start:              [8, 3, 2, 9, 7, 1, 5, 4]
                         /              \
Divide:           [8, 3, 2, 9]      [7, 1, 5, 4]
                    /      \           /      \
                  [8, 3]  [2, 9]     [7, 1]  [5, 4]
                  /  \     /  \       /  \     /  \
                 [8] [3]  [2] [9]    [7] [1]  [5] [4]

Merge sorted pairs:
                 [3, 8]  [2, 9]     [1, 7]  [4, 5]

Merge larger parts:
                [2, 3, 8, 9]         [1, 4, 5, 7]

Final merge:
                  [1, 2, 3, 4, 5, 7, 8, 9]
```

The single-item arrays are already sorted. The algorithm then combines small sorted arrays into larger sorted arrays.

**Typical complexity:** merge sort takes `O(n log n)` time and generally `O(n)` extra space for an array of `n` elements.

**Use divide and conquer when:** subproblems can be solved independently and their results can be combined efficiently. Other examples include binary search and many geometric algorithms.

#### 4. Backtracking

##### What it means

Backtracking searches through possible choices while building a solution step by step. If a partial solution violates a constraint or cannot lead to a valid answer, undo the last choice and try another.

A common pattern is:

```text
Search(state):
    if state is a complete solution:
        report or return it

    for each possible choice:
        if choice is valid:
            make the choice
            Search(updated state)
            undo the choice       ← backtrack
```

The undo step is essential when the same state is reused to explore another branch.

##### Example: finding a path through a maze

`S` is the start, `G` is the goal, `#` is a wall, and `.` is an open cell.

```text
+---+---+---+---+---+
| S | . | # | . | . |
+---+---+---+---+---+
| # | . | # | . | # |
+---+---+---+---+---+
| . | . | . | . | G |
+---+---+---+---+---+
```

One possible exploration:

```text
1. Start at S.
2. Move to an adjacent open cell.
3. Continue while the move is legal and the cell has not been visited.
4. If trapped, return to the previous cell and try another direction.
5. Stop when G is reached, or when all choices have been exhausted.
```

A backtracking search can explore a path, discover a dead end, and undo its previous move. For a real maze solver, track visited cells to avoid cycling; whether to unmark a cell during backtracking depends on the exact problem and whether paths may revisit cells.

**Use backtracking when:** you must explore combinations or arrangements subject to constraints, such as Sudoku, N-Queens, permutations, and finding paths.

**Performance:** the search may be exponential in the worst case. Pruning invalid partial solutions early can make it much faster.

#### 5. Greedy algorithms

##### What it means

A greedy algorithm chooses what seems best **right now**, then continues from that choice. It normally does not reconsider earlier decisions.

A greedy strategy is fast and simple when the problem has the right mathematical properties, but a locally best choice does not always lead to the globally best answer.

##### Example A: coin change where greedy works

Suppose the available coin values are `{10, 5, 1}` and the target is `18`.

```text
Remaining amount     Choice      Coins chosen
18                   10          [10]
 8                    5           [10, 5]
 3                    1           [10, 5, 1]
 2                    1           [10, 5, 1, 1]
 1                    1           [10, 5, 1, 1, 1]
 0                    Done        5 coins total
```

Here, repeatedly taking the largest coin that does not exceed the remaining amount gives a minimum-coin solution.

##### Example B: a case where greedy fails

Now suppose the coin values are `{1, 3, 4}` and the target is `6`.

```text
Greedy:  take 4, then 1, then 1  → 3 coins
Optimal: take 3, then 3          → 2 coins
```

So greedy is **not** a universally correct solution to coin change. It works for some coin systems, not all.

**Use greedy when:** you can justify that each local choice is safe and leads to an optimal overall solution. Well-known examples include interval scheduling (under the correct selection rule), Kruskal's minimum spanning tree algorithm, and Huffman coding.

#### 6. Dynamic programming (DP)

##### What it means

Dynamic programming is useful when a problem has:

- **Overlapping subproblems:** the same smaller question appears repeatedly.
- **Optimal substructure:** a larger answer can be built from answers to smaller subproblems.

DP stores answers to smaller subproblems, avoiding repeated work. Two common forms are:

- **Memoization (top-down):** use recursion and cache each result.
- **Tabulation (bottom-up):** fill a table from the smallest cases upward.

DP does not automatically guarantee an optimal result for every problem; the recurrence and state must correctly represent the problem.

##### Example: Fibonacci numbers

The sequence starts `0, 1, 1, 2, 3, 5, 8, ...`, with:

`F(n) = F(n - 1) + F(n - 2)` for `n >= 2`.

Plain recursion repeats many calculations:

```text
                 Fib(5)
                /      \
            Fib(4)     Fib(3)  ← repeated work
            /   \       /   \
        Fib(3) Fib(2) Fib(2) Fib(1)
          ...
```

The same values, such as `Fib(3)` and `Fib(2)`, are computed more than once.

With memoization, each value is calculated once and reused:

```text
Fib(5) needs Fib(4) and Fib(3)
Fib(4) needs Fib(3) and Fib(2)
Fib(3) is calculated once and cached
Later requests for Fib(3) reuse the cached value
```

##### C# example: bottom-up DP

```csharp
static long Fibonacci(int n)
{
    if (n < 0)
        throw new ArgumentOutOfRangeException(nameof(n));

    if (n <= 1)
        return n;

    long previous = 0; // F(0)
    long current = 1;  // F(1)

    for (int i = 2; i <= n; i++)
    {
        long next = previous + current;
        previous = current;
        current = next;
    }

    return current;
}
```

This version uses `O(n)` time and `O(1)` extra space. Note that `long` still overflows for sufficiently large Fibonacci indices.

**Use DP when:** the problem contains repeated subproblems and the answer can be assembled from smaller states. Common examples include the 0/1 knapsack problem, longest common subsequence, and many path-counting problems.

#### 7. Side-by-side comparison

| Technique | Main idea | Repeats work? | Revisits choices? | Typical example |
|---|---|---|---|---|
| Recursion | A function calls itself | It can | Not inherently | Factorial, tree traversal |
| Divide and conquer | Split, solve independently, combine | Usually avoids overlap by design | Not usually | Merge sort |
| Backtracking | Explore choices; undo invalid or unproductive choices | Often explores many states | Yes | Sudoku, N-Queens |
| Greedy | Choose the best immediate option | Usually little repeated work | Usually no | Kruskal's algorithm |
| Dynamic programming | Cache or tabulate overlapping subproblems | Avoids repeated subproblem calculations | It compares states, not necessarily paths | Knapsack, LCS |

#### 8. How to choose an approach

Ask these questions:

```text
Does the solution naturally call the same function on a smaller input?
    → Consider recursion.

Can the problem be split into independent parts and combined?
    → Consider divide and conquer.

Must you try different choices and reject invalid partial solutions?
    → Consider backtracking.

Can you prove that the best immediate choice is always safe?
    → Consider greedy.

Do smaller subproblems overlap, and can their answers be reused?
    → Consider dynamic programming.
```

These are clues, not strict rules. Some problems support more than one approach, with different trade-offs in runtime, memory, and implementation complexity.

#### 9. Key distinctions to remember

- **Recursion is a technique;** the others are problem-solving strategies or paradigms.
- **Divide and conquer** typically solves independent subproblems and combines their answers.
- **Backtracking** searches alternatives and undoes choices when a path cannot work.
- **Greedy** commits to locally attractive choices; correctness requires a proof or a known property of the problem.
- **Dynamic programming** remembers subproblem answers to avoid recalculating them; it is especially useful when subproblems overlap.

The most important skill is not memorizing labels. It is recognizing the structure of a problem, choosing a suitable approach, and checking whether the approach guarantees the answer you need.

---

<a id="c-data-structures-guide-pdf"></a>

## Source: C# Data Structures Guide

*Imported from `C# Data Structures Guide.pdf`.*

May 21, 2025 at 8:27 AM


You said:


                             list of all C# data structure like— string, date, list,
                             dictionary, set, stack, queue, praiorityQueue (min heap,
                             max heap), linked list, tree, try, tuple, record. with example
                             and all usefull function associated with them are need in
                             problem solving. as well ass list all i missed . dont miss
                             anything that we get after clicking dot after the data
                             structure

   Memory updated

ChatGPT said:

Here is a comprehensive guide to all core C# data structures, including examples and
commonly used members (properties, methods, and operators)—ideal for problem-
solving, interviews, and advanced development:




✅ 1.        string
Immutable sequence of characters.


       C#

  string name = "Afifa";



Useful members:


       C#

  name.Length
  name.ToUpper(), name.ToLower()
  name.Contains("fi")
  name.StartsWith("A"), name.EndsWith("a")
  name.IndexOf("i"), name.LastIndexOf("f")
  name.Substring(1, 2)
  name.Replace("fi", "xy")
  name.Split(' ')
  string.Join(",", array)
  string.IsNullOrEmpty(name), string.IsNullOrWhiteSpace(name)




✅ 2.   DateTime
Used for date and time manipulation.


       C#

  DateTime now = DateTime.Now;
  DateTime specific = new DateTime(2024, 12, 25);



Useful members:


       C#

  now.Year, now.Month, now.Day
  now.AddDays(5), now.AddMonths(-1)
  now.Subtract(anotherDate)
  now.ToString("yyyy-MM-dd")
  DateTime.TryParse("2024-05-01", out var dt)




✅ 3.   List<T>
Dynamic array.


       C#

  List<int> numbers = new() {1, 2, 3};



Useful members:


       C#

  numbers.Add(4), numbers.AddRange(new[] {5, 6})
  numbers.Remove(3), numbers.RemoveAt(0)
  numbers.Insert(1, 99)
  numbers.Contains(2)
  numbers.IndexOf(2)
  numbers.Sort(), numbers.Reverse()
  numbers.Count, numbers.Clear()
  numbers.ForEach(Console.WriteLine)
✅ 4.   Dictionary<TKey, TValue>
Key-value pairs.


       C#

  var dict = new Dictionary<string, int> { ["apple"] = 3 };



Useful members:


       C#

  dict.Add("banana", 5)
  dict.Remove("apple")
  dict.ContainsKey("banana")
  dict.ContainsValue(5)
  dict["banana"] = 6
  dict.Keys, dict.Values
  dict.TryGetValue("banana", out var value)




✅ 5.   HashSet<T>
Unordered collection with unique elements.


       C#

  HashSet<int> set = new() {1, 2, 3};



Useful members:


       C#

  set.Add(4), set.Remove(2)
  set.Contains(3)
  set.UnionWith(otherSet), set.IntersectWith()
  set.ExceptWith(), set.SymmetricExceptWith()




✅ 6.   Stack<T>
LIFO data structure.
       C#
  Stack<int> stack = new();
  stack.Push(1);
  int top = stack.Pop();



Useful members:


       C#

  stack.Push(5), stack.Pop()
  stack.Peek(), stack.Contains(3)
  stack.Count, stack.Clear()




✅ 7.   Queue<T>
FIFO data structure.


       C#

  Queue<string> queue = new();
  queue.Enqueue("task1");
  string next = queue.Dequeue();



Useful members:


       C#

  queue.Enqueue(), queue.Dequeue()
  queue.Peek(), queue.Contains()
  queue.Count, queue.Clear()




✅ 8.   PriorityQueue<TElement, TPriority> (Min-Heap by default)


       C#

  var pq = new PriorityQueue<string, int>();
  pq.Enqueue("task1", 1);
  pq.Enqueue("task2", 0);



Useful members:
       C#
  pq.Enqueue("item", priority)
  pq.Dequeue(), pq.Peek()
  pq.Count, pq.Clear()



➡️ Max-Heap workaround:
       C#

  pq.Enqueue("item", -priority)   // invert the priority




✅ 9.    LinkedList<T>


       C#

  LinkedList<int> list = new();
  list.AddLast(1); list.AddFirst(0);



Useful members:


       C#

  list.AddFirst(value), list.AddLast(value)
  list.AddBefore(node, value), list.AddAfter(node, value)
  list.RemoveFirst(), list.RemoveLast()
  list.Find(value), list.Contains(value)




✅ 10.       Tuple<T1, T2, ..., T7>


       C#

  var t = Tuple.Create("Afifa", 100);
  var name = t.Item1;



For better readability use ValueTuple :


       C#

  (string name, int score) = ("Afifa", 100);
✅ 11.       Record (C# 9+)
Immutable reference type with value-based equality.


       C#

  public record Person(string Name, int Age);
  var p1 = new Person("Afifa", 25);



Benefits:
      Immutable
      Structural equality
      Deconstruction




✅ 12.       Tree (Custom)


       C#

  class TreeNode {
       public int Value;
       public List<TreeNode> Children = new();
  }



Useful for hierarchical data like file systems, org charts, etc.




✅ 13.       Trie (Custom)
Prefix tree used in string searching/autocomplete.


       C#

  class TrieNode {
       public Dictionary<char, TrieNode> Children = new();
       public bool IsWord = false;
  }




✅ 14.       Array
      C#
  int[] arr = {1, 2, 3};



Useful members:


      C#

  arr.Length
  Array.Sort(arr), Array.Reverse(arr)
  Array.IndexOf(arr, 2)
  Array.Copy(source, dest, length)




✅ 15.      Span<T> / ReadOnlySpan<T>
Efficient memory slicing (C# 7.2+).


      C#

  Span<int> span = stackalloc int[3];



Use in performance-critical scenarios.




✅ 16.      IEnumerable<T> / IQueryable<T>
Used for LINQ queries and deferred execution.


      C#

  var filtered = numbers.Where(x => x > 2);




✅ 17.      ObservableCollection<T>
Collection with change notifications (used in WPF).


      C#

  ObservableCollection<string> names = new();
  names.CollectionChanged += ...
✅ 18.       SortedList<TKey, TValue> / SortedDictionary<TKey, TValue>
Keeps keys in sorted order.


       C#

  SortedList<string, int> sorted = new();




✅ 19.       ConcurrentDictionary , ConcurrentQueue , etc.
Thread-safe collections.


       C#

  ConcurrentDictionary<string, int> cache = new();




✅ 20.       ImmutableList<T> , ImmutableDictionary<TKey, TValue>
Immutable versions for functional-style programming.




✅ 21.       BitArray
Efficient binary flags.


       C#

  BitArray flags = new(10); flags[0] = true;




🔍 IntelliSense Tips
For any structure:


       C#

  var s = new Stack<int>();
  s.   // Press dot to see all members like Push, Pop, Count



Use Ctrl+. in Visual Studio or type . to explore all properties, methods, extension
methods.
🔚 Summary Table
Data Structure           Namespace                Purpose

string                   System                   Text


DateTime                 System                   Time


List<T>                  System.Collections.Gen   Dynamic array
                         eric


Dictionary<K,V>          System.Collections.Gen   Key-value
                         eric


HashSet<T>               System.Collections.Gen   Unique set
                         eric


Stack<T>                 System.Collections.Gen   LIFO
                         eric


Queue<T>                 System.Collections.Gen   FIFO
                         eric


PriorityQueue<TElement   System.Collections.Gen   Min-Heap
, TPriority>             eric


LinkedList<T>            System.Collections.Gen   Bi-directional nodes
                         eric


Tuple , ValueTuple       System                   Group values


Record                   System                   Immutable type


Array , Span<T>          System                   Performance


Trie , Tree              Custom                   Hierarchical/string search


Concurrent*              System.Collections.Con   Thread-safe
                         current


Immutable*               System.Collections.Imm   Immutable variants
                         utable
Data Structure                Namespace                        Purpose

 BitArray                     System.Collections               Bits


 SortedList ,                 System.Collections.Gen           Sorted
 SortedDictionary             eric




Would you like a downloadable PDF or code snippets in a GitHub Gist for this?

You said:


                             rewrite all with example, you listout but missed give
                             example as well as you missed many thing like array.sort()
                             or revers() with param, max heap (a,b) = > (a-b) like
                             function... array.fill(with param) linked list, double linked list
                             etc


ChatGPT said:

You're right—let’s now provide complete examples with code, IntelliSense-accessible
members, and problem-solving tips for each C# data structure. I’ll also cover missing
structures like Array.Sort(…) , Array.Fill(…) , SortedSet , Deque , MaxHeap ,
etc.




🔹 1.        string


       C#

  string name = "Afifa";
  Console.WriteLine(name.ToUpper());         // AFIFA
  Console.WriteLine(name.Contains("fi"));    // True
  Console.WriteLine(name.Substring(1, 2));   // fi



Useful members:


       C#

  Length, Contains, Substring, IndexOf, Replace, Split, StartsWith, EndsWith, ToUpper, ToLo
🔹 2.   DateTime


     C#

 DateTime now = DateTime.Now;
 DateTime birthday = new DateTime(1993, 12, 25);
 var tomorrow = now.AddDays(1);
 Console.WriteLine(birthday.DayOfWeek);   // Saturday



Useful members:


     C#

 Year, Month, Day, AddDays, AddMonths, Subtract, ToString("yyyy-MM-dd"), TryParse




🔹 3.   Array


     C#

 int[] arr = { 5, 2, 3, 1 };


 // Sort ascending
 Array.Sort(arr);
 Console.WriteLine(string.Join(", ", arr)); // 1, 2, 3, 5


 // Sort descending
 Array.Sort(arr, (a, b) => b - a);


 // Fill array
 Array.Fill(arr, 9); // All elements become 9


 // Reverse
 Array.Reverse(arr);




🔹 4.   List<T>


     C#

 List<int> numbers = new() { 1, 3, 5 };
 numbers.Add(7);
 numbers.Insert(1, 2);   // [1, 2, 3, 5, 7]
 numbers.Sort();        // Ascending
 numbers.Sort((a, b) => b - a);   // Descending



Members:


      C#

 Add, AddRange, Remove, RemoveAt, Contains, IndexOf, Insert, Sort, Reverse, Clear, Count,




🔹 5.    Dictionary<TKey, TValue>


      C#

 Dictionary<string, int> dict = new()
 {
      ["apple"] = 3,
      ["banana"] = 2
 };
 dict["banana"] = 5;
 if (dict.TryGetValue("banana", out var val)) Console.WriteLine(val);




🔹 6.    HashSet<T> – Unordered, unique


      C#

 HashSet<int> set = new() { 1, 2, 3 };
 set.Add(3);   // ignored
 set.UnionWith(new[] { 4, 5 }); // adds 4 and 5




🔹 7.    SortedSet<T> – Sorted, unique


      C#

 SortedSet<int> sorted = new() { 3, 1, 2 };
 foreach (var i in sorted) Console.WriteLine(i); // 1 2 3




🔹 8.    Stack<T> – LIFO
    C#

 Stack<int> stack = new();
 stack.Push(10);
 int last = stack.Pop();      // 10




🔹 9.    Queue<T> – FIFO


    C#

 Queue<string> queue = new();
 queue.Enqueue("first");
 string item = queue.Dequeue();   // "first"




🔹 10.    PriorityQueue<TElement, TPriority> – Min-Heap


    C#

 PriorityQueue<string, int> pq = new();
 pq.Enqueue("low", 1);
 pq.Enqueue("high", 0);
 Console.WriteLine(pq.Dequeue()); // "high"



🔸 Max Heap:
    C#

 PriorityQueue<string, int> maxHeap = new(Comparer<int>.Create((a, b) => b - a));
 maxHeap.Enqueue("high", 10);
 maxHeap.Enqueue("low", 1);




🔹 11.    LinkedList<T> – Double-linked list


    C#

 var list = new LinkedList<int>();
 list.AddFirst(1);
 list.AddLast(3);
 var two = list.AddAfter(list.First, 2);   // 1 -> 2 -> 3
🔹 12.     Tuple<T1, T2> and ValueTuple


     C#

 var tuple = Tuple.Create("Afifa", 11);
 var (name, age) = ("Afifa", 11); // ValueTuple




🔹 13.     Record – Immutable value object (C# 9+)


     C#

 public record Person(string Name, int Age);


 var p1 = new Person("Afifa", 30);
 var p2 = p1 with { Age = 31 };




🔹 14.     Tree (Custom)


     C#

 class TreeNode {
     public int Val;
     public List<TreeNode> Children = new();
 }




🔹 15.     Trie (Prefix tree for strings)


     C#

 class TrieNode {
     public Dictionary<char, TrieNode> Children = new();
     public bool IsEndOfWord;
 }




🔹 16.     Span<T> – Efficient slicing (C# 7.2+)
    C#

 Span<int> s = stackalloc int[3];
 s[0] = 1;




🔹 17.    ObservableCollection<T> – CollectionChanged for WPF


    C#

 var collection = new ObservableCollection<string>();
 collection.CollectionChanged += (s, e) => Console.WriteLine("Changed");
 collection.Add("Afifa");




🔹 18.    SortedList<TKey, TValue> vs SortedDictionary<TKey,
TValue>


    C#

 SortedList<string, int> sortedList = new() { ["a"] = 1, ["b"] = 2 };
 SortedDictionary<string, int> sortedDict = new() { ["a"] = 1, ["b"] = 2 };




🔹 19.    ConcurrentDictionary<TKey, TValue> – Thread-safe


    C#

 ConcurrentDictionary<string, int> cd = new();
 cd.TryAdd("x", 1);




🔹 20.    ImmutableList<T> – Functional programming


    C#

 var list = ImmutableList.Create<int>(1, 2, 3);
 var newList = list.Add(4); // returns new list
🔹 21.      BitArray – Efficient flags


      C#

  BitArray flags = new(8);
  flags[0] = true;
  Console.WriteLine(flags[0]); // True




✅ Summary Table
Data Structure                Usage                   Key Functions

string                        Text                    Substring, Replace, IndexOf


DateTime                      Dates/times             AddDays, Subtract, ToString


Array                         Static list             Sort, Reverse, Fill


List<T>                       Dynamic list            Add, Insert, Sort, Remove


Dictionary<K,V>               Key-value pairs         TryGetValue, ContainsKey


HashSet<T>                    Unique values           UnionWith, IntersectWith


SortedSet<T>                  Sorted unique values    Min/Max operations


Stack<T>                      LIFO                    Push, Pop, Peek


Queue<T>                      FIFO                    Enqueue, Dequeue


PriorityQueue<T>              Min/Max heap            Enqueue, Dequeue (with
                                                      comparer)


LinkedList<T>                 Bidirectional list      AddBefore, AddAfter


Tuple , ValueTuple            Grouping values         Item1, Item2 or named
                                                      deconstruction


Record                        Immutable class         with, equality, ToString


Tree , Trie                   Hierarchical / Prefix   Custom classes
                              structures
Data Structure                  Usage                          Key Functions

 Span<T>                        Memory slice (perf)            Efficient iteration


 ObservableCollection<T         WPF UI updates                 CollectionChanged
 >


 SortedList ,                   Auto-sorted                    Keys, Values
 SortedDict


 Concurrent*                    Thread-safe collections        TryAdd, TryRemove


 Immutable*                     Functional style               Add, Remove (new copy)


 BitArray                       Memory-efficient flags         Set, Get




Would you like this as a PDF, cheat sheet, or interactive notebook? I can also help you
prepare questions or practice problems based on these structures.

You said:


                                small = new PriorityQueue<int, int>
                                (Comparer<int>.Create((a, b) => b.CompareTo(a)));
                                is there any alternative


ChatGPT said:

Yes, the line:


        C#

     small = new PriorityQueue<int, int>(Comparer<int>.Create((a, b) => b.CompareTo(a)));



creates a max-heap using PriorityQueue<TElement, TPriority> by reversing the
default comparer. This is correct and idiomatic, but there are a few alternatives and
variations.




✅ Alternative 1: Use            Comparer<int>.Default.Reverse()
This is more expressive and readable:
        C#

  small = new PriorityQueue<int, int>(Comparer<int>.Default.Reverse());




    Reverse() is available on Comparer<T> in .NET 6+.




✅ Alternative 2: Explicit              new Comparer<int> with lambda
If you can't use .Reverse() (e.g., older C# version):


        C#

  small = new PriorityQueue<int, int>(
        Comparer<int>.Create((x, y) => y - x) // or y.CompareTo(x)
  );




✅ Alternative 3: Use negative priorities
Another trick, especially in competitive coding:


        C#

  PriorityQueue<int, int> pq = new();
  pq.Enqueue(10, -10); // store priority as -priority


  var max = pq.Dequeue(); // element with highest original value



But be careful—this assumes:
       You control how priorities are added.
       All priorities are numeric and convertible to negative.




🔁 Summary
Approach                          Code Example                   Notes

Default + Reverse()                Comparer<int>.Default.R       ✅ Clean, readable
                                   everse()
Approach                       Code Example                   Notes

Custom lambda                   Comparer<int>.Create((a       ✅ Common trick
                                , b) => b - a)


Use negative priorities         Enqueue(value, -              ⚠️ Manual, less safe
                                priority)




Let me know if you'd like to wrap this into a reusable helper or extension method for
MaxHeap!


                                       Today 1:54 PM


You said:


                                                        make mark down, avoid duplicate


ChatGPT said:

This response couldn’t load

You said:


                                  make markdown file , avoid duplicate information If any


Analyzing

---

## Additional Questpond question sets

### Questpond source: 10-scenario-based.md

---



---

#### Scenario-based questions

Answers below use the supplied examples. Source page numbers refer to the PDF's 1-based pages.

##### Arrays and strings

###### Sort by descending frequency, breaking ties by descending value *(PDF p.16)*

Input: `{2,2,5,6,1,3,3,6,6,5,4,5,5}`. Count each value, then sort by count descending and value descending.

```csharp
int[] values = { 2, 2, 5, 6, 1, 3, 3, 6, 6, 5, 4, 5, 5 };
var counts = new Dictionary<int, int>();
foreach (int x in values)
    counts[x] = counts.TryGetValue(x, out int n) ? n + 1 : 1;

Array.Sort(values, (a, b) =>
{
    int byFrequency = counts[b].CompareTo(counts[a]);
    return byFrequency != 0 ? byFrequency : b.CompareTo(a);
});
// {5,5,5,5,6,6,6,3,3,2,2,4,1}
```

This uses `Array.Sort` for clarity; a custom sort can compare the same two keys if built-ins are disallowed.

###### Sort an array in ascending order *(PDF p.16)*

Input `{3,7,2,4,5,1}` → `{1,2,3,4,5,7}`. For example, `Array.Sort(values)`; a comparison sort takes `O(n log n)`.

###### Remove duplicate values while preserving first-seen order *(PDF p.16)*

Input `{3,7,3,2,1,4,5,5,2,2}` → `{3,7,2,1,4,5}`.

```csharp
var seen = new HashSet<int>();
var unique = new List<int>();
foreach (int x in new[] { 3, 7, 3, 2, 1, 4, 5, 5, 2, 2 })
    if (seen.Add(x)) unique.Add(x);
// unique: 3, 7, 2, 1, 4, 5
```

###### Count each character in `"hello world"` *(PDF pp.16–17)*

Count all characters including spaces; the source's example output omits the space. In first-seen order, the non-space counts are `h:1, e:1, l:3, o:2, w:1, r:1, d:1`.

```csharp
var frequency = new Dictionary<char, int>();
foreach (char ch in "hello world")
    frequency[ch] = frequency.TryGetValue(ch, out int n) ? n + 1 : 1;
```

###### Print the most frequent character *(PDF p.17)*

For `"hello world"`, the answer is `l:3`. If multiple characters tie, return all tied characters or define a tie-break rule; none is specified in the source.

###### Encode a string as consecutive character/count pairs *(PDF p.17)*

`AACBAEFEEH` → `A2C1B1A1E1F1E2H1`. This is run-length encoding (not a count of all occurrences of each character).

```csharp
static string Encode(string s)
{
    if (s.Length == 0) return "";
    var result = new StringBuilder();
    for (int i = 0; i < s.Length;)
    {
        int j = i + 1;
        while (j < s.Length && s[j] == s[i]) j++;
        result.Append(s[i]).Append(j - i);
        i = j;
    }
    return result.ToString();
}
```

###### Decode character/count pairs *(PDF p.17)*

`A2C3B1A1` → `AACCCBA`.

```csharp
static string Decode(string encoded)
{
    var result = new StringBuilder();
    for (int i = 0; i < encoded.Length;)
    {
        char ch = encoded[i++];
        if (!char.IsAsciiLetter(ch)) throw new FormatException("Expected a letter.");
        if (i == encoded.Length || !char.IsAsciiDigit(encoded[i]))
            throw new FormatException("Expected a positive count.");
        int count = 0;
        while (i < encoded.Length && char.IsAsciiDigit(encoded[i]))
            count = checked(count * 10 + encoded[i++] - '0');
        if (count < 1) throw new FormatException("Count must be positive.");
        result.Append(ch, count);
    }
    return result.ToString();
}
```

Assumption: input consists of alternating ASCII letters and positive decimal counts; malformed input is rejected.

###### Check whether two strings are anagrams *(PDF p.20)*

Examples: `abac`/`baac`, `rat`/`tar`, and `India`/`diani` are anagrams; `abc`/`cbcaa` are not. The examples imply case-insensitive comparison. Count each character and compare counts in `O(n)` time; ignore or include spaces/punctuation according to the desired policy (not specified here).

```csharp
static bool AreAnagrams(string a, string b)
{
    if (a.Length != b.Length) return false;
    var counts = new Dictionary<char, int>();
    foreach (char raw in a)
    {
        char c = char.ToUpperInvariant(raw);
        counts[c] = counts.TryGetValue(c, out int n) ? n + 1 : 1;
    }
    foreach (char raw in b)
    {
        char c = char.ToUpperInvariant(raw);
        if (!counts.TryGetValue(c, out int n)) return false;
        if (n == 1) counts.Remove(c); else counts[c] = n - 1;
    }
    return counts.Count == 0;
}
```

##### Pattern-printing prompts

###### Print the number pattern *(PDF pp.17–18)*

The PDF supplies the output but no rule/input for generating it. Preserve the specified output:

```text
2
14
365
8 7 10 9
12 11 14 13 16
```

###### Print the 5-row asterisk pattern *(PDF p.18)*

```text
*
**
***
**
*
```

Print rows with widths `1, 2, 3, 2, 1`.

###### Print the second spaced asterisk pattern *(PDF p.18)*

The source's spacing is approximate; this retains the displayed shape:

```text
     *
 *       *
* *          *
 *       *
     *
```

The intended alignment/spacing beyond the shown sample is not defined.

##### PAN textbox masking *(PDF p.20)*

For the example PAN `XYGPN1122P`, the display is `XYG***` and the database value remains `XYGPN1122P`. Keep the unmasked value in the application model/state, persist that value, and derive the masked display from it; never replace the stored PAN with the masked text. While the user types, mask characters after the first three in the UI. A masked value cannot be used to reconstruct the original PAN, so edits should be handled with a secure re-entry flow. The PDF does not explain why only three asterisks are shown for the seven hidden characters.

##### Architecture and platform scenarios

###### How would you convert a current project to microservices? *(PDF p.20)*

First identify business capabilities and boundaries, then extract one bounded context at a time behind stable APIs/events. Avoid a big-bang rewrite: use a strangler migration, independent deployment pipelines, observability, and contract tests. Give each service clear ownership of its data; plan for distributed transactions, retries/idempotency, security, and operational overhead before splitting.

###### Design a high-traffic service with concurrent requests and protect data integrity *(PDF p.20)*

Scale stateless app instances horizontally behind a load balancer; use caching and queues where appropriate, and apply backpressure/rate limits. Protect writes with database transactions and suitable constraints/optimistic concurrency (or row-level locking where justified), make operations idempotent, and use bounded retries. Measure latency/errors/contention and test peak load; do not rely on in-process locks across multiple instances.

###### How do you write middleware in .NET Core? *(PDF p.20)*

Implement a class with `RequestDelegate` and `InvokeAsync(HttpContext)`, or use `app.Use(...)`. Call the next delegate to continue the pipeline; register it in the intended order before terminal middleware. Example:

```csharp
public sealed class TimingMiddleware
{
    private readonly RequestDelegate _next;
    public TimingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context, ILogger<TimingMiddleware> logger)
    {
        var timer = System.Diagnostics.Stopwatch.StartNew();
        await _next(context);
        logger.LogInformation("HTTP request took {ElapsedMs} ms", timer.ElapsedMilliseconds);
    }
}
// Registration: app.UseMiddleware<TimingMiddleware>();
```

###### Migrate a .NET 4.5 project to .NET 8 *(PDF p.20)*

Inventory frameworks, packages, OS/hosting assumptions, and unsupported APIs; add tests and establish a baseline. Port/retarget in stages (often first to SDK-style projects), replace incompatible dependencies/APIs, modernize configuration/DI/logging/hosting, and migrate data carefully. Validate behavior and performance in CI and a staging environment, then deploy incrementally with monitoring and rollback. .NET Framework 4.5 code may require redesign rather than a direct retarget.

###### Where do you register `AddScoped` and `AddTransient`? *(PDF p.20)*

Register services in the dependency-injection setup, typically `builder.Services` in `Program.cs` (or `ConfigureServices` in older ASP.NET Core). Use scoped for one instance per web request and transient for a new instance per resolution; choose lifetime based on dependencies and state, not convenience.

###### What design patterns have you used? *(PDF p.20)*

Give truthful, project-specific examples. Common choices include Strategy for interchangeable business rules, Factory for construction, Adapter for integrating a legacy API, Decorator for cross-cutting behavior, and Repository only when it provides a useful abstraction over persistence.

###### What is an SSL certificate? *(PDF p.20)*

An X.509 TLS certificate binds a public key to a verified identity/domain. A trusted certificate chain lets clients authenticate the server and establish encrypted TLS connections. “SSL certificate” is the common term; modern connections use TLS.

###### How is deployment carried out? *(PDF p.20)*

Build and test in CI, package a versioned artifact/container, deploy through controlled environments using configuration/secrets external to the artifact, run migrations safely, verify health/telemetry, and promote gradually with rollback available. Mention the actual hosting platform and approval process used by the team.

###### How do you connect Entity Framework to a database? *(PDF p.21)*

Install the provider, configure a `DbContext` and connection string, then inject the context. For example, with SQL Server:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("App")));
```

Keep credentials in secret/configuration management, not source control; use migrations or an approved schema deployment process.

###### As an architect, what do you ask the customer before development? *(PDF p.21)*

Clarify business goals, users, workflows, scope, success metrics, priorities, and acceptance criteria; functional and non-functional requirements (scale, availability, latency, security, accessibility); data ownership/retention/privacy; integrations and constraints; compliance; target platforms; budget/timeline; deployment/support; and risks/open decisions. Confirm assumptions and obtain sign-off on a prioritized scope.

###### What logging mechanism have you used? *(PDF p.21)*

Describe the system actually used. A sound .NET approach uses `ILogger<T>` with structured logging, centralized collection/search, correlation/trace IDs, appropriate levels, retention and alerting. Avoid logging secrets or sensitive personal data.

###### Have you worked with queues or events? *(PDF p.21)*

Describe a real implementation if applicable. In general, queues decouple producers/consumers and smooth bursts; use durable messages, retries with backoff, dead-letter handling, idempotent consumers, and observability. Events communicate that something happened; define schemas/versioning and delivery semantics explicitly.

---

### Questpond source: 12-coding-questions.md

---



---

#### Coding questions

The PDF's coding section asks for efficient solutions and often says not to rely on built-in convenience functions. Examples use C#; assume valid non-null inputs unless stated. Source page numbers are included with each prompt.

##### 1. Reverse characters in a string; reverse words *(PDF p.22)*

The listed approaches are prepend, a buffer, and a two-pointer swap. Prepending repeatedly is quadratic; use a buffer or swap:

```csharp
static string ReverseByPrepending(string s)
{
    string reversed = "";
    for (int i = 0; i < s.Length; i++) reversed = s[i] + reversed;
    return reversed; // O(n^2) due to repeated string allocation
}

static string ReverseChars(string s)
{
    char[] a = s.ToCharArray();
    for (int l = 0, r = a.Length - 1; l < r; l++, r--)
        (a[l], a[r]) = (a[r], a[l]);
    return new string(a);
}

// Reverse word order; normalize whitespace separators to one space.
static string ReverseWords(string s)
{
    var result = new StringBuilder();
    int i = s.Length - 1;
    while (i >= 0)
    {
        while (i >= 0 && char.IsWhiteSpace(s[i])) i--;
        if (i < 0) break;
        int end = i;
        while (i >= 0 && !char.IsWhiteSpace(s[i])) i--;
        if (result.Length > 0) result.Append(' ');
        for (int j = i + 1; j <= end; j++) result.Append(s[j]);
    }
    return result.ToString();
}
```

For an array, use the same two-pointer swap directly in place. Reverse-word separator behavior was unspecified; this version normalizes whitespace to single spaces.

##### 2. Find occurrence of each character *(PDF p.22)*

Use a dictionary of counts, then iterate the input and emit a character only on its first occurrence to preserve input order. This is `O(n)` time and `O(u)` space, where `u` is the number of distinct characters.

```csharp
static Dictionary<char, int> CharacterCounts(string s)
{
    var counts = new Dictionary<char, int>();
    foreach (char c in s)
        counts[c] = counts.TryGetValue(c, out int n) ? n + 1 : 1;
    return counts;
}
```

##### 3. Find the first non-repeating character *(PDF p.22)*

```csharp
static char? FirstUnique(string s)
{
    var counts = new Dictionary<char, int>();
    foreach (char c in s)
        counts[c] = counts.TryGetValue(c, out int n) ? n + 1 : 1;
    foreach (char c in s)
        if (counts[c] == 1) return c;
    return null;
}
```

##### 4. Find two array numbers that sum to a target *(PDF p.22)*

Return the first pair of values encountered; return `false` if none exists.

```csharp
static bool TryTwoSum(int[] a, int target, out int first, out int second)
{
    var seen = new HashSet<int>();
    foreach (int x in a)
    {
        int complement = target - x;
        if (seen.Contains(complement))
        {
            first = complement; second = x; return true;
        }
        seen.Add(x);
    }
    first = second = 0;
    return false;
}
```

Expected `O(n)` time, `O(n)` additional space; check integer overflow if the input domain can exceed `int`.

##### 5. Check whether a string is a palindrome *(PDF p.22)*

```csharp
static bool IsPalindrome(string s)
{
    for (int l = 0, r = s.Length - 1; l < r; l++, r--)
        if (s[l] != s[r]) return false;
    return true;
}
```

This compares exact characters. To ignore case/punctuation, normalize first; that behavior was not specified.

##### 6. Check anagrams *(PDF p.22)*

Use the character-frequency method from [the scenario answer](10-scenario-based.md#check-whether-two-strings-are-anagrams-pdf-p20). It is `O(n)` time and handles the source's `India`/`diani` example without regard to case.

##### 7. Maximum sum of a contiguous subarray of size `k` *(PDF p.22)*

Fixed-size sliding window, `O(n)` time, `O(1)` space. Require `1 <= k <= a.Length`.

```csharp
static long MaxWindowSum(int[] a, int k)
{
    if (k < 1 || k > a.Length) throw new ArgumentOutOfRangeException(nameof(k));
    long sum = 0;
    for (int i = 0; i < k; i++) sum += a[i];
    long best = sum;
    for (int i = k; i < a.Length; i++)
    {
        sum += (long)a[i] - a[i - k];
        if (sum > best) best = sum;
    }
    return best;
}
```

##### 8. Sort an array containing three unique values; bubble and insertion sorts *(PDF p.22)*

For values `lowValue`, `midValue`, `highValue`, the Dutch National Flag partition is linear:

```csharp
static void SortThreeValues(int[] a, int lowValue, int midValue, int highValue)
{
    int low = 0, i = 0, high = a.Length - 1;
    while (i <= high)
    {
        if (a[i] == lowValue)
        {
            int temp = a[low]; a[low] = a[i]; a[i] = temp;
            low++; i++;
        }
        else if (a[i] == midValue) i++;
        else if (a[i] == highValue)
        {
            int temp = a[i]; a[i] = a[high]; a[high] = temp;
            high--;
        }
        else throw new ArgumentException("Array contains a value outside the three specified values.");
    }
}
```

The source also names bubble sort and insertion sort. These general alternatives are `O(n²)`:

```csharp
static void BubbleSort(int[] a)
{
    for (int end = a.Length - 1; end > 0; end--)
    {
        bool changed = false;
        for (int i = 0; i < end; i++)
            if (a[i] > a[i + 1])
            {
                (a[i], a[i + 1]) = (a[i + 1], a[i]);
                changed = true;
            }
        if (!changed) break;
    }
}

static void InsertionSort(int[] a)
{
    for (int i = 1; i < a.Length; i++)
    {
        int value = a[i], j = i - 1;
        while (j >= 0 && a[j] > value) { a[j + 1] = a[j]; j--; }
        a[j + 1] = value;
    }
}
```

##### 9. Validate parentheses/brackets *(PDF p.22)*

Check matching `()`, `[]`, `{}` using a stack; any other character is ignored.

```csharp
static bool HasValidBrackets(string s)
{
    var stack = new Stack<char>();
    foreach (char c in s)
    {
        if (c is '(' or '[' or '{') stack.Push(c);
        else if (c is ')' or ']' or '}')
        {
            if (stack.Count == 0) return false;
            char open = stack.Pop();
            if ((c == ')' && open != '(') || (c == ']' && open != '[') ||
                (c == '}' && open != '{')) return false;
        }
    }
    return stack.Count == 0;
}
```

##### 10. Basic algorithms: prime, factorial, Fibonacci, remainder, odd/even *(PDF p.22)*

Prime test (`O(√n)`), factorial (checked `long`), and first `count` Fibonacci values:

```csharp
static bool IsPrime(int n)
{
    if (n < 2) return false;
    for (int d = 2; d <= n / d; d++) if (n % d == 0) return false;
    return true;
}

static long Factorial(int n)
{
    if (n < 0) throw new ArgumentOutOfRangeException(nameof(n));
    long result = 1;
    for (int i = 2; i <= n; i++) result = checked(result * i);
    return result; // 0! = 1; 4! = 24
}

static long[] Fibonacci(int count)
{
    if (count < 0) throw new ArgumentOutOfRangeException(nameof(count));
    var result = new long[count];
    long a = 0, b = 1;
    for (int i = 0; i < count; i++)
    {
        result[i] = a;
        if (i + 1 < count) (a, b) = (b, checked(a + b));
    }
    return result; // 0, 1, 1, 2, 3, 5, ...
}
```

Remainder is `n % divisor` (the divisor must not be zero); an integer is even exactly when `n % 2 == 0`. Whole numbers are `0, 1, 2, ...`. A prime is an integer greater than 1 divisible only by 1 and itself.

##### 11. Find the next number in each sequence *(PDF p.22)*

The source recommends checking differences, differences of differences, alternating rules, squares/cubes, gaps, primes and Fibonacci. Under the simplest visible pattern:

| Sequence | Next | Rule |
|---|---:|---|
| `4, 7, 10, 13, 16` | `19` | Add 3 |
| `1, 4, 9, 16` | `25` | Squares |
| `1, 8, 27, 64, 125` | `216` | Cubes |
| `64, 58, 52, 46` | `40` | Subtract 6 |
| `2, 4, 8, 10, 14` | `16` | Assumption: alternating `+2,+4` |
| `1, 4, 10, 19, 31` | `46` | Differences `3,6,9,12,15` |
| `1, 3, 7, 13, 21, 31` | `43` | Differences `2,4,6,8,10,12` |
| `10,21,35,56,90,145,231,360` | Not uniquely determined | No unambiguous rule is specified; many sequences fit these terms. |
| Special series (e.g. prime/Fibonacci) | Depends on the given series | Identify the defining rule first. |

##### 12. Move zeroes to the end of an array *(PDF p.22)*

Stable, in-place `O(n)` solution; nonzero order is preserved:

```csharp
static void MoveZerosToEnd(int[] a)
{
    int write = 0;
    for (int read = 0; read < a.Length; read++)
        if (a[read] != 0) a[write++] = a[read];
    while (write < a.Length) a[write++] = 0;
}
```

##### 13. Rotate an array left/right by `k` places *(PDF p.22)*

Normalize `k` modulo length; use three reversals. Empty arrays are unchanged.

```csharp
static void Rotate(int[] a, int k, bool left)
{
    if (a.Length == 0) return;
    k = ((k % a.Length) + a.Length) % a.Length;
    if (!left) k = a.Length - k;
    ReverseRange(a, 0, k - 1);
    ReverseRange(a, k, a.Length - 1);
    ReverseRange(a, 0, a.Length - 1);
}

static void ReverseRange(int[] a, int l, int r)
{
    while (l < r)
    {
        int temp = a[l]; a[l] = a[r]; a[r] = temp;
        l++; r--;
    }
}
```

For left rotation, reverse the first `k`, the remainder, then the whole array; right rotation is equivalent to left rotation by `n-k`.

##### 14. Longest substring without repeating characters *(PDF p.22)*

Sliding window, `O(n)` time:

```csharp
static int LongestUniqueSubstringLength(string s)
{
    var last = new Dictionary<char, int>();
    int start = 0, best = 0;
    for (int i = 0; i < s.Length; i++)
    {
        if (last.TryGetValue(s[i], out int previous) && previous >= start)
            start = previous + 1;
        last[s[i]] = i;
        best = Math.Max(best, i - start + 1);
    }
    return best;
}
```

##### 15. Reverse an integer without converting to a string *(PDF p.22)*

```csharp
static int ReverseNumber(int n)
{
    int result = 0;
    while (n != 0)
    {
        int digit = n % 10;
        result = checked(result * 10 + digit);
        n /= 10;
    }
    return result;
}
```

Overflow throws; handle it differently if the required contract is to return an error/sentinel.

##### 16. Remove duplicate characters from a string *(PDF p.22)*

Preserve first occurrence order:

```csharp
static string RemoveDuplicateChars(string s)
{
    var seen = new HashSet<char>();
    var result = new StringBuilder();
    foreach (char c in s) if (seen.Add(c)) result.Append(c);
    return result.ToString();
}
```

##### 17. Basic star patterns *(PDF p.22)*

For a centered pyramid of `rows` lines, print `rows - r` spaces and `2*r - 1` stars for each row `r` from 1 to `rows`. The two specific patterns shown in the PDF are reproduced in [the pattern answers](10-scenario-based.md#pattern-printing-prompts).

##### 18. Find the Nth highest salary *(PDF p.22)*

See the `DENSE_RANK` SQL solution in [SQL scenarios](11-sql-scenarios.md#find-the-nth-highest-salary-pdf-p22); it returns the Nth distinct salary.

##### 19. Swap two numbers *(PDF p.22)*

```csharp
(a, b) = (b, a);
```

The tuple assignment is clear and avoids arithmetic-overflow issues.

##### 20. Extract the surname *(PDF p.22)*

Assumption: surname is the final whitespace-separated token; cultural naming conventions may require a different rule.

```csharp
static string Surname(string fullName)
{
    int end = fullName.Length - 1;
    while (end >= 0 && char.IsWhiteSpace(fullName[end])) end--;
    int start = end;
    while (start >= 0 && !char.IsWhiteSpace(fullName[start])) start--;
    return fullName.Substring(start + 1, end - start);
}
```

##### 21. Remove all whitespace from a string *(PDF p.23)*

```csharp
static string RemoveWhitespace(string s)
{
    var result = new StringBuilder(s.Length);
    foreach (char c in s) if (!char.IsWhiteSpace(c)) result.Append(c);
    return result.ToString();
}
```

##### 22. Find the longest palindromic substring *(PDF p.23)*

Expand around every odd/even center. `O(n²)` time, `O(1)` auxiliary space:

```csharp
static string LongestPalindrome(string s)
{
    int bestStart = 0, bestLength = 0;
    for (int center = 0; center < s.Length; center++)
    {
        Expand(center, center);
        Expand(center, center + 1);
    }
    return s.Substring(bestStart, bestLength);

    void Expand(int left, int right)
    {
        while (left >= 0 && right < s.Length && s[left] == s[right])
        {
            int length = right - left + 1;
            if (length > bestLength) { bestStart = left; bestLength = length; }
            left--; right++;
        }
    }
}
```

##### 23. Print the specified output from `{1,2,3,4,5,6}` *(PDF p.23)*

The PDF gives only this output and no operation definition. One consistent interpretation is to print eight cyclic values in rows of four:

```text
1 2 3 4
5 6 1 2
```

```csharp
int[] givenArray = { 1, 2, 3, 4, 5, 6 };
for (int i = 0; i < 8; i++)
{
    Console.Write(givenArray[i % givenArray.Length]);
    Console.Write(i % 4 == 3 ? "\n" : " ");
}
```

This is an explicit assumption because the source does not state why eight values are printed.

##### 24. Find the maximum number in an array *(PDF p.23)*

```csharp
static int MaxValue(int[] a)
{
    if (a.Length == 0) throw new ArgumentException("Array must not be empty.");
    int max = a[0];
    for (int i = 1; i < a.Length; i++) if (a[i] > max) max = a[i];
    return max;
}
```

##### 25. Find duplicate records in a table *(PDF p.23)*

Group by the columns that define a duplicate and use `HAVING COUNT(*) > 1`; see the parameterized example in [SQL scenarios](11-sql-scenarios.md#find-duplicate-records-in-a-table-pdf-p23).

##### 26. Count contiguous subarrays whose product is less than `K` *(PDF p.23)*

For positive integers, use a sliding window. Example `[1,2,3,4]`, `K=10` returns `7` (the seven subarrays listed in the PDF). Time is `O(n)`.

```csharp
static long CountProductsBelowK(int[] a, long k)
{
    if (k <= 1) return 0;
    long product = 1, count = 0;
    int left = 0;
    for (int right = 0; right < a.Length; right++)
    {
        if (a[right] <= 0) throw new ArgumentException("All values must be positive.");
        while (product > (k - 1) / a[right] && left <= right)
            product /= a[left++];
        if (left > right) { product = 1; continue; }
        product *= a[right];
        count += right - left + 1;
    }
    return count;
}
```

##### 27. Choose the cheapest coupon for each required ingredient and total the cost *(PDF p.23)*

Assume each coupon is `(itemId, price)` and every required item must have at least one coupon. Keep the lowest price per item:

```csharp
static decimal CheapestTotal(
    IEnumerable<int> requiredItems,
    IEnumerable<(int ItemId, decimal Price)> coupons)
{
    var cheapest = new Dictionary<int, decimal>();
    foreach (var coupon in coupons)
    {
        if (coupon.Price < 0) throw new ArgumentException("Prices cannot be negative.");
        if (!cheapest.TryGetValue(coupon.ItemId, out decimal old) || coupon.Price < old)
            cheapest[coupon.ItemId] = coupon.Price;
    }

    decimal total = 0;
    foreach (int item in requiredItems)
    {
        if (!cheapest.TryGetValue(item, out decimal price))
            throw new InvalidOperationException($"No coupon exists for required item {item}.");
        total = checked(total + price);
    }
    return total;
}
```

##### 28. Find minimum and maximum numbers in an array *(PDF p.23)*

Single scan, `O(n)`:

```csharp
static (int Min, int Max) MinMax(int[] a)
{
    if (a.Length == 0) throw new ArgumentException("Array must not be empty.");
    int min = a[0], max = a[0];
    for (int i = 1; i < a.Length; i++)
    {
        if (a[i] < min) min = a[i];
        if (a[i] > max) max = a[i];
    }
    return (min, max);
}
```

##### Practice notes copied from the source *(PDF p.22)*

The source emphasizes avoiding convenience built-ins when performance is being assessed, discussing Big-O, camelCase/PascalCase, basic namespaces (`System.Text`, `System.Collections.Generic`), zero-based string indexing, string immutability, ASCII, loops and collections. Use a built-in implementation only when permitted; explain the algorithm and complexity either way.
