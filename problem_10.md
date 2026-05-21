# PROBLEM 11: Employee Hierarchy Using Recursive CTE

## Problem Statement

You are given an employees table where each employee has a `ManagerID` pointing to another employee (their boss). The **TOP-LEVEL employee** (CEO/owner) has `ManagerID = NULL` (no one above them).

Write a query to display each employee with their **HIERARCHY LEVEL**:
- Level 0 = Top boss (ManagerID IS NULL)
- Level 1 = Reports directly to the top boss
- Level 2 = Reports to someone at Level 1
- ...and so on

This is a **TREE TRAVERSAL** problem solved with a **RECURSIVE CTE**.

---

## Visual Hierarchy (Who Reports to Whom)

```
Jenny Jeff (ID:7, Level 0) ← TOP (ManagerID = NULL)
    ├── Samuel Pitt (ID:5, Level 1) ← reports to Jenny (ManagerID=7)
    │       └── Smith Jones (ID:2, Level 2) ← reports to Samuel (ManagerID=5)
    └── Mark Miles (ID:6, Level 1) ← reports to Jenny (ManagerID=7)
            └── Richard Robinson (ID:4, Level 2) ← reports to Mark (ManagerID=6)
                    └── Hilary Riles (ID:3, Level 3) ← reports to Richard (ManagerID=4)
                            └── Adam Owens (ID:1, Level 4) ← reports to Hilary (ManagerID=3)
```

**Chain:** Jenny → Mark → Richard → Hilary → Adam

---

## Setup

```sql
CREATE OR REPLACE TABLE employees (
    EmployeeID INT,
    EmployeeName VARCHAR(50),
    DepartmentID INT,
    ManagerID INT
);

INSERT INTO employees VALUES
(1, 'Adam Owens', 103, 3),
(2, 'Smith Jones', 102, 5),
(3, 'Hilary Riles', 101, 4),
(4, 'Richard Robinson', 103, 6),
(5, 'Samuel Pitt', 101, 7),
(6, 'Mark Miles', NULL, 7),
(7, 'Jenny Jeff', 999, NULL);
```

### Sample Data

| EmployeeID | EmployeeName | DepartmentID | ManagerID |
|------------|--------------|--------------|-----------|
| 1 | Adam Owens | 103 | 3 |
| 2 | Smith Jones | 102 | 5 |
| 3 | Hilary Riles | 101 | 4 |
| 4 | Richard Robinson | 103 | 6 |
| 5 | Samuel Pitt | 101 | 7 |
| 6 | Mark Miles | NULL | 7 |
| 7 | Jenny Jeff | 999 | NULL |

---

## Expected Output

| EMPLOYEEID | EMPLOYEENAME | MANAGERID | HIERARCHY |
|------------|--------------|-----------|-----------|
| 7 | Jenny Jeff | NULL | 0 |
| 5 | Samuel Pitt | 7 | 1 |
| 6 | Mark Miles | 7 | 1 |
| 2 | Smith Jones | 5 | 2 |
| 4 | Richard Robinson | 6 | 2 |
| 3 | Hilary Riles | 4 | 3 |
| 1 | Adam Owens | 3 | 4 |

---

## Solution: Recursive CTE

```sql
WITH RECURSIVE CTE AS 
(
-- ANCHOR: Start with the top-level employee (no manager)
SELECT EMPLOYEEID, EMPLOYEENAME, MANAGERID, 0 AS HIERARCHY
FROM EMPLOYEES WHERE MANAGERID IS NULL 
UNION ALL
-- RECURSIVE: Find employees whose ManagerID matches a previously found EmployeeID
SELECT E1.EMPLOYEEID, E1.EMPLOYEENAME, E1.MANAGERID, HIERARCHY + 1
FROM EMPLOYEES E1
INNER JOIN CTE E2
ON E1.MANAGERID = E2.EMPLOYEEID
)
SELECT * FROM CTE;
```

---

## How Recursive CTEs Work

A recursive CTE has **two parts** connected by `UNION ALL`:

| Part | Purpose |
|------|---------|
| **Anchor Query** | The starting point. Runs ONCE. Finds the root node(s). |
| **Recursive Query** | Runs repeatedly. Each iteration finds the next level by joining back to the CTE's previous results. |
| **Termination** | Stops automatically when the recursive query returns 0 rows. |

### Structure

```sql
WITH RECURSIVE cte_name AS (
    -- ANCHOR (runs once, finds starting rows)
    SELECT ... FROM table WHERE <root condition>
    
    UNION ALL
    
    -- RECURSIVE (runs repeatedly, finds next level)
    SELECT ... FROM table
    INNER JOIN cte_name ON <parent-child relationship>
)
SELECT * FROM cte_name;
```

---

## Step-by-Step Execution

### Iteration 0 (Anchor)

Find employee where `ManagerID IS NULL`:

| EMPLOYEEID | EMPLOYEENAME | MANAGERID | HIERARCHY |
|------------|--------------|-----------|-----------|
| 7 | Jenny Jeff | NULL | 0 |

*CTE now contains: {7}*

---

### Iteration 1 (Recursive)

Find employees whose `ManagerID = 7` (Jenny's ID):

| EMPLOYEEID | EMPLOYEENAME | MANAGERID | HIERARCHY |
|------------|--------------|-----------|-----------|
| 5 | Samuel Pitt | 7 | 1 |
| 6 | Mark Miles | 7 | 1 |

*CTE now contains: {7, 5, 6}*

---

### Iteration 2 (Recursive)

Find employees whose `ManagerID IN (5, 6)`:

| EMPLOYEEID | EMPLOYEENAME | MANAGERID | HIERARCHY |
|------------|--------------|-----------|-----------|
| 2 | Smith Jones | 5 | 2 |
| 4 | Richard Robinson | 6 | 2 |

*CTE now contains: {7, 5, 6, 2, 4}*

---

### Iteration 3 (Recursive)

Find employees whose `ManagerID IN (2, 4)`:

| EMPLOYEEID | EMPLOYEENAME | MANAGERID | HIERARCHY |
|------------|--------------|-----------|-----------|
| 3 | Hilary Riles | 4 | 3 |

*CTE now contains: {7, 5, 6, 2, 4, 3}*

---

### Iteration 4 (Recursive)

Find employees whose `ManagerID IN (3)`:

| EMPLOYEEID | EMPLOYEENAME | MANAGERID | HIERARCHY |
|------------|--------------|-----------|-----------|
| 1 | Adam Owens | 3 | 4 |

*CTE now contains: {7, 5, 6, 2, 4, 3, 1}*

---

### Iteration 5 (Recursive)

Find employees whose `ManagerID IN (1)`:

→ **No matches found → STOP**

---

## Breaking Down Each Part of the Query

| Part | Meaning |
|------|---------|
| `WITH RECURSIVE CTE AS` | Declare a CTE named "CTE" that is allowed to reference itself |
| `SELECT ... WHERE MANAGERID IS NULL` | **Anchor:** Find the top-level employee (root of the tree) |
| `0 AS HIERARCHY` | Assign level 0 to the root |
| `UNION ALL` | Combine anchor results with recursive results |
| `FROM EMPLOYEES E1 INNER JOIN CTE E2` | **Recursive:** Join the employees table with the CTE's current results |
| `ON E1.MANAGERID = E2.EMPLOYEEID` | Match: "my manager is someone already found in previous iterations" |
| `HIERARCHY + 1` | Each level is one more than the parent's level |
| `SELECT * FROM CTE` | Final query: return all accumulated rows from all iterations |

---

## Key Concepts

| Concept | Explanation |
|---------|-------------|
| **Recursive CTE** | A CTE that references itself. Used for hierarchical/tree data. |
| **Anchor** | The non-recursive part. Provides the starting rows (base case). |
| **Recursive member** | The part that references the CTE. Finds the next level each iteration. |
| **UNION ALL** | Required between anchor and recursive parts. Combines all levels together. |
| **Termination** | Automatic when recursive part returns 0 rows. No explicit stop needed. |
| **Tree traversal** | Walking through a tree structure level by level (Breadth-First Search). |
| **Self-referencing table** | A table with a foreign key pointing to its own primary key (ManagerID → EmployeeID). |
