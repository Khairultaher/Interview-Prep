# C# Data Types and Collections Cheat Sheet

A deduplicated reference for C#/.NET interviews and everyday
development. Examples target modern C# and .NET 6+ unless noted.

## 1. Built-in types

### Integral types

  ---------------------------------------------------------------------------
  C# type          .NET type                           Size Range
  ---------------- ------------------ --------------------- -----------------
  `sbyte`          `System.SByte`                     8-bit -128 to 127

  `byte`           `System.Byte`                      8-bit 0 to 255

  `short`          `System.Int16`                    16-bit -32,768 to 32,767

  `ushort`         `System.UInt16`                   16-bit 0 to 65,535

  `int`            `System.Int32`                    32-bit -2,147,483,648 to
                                                            2,147,483,647

  `uint`           `System.UInt32`                   32-bit 0 to
                                                            4,294,967,295

  `long`           `System.Int64`                    64-bit -9.22e18 to
                                                            9.22e18
                                                            (approximately)

  `ulong`          `System.UInt64`                   64-bit 0 to 1.84e19
                                                            (approximately)

  `nint`           `System.IntPtr`           platform-sized Signed native
                                                            integer

  `nuint`          `System.UIntPtr`          platform-sized Unsigned native
                                                            integer
  ---------------------------------------------------------------------------

``` csharp
byte pixels = 255;
int count = 42;
long largeCount = 9_000_000_000L;
uint unsignedCount = 3_000_000_000U;
nint nativeValue = 10;
```

### Floating-point and decimal types

  --------------------------------------------------------------------------
  Type                            Size          Approximate Typical use
                                                  precision 
  --------------- -------------------- -------------------- ----------------
  `float`                       32-bit     6--9 significant Graphics,
                                                     digits measurements
                                                            where
                                                            approximation is
                                                            acceptable

  `double`                      64-bit   15--17 significant General
                                                     digits scientific and
                                                            floating-point
                                                            calculations

  `decimal`            128-bit storage   28--29 significant Financial
                                                     digits calculations
                                                            requiring
                                                            decimal
                                                            arithmetic
  --------------------------------------------------------------------------

``` csharp
float ratio = 0.75f;        // f suffix
double distance = 123.456;
decimal price = 19.99m;     // m suffix
```

Floating-point types can have rounding error. `decimal` represents
decimal fractions more naturally, but it is not unlimited-precision
arithmetic.

### Other built-in types

  ------------------------------------------------------------------------
  Type                    Example                  Notes
  ----------------------- ------------------------ -----------------------
  `bool`                  `bool isReady = true;`   `true` or `false`

  `char`                  `char initial = 'A';`    One UTF-16 code unit;
                                                   not necessarily a full
                                                   Unicode character

  `string`                `string name = "Sam";`   Immutable sequence of
                                                   UTF-16 code units

  `object`                `object value = 42;`     Base type of all C#
                                                   types; value types are
                                                   boxed when converted to
                                                   `object`
  ------------------------------------------------------------------------

``` csharp
bool valid = true;
char letter = 'A';
string message = "Hello";
object boxed = 42;

Console.WriteLine(boxed.GetType()); // System.Int32
```

### Common framework value types

These are .NET types, not C# language keywords.

  -----------------------------------------------------------------------------------------
  Type                    Example                                   Useful members
  ----------------------- ----------------------------------------- -----------------------
  `DateTime`              `DateTime.UtcNow`                         `AddDays`, `AddHours`,
                                                                    `Date`, `Year`,
                                                                    `Month`, `Day`,
                                                                    `ToString`

  `DateTimeOffset`        `DateTimeOffset.UtcNow`                   `AddDays`,
                                                                    `ToUniversalTime`,
                                                                    `Offset`, `ToString`

  `TimeSpan`              `TimeSpan.FromMinutes(90)`                `TotalSeconds`,
                                                                    `TotalMinutes`, `Add`,
                                                                    `Subtract`

  `Guid`                  `Guid.NewGuid()`                          `Parse`, `TryParse`,
                                                                    `ToString`

  `decimal` helpers       `Math.Round(12.345, 2)` is for `double`   Use `decimal.Round`,
                                                                    `decimal.Floor`,
                                                                    `decimal.Ceiling` for
                                                                    `decimal`

  `DateOnly`              `DateOnly.FromDateTime(DateTime.Today)`   `AddDays`, `DayNumber`,
                                                                    `ToString`

  `TimeOnly`              `TimeOnly.FromDateTime(DateTime.Now)`     `AddHours`,
                                                                    `AddMinutes`,
                                                                    `ToString`
  -----------------------------------------------------------------------------------------

### Parsing and common numeric members

``` csharp
int n = int.Parse("123");
bool parsed = int.TryParse("123", out int result);

int max = int.MaxValue;
double root = Math.Sqrt(81);
double rounded = Math.Round(3.14159, 2);
decimal money = decimal.Round(19.995m, 2, MidpointRounding.AwayFromZero);

char.IsLetter('A');
char.IsDigit('7');
char.ToUpperInvariant('a');
```

`Parse` throws for invalid input; `TryParse` returns `false`. `Math`
methods generally operate on `double`; use the `decimal` methods when
working with decimal values.

## 2. Type categories and declarations

### Value types and reference types

-   **Value types:** built-in numeric types, `bool`, `char`, `enum`, and
    user-defined `struct`/`record struct`. Assignment copies the value.
-   **Reference types:** `class`, `string`, arrays, delegates,
    interfaces, and most collection classes. Assignment copies the
    reference.
-   `string` is a reference type but is immutable.
-   `object` and `dynamic` are different: `object` uses normal
    compile-time member checking; `dynamic` defers member binding to
    runtime.

``` csharp
int? optionalNumber = null;           // Nullable<int>
DateTime? optionalDate = null;
dynamic item = 10;
item = "now a string";

enum Status { Pending, Complete }
Status status = Status.Pending;
string statusText = status.ToString();
Status parsedStatus = Enum.Parse<Status>("Complete");
bool validStatus = Enum.TryParse("Complete", out Status parsed);
```

### User-defined and related types

  -------------------------------------------------------------------------------------------------
  Construct                           Example / purpose
  ----------------------------------- -------------------------------------------------------------
  `class`                             `class Customer { public string Name { get; set; } = ""; }`
                                      --- reference type

  `struct`                            `struct Point { public int X; public int Y; }` --- value type

  `record`                            `record Person(string Name, int Age);` --- data-oriented
                                      reference type by default

  `record struct`                     `record struct Coordinate(int X, int Y);` --- data-oriented
                                      value type

  `interface`                         `interface IRun { void Run(); }` --- contract

  `enum`                              `enum Color { Red, Green, Blue }` --- named constants

  `delegate`                          `Func<int, int> square = x => x * x;` --- callable reference

  Anonymous type                      `var x = new { Id = 1, Name = "A" };` --- compiler-generated
                                      type

  Tuple                               `(int Id, string Name) row = (1, "A");` --- grouped values

  Pointer                             `int* p` --- unsafe code only; requires an unsafe context
  -------------------------------------------------------------------------------------------------

Common delegate types:

``` csharp
Func<int, int> square = x => x * x;        // Returns a value
Action<string> print = Console.WriteLine;  // Returns void
Predicate<int> isEven = x => x % 2 == 0;   // Returns bool
```

## 3. Arrays and collection interfaces

### Array and array shapes

Arrays have a fixed length after creation.

``` csharp
int[] numbers = { 3, 1, 2 };
int[,] matrix = new int[2, 3];       // Multidimensional array
int[][] jagged = new int[2][];       // Array of arrays
jagged[0] = new[] { 1, 2 };

Array.Sort(numbers);
Array.Reverse(numbers);
int index = Array.IndexOf(numbers, 2);
int[] copy = numbers.ToArray();      // LINQ extension method
int length = numbers.Length;
```

Useful `Array` members: `Length`, `Rank`, `GetLength(dimension)`,
`GetValue`, `SetValue`, `Copy`, `CopyTo`, `Clear`, `Sort`, `Reverse`,
`IndexOf`, `Find`, `FindIndex`, `Exists`, `TrueForAll`, `Resize`
(changes the array reference passed by `ref`).

### Important collection interfaces

  -------------------------------------------------------------------------
  Interface                             Meaning
  ------------------------------------- -----------------------------------
  `IEnumerable<T>`                      Can be iterated

  `ICollection<T>`                      Basic mutable collection operations

  `IList<T>`                            Indexed collection contract

  `IReadOnlyCollection<T>`              Read-only access to count and
                                        enumeration

  `IReadOnlyList<T>`                    Read-only indexed access

  `IDictionary<TKey, TValue>`           Key/value mapping contract

  `IReadOnlyDictionary<TKey, TValue>`   Read-only key/value access

  `ISet<T>`                             Set operations contract

  `IQueryable<T>`                       Query expression intended for a
                                        provider, such as EF Core
  -------------------------------------------------------------------------

"Read-only" interfaces restrict operations through that interface; they
do not guarantee the underlying collection cannot change elsewhere.

## 4. Standard collections and their built-in members

### `List<T>`

Resizable, indexed sequence; allows duplicates.

``` csharp
var list = new List<int> { 3, 1, 2 };
list.Add(4);
list.AddRange(new[] { 5, 6 });
list.Insert(0, 9);
bool hasTwo = list.Contains(2);
int position = list.IndexOf(2);
list.Remove(3);
list.RemoveAt(0);
list.Sort();
list.Reverse();
int[] array = list.ToArray();
int count = list.Count;
list.Clear();
```

Key members: `Add`, `AddRange`, `Insert`, `InsertRange`, `Remove`,
`RemoveAt`, `RemoveAll`, `RemoveRange`, `Contains`, `IndexOf`,
`LastIndexOf`, `Find`, `FindAll`, `FindIndex`, `Exists`, `TrueForAll`,
`Sort`, `Reverse`, `BinarySearch`, `ConvertAll`, `CopyTo`, `GetRange`,
`ForEach`, `ToArray`, `Count`, `Capacity`, `Clear`.

### `ArrayList` (legacy, non-generic)

Stores `object` values, so it can mix types but may require
boxing/unboxing and casts. Prefer `List<T>` in new code.

``` csharp
var items = new System.Collections.ArrayList { 1, "text" };
items.Add(3);
items.Insert(0, "first");
items.Remove(1);
items.RemoveAt(0);
bool found = items.Contains("text");
int count = items.Count;
```

Key members: `Add`, `AddRange`, `Insert`, `InsertRange`, `Remove`,
`RemoveAt`, `RemoveRange`, `Contains`, `IndexOf`, `LastIndexOf`, `Sort`,
`Reverse`, `BinarySearch`, `CopyTo`, `ToArray`, `Count`, `Capacity`,
`Clear`.

### `Dictionary<TKey, TValue>`

Hash-based key/value collection; keys must be unique.

``` csharp
var ages = new Dictionary<string, int>();
ages.Add("Ava", 30);
ages["Ben"] = 25;                    // Adds or updates
bool exists = ages.ContainsKey("Ava");
bool gotValue = ages.TryGetValue("Ava", out int age);
ages.Remove("Ben");
var keys = ages.Keys;
var values = ages.Values;
int count = ages.Count;
```

Key members: `Add`, indexer `[]`, `TryAdd`, `TryGetValue`,
`ContainsKey`, `ContainsValue`, `Remove`, `Clear`, `EnsureCapacity`,
`TrimExcess`, `GetValueOrDefault` (extension method on supported target
frameworks), `Keys`, `Values`, `Count`.

### `SortedDictionary<TKey, TValue>` and `SortedList<TKey, TValue>`

Both keep keys sorted by comparer. `SortedDictionary` is tree-based;
`SortedList` stores sorted key/value arrays and can be efficient for
relatively small collections with infrequent inserts/removals.

``` csharp
var sorted = new SortedDictionary<int, string>
{
    [3] = "C",
    [1] = "A"
};
sorted.Add(2, "B");
bool contains = sorted.ContainsKey(2);
sorted.Remove(1);

var sortedList = new SortedList<int, string> { [2] = "B", [1] = "A" };
int keyAtZero = sortedList.Keys[0];
```

Common members: `Add`, indexer `[]`, `ContainsKey`, `ContainsValue`,
`TryGetValue`, `Remove`, `Clear`, `Keys`, `Values`, `Count`.
`SortedList` also has indexed key/value access and
`IndexOfKey`/`IndexOfValue`.

### `HashSet<T>` and `SortedSet<T>`

Sets store unique values. `HashSet<T>` is hash-based and unordered;
`SortedSet<T>` maintains sort order.

``` csharp
var set = new HashSet<int> { 1, 2, 2 };
set.Add(3);
set.Contains(2);
set.UnionWith(new[] { 3, 4 });
set.IntersectWith(new[] { 2, 4 });
set.ExceptWith(new[] { 1 });
set.SymmetricExceptWith(new[] { 4, 5 });
bool subset = set.IsSubsetOf(new[] { 2, 4, 5 });

var sortedSet = new SortedSet<int> { 5, 1, 3 };
int min = sortedSet.Min;
int max = sortedSet.Max;
var range = sortedSet.GetViewBetween(2, 5);
```

Key members: `Add`, `Remove`, `RemoveWhere`, `Contains`, `Clear`,
`CopyTo`, `SetEquals`, `IsSubsetOf`, `IsSupersetOf`, `IsProperSubsetOf`,
`IsProperSupersetOf`, `Overlaps`, `UnionWith`, `IntersectWith`,
`ExceptWith`, `SymmetricExceptWith`, `Count`. `SortedSet<T>`
additionally offers `Min`, `Max`, `GetViewBetween`, `Reverse`.

### `Stack<T>` (LIFO)

Last in, first out.

``` csharp
var stack = new Stack<int>();
stack.Push(10);
stack.Push(20);
int top = stack.Peek();       // 20, does not remove
int removed = stack.Pop();    // 20, removes it
bool hasTen = stack.Contains(10);
stack.TryPop(out int value);
stack.TryPeek(out int next);
```

Key members: `Push`, `PushRange`, `Pop`, `TryPop`, `Peek`, `TryPeek`,
`Contains`, `Clear`, `ToArray`, `CopyTo`, `Count`.

### `Queue<T>` (FIFO)

First in, first out.

``` csharp
var queue = new Queue<string>();
queue.Enqueue("first");
queue.Enqueue("second");
string next = queue.Peek();       // "first", does not remove
string removed = queue.Dequeue(); // "first", removes it
queue.TryDequeue(out string? item);
queue.TryPeek(out string? front);
```

Key members: `Enqueue`, `EnqueueRange` (modern .NET), `Dequeue`,
`TryDequeue`, `Peek`, `TryPeek`, `Contains`, `Clear`, `ToArray`,
`CopyTo`, `Count`.

### `PriorityQueue<TElement, TPriority>` (min-priority first)

Available in .NET 6+. It is a priority queue backed by a **min-heap** by
default: the smallest priority is dequeued first. It is not
automatically a stable queue for equal priorities.

``` csharp
var pq = new PriorityQueue<string, int>();
pq.Enqueue("low urgency", 10);
pq.Enqueue("urgent", 1);
string first = pq.Dequeue(); // "urgent"

pq.Enqueue("normal", 5);
pq.TryPeek(out string? element, out int priority);
pq.TryDequeue(out string? nextElement, out int nextPriority);
```

Key members: `Enqueue`, `EnqueueDequeue`, `Dequeue`, `TryDequeue`,
`Peek`, `TryPeek`, `DequeueEnqueue`, `Clear`, `EnsureCapacity`,
`TrimExcess`, `UnorderedItems`, `Count`.

### Min-heap and max-heap

.NET's built-in `PriorityQueue<TElement, TPriority>` is a min-priority
queue. There is no separate built-in public `MinHeap<T>` or `MaxHeap<T>`
collection in the base class library. Use a priority comparer for
max-priority behavior:

``` csharp
var maxHeap = new PriorityQueue<int, int>(
    Comparer<int>.Create((x, y) => y.CompareTo(x)));

maxHeap.Enqueue(10, 10);
maxHeap.Enqueue(30, 30);
maxHeap.Enqueue(20, 20);
int maximum = maxHeap.Dequeue(); // 30
```

Here the reversed comparer makes larger numeric priorities come out
first. For complex priorities, define an appropriate comparer.

### `LinkedList<T>`

Doubly linked list. Adding/removing at a known node is O(1); finding a
value is O(n). It does not support index-based access.

``` csharp
var linked = new LinkedList<string>();
linked.AddFirst("B");
linked.AddFirst("A");
linked.AddLast("D");
LinkedListNode<string>? node = linked.Find("B");
if (node is not null)
{
    linked.AddAfter(node, "C");
    linked.Remove(node);
}
```

Key members: `AddFirst`, `AddLast`, `AddBefore`, `AddAfter`, `Find`,
`FindLast`, `Remove`, `RemoveFirst`, `RemoveLast`, `Clear`, `Contains`,
`CopyTo`, `First`, `Last`, `Count`.

## 5. Additional collection families

### Immutable collections (`System.Collections.Immutable`)

Useful when a collection should not change after creation; updates
return a new collection. The package/reference may need to be added
depending on the target framework.

-   `ImmutableArray<T>`
-   `ImmutableList<T>`
-   `ImmutableDictionary<TKey, TValue>`
-   `ImmutableHashSet<T>`
-   `ImmutableSortedDictionary<TKey, TValue>`
-   `ImmutableSortedSet<T>`
-   `ImmutableQueue<T>`
-   `ImmutableStack<T>`

Typical members include `Add`, `AddRange`, `SetItem`, `Remove`,
`RemoveRange`, `Contains`, `ToBuilder`, plus the relevant lookup or
set-operation methods.

``` csharp
using System.Collections.Immutable;

ImmutableList<int> original = ImmutableList.Create(1, 2);
ImmutableList<int> updated = original.Add(3); // original remains unchanged
```

### Read-only wrappers

-   `ReadOnlyCollection<T>` --- read-only wrapper around an existing
    `IList<T>`.
-   `ReadOnlyDictionary<TKey, TValue>` --- read-only wrapper around a
    dictionary.
-   `IReadOnlyList<T>`, `IReadOnlyCollection<T>`,
    `IReadOnlyDictionary<TKey, TValue>` --- read-only access interfaces.

A wrapper does not make the underlying collection immutable; changes
made through another reference can still be visible.

### Thread-safe and producer/consumer collections

Namespace: `System.Collections.Concurrent`.

  -----------------------------------------------------------------------------------------
  Type                                   Purpose                 Common members
  -------------------------------------- ----------------------- --------------------------
  `ConcurrentDictionary<TKey, TValue>`   Concurrent key/value    `TryAdd`, `TryGetValue`,
                                         access                  `TryRemove`, `TryUpdate`,
                                                                 `GetOrAdd`, `AddOrUpdate`,
                                                                 `ContainsKey`

  `ConcurrentQueue<T>`                   Thread-safe FIFO        `Enqueue`, `TryDequeue`,
                                                                 `TryPeek`, `ToArray`,
                                                                 `Count`

  `ConcurrentStack<T>`                   Thread-safe LIFO        `Push`, `TryPop`,
                                                                 `TryPeek`, `ToArray`,
                                                                 `Count`

  `ConcurrentBag<T>`                     Thread-safe unordered   `Add`, `TryTake`,
                                         bag                     `TryPeek`, `ToArray`,
                                                                 `Count`

  `BlockingCollection<T>`                Bounded/blocking        `Add`, `TryAdd`, `Take`,
                                         producer-consumer       `TryTake`,
                                         wrapper                 `CompleteAdding`,
                                                                 `GetConsumingEnumerable`
  -----------------------------------------------------------------------------------------

Thread-safe collection operations do not automatically make a multi-step
business operation atomic.

### Specialized collections

Namespace: `System.Collections.Specialized` unless otherwise noted.

-   `BitArray` --- compact bit storage; members include `Get`, `Set`,
    `SetAll`, `And`, `Or`, `Xor`, `Not`, `Length`.
-   `OrderedDictionary` --- key/value entries accessible by key or
    index; members include `Add`, `Insert`, `Remove`, `RemoveAt`,
    `Contains`, `IndexOf`, `GetKey`, `GetByIndex`, `SetAt`.
-   `NameValueCollection` --- string keys with potentially multiple
    string values; members include `Add`, `Set`, `Get`, `GetValues`,
    `GetKey`, `Remove`, `AllKeys`.
-   `StringCollection` --- legacy string collection; members include
    `Add`, `AddRange`, `Contains`, `IndexOf`, `Insert`, `Remove`,
    `RemoveAt`.
-   `HybridDictionary` --- legacy dictionary optimized to switch
    representation for small/large collections.
-   `StringDictionary` --- legacy string-key/string-value dictionary.

For new general-purpose code, prefer generic collections such as
`List<T>` and `Dictionary<TKey, TValue>`.

### UI and data collections

-   `ObservableCollection<T>` (`System.Collections.ObjectModel`) ---
    raises change notifications for UI binding. Members include `Add`,
    `Remove`, `Move`, `Clear`; event: `CollectionChanged`.
-   `Collection<T>` --- customizable wrapper collection.
-   `ReadOnlyObservableCollection<T>` --- read-only observable wrapper.
-   `DataTable` / `DataSet` (`System.Data`) --- ADO.NET in-memory
    relational data, not general-purpose collection replacements.

### Memory and views (not ordinary collections)

-   `Span<T>` / `ReadOnlySpan<T>` --- stack-only views over contiguous
    memory; members include `Length`, `Slice`, `CopyTo`, `ToArray`,
    `IndexOf`.
-   `Memory<T>` / `ReadOnlyMemory<T>` --- heap-storable memory views
    suitable for async workflows; members include `Length`, `Slice`,
    `Span`, `ToArray`.
-   `ArraySegment<T>` --- view over part of an array; members include
    `Array`, `Offset`, `Count`, `AsSpan`, `ToArray`.
-   `Memory<T>` and `Span<T>` are not resizable collections; they view
    existing memory.

## 6. LINQ methods commonly used with collections

LINQ methods are extension methods, not members unique to `List<T>` or
`Dictionary<TKey,TValue>`. Add `using System.Linq;` when needed (often
implicit in modern .NET templates).

``` csharp
var numbers = new[] { 5, 2, 8, 2, 1 };

var filtered = numbers.Where(x => x > 2);
var projected = numbers.Select(x => x * 10);
var ordered = numbers.OrderBy(x => x);
var distinct = numbers.Distinct();
var first = numbers.First();
var maybeFirst = numbers.FirstOrDefault();
bool anyLarge = numbers.Any(x => x > 7);
bool allPositive = numbers.All(x => x > 0);
int total = numbers.Sum();
int count = numbers.Count();
var groups = numbers.GroupBy(x => x);
var lookup = numbers.ToLookup(x => x % 2);
var list = numbers.ToList();
var set = numbers.ToHashSet();
var map = numbers.Distinct().ToDictionary(x => x, x => x * x);
```

Common LINQ operators: `Where`, `Select`, `SelectMany`, `OrderBy`,
`OrderByDescending`, `ThenBy`, `ThenByDescending`, `GroupBy`, `Join`,
`GroupJoin`, `Distinct`, `DistinctBy`, `Union`, `Intersect`, `Except`,
`Concat`, `Append`, `Prepend`, `Take`, `TakeLast`, `Skip`, `SkipLast`,
`Chunk`, `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `Last`,
`LastOrDefault`, `Any`, `All`, `Contains`, `Count`, `LongCount`, `Sum`,
`Min`, `Max`, `Average`, `Aggregate`, `SequenceEqual`, `ToArray`,
`ToList`, `ToDictionary`, `ToHashSet`, `ToLookup`.

`IQueryable<T>` builds expression trees for a query provider (for
example, EF Core); `IEnumerable<T>` generally executes .NET iteration.
With EF Core, be mindful of which operators run in the database and when
materialization occurs.

## 7. Quick selection guide

  Need                                            Good starting choice
  ----------------------------------------------- --------------------------------------------------
  Fixed-size indexed data                         `T[]`
  Resizable indexed sequence                      `List<T>`
  Unique values, fast membership checks           `HashSet<T>`
  Unique values in sorted order                   `SortedSet<T>`
  Key/value lookup                                `Dictionary<TKey, TValue>`
  Key/value lookup in key order                   `SortedDictionary<TKey, TValue>`
  FIFO processing                                 `Queue<T>`
  LIFO processing                                 `Stack<T>`
  Retrieve item by priority                       `PriorityQueue<TElement, TPriority>`
  Frequent insertion/removal around known nodes   `LinkedList<T>`
  Concurrent access                               Appropriate `System.Collections.Concurrent` type
  Immutable data                                  `System.Collections.Immutable` type
  High-performance memory slicing                 `Span<T>` / `Memory<T>`

## 8. Interview reminders

-   Prefer generic collections over legacy `ArrayList`, `Hashtable`, and
    similar non-generic types.
-   `List<T>.Count` is the number of elements; an array's `Length` is
    its element count.
-   Dictionary keys must be unique; set elements must be unique
    according to the equality comparer.
-   `Queue<T>` is FIFO; `Stack<T>` is LIFO;
    `PriorityQueue<TElement, TPriority>` removes the lowest priority
    first by default.
-   A heap is a data-structure concept;
    `PriorityQueue<TElement, TPriority>` is the built-in heap-backed
    priority queue. There is no separate standard `MaxHeap<T>` type.
-   `byte` is often appropriate for binary data, `int` for typical
    integer values, `long` for larger counts, `double` for general
    floating-point math, and `decimal` for decimal financial
    calculations.
-   `uint`/`ulong` are not automatically better for non-negative values;
    use them when their unsigned semantics or range is actually needed.
-   Use `TryParse`, `TryGetValue`, `TryDequeue`, and similar `Try...`
    APIs when failure is an expected possibility.
