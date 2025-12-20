# Day 04 — Hands-on Labs (CTE‑Centric SQL)

## Objective

Practice breaking complex analytical logic into readable, testable CTE chains using the Olist dataset. The focus is on refactoring, correctness, and reasoning — not shortcuts.

---

## Dataset Used

All labs use the same canonical files:

```
data/raw/
├── customers.csv
├── orders.csv
├── order_items.csv
└── payments.csv
```

---

## Lab 01 — Refactor Nested Subqueries into CTEs

### Goal

Replace nested subqueries with a clear, step‑by‑step CTE chain.

---

### Starting Query (Intentionally Hard to Read)

```
SELECT
    order_id,
    SUM(payment_value) AS total_payment
FROM (
    SELECT
        p.order_id,
        p.payment_value
    FROM payments p
    WHERE p.payment_value > 0
) x
GROUP BY order_id
```

---

### Tasks

1. Rewrite the query using CTEs
2. Separate filtering and aggregation into different CTEs
3. Keep one logical step per CTE

---

### Constraints

* No nested SELECT statements
* Use meaningful CTE names
* Preserve the final output

---

### Expected Shape

* One row per order
* Columns:

  * order_id
  * total_payment

---

## Lab 02 — Multi‑Step Transformation Using CTE Chains

### Goal

Build a multi‑step transformation that calculates order‑level metrics.

---

### Task

Using CTEs:

1. Start from orders
2. Aggregate order_items to order level
3. Aggregate payments to order level
4. Join results back to orders

---

### Output Columns

* order_id
* order_status
* order_item_total
* payment_total

---

### Rules

* One transformation per CTE
* No joins inside aggregate CTEs
* Final SELECT only joins prepared CTEs

---

## Lab 03 — Filtering at the Right Stage

### Goal

Understand why filtering must happen at the correct step.

---

### Scenario

You want total payment per **delivered order only**.

---

### Task

1. Create a CTE that filters delivered orders
2. Create a separate CTE that aggregates payments
3. Join them together

---

### Question

* What happens if you filter delivered orders **after** aggregation?

---

## Lab 04 — CTEs for Reviewability

### Goal

Practice writing SQL that is easy to review in pull requests.

---

### Task

Refactor the following logic into CTEs:

* Join orders to customers
* Filter orders from 2018
* Count orders per customer

---

### Constraints

* No inline filters inside JOINs
* One business rule per CTE

---

### Expected Output

* customer_id
* customer_city
* order_count

---

## Lab 05 — Anti‑Patterns to Fix

### Starting Query

```
SELECT *
FROM orders o
JOIN (
    SELECT order_id, SUM(price) AS total
    FROM order_items
    GROUP BY order_id
) t
ON o.order_id = t.order_id
WHERE o.order_status = 'delivered'
```

---

### Tasks

1. Remove SELECT *
2. Move aggregation into a named CTE
3. Make grain explicit

---

## Completion Criteria

You should be able to:

* Explain every CTE in one sentence
* Identify where each business rule is applied
* Review someone else’s SQL without running it

Do not proceed to Day 05 until these patterns feel natural.
