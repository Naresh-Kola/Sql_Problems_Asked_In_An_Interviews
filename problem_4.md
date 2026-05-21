# PROBLEM: Calculate Daily Working Hours Per Employee

Given employee punch-in (intime) and punch-out (outtime) records, calculate the **TOTAL HOURS WORKED PER EMPLOYEE PER DAY**.

## EDGE CASES:
1. **Normal shift:** intime and outtime on same day
2. **Overnight shift:** intime PM, outtime next day AM (crosses midnight) → Split hours between TWO calendar days
3. **Multiple shifts in one day** (employee leaves and comes back)
4. **Very early morning shifts** (starts 4 AM)
5. **24-hour shift** spanning full day

---

## Setup: Create Table

```sql
CREATE OR REPLACE TABLE employee_attendance (
    emp_id INT,
    emp_name VARCHAR(50),
    intime TIMESTAMP,
    outtime TIMESTAMP
);
```

## Setup: Insert Sample Data

```sql
INSERT INTO employee_attendance (emp_id, emp_name, intime, outtime) VALUES
-- RAHUL: Works multiple days, mix of day and night shifts
(1, 'Rahul', '2024-06-01 09:00:00', '2024-06-01 18:00:00'),  -- Day shift: 9 hrs on Jun 1
(1, 'Rahul', '2024-06-02 10:00:00', '2024-06-02 19:30:00'),  -- Day shift: 9.5 hrs on Jun 2
(1, 'Rahul', '2024-06-03 20:00:00', '2024-06-04 05:00:00'),  -- Night shift: 4 hrs Jun 3 + 5 hrs Jun 4
(1, 'Rahul', '2024-06-04 18:00:00', '2024-06-04 22:00:00'),  -- Day shift: 4 hrs on Jun 4 (2nd shift same day)
(1, 'Rahul', '2024-06-05 09:00:00', '2024-06-05 17:00:00'),  -- Day shift: 8 hrs on Jun 5

-- PRIYA: Mostly night shifts (crosses midnight frequently)
(2, 'Priya', '2024-06-01 22:00:00', '2024-06-02 06:00:00'),  -- Night: 2 hrs Jun 1 + 6 hrs Jun 2
(2, 'Priya', '2024-06-02 21:00:00', '2024-06-03 05:30:00'),  -- Night: 3 hrs Jun 2 + 5.5 hrs Jun 3
(2, 'Priya', '2024-06-03 22:00:00', '2024-06-04 07:00:00'),  -- Night: 2 hrs Jun 3 + 7 hrs Jun 4
(2, 'Priya', '2024-06-05 19:00:00', '2024-06-06 04:00:00'),  -- Night: 5 hrs Jun 5 + 4 hrs Jun 6
(2, 'Priya', '2024-06-06 20:00:00', '2024-06-07 03:00:00'),  -- Night: 4 hrs Jun 6 + 3 hrs Jun 7

-- AMIT: Regular day shifts, consistent 8 hours
(3, 'Amit', '2024-06-01 08:00:00', '2024-06-01 16:00:00'),   -- 8 hrs Jun 1
(3, 'Amit', '2024-06-02 08:30:00', '2024-06-02 17:00:00'),   -- 8.5 hrs Jun 2
(3, 'Amit', '2024-06-03 09:00:00', '2024-06-03 18:00:00'),   -- 9 hrs Jun 3
(3, 'Amit', '2024-06-04 08:00:00', '2024-06-04 16:30:00'),   -- 8.5 hrs Jun 4
(3, 'Amit', '2024-06-05 08:00:00', '2024-06-05 16:00:00'),   -- 8 hrs Jun 5

-- KAVITA: Mix of shifts + edge case (shift ending exactly at midnight)
(4, 'Kavita', '2024-06-01 14:00:00', '2024-06-02 00:00:00'), -- 10 hrs on Jun 1 (ends at midnight exactly)
(4, 'Kavita', '2024-06-02 09:00:00', '2024-06-02 17:00:00'), -- 8 hrs Jun 2
(4, 'Kavita', '2024-06-03 23:00:00', '2024-06-04 08:00:00'), -- Night: 1 hr Jun 3 + 8 hrs Jun 4
(4, 'Kavita', '2024-06-05 06:00:00', '2024-06-05 14:00:00'), -- Early morning: 8 hrs Jun 5
(4, 'Kavita', '2024-06-05 19:00:00', '2024-06-06 02:00:00'), -- Night: 5 hrs Jun 5 + 2 hrs Jun 6

-- DEEPAK: Has a 24-hour shift + normal shifts
(5, 'Deepak', '2024-06-01 08:00:00', '2024-06-02 08:00:00'), -- 24hr: 16 hrs Jun 1 + 8 hrs Jun 2
(5, 'Deepak', '2024-06-03 09:00:00', '2024-06-03 18:00:00'), -- 9 hrs Jun 3
(5, 'Deepak', '2024-06-04 22:00:00', '2024-06-05 06:00:00'), -- Night: 2 hrs Jun 4 + 6 hrs Jun 5
(5, 'Deepak', '2024-06-05 20:00:00', '2024-06-06 04:30:00'), -- Night: 4 hrs Jun 5 + 4.5 hrs Jun 6

-- SNEHA: Edge case - multiple short shifts in same day
(6, 'Sneha', '2024-06-01 08:00:00', '2024-06-01 12:00:00'),  -- 4 hrs morning Jun 1
(6, 'Sneha', '2024-06-01 13:00:00', '2024-06-01 17:00:00'),  -- 4 hrs afternoon Jun 1
(6, 'Sneha', '2024-06-01 20:00:00', '2024-06-01 23:00:00'),  -- 3 hrs evening Jun 1
(6, 'Sneha', '2024-06-02 23:30:00', '2024-06-03 07:30:00'),  -- Night: 0.5 hrs Jun 2 + 7.5 hrs Jun 3
(6, 'Sneha', '2024-06-04 09:00:00', '2024-06-04 17:30:00');  -- 8.5 hrs Jun 4
```

---

## Expected Output

Total hours worked PER EMPLOYEE PER DAY:

| EMP_ID | EMP_NAME | WORK_DATE  | TOTAL_HOURS | Explanation |
|--------|----------|------------|-------------|-------------|
| 1 | Rahul | 2024-06-01 | 9.00 | |
| 1 | Rahul | 2024-06-02 | 9.50 | |
| 1 | Rahul | 2024-06-03 | 4.00 | ← only 4 hrs (20:00 to midnight) |
| 1 | Rahul | 2024-06-04 | 9.00 | ← 5 hrs (midnight to 05:00) + 4 hrs (18:00 to 22:00) |
| 1 | Rahul | 2024-06-05 | 8.00 | |
| 2 | Priya | 2024-06-01 | 2.00 | ← 22:00 to midnight |
| 2 | Priya | 2024-06-02 | 9.00 | ← 6 hrs (00:00-06:00) + 3 hrs (21:00-midnight) |
| 2 | Priya | 2024-06-03 | 7.50 | ← 5.5 hrs (00:00-05:30) + 2 hrs (22:00-midnight) |
| 2 | Priya | 2024-06-04 | 7.00 | ← 7 hrs (00:00-07:00) |
| 2 | Priya | 2024-06-05 | 5.00 | ← 5 hrs (19:00-midnight) |
| 2 | Priya | 2024-06-06 | 8.00 | ← 4 hrs (00:00-04:00) + 4 hrs (20:00-midnight) |
| 2 | Priya | 2024-06-07 | 3.00 | ← 3 hrs (00:00-03:00) |
| 3 | Amit | 2024-06-01 | 8.00 | |
| 3 | Amit | 2024-06-02 | 8.50 | |
| 3 | Amit | 2024-06-03 | 9.00 | |
| 3 | Amit | 2024-06-04 | 8.50 | |
| 3 | Amit | 2024-06-05 | 8.00 | |
| 4 | Kavita | 2024-06-01 | 10.00 | ← ends exactly at midnight (all on Jun 1) |
| 4 | Kavita | 2024-06-02 | 8.00 | |
| 4 | Kavita | 2024-06-03 | 1.00 | ← 23:00 to midnight |
| 4 | Kavita | 2024-06-04 | 8.00 | ← midnight to 08:00 |
| 4 | Kavita | 2024-06-05 | 13.00 | ← 8 hrs (06:00-14:00) + 5 hrs (19:00-midnight) |
| 4 | Kavita | 2024-06-06 | 2.00 | ← midnight to 02:00 |
| 5 | Deepak | 2024-06-01 | 16.00 | ← 08:00 to midnight |
| 5 | Deepak | 2024-06-02 | 8.00 | ← midnight to 08:00 |
| 5 | Deepak | 2024-06-03 | 9.00 | |
| 5 | Deepak | 2024-06-04 | 2.00 | ← 22:00 to midnight |
| 5 | Deepak | 2024-06-05 | 10.00 | ← 6 hrs (00:00-06:00) + 4 hrs (20:00-midnight) |
| 5 | Deepak | 2024-06-06 | 4.50 | ← midnight to 04:30 |
| 6 | Sneha | 2024-06-01 | 11.00 | ← 4+4+3 hrs (three shifts same day) |
| 6 | Sneha | 2024-06-02 | 0.50 | ← 23:30 to midnight |
| 6 | Sneha | 2024-06-03 | 7.50 | ← midnight to 07:30 |
| 6 | Sneha | 2024-06-04 | 8.50 | |

---

## EXPLANATION: STEP-BY-STEP LOGIC

### THE CORE IDEA:

If a shift crosses midnight, **SPLIT** it into TWO rows:
- **Row 1:** intime → midnight (belongs to Day 1)
- **Row 2:** midnight → outtime (belongs to Day 2)

Then `GROUP BY (emp_id, date)` and `SUM` hours.

---

### EXAMPLE WALKTHROUGH: Kavita works 2024-06-01 19:00 → 2024-06-02 02:00

**STEP 1: CHECK — Is it overnight?**
```
DATE_TRUNC('DAY', intime)  = 2024-06-01
DATE_TRUNC('DAY', outtime) = 2024-06-02
They are DIFFERENT → YES, it crosses midnight
```

**STEP 2: PART 1 — Calculate hours on the INTIME day (Jun 1)**
```
work_date  = DATE_TRUNC('DAY', intime) = 2024-06-01
end_point  = DATEADD('DAY', 1, '2024-06-01 00:00') = 2024-06-02 00:00 (midnight)
hours      = DATEDIFF('MINUTE', '19:00', '00:00 next day') / 60 = 300/60 = 5 hours
Result row: (Kavita, 2024-06-01, 5.0)
```

**STEP 3: PART 2 — Calculate hours on the OUTTIME day (Jun 2)**
```
work_date   = DATE_TRUNC('DAY', outtime) = 2024-06-02
start_point = DATE_TRUNC('DAY', outtime) = 2024-06-02 00:00 (midnight)
hours       = DATEDIFF('MINUTE', '00:00', '02:00') / 60 = 120/60 = 2 hours
Result row: (Kavita, 2024-06-02, 2.0)
```

**STEP 4: UNION ALL combines both rows:**
```
| Kavita | 2024-06-01 | 5.0 |  ← from Part 1
| Kavita | 2024-06-02 | 2.0 |  ← from Part 2
```

**STEP 5: GROUP BY emp_id, work_date → SUM(hours)**
```
If Kavita also has a day shift on Jun 2 (09:00-17:00 = 8 hrs):
| Kavita | 2024-06-02 | 2.0 + 8.0 = 10.0 |
```

---

### SAME-DAY SHIFT EXAMPLE: Rahul works 2024-06-01 09:00 → 18:00

**STEP 1: CHECK — Is it overnight?**
```
DATE_TRUNC('DAY', intime)  = 2024-06-01
DATE_TRUNC('DAY', outtime) = 2024-06-01
They are SAME → NO, normal day shift
```

**STEP 2: PART 1 — Since same day, CASE picks outtime directly**
```
work_date = 2024-06-01
end_point = outtime = 2024-06-01 18:00 (no midnight split needed)
hours     = DATEDIFF('MINUTE', '09:00', '18:00') / 60 = 540/60 = 9 hours
Result row: (Rahul, 2024-06-01, 9.0)
```

**STEP 3: PART 2 — WHERE clause filters this out (same day = no Part 2 row)**

---

### EDGE CASE: Shift ending exactly at midnight (Kavita 14:00 → 00:00 next day)

```
DATE_TRUNC('DAY', intime)  = 2024-06-01
DATE_TRUNC('DAY', outtime) = 2024-06-02  (midnight = start of next day)
→ Treated as overnight! Part 1 = 10 hrs, Part 2 = 0 hrs
→ WHERE hours_worked > 0 filters out the 0-hour Part 2 row
→ Final: Kavita, Jun 1, 10 hours ✓
```

---

### VISUAL SUMMARY:

```
Original row:    |------- intime ========== outtime -------|
                 |    Day 1     |  midnight  |    Day 2    |
                                    ↓
Part 1 row:     |--- intime ===|  (Day 1 hours)
Part 2 row:                    |=== outtime ---|  (Day 2 hours)
                                    ↓
GROUP BY date:  Day 1: SUM(all Part 1 + any Part 2 landing here)
                Day 2: SUM(all Part 1 + any Part 2 landing here)
```

---

## SOLUTION

Split each shift into per-day segments, then sum per employee per day:

```sql
WITH split_shifts AS (
    -- Part 1: Hours on the INTIME day (from intime to midnight or outtime, whichever is earlier)
    SELECT
        emp_id,
        emp_name,
        DATE_TRUNC('DAY', intime)::DATE AS work_date,
        DATEDIFF('MINUTE', intime, 
            CASE 
                WHEN DATE_TRUNC('DAY', intime) = DATE_TRUNC('DAY', outtime) THEN outtime
                ELSE DATEADD('DAY', 1, DATE_TRUNC('DAY', intime))
            END
        ) / 60.0 AS hours_worked
    FROM employee_attendance

    UNION ALL

    -- Part 2: Hours on the OUTTIME day (from midnight to outtime) — only for overnight shifts
    SELECT
        emp_id,
        emp_name,
        DATE_TRUNC('DAY', outtime)::DATE AS work_date,
        DATEDIFF('MINUTE', DATE_TRUNC('DAY', outtime), outtime) / 60.0 AS hours_worked
    FROM employee_attendance
    WHERE DATE_TRUNC('DAY', intime) != DATE_TRUNC('DAY', outtime)
)
SELECT
    emp_id,
    emp_name,
    work_date,
    ROUND(SUM(hours_worked), 2) AS total_hours
FROM split_shifts
WHERE hours_worked > 0
GROUP BY emp_id, emp_name, work_date
ORDER BY emp_id, work_date;
```

---

## ALTERNATE VERSION (Without emp_name column)

### Setup:

```sql
CREATE OR REPLACE TABLE employee_attendance (
    emp_id INT,
    intime TIMESTAMP,
    outtime TIMESTAMP
);

INSERT INTO employee_attendance (emp_id, intime, outtime) VALUES
(101, '2026-05-20 09:00:00', '2026-05-20 18:00:00'),
(101, '2026-05-20 22:00:00', '2026-05-21 02:00:00'),
(102, '2026-05-20 04:00:00', '2026-05-20 12:00:00'),
(103, '2026-05-20 08:00:00', '2026-05-21 08:00:00'),
(101, '2026-05-21 10:00:00', '2026-05-21 15:00:00');
```

### Solution:

```sql
WITH CTE AS (
    SELECT 
        EMP_ID,
        DATE_TRUNC('DAY', INTIME) AS WORK_DAY,
        DATEDIFF('HOURS', INTIME,
            CASE WHEN DATE_TRUNC('DAY', INTIME) = DATE_TRUNC('DAY', OUTTIME) THEN OUTTIME
            ELSE DATEADD('DAY', 1, DATE_TRUNC('DAY', INTIME)) END
        ) AS HOURS
    FROM employee_attendance

    UNION ALL

    SELECT 
        EMP_ID,
        DATE_TRUNC('DAY', OUTTIME) AS WORK_DAY,
        DATEDIFF('HOURS', DATE_TRUNC('DAY', OUTTIME),
            CASE WHEN DATE_TRUNC('DAY', INTIME) != DATE_TRUNC('DAY', OUTTIME) THEN OUTTIME END
        ) AS HOURS
    FROM employee_attendance
)
SELECT EMP_ID, WORK_DAY, SUM(HOURS) AS WORKING_HOURS 
FROM CTE
GROUP BY EMP_ID, WORK_DAY
ORDER BY EMP_ID, WORK_DAY;
```
