# 02 — CTE Patterns for dbt

## Objective

Show repeatable CTE patterns used in dbt models so transformations remain consistent, testable, and easy to extend across projects.

---

## Pattern: Source Standardization CTE

The first CTE in most dbt models standardizes raw inputs.

Purpose:

* Rename columns
* Cast data types
* Apply light, mechanical cleanup

Example:

```
WITH source_orders AS (
    SELECT
        order_id,
        customer_id,
        CAST(order_date AS DATE) AS order_date,
        CAST(amount AS NUMBER(10,2)) AS order_amount
    FROM raw.orders
)
SELECT *
FROM source_orders
```

This isolates raw data quirks from downstream logic.

---

## Pattern: Filter CTE

Filters that define business scope should live in their own CTE.

Example:

```
WITH source_orders AS (
    SELECT * FROM raw.orders
),
completed_orders AS (
    SELECT *
    FROM source_orders
    WHERE order_status = 'COMPLETED'
)
SELECT *
FROM completed_orders
```

This makes business rules explicit and reviewable.

---

## Pattern: Enrichment CTE

Enrichment CTEs join additional attributes without changing grain.

Example:

```
WITH orders AS (
    SELECT * FROM stg_orders
),
customers AS (
    SELECT * FROM stg_customers
),
enriched_orders AS (
    SELECT
        orders.order_id,
        orders.order_date,
        customers.customer_name
    FROM orders
    JOIN customers
      ON orders.customer_id = customers.customer_id
)
SELECT *
FROM enriched_orders
```

Enrichment adds context but preserves row meaning.

---

## Pattern: Aggregation CTE

Aggregations should always be isolated in their own CTE.

Example:

```
WITH orders AS (
    SELECT * FROM stg_orders
),
daily_revenue AS (
    SELECT
        order_date,
        SUM(order_amount) AS total_revenue
    FROM orders
    GROUP BY order_date
)
SELECT *
FROM daily_revenue
```

This makes grain changes explicit.

---

## Pattern: Final Select CTE

The final SELECT should be simple and boring.

Good pattern:

```
SELECT *
FROM daily_revenue
```

Avoid logic in the final SELECT. All logic should exist in prior CTEs.

---

## Pattern: Validation CTE (Optional)

Sometimes a temporary validation step helps verify assumptions.

Example:

```
WITH orders AS (
    SELECT * FROM stg_orders
),
validation AS (
    SELECT
        COUNT(*) AS row_count
    FROM orders
)
SELECT *
FROM validation
```

This is useful during development and removed later.

---

## Anti-Pattern: One Giant CTE

Avoid putting everything into one large CTE.

Problems:

* Hard to review
* Hard to test
* Small changes have large impact

If a CTE needs scrolling to understand, split it.

---

## Anti-Pattern: Logic in Final SELECT

Bad pattern:

```
SELECT
    order_date,
    SUM(order_amount)
FROM stg_orders
GROUP BY order_date
```

This hides aggregation logic at the end.

Prefer:

* Aggregation in its own CTE
* Final SELECT only references named results

---

## How These Patterns Scale in dbt

Across a dbt project:

* Staging models use 1–2 simple CTEs
* Intermediate models use structured chains
* Mart models use clear aggregation boundaries

Consistency matters more than cleverness.
