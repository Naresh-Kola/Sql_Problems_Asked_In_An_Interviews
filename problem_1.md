# SQL Problem 1: Cricket Match Points Table

## Problem Statement

Given a table of cricket matches between teams, generate a **points table** showing each team's total matches played, wins, losses, and points (2 points per win).

---

## Input Table: `SQL.PROBLEMS.MATCHES`

| TEAM1 | TEAM2 | WINNER |
|-------|-------|--------|
| India | Australia | India |
| India | England | England |
| Australia | England | Australia |
| India | South Africa | India |
| South Africa | Australia | South Africa |
| England | South Africa | England |
| India | New Zealand | India |
| Australia | New Zealand | Australia |
| England | New Zealand | New Zealand |
| South Africa | New Zealand | South Africa |
| India | Australia | Australia |
| England | India | India |

### Key Observations:
- Each row represents one match between TEAM1 and TEAM2
- A team can appear in **either** TEAM1 or TEAM2 column
- The WINNER column tells who won that match
- There are no draws/ties — every match has a winner

---

## Expected Output

| TEAM_NAME | PLAYED | WON | LOST | POINTS |
|-----------|--------|-----|------|--------|
| India | 6 | 4 | 2 | 8 |
| Australia | 5 | 3 | 2 | 6 |
| England | 5 | 2 | 3 | 4 |
| South Africa | 4 | 2 | 2 | 4 |
| New Zealand | 4 | 1 | 3 | 2 |

---

## The Challenge

The main difficulty is: **a team's matches are split across two columns (TEAM1 and TEAM2).**

For example, India appears as:
- TEAM1 in: India vs Australia, India vs England, India vs South Africa, India vs New Zealand
- TEAM2 in: England vs India

So India played 6 total matches (4 as TEAM1 + 2 as TEAM2... wait, let's count):
- Row 1: India vs Australia (TEAM1) ✓
- Row 2: India vs England (TEAM1) ✓
- Row 4: India vs South Africa (TEAM1) ✓
- Row 7: India vs New Zealand (TEAM1) ✓
- Row 11: India vs Australia (TEAM1) ✓
- Row 12: England vs India (TEAM2) ✓

**Total: 6 matches played by India**

You need to "unpivot" or "flatten" TEAM1 and TEAM2 into a single column to count all matches per team.

---

## Solution 1: CTE Approach (Multiple CTEs)

```sql
WITH CTE AS 
(
    SELECT WINNER, COUNT(WINNER) AS WON, COUNT(WINNER) * 2 AS POINTS
    FROM SQL.PROBLEMS.MATCHES
    GROUP BY WINNER
)
,CTE1 AS
(
    SELECT TEAM1 FROM SQL.PROBLEMS.MATCHES
    UNION ALL
    SELECT TEAM2 FROM SQL.PROBLEMS.MATCHES
)
,CTE2 AS (
    SELECT TEAM1, COUNT(*) AS PLAYED FROM CTE1
    GROUP BY TEAM1
)
SELECT 
    CTE2.TEAM1 AS TEAM_NAME,
    CTE2.PLAYED,
    CTE.WON,
    (CTE2.PLAYED - CTE.WON) AS LOST,
    CTE.POINTS
FROM CTE 
INNER JOIN CTE2
ON CTE.WINNER = CTE2.TEAM1
ORDER BY POINTS DESC;
```

### How it works (step by step):

**CTE (wins calculation):**
- Groups by WINNER column → counts how many times each team won
- India won 4 times → WON=4, POINTS=8

**CTE1 (all team appearances):**
- UNION ALL combines TEAM1 and TEAM2 into one column
- This gives every team appearance (24 rows for 12 matches)

**CTE2 (matches played):**
- Counts how many times each team appears in CTE1
- India appears 6 times → PLAYED=6

**Final SELECT:**
- Joins wins (CTE) with played count (CTE2)
- Calculates LOST = PLAYED - WON

---

## Solution 2: Self Join

```sql
SELECT 
    t.TEAM_NAME,
    COUNT(*) AS PLAYED,
    SUM(CASE WHEN m.WINNER = t.TEAM_NAME THEN 1 ELSE 0 END) AS WON,
    SUM(CASE WHEN m.WINNER != t.TEAM_NAME THEN 1 ELSE 0 END) AS LOST,
    SUM(CASE WHEN m.WINNER = t.TEAM_NAME THEN 2 ELSE 0 END) AS POINTS
FROM SQL.PROBLEMS.MATCHES m
JOIN (
    SELECT TEAM1 AS TEAM_NAME FROM SQL.PROBLEMS.MATCHES
    UNION
    SELECT TEAM2 AS TEAM_NAME FROM SQL.PROBLEMS.MATCHES
) t
    ON t.TEAM_NAME = m.TEAM1 OR t.TEAM_NAME = m.TEAM2
GROUP BY t.TEAM_NAME
ORDER BY POINTS DESC;
```

### How it works:

1. **Inner subquery** — Gets distinct list of all teams using UNION (not UNION ALL) on TEAM1 and TEAM2
2. **JOIN condition** — Joins each team back to matches where it appears as either TEAM1 OR TEAM2
3. **CASE expressions** — For each match row:
   - If WINNER = team name → count as a win (+1 WON, +2 POINTS)
   - If WINNER != team name → count as a loss (+1 LOST)
4. **COUNT(*)** — Total joined rows per team = total matches played

---

## Solution 3: UNION ALL Unpivot (Simplest)

```sql
SELECT 
    TEAM_NAME,
    COUNT(*) AS PLAYED,
    SUM(CASE WHEN TEAM_NAME = WINNER THEN 1 ELSE 0 END) AS WON,
    SUM(CASE WHEN TEAM_NAME != WINNER THEN 1 ELSE 0 END) AS LOST,
    SUM(CASE WHEN TEAM_NAME = WINNER THEN 2 ELSE 0 END) AS POINTS
FROM (
    SELECT TEAM1 AS TEAM_NAME, WINNER FROM SQL.PROBLEMS.MATCHES
    UNION ALL
    SELECT TEAM2 AS TEAM_NAME, WINNER FROM SQL.PROBLEMS.MATCHES
)
GROUP BY TEAM_NAME
ORDER BY POINTS DESC;
```

### How it works:

1. **UNION ALL subquery** — Creates one row per team per match:
   - First SELECT: takes TEAM1 as team name + who won
   - Second SELECT: takes TEAM2 as team name + who won
   - Result: 24 rows (12 matches × 2 teams each)

2. **Outer query** — For each team:
   - COUNT(*) = total matches played
   - CASE WHEN TEAM_NAME = WINNER = that team won
   - CASE WHEN TEAM_NAME != WINNER = that team lost

**Why this is simplest:** Single pass, no joins needed, easy to read.

---

## Solution 4: LATERAL FLATTEN (Snowflake-Specific)

```sql
SELECT 
    f.VALUE::STRING AS TEAM_NAME,
    COUNT(*) AS PLAYED,
    SUM(CASE WHEN f.VALUE::STRING = m.WINNER THEN 1 ELSE 0 END) AS WON,
    SUM(CASE WHEN f.VALUE::STRING != m.WINNER THEN 1 ELSE 0 END) AS LOST,
    SUM(CASE WHEN f.VALUE::STRING = m.WINNER THEN 2 ELSE 0 END) AS POINTS
FROM SQL.PROBLEMS.MATCHES m,
LATERAL FLATTEN(INPUT => ARRAY_CONSTRUCT(m.TEAM1, m.TEAM2)) f
GROUP BY TEAM_NAME
ORDER BY POINTS DESC;
```

### How it works:

1. **ARRAY_CONSTRUCT(m.TEAM1, m.TEAM2)** — Puts both teams into an array: `['India', 'Australia']`
2. **LATERAL FLATTEN** — Explodes the array into 2 rows (one per team)
3. **f.VALUE::STRING** — Extracts each team name from the flattened array
4. Same CASE logic as Solution 3

**Advantage:** No UNION ALL needed, Snowflake handles the unpivot natively.

---

## Concept Summary

| Concept | Explanation |
|---------|-------------|
| **The core problem** | A team appears in TEAM1 or TEAM2 — you must combine both |
| **Unpivot** | Converting columns (TEAM1, TEAM2) into rows |
| **UNION ALL** | Stacks two SELECTs vertically (keeps duplicates) |
| **UNION** | Stacks two SELECTs vertically (removes duplicates) |
| **LATERAL FLATTEN** | Snowflake function to explode arrays into rows |
| **CASE WHEN** | Conditional logic to count wins/losses per row |
| **Self Join** | Joining a table (or derived table) back to itself |

---

## SQL Problem 2: IPL 2026 Points Table

Same logic applied to IPL 2026 data with 10 teams and 15 matches. The table has additional columns (MATCH_NO, VENUE) but the solution approach is identical — unpivot TEAM1/TEAM2 and aggregate.
