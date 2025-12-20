# Day 04 — CTE-Centric SQL for dbt

## Objective

Establish a CTE-first approach to writing analytical SQL so transformations are modular, reviewable, and easy to extend in dbt models.

---

## Why CTEs Matter in Analytics SQL

As transformations grow, single-query SQL becomes hard to understand and risky to change.

Common problems without CTEs:

* Deeply nested subqueries
* Logic mixed with joins and filters
* Hard to review changes in pull requests

CTEs allow you to break a transformation into clear, named steps.

---

## CTEs as Transformation Steps

A CTE should represent **one logical step** in the transformation.

Example mental model:

```
Raw Data
   ↓
Standardize Columns
   ↓
Apply Business Rules
   ↓
Final Output
```

Each step becomes a CTE with a meaningful name.

---

## Example: From Nested SQL to CTEs

Hard-to-review pattern:

```
SELECT
    o.order_id,
    o.order_date,
    c.customer_name
FROM (
    SELECT * FROM orders WHERE status = 'COMPLETED'
) o
JOIN (
    SELECT * FROM customers WHERE is_active = TRUE
) c
  ON o.customer_id = c.customer_id
```

CTE-centric pattern:

```
WITH completed_orders AS (
    SELECT *
    FROM orders
    WHERE status = 'COMPLETED'
),
active_customers AS (
    SELECT *
    FROM customers
    WHERE is_active = TRUE
)
SELECT
    completed_orders.order_id,
    completed_orders.order_date,
    active_customers.customer_name
FROM completed_orders
JOIN active_customers
  ON completed_orders.customer_id = active_customers.customer_id
```

The intent of each step is visible immediately.

---

## Why dbt Encourages CTE-Based SQL

dbt models are reviewed, versioned, and reused.

CTE-centric SQL helps because:

* Each step can be validated independently
* Changes are localized to one section
* Reviewers can reason about intent, not syntax

This style scales across teams.

---

## When to Use Subqueries

Subqueries are not forbidden, but they should be limited.

Acceptable use cases:

* Simple existence checks
* Very small, obvious lookups

Avoid subqueries when:

* Logic spans multiple steps
* The same logic may be reused

If a subquery needs a comment to explain it, it should be a CTE.

---

## How This Day Fits in the Course

CTE-centric SQL is used in:

* Staging models
* Intermediate models
* Fact and dimension models

All dbt models later in the course follow this structure.

---

## How to Use This Day

Adopt the rule:

* If logic has a name, it deserves a CTE

This keeps models readable even as complexity grows.
