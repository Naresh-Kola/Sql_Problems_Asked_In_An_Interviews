# SQL Problem 3: Pizza Topping Combinations

## Problem Statement

Given a table of pizza toppings with their costs, generate **all possible 3-topping pizza combinations** with their total cost, sorted by cost (highest first).

---

## Input Table: `PIZZA_TOPPINGS`

| TOPPING_NAME | INGREDIENT_COST |
|--------------|-----------------|
| Pepperoni | 0.50 |
| Sausage | 0.70 |
| Chicken | 0.55 |
| Extra Cheese | 0.40 |

---

## Expected Output

| PIZZA | TOTAL_COST |
|-------|------------|
| Chicken,Pepperoni,Sausage | 1.75 |
| Chicken,Extra Cheese,Sausage | 1.65 |
| Extra Cheese,Pepperoni,Sausage | 1.60 |
| Chicken,Extra Cheese,Pepperoni | 1.45 |

---

## The Challenge

You need to:
1. Pick **3 toppings** from 4 available (combinations, not permutations)
2. **No duplicates** — "Chicken,Pepperoni,Sausage" and "Sausage,Chicken,Pepperoni" are the SAME pizza
3. **No repeated toppings** — can't use Pepperoni twice
4. Calculate total cost by summing the 3 ingredient costs
5. Display toppings in **alphabetical order** within each combination

### Math behind it:
- 4 toppings, choose 3 = C(4,3) = 4 combinations
- If we had 10 toppings choosing 3 = C(10,3) = 120 combinations

### The Key Trick: `T1.TOPPING_NAME < T2.TOPPING_NAME`
This comparison ensures:
- **No duplicates** — forces alphabetical ordering (Chicken < Pepperoni < Sausage)
- **No self-joins** — a topping can't pair with itself
- **No reverse pairs** — "A,B,C" exists but "C,B,A" does not

---

## Solution 1: Self Join with `<` Operator

```sql
SELECT 
    CONCAT(T1.TOPPING_NAME, ',', T2.TOPPING_NAME, ',', T3.TOPPING_NAME) AS PIZZA,
    (T1.INGREDIENT_COST + T2.INGREDIENT_COST + T3.INGREDIENT_COST) AS TOTAL_COST
FROM PIZZA_TOPPINGS T1
INNER JOIN PIZZA_TOPPINGS T2
    ON T1.TOPPING_NAME < T2.TOPPING_NAME
INNER JOIN PIZZA_TOPPINGS T3
    ON T2.TOPPING_NAME < T3.TOPPING_NAME
ORDER BY TOTAL_COST DESC, PIZZA;
```

### How it works (step by step):

**Step 1: First self-join (T1 JOIN T2)**
- Joins the table to itself
- `T1.TOPPING_NAME < T2.TOPPING_NAME` ensures T1 comes before T2 alphabetically
- This gives all **2-topping pairs** without duplicates:
  - Chicken < Extra Cheese ✗ (C > E is false... wait, let's check)
  - Actually in alphabetical: C < E < P < S
  - Chicken < Extra Cheese ✓
  - Chicken < Pepperoni ✓
  - Chicken < Sausage ✓
  - Extra Cheese < Pepperoni ✓
  - Extra Cheese < Sausage ✓
  - Pepperoni < Sausage ✓
  - = 6 pairs (C(4,2) = 6) ✓

**Step 2: Second self-join (T2 JOIN T3)**
- `T2.TOPPING_NAME < T3.TOPPING_NAME` ensures T3 comes after T2
- Combined with step 1: T1 < T2 < T3 (strict alphabetical ordering)
- This guarantees exactly one representation of each 3-topping combo

**Step 3: CONCAT**
- Combines the 3 topping names with commas
- Since T1 < T2 < T3, they're already alphabetically sorted

**Step 4: Sum costs**
- Adds all 3 ingredient costs

**Example trace for one combination:**
```
T1 = Chicken (0.55)
T2 = Pepperoni (0.50)    → Chicken < Pepperoni ✓
T3 = Sausage (0.70)      → Pepperoni < Sausage ✓

PIZZA = "Chicken,Pepperoni,Sausage"
TOTAL_COST = 0.55 + 0.50 + 0.70 = 1.75
```

---

## Solution 2: CROSS JOIN with WHERE Filter

```sql
SELECT 
    CONCAT(T1.TOPPING_NAME, ',', T2.TOPPING_NAME, ',', T3.TOPPING_NAME) AS PIZZA,
    (T1.INGREDIENT_COST + T2.INGREDIENT_COST + T3.INGREDIENT_COST) AS TOTAL_COST
FROM PIZZA_TOPPINGS T1
CROSS JOIN PIZZA_TOPPINGS T2, PIZZA_TOPPINGS T3
WHERE T1.TOPPING_NAME < T2.TOPPING_NAME
AND T2.TOPPING_NAME < T3.TOPPING_NAME
ORDER BY TOTAL_COST DESC, PIZZA;
```

### How it works:

1. **CROSS JOIN** — Creates ALL possible combinations (4 × 4 × 4 = 64 rows)
2. **WHERE T1 < T2 AND T2 < T3** — Filters down to only valid combinations (4 rows)

### Difference from Solution 1:
- Solution 1: Filter in JOIN condition (ON clause) — filters DURING join
- Solution 2: Filter in WHERE clause — joins everything first, then filters

**Performance:** Solution 1 is slightly better for large tables (filters earlier), but for small tables both are identical.

---

## Why `<` Works for Combinations

Consider what happens WITHOUT the `<` condition:

```
-- Without any filter (CROSS JOIN gives 64 rows for 4 toppings):
Pepperoni, Pepperoni, Pepperoni  ← self-repeat (invalid)
Pepperoni, Sausage, Chicken      ← same combo as below
Chicken, Pepperoni, Sausage      ← same combo as above (duplicate)
Sausage, Chicken, Pepperoni      ← same combo again (duplicate)
```

With `T1 < T2 < T3`:
```
-- Only ONE ordering survives:
Chicken, Pepperoni, Sausage      ← ✓ (C < P < S)
```

All permutations of the same set are eliminated because only the alphabetically sorted one satisfies `<`.

---

## Concept Summary

| Concept | Explanation |
|---------|-------------|
| **Self Join** | Joining a table to itself (same table used multiple times with aliases T1, T2, T3) |
| **Combinations vs Permutations** | Combinations = order doesn't matter (ABC = CBA). Permutations = order matters |
| **The `<` trick** | Forces one specific ordering → eliminates duplicates without DISTINCT |
| **CROSS JOIN** | Every row × every row (Cartesian product) |
| **INNER JOIN with `<`** | Same result as CROSS JOIN + WHERE, but filters during join |
| **C(n,k) formula** | Number of combinations = n! / (k! × (n-k)!) |
| **CONCAT** | Combines strings together with a separator |

---

## Variations

### What if you want 2-topping pizzas?

```sql
SELECT 
    CONCAT(T1.TOPPING_NAME, ',', T2.TOPPING_NAME) AS PIZZA,
    (T1.INGREDIENT_COST + T2.INGREDIENT_COST) AS TOTAL_COST
FROM PIZZA_TOPPINGS T1
INNER JOIN PIZZA_TOPPINGS T2
    ON T1.TOPPING_NAME < T2.TOPPING_NAME
ORDER BY TOTAL_COST DESC;
```
Result: C(4,2) = 6 combinations

### What if you want ALL size combinations (2, 3, or 4 toppings)?

```sql
-- 4-topping pizza (only 1 combination with 4 toppings):
SELECT 
    CONCAT(T1.TOPPING_NAME, ',', T2.TOPPING_NAME, ',', T3.TOPPING_NAME, ',', T4.TOPPING_NAME) AS PIZZA,
    (T1.INGREDIENT_COST + T2.INGREDIENT_COST + T3.INGREDIENT_COST + T4.INGREDIENT_COST) AS TOTAL_COST
FROM PIZZA_TOPPINGS T1
INNER JOIN PIZZA_TOPPINGS T2 ON T1.TOPPING_NAME < T2.TOPPING_NAME
INNER JOIN PIZZA_TOPPINGS T3 ON T2.TOPPING_NAME < T3.TOPPING_NAME
INNER JOIN PIZZA_TOPPINGS T4 ON T3.TOPPING_NAME < T4.TOPPING_NAME
ORDER BY TOTAL_COST DESC;
```
