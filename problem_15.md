# Problem 23: Count Delayed Orders Per Delivery Partner

## Question

Given an `order_details` table with order time, delivery time, and predicted delivery duration (in minutes), find the number of delayed orders per delivery partner. An order is considered **delayed** if the actual delivery time exceeds the predicted delivery time.

## Table Structure

```sql
CREATE TABLE order_details (
    orderid INT PRIMARY KEY,
    custid INT,
    city VARCHAR(50),
    order_date DATE,
    del_partner VARCHAR(50),
    order_time TIME,
    deliver_time TIME,
    predicted_time INT,  -- in minutes
    aov DECIMAL(10, 2)
);
```

## Sample Data

| orderid | custid | city      | order_date | del_partner | order_time | deliver_time | predicted_time | aov    |
|---------|--------|-----------|------------|-------------|------------|--------------|----------------|--------|
| 1       | 101    | Bangalore | 2024-01-01 | PartnerA    | 10:00:00   | 11:30:00     | 60             | 100.00 |
| 2       | 102    | Chennai   | 2024-01-02 | PartnerB    | 12:00:00   | 13:15:00     | 45             | 200.00 |
| 3       | 103    | Bangalore | 2024-01-03 | PartnerA    | 14:00:00   | 15:45:00     | 60             | 300.00 |
| 4       | 104    | Chennai   | 2024-01-04 | PartnerB    | 16:00:00   | 17:30:00     | 90             | 400.00 |

## Solution

```sql
SELECT DEL_PARTNER, COUNT(*) AS DELAYED_ORDERS
FROM ORDER_DETAILS
WHERE DELIVER_TIME > DATEADD(MINUTE, PREDICTED_TIME, ORDER_TIME)
GROUP BY DEL_PARTNER;
```

## Explanation

### The Error (Original Query)

```sql
SELECT DEL_PARTNER, COUNT(*) AS DELAYED_ORDERS
FROM ORDER_DETAILS
GROUP BY DEL_PARTNER
HAVING DATEADD(HOUR,(PREDICTED_TIME / 60), ORDER_TIME) <= DELIVER_TIME;
```

This produced: `SQL compilation error: [ORDER_DETAILS.PREDICTED_TIME] is not a valid group by expression`

### Why It Failed

The `HAVING` clause is evaluated **after** `GROUP BY`. At that point, individual row values like `PREDICTED_TIME` and `ORDER_TIME` no longer exist — only grouped/aggregated values are accessible. Since these columns aren't in the `GROUP BY` and aren't wrapped in aggregate functions, Snowflake rejects them.

### The Fix

1. **Use `WHERE` instead of `HAVING`** — The filtering condition checks individual rows (is this specific order late?), so it must go in `WHERE` which runs *before* grouping.

2. **Use `MINUTE` instead of `HOUR`** — `PREDICTED_TIME` is stored in minutes, so `DATEADD(MINUTE, PREDICTED_TIME, ORDER_TIME)` gives the expected delivery time.

### Key Concept: WHERE vs HAVING

| Clause   | Runs          | Can Reference              |
|----------|---------------|----------------------------|
| `WHERE`  | Before GROUP BY | Individual row columns     |
| `HAVING` | After GROUP BY  | GROUP BY columns & aggregates |

### How the Logic Works

For Order 1:
- `ORDER_TIME` = 10:00, `PREDICTED_TIME` = 60 min
- Expected delivery = `DATEADD(MINUTE, 60, '10:00:00')` = **11:00:00**
- Actual `DELIVER_TIME` = **11:30:00**
- 11:30 > 11:00 → **Delayed!**
