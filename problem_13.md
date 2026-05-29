# Problem 21: UPDATE and DELETE with Multiple Table Joins

## Table Setup

```sql
CREATE OR REPLACE TABLE departments (
    dept_id   INT PRIMARY KEY,
    dept_name VARCHAR(50)
);

CREATE OR REPLACE TABLE employees (
    emp_id    INT PRIMARY KEY,
    emp_name  VARCHAR(100),
    dept_id   INT,
    salary    DECIMAL(10,2),
    hire_date DATE
);

CREATE OR REPLACE TABLE performance_reviews (
    review_id   INT PRIMARY KEY,
    emp_id      INT,
    review_year INT,
    rating      VARCHAR(20)
);

INSERT INTO departments VALUES
    (1, 'Engineering'),
    (2, 'Sales'),
    (3, 'HR'),
    (4, 'Finance');

INSERT INTO employees VALUES
    (101, 'Alice',   1, 85000, '2019-03-15'),
    (102, 'Bob',     1, 72000, '2020-07-01'),
    (103, 'Charlie', 2, 60000, '2018-01-10'),
    (104, 'Diana',   2, 55000, '2021-06-20'),
    (105, 'Eve',     3, 48000, '2017-11-05'),
    (106, 'Frank',   3, 52000, '2022-02-14'),
    (107, 'Grace',   4, 90000, '2016-08-22'),
    (108, 'Henry',   4, 78000, '2023-01-30');

INSERT INTO performance_reviews VALUES
    (1, 101, 2024, 'Excellent'),
    (2, 102, 2024, 'Good'),
    (3, 103, 2024, 'Poor'),
    (4, 104, 2024, 'Good'),
    (5, 105, 2024, 'Poor'),
    (6, 106, 2024, 'Excellent'),
    (7, 107, 2024, 'Good'),
    (8, 108, 2024, 'Excellent');
```

---

## Question 1: UPDATE with Multiple Tables

> Give a 20% salary raise to all employees who received an 'Excellent' rating in 2024 and belong to a department whose name starts with 'E' or 'F'.
>
> (Join: employees + performance_reviews + departments)

### Solution

```sql
UPDATE employees E 
SET SALARY = SALARY * 1.20
FROM departments D, performance_reviews P
WHERE E.DEPT_ID = D.DEPT_ID
  AND P.EMP_ID = E.EMP_ID
  AND (D.DEPT_NAME LIKE 'E%' OR D.DEPT_NAME LIKE 'F%')
  AND P.RATING = 'Excellent'
  AND P.REVIEW_YEAR = 2024;
```

### Explanation

- **Syntax:** `UPDATE ... SET ... FROM table1, table2 WHERE ...`
- Joins 3 tables via comma-separated list in `FROM`
- All join conditions go in `WHERE`
- Affected rows: Alice (Engineering, Excellent) and Henry (Finance, Excellent)

---

## Question 2: DELETE with Multiple Tables

> Delete all employees who received a 'Poor' rating in 2024 and belong to departments that have fewer than 3 employees.
>
> (Join: employees + performance_reviews + departments)

### Solution

```sql
DELETE FROM employees E
USING departments D, performance_reviews P
WHERE E.DEPT_ID = D.DEPT_ID
  AND P.EMP_ID = E.EMP_ID
  AND P.RATING = 'Poor'
  AND P.REVIEW_YEAR = 2024
  AND E.DEPT_ID IN (SELECT DEPT_ID FROM employees GROUP BY DEPT_ID HAVING COUNT(*) < 3);
```

### Explanation

- **Syntax:** `DELETE FROM ... USING table1, table2 WHERE ...`
- Joins 3 tables via comma-separated list in `USING`
- Uses a subquery to check department size
- Affected rows: Charlie (Sales, Poor, dept has 2 employees) and Eve (HR, Poor, dept has 2 employees)

---

## Key Snowflake Syntax Rules

| Operation | Join Keyword | Syntax |
|-----------|-------------|--------|
| UPDATE | `FROM` | `UPDATE t SET ... FROM t2, t3 WHERE ...` |
| DELETE | `USING` | `DELETE FROM t USING t2, t3 WHERE ...` |
| SELECT | `JOIN` | `SELECT ... FROM t JOIN t2 ON ... JOIN t3 ON ...` |

- `INNER JOIN` after `SET` or `DELETE FROM` is **not supported**
- All join conditions must be in the `WHERE` clause
