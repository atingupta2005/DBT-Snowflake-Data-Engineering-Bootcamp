# 01 — Clean Analytical SQL

## Objective

Define clear, repeatable SQL writing standards so analytical queries are easy to read, review, and safely reused in dbt models.

---

## Why Clean SQL Is Non‑Negotiable

Analytical SQL is read far more often than it is written.

In dbt projects:

* Multiple people review the same model
* Small logic changes affect downstream metrics
* Errors often produce *plausible but wrong* results

Readable SQL reduces review time and prevents silent data issues.

---

## One Logical Step per Line

Each selected column should represent one clear idea.

Bad example:

```
SELECT order_id, order_date, amount*tax_rate AS total FROM orders
```

Issues:

* Hard to scan
* Logic hidden inline

Better example:

```
SELECT
    order_id,
    order_date,
    amount,
    tax_rate,
    amount * tax_rate AS tax_amount
FROM orders
```

Intent is visible without reading expressions carefully.

---

## Explicit Column Selection

Avoid `SELECT *` in analytical SQL.

Problems with `SELECT *`:

* Schema changes silently alter results
* Column order becomes unpredictable
* Reviewers cannot see intent

Example failure:

* A new column is added upstream
* Downstream logic changes unintentionally

Always select only required columns.

---

## Consistent Naming Conventions

Column names should describe *what the value represents*, not how it was calculated.

Example:

* Prefer `order_total_amount`
* Avoid `amt1`, `calc_val`, `x`

Good naming reduces the need for comments.

---

## Table Aliasing Rules

Aliases must improve clarity, not reduce it.

Bad example:

```
FROM orders o
JOIN customers c ON o.id = c.id
```

Issues:

* Ambiguous join condition
* Aliases hide intent

Better example:

```
FROM orders AS orders
JOIN customers AS customers
  ON orders.customer_id = customers.customer_id
```

In dbt models, clarity matters more.

---

## Formatting Joins for Review

Joins should be visually scannable.

Recommended pattern:

```
FROM orders
LEFT JOIN customers
  ON orders.customer_id = customers.customer_id
LEFT JOIN payments
  ON orders.order_id = payments.order_id
```

This layout makes it easy to:

* Verify join keys
* Spot missing conditions

---

## Avoid Hidden Logic in WHERE Clauses

WHERE clauses should filter data, not transform it.

Bad example:

```
WHERE TO_DATE(created_at) = '2024-01-01'
```

Better example:

```
WHERE created_at >= '2024-01-01'
  AND created_at <  '2024-01-02'
```

This improves correctness and query pruning.
 - Crucial, automatic performance optimization technique that minimizes the amount of data scanned during query execution
---

## Commenting Guidelines

Use comments sparingly and purposefully.

Good use:

* Explaining business rules
* Explaining non-obvious filters

Bad use:

* Restating column names
* Explaining obvious SQL syntax

Example:

```
-- Orders cancelled before fulfillment are excluded from revenue
WHERE order_status != 'CANCELLED'
```

---

## SQL Review Checklist

When reviewing analytical SQL, always check:

* Are all selected columns intentional?
* Are joins explicit and correct?
* Is formatting consistent?
* Is business logic visible?

If a query is hard to explain, it is not ready for production.

---

## How This Applies to dbt

Every dbt model is reviewed, versioned, and reused.

Clean SQL enables:

* Reliable refactoring
* Effective testing
* Clear lineage

Poor SQL multiplies downstream complexity.

## SQL Review Checklist

When reviewing analytical SQL, always check:

* Are all selected columns intentional?
* Are joins explicit and correct?
* Is formatting consistent?
* Is business logic visible?

If a query is hard to explain, it is not ready for production.

---

## How This Applies to dbt

Every dbt model is reviewed, versioned, and reused.

Clean SQL enables:

* Reliable refactoring
* Effective testing
* Clear lineage

Poor SQL multiplies downstream complexity.
