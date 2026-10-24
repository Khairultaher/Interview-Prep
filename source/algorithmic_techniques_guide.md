# Core Algorithmic Techniques: A Practical Guide

This guide explains **recursion, divide and conquer, backtracking, greedy algorithms, and dynamic programming** with self-contained text diagrams, examples, C# code, and a comparison table.

## 1. The big picture

These terms describe related but different things:

- **Recursion** is a programming technique: a function calls itself.
- **Divide and conquer** is a strategy: split a problem into independent smaller problems, solve them, then combine their answers.
- **Backtracking** is a search strategy: build a candidate solution, abandon it when it cannot work, and try another choice.
- **Greedy** is a decision strategy: make the best-looking choice at each step, without revisiting earlier choices.
- **Dynamic programming (DP)** is an optimization strategy: solve overlapping subproblems and save their answers so they are not repeatedly recomputed.

A technique is not necessarily an alternative to all the others. For example, divide and conquer and backtracking often use recursion. Dynamic programming can also be implemented recursively with memoization.

## 2. Recursion

### What it means

A recursive function solves a problem by calling itself on a smaller input. It needs:

1. A **base case** that stops the calls.
2. A **recursive case** that makes progress toward the base case.

Without a reachable base case, the calls may continue until a stack overflow occurs.

### Example: factorial

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

### Call-and-return diagram for `Factorial(4)`

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

## 3. Divide and conquer

### What it means

Divide and conquer breaks a problem into smaller, usually independent subproblems, solves them, and combines their results.

The usual pattern is:

1. **Divide** the input.
2. **Conquer** each part, often recursively.
3. **Combine** the partial answers.

### Example: merge sort

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

## 4. Backtracking

### What it means

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

### Example: finding a path through a maze

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

## 5. Greedy algorithms

### What it means

A greedy algorithm chooses what seems best **right now**, then continues from that choice. It normally does not reconsider earlier decisions.

A greedy strategy is fast and simple when the problem has the right mathematical properties, but a locally best choice does not always lead to the globally best answer.

### Example A: coin change where greedy works

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

### Example B: a case where greedy fails

Now suppose the coin values are `{1, 3, 4}` and the target is `6`.

```text
Greedy:  take 4, then 1, then 1  → 3 coins
Optimal: take 3, then 3          → 2 coins
```

So greedy is **not** a universally correct solution to coin change. It works for some coin systems, not all.

**Use greedy when:** you can justify that each local choice is safe and leads to an optimal overall solution. Well-known examples include interval scheduling (under the correct selection rule), Kruskal's minimum spanning tree algorithm, and Huffman coding.

## 6. Dynamic programming (DP)

### What it means

Dynamic programming is useful when a problem has:

- **Overlapping subproblems:** the same smaller question appears repeatedly.
- **Optimal substructure:** a larger answer can be built from answers to smaller subproblems.

DP stores answers to smaller subproblems, avoiding repeated work. Two common forms are:

- **Memoization (top-down):** use recursion and cache each result.
- **Tabulation (bottom-up):** fill a table from the smallest cases upward.

DP does not automatically guarantee an optimal result for every problem; the recurrence and state must correctly represent the problem.

### Example: Fibonacci numbers

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

### C# example: bottom-up DP

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

## 7. Side-by-side comparison

| Technique | Main idea | Repeats work? | Revisits choices? | Typical example |
|---|---|---|---|---|
| Recursion | A function calls itself | It can | Not inherently | Factorial, tree traversal |
| Divide and conquer | Split, solve independently, combine | Usually avoids overlap by design | Not usually | Merge sort |
| Backtracking | Explore choices; undo invalid or unproductive choices | Often explores many states | Yes | Sudoku, N-Queens |
| Greedy | Choose the best immediate option | Usually little repeated work | Usually no | Kruskal's algorithm |
| Dynamic programming | Cache or tabulate overlapping subproblems | Avoids repeated subproblem calculations | It compares states, not necessarily paths | Knapsack, LCS |

## 8. How to choose an approach

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

## 9. Key distinctions to remember

- **Recursion is a technique;** the others are problem-solving strategies or paradigms.
- **Divide and conquer** typically solves independent subproblems and combines their answers.
- **Backtracking** searches alternatives and undoes choices when a path cannot work.
- **Greedy** commits to locally attractive choices; correctness requires a proof or a known property of the problem.
- **Dynamic programming** remembers subproblem answers to avoid recalculating them; it is especially useful when subproblems overlap.

The most important skill is not memorizing labels. It is recognizing the structure of a problem, choosing a suitable approach, and checking whether the approach guarantees the answer you need.
