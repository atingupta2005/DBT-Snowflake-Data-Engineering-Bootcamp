# Day 03 — Hands-on Labs (SQL Discipline)

## Objective

Apply clean analytical SQL and correct join patterns using the Olist dataset. These labs focus on correctness, readability, and grain safety before introducing dbt.

---

## Dataset Used

All labs use CSVs from:

```
data/raw/
├── customers.csv
├── orders.csv
├── order_items.csv
└── payments.csv
```

Load these CSVs into your SQL environment of choice (Snowflake recommended later, any SQL engine is fine for now).

See [load-csv.md](load-csv.md) for instructions on loading the files.

---

## Lab 01 — Clean Analytical SQL

### Goal

Rewrite poorly written SQL into clean, reviewable analytical SQL.

---

### Starting Query (Intentionally Bad)

```
SELECT * FROM orders o, customers c WHERE o.customer_id=c.customer_id
```

Problems to identify:

* SELECT * usage
* Implicit join
* No clear intent
* Hard to review

---

### Tasks

1. Rewrite the query using:

   * Explicit column selection
   * Explicit JOIN syntax
   * Clear formatting
2. Return only:

   * order_id
   * order_purchase_timestamp
   * customer_unique_id
   * customer_city

---

### Constraints

* Do not use SELECT *
* Use table names or meaningful aliases
* One column per line

---

### Expected Shape

* One row per order
* No duplicated orders

---

## Lab 02 — Join Grain and Duplication

### Goal

Understand how one-to-many joins can silently break metrics.

---

### Step 1 — Naive Join

Run the following:

```
SELECT
    o.order_id,
    o.order_status,
    i.price
FROM orders o
JOIN order_items i
  ON o.order_id = i.order_id
```

Observe:

* Number of rows returned
* Repeated order_id values

---

### Step 2 — Identify the Problem

Answer:

* What is the grain of `orders`?
* What is the grain of `order_items`?
* Why does this join multiply rows?

---

### Step 3 — Correct the Join

Rewrite the logic so the result has:

* One row per order
* Total item value per order

Hints:

* Aggregate order_items first
* Join aggregated results to orders

---

### Expected Shape

Columns:

* order_id
* order_status
* order_total_amount

---

## Lab 03 — LEFT JOIN vs INNER JOIN

### Goal

Understand how join type affects row preservation.

---

### Task

1. Join orders with payments using INNER JOIN
2. Count number of orders returned
3. Repeat using LEFT JOIN

```
SELECT COUNT(*) FROM ...
```

---

### Questions to Answer

* Why do counts differ?
* Which join is correct for revenue reporting?

---

## Lab 04 — Join Filters Placement

### Goal

Understand why filter placement matters.

---

### Starting Query

```
SELECT
    o.order_id,
    p.payment_type
FROM orders o
LEFT JOIN payments p
  ON o.order_id = p.order_id
WHERE p.payment_type = 'credit_card'
```

---

### Task

1. Run the query and observe row count
2. Move the filter into the JOIN condition
3. Compare results

---

### Question

* Why does the WHERE clause change LEFT JOIN behavior?

---

## Completion Criteria

You should be able to explain:

* Why SELECT * is dangerous
* How joins change grain
* Why LEFT JOIN filters belong in ON

Do not proceed to Day 04 until these concepts are clear.
