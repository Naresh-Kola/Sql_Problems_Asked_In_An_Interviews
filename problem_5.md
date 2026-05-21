# PROBLEM: Microsoft Azure Supercloud Customer

A Microsoft Azure **Supercloud customer** is defined as a customer who has purchased at least one product from **EVERY product category** listed in the products table.

Write a query that identifies the customer IDs of these Supercloud customers.

---

## Tables

### `products` Table

| Column Name | Type |
|-------------|------|
| product_id | integer |
| product_category | string |
| product_name | string |

### `customer_contracts` Table

| Column Name | Type |
|-------------|------|
| customer_id | integer |
| product_id | integer |
| amount | integer |

---

## Logic

1. Find total distinct categories in products table
2. For each customer, count how many distinct categories they purchased from
3. If a customer's distinct category count = total categories → **Supercloud**

---

## Setup: Create Tables

```sql
CREATE OR REPLACE TABLE products (
    product_id INT,
    product_category VARCHAR(50),
    product_name VARCHAR(100)
);

CREATE OR REPLACE TABLE customer_contracts (
    customer_id INT,
    product_id INT,
    amount INT
);
```

## Setup: Insert Data

### Products

```sql
INSERT INTO products (product_id, product_category, product_name) VALUES
(1, 'Analytics', 'Azure Databricks'),
(2, 'Analytics', 'Azure Stream Analytics'),
(4, 'Containers', 'Azure Kubernetes Service'),
(5, 'Containers', 'Azure Service Fabric'),
(6, 'Compute', 'Virtual Machines'),
(7, 'Compute', 'Azure Functions');
```

| product_id | product_category | product_name |
|------------|-----------------|--------------|
| 1 | Analytics | Azure Databricks |
| 2 | Analytics | Azure Stream Analytics |
| 4 | Containers | Azure Kubernetes Service |
| 5 | Containers | Azure Service Fabric |
| 6 | Compute | Virtual Machines |
| 7 | Compute | Azure Functions |

### Customer Contracts

```sql
INSERT INTO customer_contracts (customer_id, product_id, amount) VALUES
-- Customer 1: buys from Analytics(1), Containers(5), Compute(6) → ALL 3 categories ✓ SUPERCLOUD
(1, 1, 1000),
(1, 5, 2000),
(1, 6, 1500),
-- Customer 2: buys from Analytics(2) only → 1 out of 3 categories ✗
(2, 2, 3000),
-- Customer 3: buys from Analytics(1) and Containers(4) → 2 out of 3 categories ✗
(3, 1, 2500),
(3, 4, 3000),
-- Customer 4: buys from Analytics(2), Containers(5), Compute(7) → ALL 3 categories ✓ SUPERCLOUD
(4, 2, 1000),
(4, 5, 2000),
(4, 7, 3500),
-- Customer 5: buys multiple from same category → Compute(6,7) only → 1 category ✗
(5, 6, 4000),
(5, 7, 1500);
```

| customer_id | product_id | amount | Category Covered |
|-------------|-----------|--------|-----------------|
| 1 | 1 | 1000 | Analytics |
| 1 | 5 | 2000 | Containers |
| 1 | 6 | 1500 | Compute |
| 2 | 2 | 3000 | Analytics |
| 3 | 1 | 2500 | Analytics |
| 3 | 4 | 3000 | Containers |
| 4 | 2 | 1000 | Analytics |
| 4 | 5 | 2000 | Containers |
| 4 | 7 | 3500 | Compute |
| 5 | 6 | 4000 | Compute |
| 5 | 7 | 1500 | Compute |

---

## Expected Output

| customer_id |
|-------------|
| 1 |
| 4 |

### Explanation

| Customer | Categories Purchased | Count | Supercloud? |
|----------|---------------------|-------|-------------|
| 1 | Analytics + Containers + Compute | 3 of 3 | ✓ YES |
| 2 | Analytics | 1 of 3 | ✗ NO |
| 3 | Analytics + Containers | 2 of 3 | ✗ NO |
| 4 | Analytics + Containers + Compute | 3 of 3 | ✓ YES |
| 5 | Compute (2 products, same category) | 1 of 3 | ✗ NO |

---

## Solution

```sql
SELECT CUSTOMER_ID
FROM PRODUCTS T1
INNER JOIN CUSTOMER_CONTRACTS T2
ON T1.PRODUCT_ID = T2.PRODUCT_ID
GROUP BY CUSTOMER_ID
HAVING COUNT(DISTINCT PRODUCT_CATEGORY) =
    (SELECT COUNT(DISTINCT PRODUCT_CATEGORY) FROM PRODUCTS);
```

### How It Works

1. **JOIN** products and customer_contracts on `product_id` to get each customer's purchased categories
2. **GROUP BY** customer_id to aggregate per customer
3. **HAVING** filters only customers whose distinct category count equals the total number of categories in the products table
