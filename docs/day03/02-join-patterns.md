# 02 — Join Patterns

## Objective

Explain how to design correct joins in analytical SQL so results are accurate, reviewable, and stable when reused in dbt models.

---

## Why Joins Are the Biggest Source of Errors

Most analytics bugs come from joins, not calculations.

Common symptoms:

* Revenue suddenly doubles
* Counts change after adding a new table
* Metrics differ across dashboards

These issues usually come from mismatched grain or incorrect join conditions.

---

## Understand Table Grain Before Joining

Every table has a **grain**: what one row represents.

Examples:

* `orders` → one row per order
* `order_items` → one row per item per order
* `payments` → one row per payment attempt

Joining tables without aligning grain produces incorrect results.

---

## One-to-One Joins

A one-to-one join means each row matches at most one row on the other side.

Example:

* `orders` joined to `customers`

```
orders (1 row per order)
customers (1 row per customer)
```

Correct pattern:

```
FROM orders
JOIN customers
  ON orders.customer_id = customers.customer_id
```

This join does not multiply rows.

---

## One-to-Many Joins

One-to-many joins are common and dangerous.

Example:

* One order has many order items

```
orders (1 row per order)
order_items (many rows per order)
```

Naive join:

```
FROM orders
JOIN order_items
  ON orders.order_id = order_items.order_id
```

Result:

* One order row becomes many rows
* Measures at order level are duplicated

---

## Safe Patterns for One-to-Many

If you need order-level metrics, aggregate first.

Correct pattern:

```
WITH items_agg AS (
    SELECT
        order_id,
        SUM(item_amount) AS order_item_total
    FROM order_items
    GROUP BY order_id
)
SELECT
    orders.order_id,
    items_agg.order_item_total
FROM orders
LEFT JOIN items_agg
  ON orders.order_id = items_agg.order_id
```

This preserves the grain of `orders`.

---

## Many-to-Many Joins

Many-to-many joins are almost always incorrect in analytics.

Example:

* Orders joined to promotions
* Promotions joined to products

Symptoms:

* Exploding row counts
* Impossible-to-explain metrics

If you think you need a many-to-many join, stop and redesign the model.

---

## LEFT JOIN vs INNER JOIN

INNER JOIN:

* Drops rows without a match

LEFT JOIN:

* Preserves all rows from the left table

Example:

* Orders without payments

```
FROM orders
LEFT JOIN payments
  ON orders.order_id = payments.order_id
```

This keeps unpaid orders in the result.

---

## Detecting Join Issues

When reviewing joins, always ask:

* Can this join multiply rows?
* Are filters applied in the correct place?

Simple validation:

* Compare row counts before and after joins

---

## How This Applies to dbt Models

In dbt:

* Each model should have a clearly defined grain
* Grain must not change accidentally downstream

Bad joins propagate errors across the DAG.

Correct joins make downstream models predictable and testable.
