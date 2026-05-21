# PROBLEM 12: Split Colon-Separated Values into Individual Rows

## Problem Statement

You have a table of club members where the `EDU` column contains multiple education codes separated by **colons (`:`)**.  
Some members have `NULL` in EDU (no education codes).

Write a query to split the EDU column so each education code gets its own row, while keeping the `Club_Id` and `Member_Id` associated.

**NOTE:** Rows where `EDU IS NULL` should be excluded (no codes to split).

### Example

**Input row:**
```
Club_Id=1002, Member_Id=215, EDU='CD:CI:CM'
```

**Output:** 3 separate rows:
```
(1002, 215, 'CD')
(1002, 215, 'CI')
(1002, 215, 'CM')
```

---

## Setup

```sql
CREATE OR REPLACE TABLE club_members (
    Club_Id INT,
    Member_Id INT,
    EDU VARCHAR(50)
);

INSERT INTO club_members VALUES
(1001, 210, NULL),
(1001, 211, 'MM:CI'),
(1002, 215, 'CD:CI:CM'),
(1002, 216, 'CL:CM'),
(1002, 217, 'MM:CM'),
(1003, 255, NULL),
(1001, 216, 'CO:CD:CL:MM'),
(1002, 210, NULL);
```

### Sample Data

| Club_Id | Member_Id | EDU |
|---------|-----------|-----|
| 1001 | 210 | NULL |
| 1001 | 211 | MM:CI |
| 1002 | 215 | CD:CI:CM |
| 1002 | 216 | CL:CM |
| 1002 | 217 | MM:CM |
| 1003 | 255 | NULL |
| 1001 | 216 | CO:CD:CL:MM |
| 1002 | 210 | NULL |

---

## Expected Output

| CLUB_ID | MEMBER_ID | EDU |
|---------|-----------|-----|
| 1001 | 211 | MM |
| 1001 | 211 | CI |
| 1002 | 215 | CD |
| 1002 | 215 | CI |
| 1002 | 215 | CM |
| 1002 | 216 | CL |
| 1002 | 216 | CM |
| 1002 | 217 | MM |
| 1002 | 217 | CM |
| 1001 | 216 | CO |
| 1001 | 216 | CD |
| 1001 | 216 | CL |
| 1001 | 216 | MM |

---

## Solution: LATERAL SPLIT_TO_TABLE

```sql
SELECT
    c.Club_Id,
    c.Member_Id,
    TRIM(s.VALUE) AS EDU
FROM club_members c,
    LATERAL SPLIT_TO_TABLE(c.EDU, ':') s
WHERE c.EDU IS NOT NULL;
```

---

## How It Works

### Breaking Down Each Part

| Part | Meaning |
|------|---------|
| `FROM club_members c` | The main table — one row per member |
| `,` | Implicit CROSS JOIN (connects left table to the table function) |
| `LATERAL` | "For each row of club_members, run the function using that row's data" |
| `SPLIT_TO_TABLE(c.EDU, ':')` | Split the EDU column by `:` delimiter, return one row per segment |
| `s` | Alias for the split output (gives access to `s.VALUE`, `s.INDEX`, `s.SEQ`) |
| `TRIM(s.VALUE)` | Remove any leading/trailing spaces from the split segment |
| `WHERE c.EDU IS NOT NULL` | Skip rows with no education codes (can't split NULL) |

### SPLIT_TO_TABLE Output Columns

| Column | Description |
|--------|-------------|
| `s.seq` | Unique number identifying the source row |
| `s.index` | Position of this segment in the split (1-based) |
| `s.value` | The actual string segment after splitting |

---

## Step-by-Step Walkthrough

### For Member_Id=216, EDU='CO:CD:CL:MM'

`SPLIT_TO_TABLE('CO:CD:CL:MM', ':')` produces:

| s.INDEX | s.VALUE |
|---------|---------|
| 1 | CO |
| 2 | CD |
| 3 | CL |
| 4 | MM |

Each row is joined back to `(Club_Id=1001, Member_Id=216)`:

| CLUB_ID | MEMBER_ID | EDU |
|---------|-----------|-----|
| 1001 | 216 | CO |
| 1001 | 216 | CD |
| 1001 | 216 | CL |
| 1001 | 216 | MM |

---

### For Member_Id=211, EDU='MM:CI'

`SPLIT_TO_TABLE('MM:CI', ':')` produces:

| s.INDEX | s.VALUE |
|---------|---------|
| 1 | MM |
| 2 | CI |

Joined back to `(Club_Id=1001, Member_Id=211)`:

| CLUB_ID | MEMBER_ID | EDU |
|---------|-----------|-----|
| 1001 | 211 | MM |
| 1001 | 211 | CI |

---

### For Member_Id=210, EDU=NULL

`WHERE c.EDU IS NOT NULL` → **row is filtered out** (nothing to split).

---

## Visual Summary

```
Original row:
┌─────────┬───────────┬─────────────────┐
│ 1002    │ 215       │ CD:CI:CM        │
└─────────┴───────────┴─────────────────┘
                              │
              LATERAL SPLIT_TO_TABLE(EDU, ':')
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                   'CD'      'CI'      'CM'
                    │         │         │
                    ▼         ▼         ▼
┌─────────┬───────────┬─────┐
│ 1002    │ 215       │ CD  │
│ 1002    │ 215       │ CI  │
│ 1002    │ 215       │ CM  │
└─────────┴───────────┴─────┘
```

---

## Why LATERAL is Required

Without `LATERAL`, the `SPLIT_TO_TABLE` function **cannot see** `c.EDU` — it can't reference columns from the left table. `LATERAL` creates that "for each row" connection, allowing the function to use a different value per row.
