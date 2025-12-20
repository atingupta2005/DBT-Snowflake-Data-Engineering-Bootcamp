# 03 — Subqueries and Best Practices

## Objective

Clarify when subqueries are acceptable in analytical SQL, when they introduce risk, and how to refactor them into clearer CTE-based patterns for dbt models.

---

## Subqueries Are Not the Enemy

Subqueries are a valid SQL feature. Problems arise when they are overused or hide intent.

Good usage:

* Simple existence checks
* Small, obvious lookups

Risky usage:

* Multi-step transformations
* Complex joins inside subqueries

The rule is about **clarity**, not prohibition.

---

## Acceptable Pattern: EXISTS Checks

EXISTS subqueries are often clearer than joins.

Example:

```
SELECT
    order_id
FROM orders
WHERE EXISTS (
    SELECT 1
    FROM payments
    WHERE payments.order_id = orders.order_id
      AND payments.status = 'SUCCESS'
)
```

Why this works:

* Intent is clear
* No risk of row multiplication
* No additional columns introduced

---

## Acceptable Pattern: Scalar Lookups

Small scalar subqueries can be acceptable.

Example:

```
SELECT
    order_id,
    order_amount,
    (
        SELECT tax_rate
        FROM tax_config
        WHERE country = orders.country
    ) AS tax_rate
FROM orders
```

Use this only when:

* The lookup table is guaranteed to return one row
* The logic is obvious

Otherwise, use a join.

---

## Risky Pattern: Transformations Inside Subqueries

Example:

```
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE created_at >= '2024-01-01'
      AND status = 'ACTIVE'
      AND country = 'IN'
)
```

Problems:

* Business logic is hidden
* Hard to test independently
* Hard to extend later

This logic deserves its own CTE.

---

## Refactoring Subqueries into CTEs

Refactored version:

```
WITH active_indian_customers AS (
    SELECT customer_id
    FROM customers
    WHERE created_at >= '2024-01-01'
      AND status = 'ACTIVE'
      AND country = 'IN'
)
SELECT
    orders.order_id
FROM orders
JOIN active_indian_customers
  ON orders.customer_id = active_indian_customers.customer_id
```

Benefits:

* Named business concept
* Reusable logic
* Easier to test and review

---

## Subqueries vs CTEs in dbt Context

In dbt:

* Models are version-controlled
* Changes are reviewed in pull requests

CTEs win because:

* Reviewers can comment on individual steps
* Logic changes are localized
* Tests can be added downstream

Subqueries hide too much context.

---

## Anti-Pattern: Deeply Nested Subqueries

Example:

```
SELECT *
FROM (
    SELECT *
    FROM (
        SELECT * FROM orders
    ) o
) x
```

This adds no value and reduces readability.

Flatten immediately.

---

## Anti-Pattern: Repeating the Same Subquery

Example:

```
SELECT
    (SELECT COUNT(*) FROM orders WHERE status = 'COMPLETED') AS completed_orders,
    (SELECT SUM(amount) FROM orders WHERE status = 'COMPLETED') AS revenue
```

Refactor into a CTE once and reuse it.

---

## Practical Decision Rule

When writing SQL, ask:

* Does this logic have a name?
* Would I want to test it?
* Would I reuse it?

If yes, it should be a CTE, not a subquery.

---

## How This Applies to dbt Models

In dbt:

* Subqueries are acceptable for small, obvious checks
* Business logic belongs in named CTEs

This keeps models readable and change-safe as they grow.
