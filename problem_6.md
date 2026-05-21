# PROBLEM 5: Remove Duplicate Records (Keep One Copy)

Multiple approaches from most efficient to alternatives.

---

## Setup: Create Table with Duplicate Records

```sql
CREATE OR REPLACE TABLE EMPLOYEES (
    id INT,
    name VARCHAR(50),
    department VARCHAR(50),
    salary NUMBER(10,2)
);

INSERT INTO EMPLOYEES VALUES
(1, 'Alice', 'Engineering', 90000),
(1, 'Alice', 'Engineering', 90000),   -- duplicate
(1, 'Alice', 'Engineering', 90000),   -- duplicate
(2, 'Bob', 'Marketing', 75000),
(2, 'Bob', 'Marketing', 75000),       -- duplicate
(3, 'Charlie', 'Sales', 80000),
(4, 'Diana', 'Engineering', 95000),
(4, 'Diana', 'Engineering', 95000),   -- duplicate
(4, 'Diana', 'Engineering', 95000),   -- duplicate
(4, 'Diana', 'Engineering', 95000),   -- duplicate
(5, 'Eve', 'Marketing', 72000);
```

### Verify Duplicates Exist

```sql
SELECT *, COUNT(*) OVER (PARTITION BY id, name, department, salary) AS dup_count
FROM EMPLOYEES
ORDER BY id;
```

### Original Data (11 rows)

| ID | NAME | DEPARTMENT | SALARY | Copies |
|----|------|-----------|--------|--------|
| 1 | Alice | Engineering | 90000 | 3 |
| 2 | Bob | Marketing | 75000 | 2 |
| 3 | Charlie | Sales | 80000 | 1 |
| 4 | Diana | Engineering | 95000 | 4 |
| 5 | Eve | Marketing | 72000 | 1 |

### Expected Output After Deduplication (5 rows)

| ID | NAME | DEPARTMENT | SALARY |
|----|------|-----------|--------|
| 1 | Alice | Engineering | 90000 |
| 2 | Bob | Marketing | 75000 |
| 3 | Charlie | Sales | 80000 |
| 4 | Diana | Engineering | 95000 |
| 5 | Eve | Marketing | 72000 |

---

## METHOD 1: CTAS WITH ROW_NUMBER (Most Efficient on Snowflake)

**WHY MOST EFFICIENT:**
- Single table scan
- No self-join
- Snowflake optimizes window functions very well
- Atomic swap (no downtime)

```sql
CREATE OR REPLACE TABLE EMPLOYEES AS
SELECT id, name, department, salary
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY id, name, department, salary
            ORDER BY id
        ) AS rn
    FROM SQL.PROBLEMS.EMPLOYEES
)
WHERE rn = 1;

SELECT * FROM EMPLOYEES ORDER BY id;
```

### How Method 1 Works (Step-by-Step)

**STEP 1:** `ROW_NUMBER()` assigns a sequential number WITHIN each group of identical rows (`PARTITION BY` all columns):

| ID | NAME | DEPARTMENT | SALARY | RN | Action |
|----|------|-----------|--------|----|--------|
| 1 | Alice | Engineering | 90000 | 1 | KEEP (first in group) |
| 1 | Alice | Engineering | 90000 | 2 | DISCARD |
| 1 | Alice | Engineering | 90000 | 3 | DISCARD |
| 2 | Bob | Marketing | 75000 | 1 | KEEP |
| 2 | Bob | Marketing | 75000 | 2 | DISCARD |
| 3 | Charlie | Sales | 80000 | 1 | KEEP (only one, no dup) |
| 4 | Diana | Engineering | 95000 | 1 | KEEP |
| 4 | Diana | Engineering | 95000 | 2 | DISCARD |
| 4 | Diana | Engineering | 95000 | 3 | DISCARD |
| 4 | Diana | Engineering | 95000 | 4 | DISCARD |
| 5 | Eve | Marketing | 72000 | 1 | KEEP |

**STEP 2:** `WHERE rn = 1` filters to keep ONLY the first row per group → 5 rows remain.

**STEP 3:** `CREATE OR REPLACE TABLE ... AS` replaces the original table with this deduplicated result. This is **ATOMIC** — the old table is dropped and the new one created in a single transaction. No moment where the table is empty or missing.

### Execution Flow in Snowflake

1. Scan EMPLOYEES once (reads all 11 rows from micro-partitions)
2. Hash-partition rows by (id, name, department, salary)
3. Assign ROW_NUMBER within each partition (O(n) per partition)
4. Filter rn = 1 (discard 6 duplicate rows)
5. Write 5 rows into new micro-partitions
6. Atomically swap metadata to point table name at new data
7. Old micro-partitions marked for garbage collection

### Why It's Fast

- **ONE** scan of source data (not two like a self-join DELETE)
- Window functions run IN-MEMORY per partition (no spill for small groups)
- No index lookups, no row-by-row deletion
- Snowflake parallelizes across all micro-partitions simultaneously
- CTAS writes new data sequentially (no random I/O)
- **Cost:** O(n) time, O(n) space for the new table, then old table storage is reclaimed

---

## METHOD 2: INSERT OVERWRITE WITH DISTINCT (Simple & Efficient)

```sql
INSERT OVERWRITE INTO SQL.PROBLEMS.EMPLOYEES
SELECT DISTINCT id, name, department, salary
FROM SQL.PROBLEMS.EMPLOYEES;

SELECT * FROM SQL.PROBLEMS.EMPLOYEES ORDER BY id;
```

### How It Works

- `SELECT DISTINCT` removes all duplicate rows
- `INSERT OVERWRITE` replaces the entire table contents atomically
- No temp table needed, no subquery complexity
- Single scan of the data

---

## METHOD 3: SWAP WITH TEMP TABLE (Safe for Production)

```sql
-- Step 1: Create temp table with distinct records
CREATE OR REPLACE TEMPORARY TABLE SQL.PROBLEMS.EMPLOYEES_DEDUP AS
SELECT DISTINCT id, name, department, salary
FROM SQL.PROBLEMS.EMPLOYEES;

-- Step 2: Swap the tables (atomic, instant, preserves grants)
ALTER TABLE SQL.PROBLEMS.EMPLOYEES SWAP WITH SQL.PROBLEMS.EMPLOYEES_DEDUP;

-- Step 3: Drop the old data (now in the temp table name)
DROP TABLE SQL.PROBLEMS.EMPLOYEES_DEDUP;

SELECT * FROM SQL.PROBLEMS.EMPLOYEES ORDER BY id;
```

### How It Works

1. **Create temp table** with only distinct rows (deduplicated copy)
2. **SWAP** atomically exchanges the data between the two tables — this is instant regardless of table size because it only swaps metadata pointers
3. **DROP** the temp table (which now holds the old duplicated data)

### Why Use This in Production

- Preserves all grants/permissions on the original table
- Zero downtime (SWAP is instant)
- Rollback possible if you don't drop the temp table immediately

---

## METHOD 4: QUALIFY CLAUSE (Snowflake-Specific, Cleanest Syntax)

```sql
CREATE OR REPLACE TABLE SQL.PROBLEMS.EMPLOYEES AS
SELECT id, name, department, salary
FROM SQL.PROBLEMS.EMPLOYEES
QUALIFY ROW_NUMBER() OVER (PARTITION BY id, name, department, salary ORDER BY id) = 1;

SELECT * FROM SQL.PROBLEMS.EMPLOYEES ORDER BY id;
```

### How It Works

- `QUALIFY` is Snowflake's elegant filter on window functions
- Eliminates the need for a subquery/CTE
- Same logic as Method 1 but in fewer lines
- `QUALIFY` filters AFTER the window function is computed (like `HAVING` filters after `GROUP BY`)

---

## METHOD 5: GROUP BY (When All Columns Define Uniqueness)

```sql
CREATE OR REPLACE TABLE SQL.PROBLEMS.EMPLOYEES AS
SELECT id, name, department, salary
FROM SQL.PROBLEMS.EMPLOYEES
GROUP BY id, name, department, salary;

SELECT * FROM SQL.PROBLEMS.EMPLOYEES ORDER BY id;
```

### How It Works

- `GROUP BY` all columns collapses identical rows into one
- Simple but can't pick a specific row to keep (no ordering control)
- Works only when ALL columns together define the duplicate

---

## Efficiency Ranking (Best to Worst on Snowflake)

| Rank | Method | Why |
|------|--------|-----|
| 1 | CTAS + QUALIFY | Single scan, no subquery, Snowflake-native |
| 2 | CTAS + ROW_NUMBER | Single scan, works on all platforms |
| 3 | INSERT OVERWRITE + DISTINCT | Single scan, no temp table needed |
| 4 | SWAP with temp table | Atomic, safe for production, preserves grants/permissions |
| 5 | GROUP BY | Simple but can't pick specific row to keep |
