# Day 02 — Snowflake Essentials for Analytics Engineering

## Objective

Build a clear mental model of how Snowflake works internally so dbt models can be designed for correctness, performance, and cost control.

---

## Why Snowflake Matters in This Course

In this course, Snowflake is not treated as a generic database. It is the **execution engine** for all dbt transformations.

Every dbt design decision depends on understanding:

* How Snowflake stores data
* How compute is allocated
* How queries are optimized and charged

Without this understanding, dbt projects work initially but fail at scale.

---

## What This Day Covers

Day 02 focuses on Snowflake concepts that directly influence dbt behavior.

Topics covered:

* Snowflake’s separation of storage, compute, and services
* How warehouses execute dbt models
* How data is stored and pruned during queries

Examples throughout this day use:

* Raw tables (`raw.orders`)
* dbt-managed schemas (`analytics_dev.dbt_<user>`)
* Dedicated dbt warehouses

---

## What This Day Does Not Cover

To stay focused, the following are intentionally excluded:

* Snowflake account setup
* Advanced admin features
* Security hardening
* Snowflake-specific SQL extensions

Those topics are not required to design effective dbt transformations.

---

## Mental Model to Keep in Mind

```
Storage is shared
Compute is disposable
Transformations are repeatable
```

All Snowflake + dbt design decisions in later modules assume this model.

---

## How This Day Fits in the Course Flow

```
Day 01 → ELT concepts and dbt role
Day 02 → Snowflake internals for ELT
Day 03 → SQL discipline for analytics
Day 04 → CTE-based transformations
```

Day 02 bridges abstract ELT concepts and practical SQL modeling.

---

## How to Use These Documents

Read the documents in order:

1. Snowflake architecture
2. Snowflake setup for dbt
3. Storage and query behavior

Do not skip ahead. Each topic builds on the previous one.

---

These concepts will be referenced repeatedly during hands-on labs later.
