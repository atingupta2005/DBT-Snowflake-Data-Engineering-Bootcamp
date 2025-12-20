# 01 — CTE Fundamentals

## Objective

Explain how Common Table Expressions (CTEs) work in analytical SQL and how to use them correctly as building blocks for dbt models.

---

## What a CTE Is

A CTE is a named, temporary result set defined at the start of a query.

Basic form:

```
WITH some_name AS (
    SELECT ...
)
SELECT ...
FROM some_name
```

The CTE exists only for the duration of the query. It does not create a table or view.

---

## Why CTEs Improve Analytical SQL

CTEs make transformations explicit.

Compare the two approaches:

Nested logic:

```
SELECT
    order_id
FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE country = 'IN'
)
```

CTE-based logic:

```
WITH indian_customers AS (
    SELECT customer_id
    FROM customers
    WHERE country = 'IN'
)
SELECT
    orders.order_id
FROM orders
JOIN indian_customers
  ON orders.customer_id = indian_customers.customer_id
```

The second version shows intent clearly and is easier to review.

---

## One Logical Step per CTE

Each CTE should do exactly one thing.

Good pattern:

* One CTE to filter
* One CTE to standardize
* One CTE to aggregate

Bad pattern:

* Filtering, joining, and aggregating in one CTE

Example:

```
WITH filtered_orders AS (
    SELECT *
    FROM orders
    WHERE order_status = 'COMPLETED'
),
aggregated_orders AS (
    SELECT
        customer_id,
        COUNT(*) AS order_count
    FROM filtered_orders
    GROUP BY customer_id
)
SELECT *
FROM aggregated_orders
```

---

## Ordering of CTEs

CTEs should be ordered top-to-bottom following the transformation flow.

```
Raw → Clean → Enrich → Aggregate → Final
```

Avoid jumping back and forth between levels of logic.

Bad ordering:

* Aggregate first
* Filter later

Correct ordering makes reviews predictable.

---

## Naming CTEs

CTE names should describe the result, not the operation.

Bad names:

* tmp
* step1
* data

Good names:

* filtered_orders
* active_customers
* daily_revenue

A reader should understand the query flow by reading CTE names alone.

---

## Reusing CTEs

CTEs can be referenced multiple times in the same query.

Example:

```
WITH completed_orders AS (
    SELECT *
    FROM orders
    WHERE order_status = 'COMPLETED'
)
SELECT COUNT(*) FROM completed_orders
UNION ALL
SELECT SUM(amount) FROM completed_orders
```

This avoids duplicating logic.

---

## CTEs and Performance

CTEs are a readability tool, not a performance tool.

In Snowflake:

* The optimizer decides execution strategy
* Multiple references to a CTE do not necessarily re-scan data

Do not avoid CTEs for fear of performance.

---

## Common CTE Mistakes

Example 1: Overloading a CTE

* Too much logic in one CTE
* Hard to change safely

Example 2: Using CTEs for constants

* Hard-coded values hidden in CTEs

Example 3: Deep CTE chains without purpose

* Many steps that do trivial changes

Each CTE should justify its existence.

---

## How This Applies to dbt

In dbt:

* Each model is already a named transformation
* CTEs define steps *within* the model

Staging models usually have:

* Few, simple CTEs

Intermediate models:

* Longer CTE chains

Fact models:

* Final aggregation CTE followed by a simple SELECT
