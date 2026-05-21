# PROBLEM 7: Forward-Fill NULL Values (Gap-Filling)

## Problem Statement

Given a table where **category** is only filled for the first brand in each group and `NULL` for the rest, write a query to fill every NULL category with the correct category from the row above it.

This is a classic **"forward-fill"** / **"gap-filling"** problem commonly asked in SQL interviews.

---

## Input vs Expected Output

| category | brand_name | → | category (filled) | brand_name |
|----------|-----------|---|-------------------|-----------|
| Beverages | Coca Cola | → | Beverages | Coca Cola |
| NULL | Pepsi | → | **Beverages** | Pepsi |
| NULL | Sprite | → | **Beverages** | Sprite |
| NULL | Fanta | → | **Beverages** | Fanta |
| Snacks | Lays | → | Snacks | Lays |
| NULL | Doritos | → | **Snacks** | Doritos |
| NULL | Kurkure | → | **Snacks** | Kurkure |

---

## Setup

```sql
CREATE OR REPLACE TABLE Products (
    category    VARCHAR(50),
    brand_name  VARCHAR(50)
);

INSERT INTO Products (category, brand_name) VALUES ('Beverages', 'Coca Cola');
INSERT INTO Products (category, brand_name) VALUES (NULL, 'Pepsi');
INSERT INTO Products (category, brand_name) VALUES (NULL, 'Sprite');
INSERT INTO Products (category, brand_name) VALUES (NULL, 'Fanta');
INSERT INTO Products (category, brand_name) VALUES ('Snacks', 'Lays');
INSERT INTO Products (category, brand_name) VALUES (NULL, 'Doritos');
INSERT INTO Products (category, brand_name) VALUES (NULL, 'Kurkure');
```

---

## SOLUTION 1: LAST_VALUE with IGNORE NULLS (Best Approach)

```sql
SELECT
    LAST_VALUE(CATEGORY IGNORE NULLS) OVER (
        ORDER BY (SELECT NULL)
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS category,
    BRAND_NAME
FROM PRODUCTS;
```

### How It Works — Row by Row

| Row | category (original) | Window scans rows 1→current | LAST non-NULL found | Result |
|-----|--------------------|-----------------------------|---------------------|--------|
| 1 | Beverages | [Beverages] | Beverages | Beverages |
| 2 | NULL | [Beverages, NULL] | Beverages | Beverages |
| 3 | NULL | [Beverages, NULL, NULL] | Beverages | Beverages |
| 4 | NULL | [Beverages, NULL, NULL, NULL] | Beverages | Beverages |
| 5 | Snacks | [Beverages, NULL, NULL, NULL, Snacks] | Snacks | Snacks |
| 6 | NULL | [..., Snacks, NULL] | Snacks | Snacks |
| 7 | NULL | [..., Snacks, NULL, NULL] | Snacks | Snacks |

### Breaking Down Each Part

| Part | Meaning |
|------|---------|
| `LAST_VALUE(CATEGORY IGNORE NULLS)` | Get the last value of the `category` column, but skip over any NULLs. Only considers actual non-NULL values. |
| `OVER (...)` | Defines the window (which rows to look at for each calculation) |
| `ORDER BY (SELECT NULL)` | Process rows in insertion order (no specific column to sort by). In production, use a proper ID column. |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | For each row, look at ALL rows from the very first row up to the current row. This is the "look backward" window. |

### Visual Explanation

```
Row 1: [Beverages] Coca Cola    ← Window: [row1]         → Last non-NULL = "Beverages"
Row 2: [NULL]      Pepsi        ← Window: [row1, row2]   → Last non-NULL = "Beverages" (skips NULL)
Row 3: [NULL]      Sprite       ← Window: [row1..row3]   → Last non-NULL = "Beverages" (skips NULLs)
Row 4: [NULL]      Fanta        ← Window: [row1..row4]   → Last non-NULL = "Beverages" (skips NULLs)
Row 5: [Snacks]    Lays         ← Window: [row1..row5]   → Last non-NULL = "Snacks" (newest non-NULL!)
Row 6: [NULL]      Doritos      ← Window: [row1..row6]   → Last non-NULL = "Snacks" (skips NULL)
Row 7: [NULL]      Kurkure      ← Window: [row1..row7]   → Last non-NULL = "Snacks" (skips NULL)
```

### Why `IGNORE NULLS` is the Key

Without `IGNORE NULLS`:
- `LAST_VALUE` would return the actual last value including NULLs
- Row 2 would get `NULL` (its own value) instead of `Beverages`

With `IGNORE NULLS`:
- `LAST_VALUE` skips NULLs and returns the most recent **real** value
- This creates the "carry forward" behavior we need

---

## SOLUTION 2: COUNT to Create Groups + MAX to Fill

```sql
SELECT
    MAX(category) OVER (
        PARTITION BY grp
    ) AS category,
    brand_name
FROM (
    SELECT
        category,
        brand_name,
        COUNT(category) OVER (
            ORDER BY (SELECT NULL)
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS grp
    FROM Products
);
```

### How It Works — Two Steps

#### Step 1: Create Group Numbers using COUNT

`COUNT(category)` only counts **non-NULL** values. As a running count, it increments ONLY when a new non-NULL category appears:

| Row | category | brand_name | Running COUNT (grp) | Why |
|-----|----------|-----------|--------------------|----|
| 1 | Beverages | Coca Cola | 1 | Non-NULL found → count becomes 1 |
| 2 | NULL | Pepsi | 1 | NULL → count stays 1 |
| 3 | NULL | Sprite | 1 | NULL → count stays 1 |
| 4 | NULL | Fanta | 1 | NULL → count stays 1 |
| 5 | Snacks | Lays | 2 | Non-NULL found → count becomes 2 |
| 6 | NULL | Doritos | 2 | NULL → count stays 2 |
| 7 | NULL | Kurkure | 2 | NULL → count stays 2 |

**Key Insight:** `COUNT(column)` ignores NULLs by default! So it only increments when it sees a real category value. This naturally creates group boundaries.

#### Step 2: Fill with MAX within each group

Now we have groups (grp=1, grp=2). Within each group, there is exactly ONE non-NULL category value. `MAX(category)` picks that value for every row in the group:

| Row | grp | category values in group | MAX(category) | Result |
|-----|-----|-------------------------|---------------|--------|
| 1 | 1 | Beverages, NULL, NULL, NULL | Beverages | Beverages |
| 2 | 1 | Beverages, NULL, NULL, NULL | Beverages | Beverages |
| 3 | 1 | Beverages, NULL, NULL, NULL | Beverages | Beverages |
| 4 | 1 | Beverages, NULL, NULL, NULL | Beverages | Beverages |
| 5 | 2 | Snacks, NULL, NULL | Snacks | Snacks |
| 6 | 2 | Snacks, NULL, NULL | Snacks | Snacks |
| 7 | 2 | Snacks, NULL, NULL | Snacks | Snacks |

### Breaking Down Each Part

| Part | Meaning |
|------|---------|
| `COUNT(category)` | Counts non-NULL values only (NULLs are automatically ignored by COUNT) |
| `OVER (ORDER BY ... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` | Running/cumulative count from first row to current row |
| `AS grp` | This running count becomes our group identifier |
| `MAX(category) OVER (PARTITION BY grp)` | Within each group, find the maximum (= the only non-NULL) category value |
| `PARTITION BY grp` | Calculate MAX separately for each group number |

### Why MAX Works Here

Within each group, the values are: `['Beverages', NULL, NULL, NULL]`
- `MAX` ignores NULLs
- `MAX('Beverages')` = 'Beverages'
- If there were multiple non-NULL values, MAX picks alphabetically highest (but here there's always exactly one)

---

## Comparison

| Solution | Scans | Complexity | Best For |
|----------|-------|-----------|----------|
| 1. LAST_VALUE IGNORE NULLS | 1 scan | Simple, 1 window function | Snowflake, Oracle, modern DBs that support IGNORE NULLS |
| 2. COUNT groups + MAX | 1 scan | 2 window functions (subquery) | Portable. Works on DBs without IGNORE NULLS (older MySQL, SQL Server) |

**Recommendation:** Solution 1 (LAST_VALUE IGNORE NULLS) — cleanest, most readable, most efficient.

---

## Important Notes

### About `ORDER BY (SELECT NULL)`

- This tells the window function to process rows in their "natural" order (insertion order)
- Snowflake generally preserves insertion order within a micro-partition
- **For production use**, always add a proper ordering column (ID, timestamp, sequence) to guarantee order
- Without `ORDER BY`, the window function has no defined order and results are unpredictable

### About `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

```
UNBOUNDED PRECEDING = Start from the very first row in the partition
CURRENT ROW         = Stop at the row being processed right now

Together: "Look at all rows from the beginning up to where I am now"
```

This creates a **growing window** that expands as we move down the rows:
```
Row 1: window = [row 1]
Row 2: window = [row 1, row 2]
Row 3: window = [row 1, row 2, row 3]
...and so on
```

### Why COUNT(column) Ignores NULLs

This is standard SQL behavior:
- `COUNT(*)` → counts ALL rows including NULLs
- `COUNT(column_name)` → counts only rows where that column is NOT NULL
- This is why `COUNT(category)` naturally creates our groups — it only increments on non-NULL values
