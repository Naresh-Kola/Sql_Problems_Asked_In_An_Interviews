# Problem 20: Correlated Subqueries — Employees Above Department Average

## Problem Statement

Find employees whose salary is greater than the average salary of their department. Then demonstrate how to UPDATE and DELETE using the same logic with multiple tables.

---

## Table Setup

```sql
CREATE OR REPLACE TABLE employee_salary (
    employee_id   INT            PRIMARY KEY,
    name          VARCHAR(100)   NOT NULL,
    department    VARCHAR(50)    NOT NULL,
    salary        DECIMAL(10, 2) NOT NULL
);

INSERT INTO employee_salary (employee_id, name, department, salary)
VALUES
    (1,  'Alice',   'Engineering', 90000),
    (2,  'Bob',     'Engineering', 70000),
    (3,  'Charlie', 'Engineering', 80000),
    (4,  'Diana',   'HR',          55000),
    (5,  'Eve',     'HR',          48000),
    (6,  'Frank',   'HR',          62000),
    (7,  'Grace',   'Finance',     75000),
    (8,  'Henry',   'Finance',     68000),
    (9,  'Ivy',     'Finance',     72000);
```

---

## Solution 1: Window Function (Best)

```sql
SELECT EMPLOYEE_ID, NAME, SALARY, AV_G FROM (
  SELECT *, AVG(SALARY) OVER(PARTITION BY DEPARTMENT) AS AV_G
  FROM EMPLOYEE_SALARY
) WHERE SALARY > AV_G;
```

---

## Solution 2: Correlated Subquery

```sql
SELECT E1.*
FROM employee_salary E1
WHERE SALARY > (
  SELECT AVG(SALARY) FROM employee_salary E2
  WHERE E1.DEPARTMENT = E2.DEPARTMENT
);
```

---

## Solution 3: INNER JOIN with Derived Table

```sql
SELECT E1.*
FROM EMPLOYEE_SALARY E1
INNER JOIN (
  SELECT DEPARTMENT, AVG(SALARY) AS AV_G
  FROM EMPLOYEE_SALARY
  GROUP BY DEPARTMENT
) E2
ON E1.DEPARTMENT = E2.DEPARTMENT
WHERE E1.SALARY > E2.AV_G
ORDER BY E1.EMPLOYEE_ID;
```

---

## UPDATE: Double Salary for Above-Average Employees

### Using FROM (join approach)

```sql
UPDATE employee_salary E1
SET SALARY = SALARY * 2
FROM (SELECT DEPARTMENT, AVG(SALARY) AS AV_G FROM EMPLOYEE_SALARY GROUP BY DEPARTMENT) E2
WHERE E1.DEPARTMENT = E2.DEPARTMENT
AND E1.SALARY > E2.AV_G;
```

### Using Correlated Subquery

```sql
UPDATE employee_salary E1
SET SALARY = SALARY * 2 
WHERE SALARY > (
  SELECT AVG(SALARY) FROM employee_salary E2
  WHERE E1.DEPARTMENT = E2.DEPARTMENT
);
```

---

## DELETE: Remove Above-Average Employees

### Using USING (join approach)

```sql
DELETE FROM employee_salary E1
USING (SELECT DEPARTMENT, AVG(SALARY) AS AV_G FROM EMPLOYEE_SALARY GROUP BY DEPARTMENT) E2
WHERE E1.DEPARTMENT = E2.DEPARTMENT AND E1.SALARY > E2.AV_G;
```

### Using Correlated Subquery

```sql
DELETE FROM employee_salary E1
WHERE SALARY > (
  SELECT AVG(SALARY) FROM employee_salary E2
  WHERE E1.DEPARTMENT = E2.DEPARTMENT
);
```

---

## Step-by-Step Execution of the Correlated Subquery

This is a **CORRELATED SUBQUERY** — the inner query depends on the outer query. It executes once PER ROW of the outer query.

### Walkthrough

| Step | Row | Department | Salary | Dept Avg | Salary > Avg? | Result |
|------|-----|-----------|--------|----------|---------------|--------|
| 1 | Alice | Engineering | 90000 | 80000 | ✅ YES | Included |
| 2 | Bob | Engineering | 70000 | 80000 | ❌ NO | Excluded |
| 3 | Charlie | Engineering | 80000 | 80000 | ❌ NO | Excluded |
| 4 | Diana | HR | 55000 | 55000 | ❌ NO | Excluded |
| 5 | Eve | HR | 48000 | 55000 | ❌ NO | Excluded |
| 6 | Frank | HR | 62000 | 55000 | ✅ YES | Included |
| 7 | Grace | Finance | 75000 | 71667 | ✅ YES | Included |
| 8 | Henry | Finance | 68000 | 71667 | ❌ NO | Excluded |
| 9 | Ivy | Finance | 72000 | 71667 | ✅ YES | Included |

### Final Result

| EMPLOYEE_ID | NAME | DEPARTMENT | SALARY |
|-------------|------|------------|--------|
| 1 | Alice | Engineering | 90000 |
| 6 | Frank | HR | 62000 |
| 7 | Grace | Finance | 75000 |
| 9 | Ivy | Finance | 72000 |

---

## Execution Flow Diagram

```
┌──────────────┐
│ OUTER QUERY  │──── For each row in E1:
│ (Full Scan)  │
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│ INNER SUBQUERY   │──── Compute AVG(SALARY)
│ (Per-row scan of │     WHERE dept matches
│  E2 for matching │     current E1 row
│  department)     │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ FILTER           │──── E1.SALARY > AVG result?
│                  │     YES → include in output
│                  │     NO  → skip
└──────────────────┘
```

---

## Key Concepts

| Type | Description |
|------|-------------|
| **Correlated** | Inner query references outer query (`E1.DEPARTMENT`). Executes ONCE PER ROW. |
| **Non-Correlated** | Inner query is independent. Executes ONCE total, result reused for all rows. |

> **Note:** Snowflake's optimizer may internally rewrite correlated subqueries as JOINs for better performance, but logically they execute as described above.

---

## Snowflake Syntax Summary

| Operation | Keyword | Example |
|-----------|---------|---------|
| SELECT | `JOIN ... ON` | `SELECT ... FROM t1 JOIN t2 ON ...` |
| UPDATE | `FROM` | `UPDATE t1 SET ... FROM t2 WHERE ...` |
| DELETE | `USING` | `DELETE FROM t1 USING t2 WHERE ...` |
