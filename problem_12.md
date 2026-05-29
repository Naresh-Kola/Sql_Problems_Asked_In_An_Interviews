# Problem 22: Delete Duplicate Records (Keep Latest)

## Problem Statement

The `employee_records` table has entries where the same employee appears more than once with the same name, email, and department combination.

**Task:** Delete all duplicate rows but retain only the one with the highest `id` (latest inserted) for each duplicate group.

---

## Table Setup

```sql
CREATE OR REPLACE TABLE employee_records (
    id           INT            PRIMARY KEY AUTOINCREMENT,
    name         VARCHAR(100)   NOT NULL,
    email        VARCHAR(150)   NOT NULL,
    department   VARCHAR(50)    NOT NULL,
    created_at   TIMESTAMP      DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO employee_records (name, email, department)
VALUES
  ('Alice',   'alice@company.com',  'Engineering'),
  ('Alice',   'alice@company.com',  'Engineering'),
  ('Alice',   'alice@company.com',  'Engineering'),
  ('Bob',     'bob@company.com',    'HR'),
  ('Bob',     'bob@company.com',    'HR'),
  ('Charlie', 'charlie@company.com','Finance'),
  ('Diana',   'diana@company.com',  'Finance'),
  ('Diana',   'diana@company.com',  'Finance'),
  ('Eve',     'eve@company.com',    'Engineering');
```

---

## Find Duplicates

```sql
SELECT name, email, department, COUNT(*)
FROM employee_records
GROUP BY name, email, department
HAVING COUNT(*) > 1;
```

---

## Solutions: 6 Ways to Delete Duplicates (Ordered by Efficiency)

### Method 1: NOT IN with MAX (Most Efficient, Simplest)

```sql
DELETE FROM employee_records
WHERE id NOT IN (
  SELECT MAX(id)
  FROM employee_records
  GROUP BY name, email, department
);
```

**Why best:** Single aggregate scan, no correlated subquery, no window functions. Optimizer handles it in one pass.

---

### Method 2: ROW_NUMBER Subquery (Efficient, Flexible)

```sql
DELETE FROM employee_records
WHERE id IN (
  SELECT id FROM (
    SELECT id, ROW_NUMBER() OVER(PARTITION BY name, email, department ORDER BY id DESC) AS rn
    FROM employee_records
  ) WHERE rn > 1
);
```

**Why good:** Flexible — can easily change tie-breaking logic (e.g., keep earliest, keep by date, etc.)

---

### Method 3: Correlated Subquery with != MAX (Moderate)

```sql
DELETE FROM employee_records E1
WHERE id != (
  SELECT MAX(id) FROM employee_records E2
  WHERE E1.name = E2.name
    AND E1.email = E2.email
    AND E1.department = E2.department
);
```

**Trade-off:** Readable but runs the subquery per row (optimizer may rewrite as join).

---

### Method 4: EXISTS with a Better Duplicate (Moderate)

```sql
DELETE FROM employee_records E1
WHERE EXISTS (
  SELECT 1 FROM employee_records E2
  WHERE E1.name = E2.name
    AND E1.email = E2.email
    AND E1.department = E2.department
    AND E2.id > E1.id
);
```

**Why useful:** Clear intent — "delete this row if a better one exists."

---

### Method 5: USING with Aggregated Subquery (Multi-Table Pattern)

```sql
DELETE FROM employee_records E1
USING (
  SELECT name, email, department, MAX(id) AS max_id
  FROM employee_records
  GROUP BY name, email, department
  HAVING COUNT(*) > 1
) E2
WHERE E1.name = E2.name
  AND E1.email = E2.email
  AND E1.department = E2.department
  AND E1.id != E2.max_id;
```

**Why useful:** Demonstrates the `USING` syntax; only touches groups that actually have duplicates.

---

### Method 6: SWAP TABLE (Best for Massive Tables)

```sql
CREATE OR REPLACE TABLE employee_records_clean AS
SELECT * FROM employee_records
QUALIFY ROW_NUMBER() OVER(PARTITION BY name, email, department ORDER BY id DESC) = 1;

ALTER TABLE employee_records_clean SWAP WITH employee_records;
DROP TABLE employee_records_clean;
```

**Why best for large data:** Avoids row-by-row deletion entirely. Rebuilds a clean table in one scan, then atomic swap. No transaction log bloat.

---

## Summary Table

| # | Method | Performance | Best For |
|---|--------|-------------|----------|
| 1 | NOT IN + MAX | ⭐⭐⭐⭐⭐ | Small-medium tables, simple dedup |
| 2 | ROW_NUMBER subquery | ⭐⭐⭐⭐ | Complex tie-breaking logic |
| 3 | Correlated != MAX | ⭐⭐⭐ | Readability |
| 4 | EXISTS | ⭐⭐⭐ | Clear intent |
| 5 | USING + aggregate | ⭐⭐⭐ | Multi-table patterns |
| 6 | SWAP TABLE | ⭐⭐⭐⭐⭐ | Massive tables, bulk dedup |

---

## Key Snowflake Notes

- CTEs **cannot** be used before DELETE (`WITH cte AS (...) DELETE ...` is invalid)
- DELETE with joins uses **`USING`** (not `INNER JOIN`)
- UPDATE with joins uses **`FROM`** (not `INNER JOIN`)
- `QUALIFY` is Snowflake-specific and eliminates the need for subqueries with window functions
