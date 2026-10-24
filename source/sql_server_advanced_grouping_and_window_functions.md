# SQL Server: Grouping, PIVOT/UNPIVOT, and Window Functions

A compact, runnable reference with nonduplicated examples. All queries use T-SQL.

## 1. GROUPING SETS, ROLLUP, and CUBE

These extend `GROUP BY` to return multiple aggregation levels in one query.

```sql
DROP TABLE IF EXISTS dbo.DepartmentSales;
CREATE TABLE dbo.DepartmentSales (
    Department varchar(20),
    SalesYear int,
    Amount int
);
INSERT INTO dbo.DepartmentSales VALUES
('HR', 2023, 100), ('HR', 2024, 200),
('IT', 2023, 300), ('IT', 2024, 400);
```

| Feature | Groupings generated for `(Department, SalesYear)` | Purpose |
|---|---|---|
| `GROUPING SETS` | Only those you specify | Custom summaries |
| `ROLLUP` | `(Department, SalesYear)`, `(Department)`, `()` | Hierarchical subtotals and grand total |
| `CUBE` | `(Department, SalesYear)`, `(Department)`, `(SalesYear)`, `()` | Every combination |

```sql
-- Custom groupings
SELECT Department, SalesYear, SUM(Amount) AS Total
FROM dbo.DepartmentSales
GROUP BY GROUPING SETS ((Department, SalesYear), (Department), ());

-- Hierarchical totals
SELECT Department, SalesYear, SUM(Amount) AS Total
FROM dbo.DepartmentSales
GROUP BY ROLLUP (Department, SalesYear);

-- All grouping combinations
SELECT Department, SalesYear, SUM(Amount) AS Total
FROM dbo.DepartmentSales
GROUP BY CUBE (Department, SalesYear);
```

`NULL` in subtotal rows represents an aggregated-away dimension, but source data can also contain real `NULL`s. Use `GROUPING(Department)` or `GROUPING_ID(Department, SalesYear)` to distinguish them. `ORDER BY` is needed for a guaranteed display order.

## 2. PIVOT and UNPIVOT

- **PIVOT**: rows → columns, with an aggregation.
- **UNPIVOT**: columns → rows. SQL Server `UNPIVOT` excludes `NULL` values.

### PIVOT: separate source table

```sql
DROP TABLE IF EXISTS dbo.SalesLong;
CREATE TABLE dbo.SalesLong (
    SalesYear int,
    Product varchar(20),
    Amount int
);
INSERT INTO dbo.SalesLong VALUES
(2023, 'Apple', 100), (2023, 'Banana', 150), (2023, 'Mango', 120),
(2024, 'Apple', 110), (2024, 'Banana', 160), (2024, 'Mango', 130);

SELECT SalesYear, [Apple], [Banana], [Mango]
FROM (SELECT SalesYear, Product, Amount FROM dbo.SalesLong) AS src
PIVOT (
    SUM(Amount) FOR Product IN ([Apple], [Banana], [Mango])
) AS p
ORDER BY SalesYear;
```

| SalesYear | Apple | Banana | Mango |
|---|---:|---:|---:|
| 2023 | 100 | 150 | 120 |
| 2024 | 110 | 160 | 130 |

### UNPIVOT: independent wide table

```sql
DROP TABLE IF EXISTS dbo.SalesWide;
CREATE TABLE dbo.SalesWide (
    SalesYear int,
    Apple int,
    Banana int,
    Mango int
);
INSERT INTO dbo.SalesWide VALUES
(2023, 100, 150, 120),
(2024, 110, 160, 130);

SELECT SalesYear, Product, Amount
FROM dbo.SalesWide
UNPIVOT (
    Amount FOR Product IN ([Apple], [Banana], [Mango])
) AS u
ORDER BY SalesYear, Product;
```

This returns one row per `(SalesYear, Product)` with its amount. `PIVOT` aggregates duplicate source rows, so pivoting and unpivoting are not always lossless inverses.

## 3. Window-function syntax and supporting clauses

Window functions calculate values across related rows **without collapsing the original rows**.

```sql
function_expression OVER (
    [PARTITION BY column_list]
    [ORDER BY sort_expression]
    [ROWS BETWEEN frame_start AND frame_end]
)
```

This is a conceptual template: the allowed clauses depend on the specific function.

| Clause or keyword | Meaning |
|---|---|
| `OVER (...)` | Defines the window |
| `PARTITION BY` | Restarts calculations for each group; optional |
| `ORDER BY` inside `OVER` | Defines processing order; required by ranking, offset and most analytic functions |
| `ROWS` | Defines a frame by physical row positions |
| `RANGE` | Groups rows with equal ordering values (peers); SQL Server supports only limited RANGE boundaries |
| `UNBOUNDED PRECEDING` | First row of partition |
| `n PRECEDING` | n rows before current row (`ROWS` only for numeric offsets) |
| `CURRENT ROW` | Current row (`RANGE` also includes peers at its boundary) |
| `n FOLLOWING` | n rows after current row (`ROWS` only for numeric offsets) |
| `UNBOUNDED FOLLOWING` | Last row of partition |

**Important:** `LAG`, `LEAD`, and ranking functions do **not** accept a `ROWS BETWEEN` frame in SQL Server. For ordered aggregate windows, explicitly specify `ROWS` when row-by-row behavior is intended; otherwise the default can be `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

## 4. Window-function catalog (SQL Server)

| Category | Functions | What they do |
|---|---|---|
| Ranking | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` | Number, rank, or bucket ordered rows |
| Offset | `LAG`, `LEAD` | Read previous/next row by offset |
| Value | `FIRST_VALUE`, `LAST_VALUE` | Read first/last value in the applicable window frame |
| Distribution | `PERCENT_RANK`, `CUME_DIST` | Relative rank and cumulative distribution |
| Percentile | `PERCENTILE_CONT`, `PERCENTILE_DISC` | Interpolated/discrete percentile; use `WITHIN GROUP (...) OVER (...)` |
| Aggregates usable as windows | `SUM`, `AVG`, `MIN`, `MAX`, `COUNT`, `COUNT_BIG`, `STDEV`, `STDEVP`, `VAR`, `VARP` | Aggregation without reducing row count |
| Additional aggregate | `STRING_AGG` | **Not** supported with `OVER` in SQL Server |

**Version notes:** `PERCENTILE_CONT` and `PERCENTILE_DISC` are available well before SQL Server 2022. `NTH_VALUE` is **not** a SQL Server T-SQL function (it exists in other databases). SQL Server does **not** support `EXCLUDE CURRENT ROW` window-frame syntax. SQL Server 2022+ (compatibility level 160+) supports a reusable `WINDOW` clause; availability depends on version/compatibility.

## 5. Runnable window examples: frames, LAG, and LEAD

```sql
DROP TABLE IF EXISTS dbo.MonthlySales;
CREATE TABLE dbo.MonthlySales (
    SalesPerson varchar(20),
    SalesMonth int,
    Amount int
);
INSERT INTO dbo.MonthlySales VALUES
('Alice', 1, 100), ('Alice', 2, 150), ('Alice', 3, 200),
('Bob', 1, 120), ('Bob', 2, 180), ('Bob', 3, 160);
```

### Three frame types in one query

```sql
SELECT SalesPerson, SalesMonth, Amount,
    SUM(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS RunningTotal,
    SUM(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
    ) AS ForwardTotal,
    SUM(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS FullPartitionTotal
FROM dbo.MonthlySales
ORDER BY SalesPerson, SalesMonth;
```

| SalesPerson | Month | Amount | RunningTotal | ForwardTotal | FullPartitionTotal |
|---|---:|---:|---:|---:|---:|
| Alice | 1 | 100 | 100 | 450 | 450 |
| Alice | 2 | 150 | 250 | 350 | 450 |
| Alice | 3 | 200 | 450 | 200 | 450 |
| Bob | 1 | 120 | 120 | 460 | 460 |
| Bob | 2 | 180 | 300 | 340 | 460 |
| Bob | 3 | 160 | 460 | 160 | 460 |

### Previous and next rows with LAG / LEAD

```sql
SELECT SalesPerson, SalesMonth, Amount,
    LAG(Amount, 1, 0) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
    ) AS PreviousAmount,
    LEAD(Amount, 1, 0) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
    ) AS NextAmount
FROM dbo.MonthlySales
ORDER BY SalesPerson, SalesMonth;
```

| SalesPerson | Month | Amount | PreviousAmount | NextAmount |
|---|---:|---:|---:|---:|
| Alice | 1 | 100 | 0 | 150 |
| Alice | 2 | 150 | 100 | 200 |
| Alice | 3 | 200 | 150 | 0 |
| Bob | 1 | 120 | 0 | 180 |
| Bob | 2 | 180 | 120 | 160 |
| Bob | 3 | 160 | 180 | 0 |

`1` is the offset; `0` is the fallback when there is no previous/next row. These functions use row position within the partition, not a window frame.

### Moving average: one row before through one row after

```sql
SELECT SalesPerson, SalesMonth, Amount,
    AVG(CAST(Amount AS decimal(10,2))) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS ThreeRowMovingAverage
FROM dbo.MonthlySales
ORDER BY SalesPerson, SalesMonth;
```

At partition boundaries, the frame contains fewer than three rows.

### LAST_VALUE gotcha

```sql
SELECT SalesPerson, SalesMonth, Amount,
    LAST_VALUE(Amount) OVER (
        PARTITION BY SalesPerson ORDER BY SalesMonth
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS LastMonthAmount
FROM dbo.MonthlySales;
```

The explicit full frame is important: without it, `LAST_VALUE` commonly returns the current row's value rather than the partition's final value.

## 6. Interview reminders

- `GROUP BY` reduces rows; `PARTITION BY` inside `OVER` keeps rows.
- `ROLLUP` follows the listed hierarchy; `CUBE` generates all combinations; `GROUPING SETS` lets you choose.
- `PIVOT` converts values to columns; `UNPIVOT` converts columns to values.
- `RANK` leaves gaps after ties; `DENSE_RANK` does not; `ROW_NUMBER` assigns unique sequential numbers.
- Add a unique tie-breaker to window `ORDER BY` when deterministic row positioning matters.
- Window-frame clauses work with applicable aggregate/value functions, **not** with `LAG`/`LEAD`.
