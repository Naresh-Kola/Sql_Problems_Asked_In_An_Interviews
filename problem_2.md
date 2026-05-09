# SQL Problem 2: IPL 2026 Points Table

## Problem Statement

Given a table of IPL 2026 match results (first 15 matches), generate the **IPL Points Table** showing each team's matches played, wins, losses, and points (2 points per win), ordered by points descending.

---

## Input Table: `SQL.PROBLEMS.IPL_2026_MATCHES`

| MATCH_NO | TEAM1 | TEAM2 | WINNER | VENUE |
|----------|-------|-------|--------|-------|
| 1 | Mumbai Indians | Chennai Super Kings | Mumbai Indians | Wankhede Stadium |
| 2 | Royal Challengers Bengaluru | Kolkata Knight Riders | Kolkata Knight Riders | M Chinnaswamy Stadium |
| 3 | Delhi Capitals | Rajasthan Royals | Rajasthan Royals | Arun Jaitley Stadium |
| 4 | Sunrisers Hyderabad | Punjab Kings | Sunrisers Hyderabad | Rajiv Gandhi Stadium |
| 5 | Gujarat Titans | Lucknow Super Giants | Gujarat Titans | Narendra Modi Stadium |
| 6 | Chennai Super Kings | Royal Challengers Bengaluru | Chennai Super Kings | MA Chidambaram Stadium |
| 7 | Kolkata Knight Riders | Mumbai Indians | Mumbai Indians | Eden Gardens |
| 8 | Rajasthan Royals | Sunrisers Hyderabad | Rajasthan Royals | Sawai Mansingh Stadium |
| 9 | Punjab Kings | Delhi Capitals | Delhi Capitals | PCA Stadium |
| 10 | Lucknow Super Giants | Chennai Super Kings | Chennai Super Kings | BRSABV Ekana Stadium |
| 11 | Mumbai Indians | Gujarat Titans | Gujarat Titans | Wankhede Stadium |
| 12 | Royal Challengers Bengaluru | Rajasthan Royals | Royal Challengers Bengaluru | M Chinnaswamy Stadium |
| 13 | Kolkata Knight Riders | Sunrisers Hyderabad | Kolkata Knight Riders | Eden Gardens |
| 14 | Delhi Capitals | Lucknow Super Giants | Delhi Capitals | Arun Jaitley Stadium |
| 15 | Punjab Kings | Mumbai Indians | Mumbai Indians | PCA Stadium |

### Key Observations:
- 10 IPL teams, 15 matches played so far
- Each team plays as either TEAM1 (home) or TEAM2 (away)
- WINNER column indicates the winning team
- No ties in T20 cricket (Super Over decides the winner)
- Not all teams have played the same number of matches yet

---

## Expected Output

| TEAM_NAME | PLAYED | WON | LOST | POINTS |
|-----------|--------|-----|------|--------|
| Mumbai Indians | 4 | 3 | 1 | 6 |
| Rajasthan Royals | 3 | 2 | 1 | 4 |
| Chennai Super Kings | 3 | 2 | 1 | 4 |
| Gujarat Titans | 2 | 2 | 0 | 4 |
| Delhi Capitals | 3 | 2 | 1 | 4 |
| Kolkata Knight Riders | 3 | 2 | 1 | 4 |
| Sunrisers Hyderabad | 3 | 1 | 2 | 2 |
| Royal Challengers Bengaluru | 3 | 1 | 2 | 2 |
| Lucknow Super Giants | 3 | 0 | 3 | 0 |
| Punjab Kings | 3 | 0 | 3 | 0 |

---

## The Challenge

Same core problem as Problem 1:
- A team can appear in **TEAM1** or **TEAM2** column
- You must combine both to get total matches per team
- Then count wins/losses using the WINNER column

**Additional complexity vs Problem 1:**
- 10 teams instead of 5
- Unequal matches played (Gujarat Titans played only 2, Mumbai Indians played 4)
- Extra columns (MATCH_NO, VENUE) that are irrelevant to the points table

---

## Solution: UNION ALL Unpivot

```sql
WITH CTE AS 
(
    SELECT TEAM1, WINNER FROM SQL.PROBLEMS.IPL_2026_MATCHES
    UNION ALL 
    SELECT TEAM2, WINNER FROM SQL.PROBLEMS.IPL_2026_MATCHES
)
SELECT
    TEAM1 AS TEAM_NAME,
    COUNT(TEAM1) AS PLAYED,
    SUM(CASE WHEN TEAM1 = WINNER THEN 1 ELSE 0 END) AS WON,
    SUM(CASE WHEN TEAM1 != WINNER THEN 1 ELSE 0 END) AS LOST,
    SUM(CASE WHEN TEAM1 = WINNER THEN 2 ELSE 0 END) AS POINTS
FROM CTE 
GROUP BY TEAM1
ORDER BY POINTS DESC;
```

> **Note:** Remove the trailing comma after POINTS in the SELECT (syntax error in original).

### How it works (step by step):

**Step 1: CTE — Unpivot teams into single column**

```sql
SELECT TEAM1, WINNER FROM SQL.PROBLEMS.IPL_2026_MATCHES
UNION ALL 
SELECT TEAM2, WINNER FROM SQL.PROBLEMS.IPL_2026_MATCHES
```

- First SELECT: Takes all TEAM1 values + who won (15 rows)
- Second SELECT: Takes all TEAM2 values + who won (15 rows)
- Result: 30 rows total (15 matches × 2 teams each)

**Example of CTE output:**

| TEAM1 | WINNER |
|-------|--------|
| Mumbai Indians | Mumbai Indians |
| Royal Challengers Bengaluru | Kolkata Knight Riders |
| Delhi Capitals | Rajasthan Royals |
| ... | ... |
| Chennai Super Kings | Mumbai Indians |
| Kolkata Knight Riders | Kolkata Knight Riders |
| Rajasthan Royals | Rajasthan Royals |
| ... | ... |

**Step 2: COUNT — Matches played**

```sql
COUNT(TEAM1) AS PLAYED
```

- Groups by team name, counts how many rows each team has
- Mumbai Indians appears 4 times in CTE → PLAYED = 4
- Gujarat Titans appears 2 times → PLAYED = 2

**Step 3: CASE WHEN — Wins and Losses**

```sql
SUM(CASE WHEN TEAM1 = WINNER THEN 1 ELSE 0 END) AS WON
SUM(CASE WHEN TEAM1 != WINNER THEN 1 ELSE 0 END) AS LOST
```

For each row in the CTE:
- If the team name equals WINNER → that's a win (+1)
- If the team name does NOT equal WINNER → that's a loss (+1)

**Example trace for Mumbai Indians (4 rows in CTE):**

| Row | TEAM1 | WINNER | WON? | LOST? |
|-----|-------|--------|------|-------|
| Match 1 | Mumbai Indians | Mumbai Indians | ✓ (1) | ✗ (0) |
| Match 7 | Mumbai Indians | Mumbai Indians | ✓ (1) | ✗ (0) |
| Match 11 | Mumbai Indians | Gujarat Titans | ✗ (0) | ✓ (1) |
| Match 15 | Mumbai Indians | Mumbai Indians | ✓ (1) | ✗ (0) |
| **Total** | | | **3** | **1** |

**Step 4: Points calculation**

```sql
SUM(CASE WHEN TEAM1 = WINNER THEN 2 ELSE 0 END) AS POINTS
```

- 2 points per win
- Mumbai Indians: 3 wins × 2 = 6 points

---

## Difference from Problem 1

| Aspect | Problem 1 | Problem 2 |
|--------|-----------|-----------|
| Sport | Cricket World Cup | IPL T20 |
| Teams | 5 | 10 |
| Matches | 12 | 15 |
| Table columns | 3 (TEAM1, TEAM2, WINNER) | 5 (MATCH_NO, TEAM1, TEAM2, WINNER, VENUE) |
| Matches per team | Equal (4-6 each) | Unequal (2-4 each) |
| Solution approach | Same | Same |

The **core logic is identical** — unpivot TEAM1/TEAM2, then aggregate with CASE WHEN.

---

## Bug in Original Query

The original SQL has a **trailing comma** after POINTS:

```sql
SUM(CASE WHEN TEAM1 = WINNER THEN 2 ELSE 0 END) AS POINTS,  ← REMOVE THIS COMMA
FROM CTE
```

This causes a syntax error. Remove the comma before `FROM`.

---

## Concept Summary

| Concept | Explanation |
|---------|-------------|
| **UNION ALL** | Combines TEAM1 and TEAM2 into one column (keeps all rows) |
| **CTE** | Common Table Expression — temporary named result set |
| **CASE WHEN** | Conditional logic: if team = winner → win, else → loss |
| **GROUP BY** | Aggregates all rows per team into one summary row |
| **COUNT** | Total rows per team = total matches played |
| **SUM with CASE** | Counts specific conditions (wins, losses) |
| **ORDER BY DESC** | Highest points first (like a real points table) |
