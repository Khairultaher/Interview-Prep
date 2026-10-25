# Data Structures and Algorithms

Each concept is explained once. Where several notes covered the same topic, they have been merged into a single section, and related topics in other guides are linked instead of repeated.

## Contents

- [C# Data Structures, Heaps, Comparers, and Sorting — Interview Cheat Sheet](#csharp-data-structures-sorting-interview-cheatsheet-md)
- [Core Algorithmic Techniques: A Practical Guide](#algorithmic-techniques-guide-md)
- [Other Handy Types](#c-data-structures-guide-pdf)
- [Additional Questpond question sets](#additional-questpond-question-sets)

---

<a id="csharp-data-structures-sorting-interview-cheatsheet-md"></a>

## C# Data Structures, Heaps, Comparers, and Sorting — Interview Cheat Sheet

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

#### 6. Everyday collection operations and LINQ

Member-by-member examples for `List<T>`, `Dictionary`, `HashSet`, `Queue`, `Stack`, and the common LINQ operators are in [C# Language and Code Quality → Standard collections](01-csharp-language-and-quality.md#csharp-data-types-and-collections-cheatsheet-md). Two interview-specific reminders:

- Set methods such as `UnionWith`, `IntersectWith`, and `ExceptWith` **mutate** the receiver; LINQ `Union`/`Intersect`/`Except` return a new sequence.
- Frequency count in one line: `numbers.GroupBy(n => n).ToDictionary(g => g.Key, g => g.Count())`. In hot paths, a plain loop can be clearer and avoids LINQ allocations.

#### 7. Binary search on sorted data

```csharp
int[] sorted = { 1, 3, 5, 7, 9 };
int index = Array.BinarySearch(sorted, 7); // index 3 if found
int missing = Array.BinarySearch(sorted, 6); // negative encoded insertion point
```

The input must already be sorted with the same ordering used for the search. For insertion point when not found: `int insertionIndex = ~missing;`.

#### 8. Interview patterns to recognize

- **Fast membership / deduplication:** `HashSet<T>`.
- **Count frequencies / pair lookup:** `Dictionary<TKey,TValue>`.
- **BFS / shortest path in an unweighted graph:** `Queue<T>`.
- **DFS / matching brackets / monotonic stack:** `Stack<T>`.
- **Repeated minimum/maximum, Dijkstra, scheduling, Top-K:** `PriorityQueue<TElement,TPriority>`.
- **Sorted unique values:** `SortedSet<T>`.
- **Sort once, then binary search / two pointers:** array or list + sort.
- **Sliding window:** array/list plus two pointers; sometimes a dictionary or deque-like structure.
- **Custom ordering:** comparison lambda, `Comparer<T>.Create`, or `IComparer<T>`.

#### 9. Pitfalls worth remembering

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

#### 10. Quick templates

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

## Core Algorithmic Techniques: A Practical Guide

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

## Other Handy Types

`List<T>`, `Dictionary`, sets, `Stack`, `Queue`, `LinkedList`, concurrent/immutable collections, `Span<T>`, and `ObservableCollection<T>` are covered once in [C# Language and Code Quality → Data Types and Collections](01-csharp-language-and-quality.md#csharp-data-types-and-collections-cheatsheet-md). Sorting and heaps are covered above. This section only adds what isn't covered elsewhere.

> **Watch out:** `Comparer<int>.Default.Reverse()` is **not** a .NET API. For a max-heap, use `Comparer<int>.Create((a, b) => b.CompareTo(a))`. Negating priorities only works when the value can't be `int.MinValue`.

### `string` — immutable sequence of characters

```csharp
string name = "Afifa";

int length = name.Length;
string upper = name.ToUpper();
bool contains = name.Contains("fi");
bool startsWith = name.StartsWith("A");
int firstIndex = name.IndexOf("i");
string part = name.Substring(1, 2);
string replaced = name.Replace("fi", "xy");
string[] words = name.Split(' ');
string joined = string.Join(", ", words);
bool isEmpty = string.IsNullOrEmpty(name);
bool isBlank = string.IsNullOrWhiteSpace(name);
```

Every "modifying" method returns a **new** string. In loops, use `StringBuilder` to avoid creating lots of temporary strings.

### Custom trees and tries

The base library has no general-purpose tree or trie type, so you build them yourself. Trees model hierarchies; a trie (prefix tree) is great for prefix lookups and autocomplete.

```csharp
class TreeNode
{
    public int Value { get; set; }
    public List<TreeNode> Children { get; } = new();
}

class TrieNode
{
    public Dictionary<char, TrieNode> Children { get; } = new();
    public bool IsEndOfWord { get; set; }
}
```

**Tip:** in any IDE, type a variable name followed by `.` to see all available members in IntelliSense. It's the fastest way to discover APIs.

## Additional Questpond question sets

### Scenario-based questions

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

### Coding questions

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

Use the character-frequency method from [the scenario answer](#check-whether-two-strings-are-anagrams-pdf-p20). It is `O(n)` time and handles the source's `India`/`diani` example without regard to case.

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

For a centered pyramid of `rows` lines, print `rows - r` spaces and `2*r - 1` stars for each row `r` from 1 to `rows`. The two specific patterns shown in the PDF are reproduced in [the pattern answers](#pattern-printing-prompts).

##### 18. Find the Nth highest salary *(PDF p.22)*

See the `DENSE_RANK` SQL solution in [SQL scenarios](05-efcore-linq-and-sql.md#find-the-nth-highest-salary-pdf-p22); it returns the Nth distinct salary.

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

Group by the columns that define a duplicate and use `HAVING COUNT(*) > 1`; see the parameterized example in [SQL scenarios](05-efcore-linq-and-sql.md#find-duplicate-records-in-a-table-pdf-p23).

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
