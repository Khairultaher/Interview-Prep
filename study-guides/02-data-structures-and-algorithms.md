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
