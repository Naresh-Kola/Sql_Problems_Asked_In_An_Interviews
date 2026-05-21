# PROBLEM 10: Capital Gain/Loss Per Stock

## Problem Statement

You are given a table of stock buy/sell transactions. Each row represents either a **'Buy'** or **'Sell'** operation for a stock on a given day at a given price.

Write a query to report the **CAPITAL GAIN OR LOSS** for each stock.

**Formula:**
```
Capital Gain/Loss = Total Sell Revenue - Total Buy Cost
```
- Positive value = Profit
- Negative value = Loss

**Return:** `stock_name`, `capita_gain_loss`  
**Order:** Any order is acceptable

---

## Setup

```sql
CREATE or replace TABLE stocks (
    stock_name VARCHAR(50),
    operation VARCHAR(10),
    operation_day INT,
    price INT
);

INSERT INTO stocks VALUES
('Leetcode', 'Buy', 1, 1000),
('Corona Masks', 'Buy', 2, 10),
('Leetcode', 'Sell', 5, 9000),
('Handbags', 'Buy', 17, 30000),
('Corona Masks', 'Sell', 3, 1010),
('Corona Masks', 'Buy', 4, 1000),
('Corona Masks', 'Sell', 5, 500),
('Corona Masks', 'Buy', 6, 1000),
('Handbags', 'Sell', 29, 7000),
('Corona Masks', 'Sell', 10, 10000);
```

### Sample Data

| stock_name | operation | operation_day | price |
|------------|-----------|---------------|-------|
| Leetcode | Buy | 1 | 1000 |
| Corona Masks | Buy | 2 | 10 |
| Leetcode | Sell | 5 | 9000 |
| Handbags | Buy | 17 | 30000 |
| Corona Masks | Sell | 3 | 1010 |
| Corona Masks | Buy | 4 | 1000 |
| Corona Masks | Sell | 5 | 500 |
| Corona Masks | Buy | 6 | 1000 |
| Handbags | Sell | 29 | 7000 |
| Corona Masks | Sell | 10 | 10000 |

---

## Expected Output

| STOCK_NAME | CAPITA_GAIN_LOSS | Explanation |
|------------|-----------------|-------------|
| Leetcode | 8000 | Sold 9000, Bought 1000 → +8000 |
| Corona Masks | 9500 | Sold 1010+500+10000=11510, Bought 10+1000+1000=2010 → +9500 |
| Handbags | -23000 | Sold 7000, Bought 30000 → -23000 |

---

## SOLUTION 1: SUM with Separate CASE for Buy and Sell

```sql
SELECT
    stock_name,
    SUM(CASE WHEN operation = 'Sell' THEN price END)
    - SUM(CASE WHEN operation = 'Buy' THEN price END) AS capita_gain_loss
FROM stocks
GROUP BY stock_name;
```

### How It Works

- `SUM(CASE WHEN 'Sell' THEN price END)` → total revenue from selling
- `SUM(CASE WHEN 'Buy' THEN price END)` → total cost of buying
- Subtract buy total from sell total = net gain/loss
- `GROUP BY stock_name` to get one row per stock

### Step-by-Step for "Corona Masks"

| Row | Operation | Price | Sell SUM | Buy SUM |
|-----|-----------|-------|----------|---------|
| 1 | Buy | 10 | — | 10 |
| 2 | Sell | 1010 | 1010 | — |
| 3 | Buy | 1000 | — | 1000 |
| 4 | Sell | 500 | 500 | — |
| 5 | Buy | 1000 | — | 1000 |
| 6 | Sell | 10000 | 10000 | — |
| **Total** | | | **11510** | **2010** |

**Result:** 11510 - 2010 = **9500**

---

## SOLUTION 2: Single SUM with Sign Flip (More Concise)

```sql
SELECT
    stock_name,
    SUM(CASE WHEN operation = 'Sell' THEN price ELSE price * -1 END) AS capita_gain_loss
FROM stocks
GROUP BY stock_name;
```

### How It Works

- If operation = 'Sell' → price is **positive** (money IN)
- If operation = 'Buy' → price * -1 (money OUT, so **negate** it)
- SUM of all = net gain/loss
- No subtraction needed — the sign handles it

### Step-by-Step for "Handbags"

| Row | Operation | Price | Adjusted Value |
|-----|-----------|-------|----------------|
| 1 | Buy | 30000 | -30000 (negated) |
| 2 | Sell | 7000 | +7000 (as-is) |
| **SUM** | | | **-23000** |

---

## Comparison

| Solution | Approach | Pros |
|----------|----------|------|
| 1 | Two separate SUMs, then subtract | Easier to read, explicit intent |
| 2 | Single SUM with sign flip | More concise, one aggregation pass |

Both are equally efficient — the database optimizer handles them the same way.
