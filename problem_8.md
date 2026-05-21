# LATERAL FLATTEN — Exploding JSON Arrays into Rows

## Problem

Each order has an `ITEMS` column holding a JSON array of objects. We need to turn each array element into its own row so we can query `qty` and `sku` as normal columns.

**Solution:** `LATERAL FLATTEN` explodes the array — one row per element.

---

## Step 1: Create the ORDERS Table

The `VARIANT` column stores semi-structured JSON data (arrays, objects, etc.)

```sql
CREATE OR REPLACE TABLE ORDERS (
    order_id     INT,
    customer_id  VARCHAR(10),
    items        VARIANT          -- holds a JSON array of {qty, sku} objects
);
```

---

## Step 2: Insert Data

`PARSE_JSON` converts a JSON string into a VARIANT value. Each row's items column holds an ARRAY of OBJECTS: `[{"qty":N, "sku":"X"}, ...]`

```sql
INSERT INTO ORDERS (order_id, customer_id, items)
SELECT 101, 'C001', PARSE_JSON('[{"qty": 2, "sku": "A1"}, {"qty": 1, "sku": "B9"}]')
UNION ALL
SELECT 102, 'C002', PARSE_JSON('[{"qty": 1, "sku": "A1"}, {"qty": 4, "sku": "C3"}]');
```

---

## Step 3: View Raw Data

Notice `items` is a single JSON array per row:

```sql
SELECT * FROM ORDERS;
```

| order_id | customer_id | items |
|----------|-------------|-------|
| 101 | C001 | [{"qty":2,"sku":"A1"},{"qty":1,"sku":"B9"}] |
| 102 | C002 | [{"qty":1,"sku":"A1"},{"qty":4,"sku":"C3"}] |

---

## Step 4: LATERAL FLATTEN — Explode the JSON Array

```sql
SELECT
    O.order_id,
    O.customer_id,
    F.value:qty::NUMBER AS quanty,
    F.value:sku::STRING AS sku
FROM ORDERS O,
LATERAL FLATTEN (INPUT => O.ITEMS) F;
```

---

## How It Works — Breaking Down Every Part

```
LATERAL FLATTEN(INPUT => O.ITEMS) F
│       │               │         └── F = alias for the flattened output
│       │               └── O.ITEMS = the JSON array to explode
│       └── FLATTEN = table function that produces one row per array element
└── LATERAL = means "for each row of ORDERS, run FLATTEN on that row's ITEMS"
```

### Each Keyword Explained

| Part | Meaning |
|------|---------|
| `LATERAL` | "For each row in the left table, execute the function on the right using that row's data." Without LATERAL, the function can't reference columns from ORDERS. |
| `FLATTEN` | A Snowflake table function that takes an array/object and produces one output row per element. |
| `INPUT => O.ITEMS` | The input to flatten — the JSON array column from the current ORDERS row. |
| `F` | Alias for the flattened result. Gives us access to `F.value`, `F.index`, `F.key`, etc. |
| `F.value` | The current array element (e.g., `{"qty": 2, "sku": "A1"}`) |
| `F.value:qty` | Extract the `qty` key from the JSON object (colon `:` is the JSON path operator) |
| `::NUMBER` | Cast the extracted value from VARIANT to NUMBER type |
| `::STRING` | Cast the extracted value from VARIANT to STRING type |
| `,` (comma join) | The comma between `ORDERS O` and `LATERAL FLATTEN` is an implicit CROSS JOIN. Each order row is joined with its own flattened array elements. |

---

## Before vs After FLATTEN

### Before (2 rows — 2 items packed per row):

```
Order 101 → [{"qty":2,"sku":"A1"}, {"qty":1,"sku":"B9"}]
Order 102 → [{"qty":1,"sku":"A1"}, {"qty":4,"sku":"C3"}]
```

### After (4 rows — one per array element):

| ORDER_ID | CUSTOMER_ID | QUANTY | SKU |
|----------|-------------|--------|-----|
| 101 | C001 | 2 | A1 |
| 101 | C001 | 1 | B9 |
| 102 | C002 | 1 | A1 |
| 102 | C002 | 4 | C3 |

---

## Visual Walkthrough

```
ORDERS table row:
┌──────────┬─────────────┬─────────────────────────────────────────────────┐
│ order_id │ customer_id │ items                                           │
├──────────┼─────────────┼─────────────────────────────────────────────────┤
│ 101      │ C001        │ [{"qty":2,"sku":"A1"}, {"qty":1,"sku":"B9"}]   │
└──────────┴─────────────┴─────────────────────────────────────────────────┘
                                    │
                          LATERAL FLATTEN
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
              F.value = {"qty":2,"sku":"A1"}   F.value = {"qty":1,"sku":"B9"}
                         │                     │
                         ▼                     ▼
┌──────────┬─────────────┬────────┬─────┐  ┌──────────┬─────────────┬────────┬─────┐
│ 101      │ C001        │ 2      │ A1  │  │ 101      │ C001        │ 1      │ B9  │
└──────────┴─────────────┴────────┴─────┘  └──────────┴─────────────┴────────┴─────┘
```

---

## FLATTEN Output Columns (Available via alias F)

| Column | Description |
|--------|-------------|
| `F.seq` | Sequence number (unique across all input rows) |
| `F.key` | Key name (for objects) or NULL (for arrays) |
| `F.path` | Path to this element in the original structure |
| `F.index` | Array index (0-based) of the current element |
| `F.value` | The actual value of the current element |
| `F.this` | The entire array/object being flattened |

---

## Key Concepts for Non-SQL Users

| Concept | Explanation |
|---------|-------------|
| **VARIANT** | A Snowflake data type that can hold any JSON structure (arrays, objects, strings, numbers). Think of it as a "flexible container." |
| **PARSE_JSON()** | Converts a text string like `'[{"a":1}]'` into a proper VARIANT value that Snowflake can navigate. |
| **Semi-structured data** | Data without a fixed schema — JSON, XML, Avro, Parquet. Snowflake stores it in VARIANT columns. |
| **`:` (colon operator)** | Extracts a key from a JSON object. `F.value:qty` means "get the value of the 'qty' key from the JSON object in F.value." |
| **`::TYPE` (cast)** | Converts a VARIANT value to a specific SQL type. Needed because JSON values are generic VARIANT until you cast them. |
| **LATERAL** | Allows a table function to reference columns from the preceding table in the FROM clause. Without it, FLATTEN can't see `O.ITEMS`. |
| **Table function** | A function that returns multiple rows (like a virtual table). FLATTEN is one. Others: GENERATOR, SPLIT_TO_TABLE. |
