# Day 03 — SQL Foundations for Analytics Engineering

## Objective

Establish strict SQL writing standards so analytical transformations are readable, correct, and maintainable when used in dbt projects.

---

## Why SQL Discipline Matters

In modern ELT, SQL is not exploratory or temporary. It becomes production code.

Poor SQL practices cause:

* Incorrect joins and double counting
* Queries that are hard to review or debug
* Models that break silently when data changes

Unlike application code, SQL errors often produce *wrong results* instead of failures.

---

## SQL as Production Code

SQL in analytics must follow the same discipline as application code.

Expectations:

* Clear intent
* Predictable structure
* Explicit logic

Example problem:

* A query works for today’s data
* A new column or duplicate record appears
* Metrics drift without obvious errors

SQL discipline reduces this risk.

---

## Scope of This Day

This day focuses only on **foundational SQL patterns** required for dbt.

Included:

* Clean formatting and naming
* Join correctness
* Avoiding ambiguous logic

Excluded:

* Window functions
* Advanced analytics SQL
* Vendor-specific SQL extensions

Those topics appear later where they are used.

---

## Mental Model for Writing Analytical SQL

```
Readable → Reviewable → Testable → Reliable
```

If a query is hard to read, it is hard to validate.

---

## Example of Intent-First SQL

Bad pattern:

```
SELECT *
FROM orders o
JOIN customers c ON o.id = c.id
```

Issues:

* Ambiguous join condition
* Unclear column origin

Better pattern:

```
SELECT
    o.order_id,
    o.order_date,
    c.customer_id,
    c.customer_name
FROM orders o
JOIN customers c
  ON o.customer_id = c.customer_id
```

Intent is explicit and reviewable.

---

## How This Day Fits in the Course

```
Day 01 → ELT concepts
Day 02 → Snowflake internals
Day 03 → SQL discipline
Day 04 → CTE-based transformations
Day 05 → dbt project structure
```

Without strong SQL foundations, dbt models become fragile.

---

## How to Use This Day

Read and apply these rules consistently.

Every dbt model written later in the course assumes:

* Clean SQL
* Explicit joins
* Predictable formatting

Inconsistency here multiplies complexity later.
