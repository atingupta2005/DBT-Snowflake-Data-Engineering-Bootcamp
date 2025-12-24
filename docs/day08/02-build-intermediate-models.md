# 02 — Build Intermediate Models

With staging models in place, we now build **intermediate models**.

Intermediate models exist to combine multiple staged tables into clean, predictable inputs for marts. This is where joins and CTE discipline matter.

In this file, we will do three things:

1. Choose intermediate model(s) and their purpose
2. Build intermediate models using `ref()` and CTE chains
3. Run intermediate models and confirm they build cleanly

## Rules for intermediate models

Intermediate models are where we start shaping data, but we still avoid “final” outputs.

Allowed:

* Joining multiple staging models
* Consolidation and enrichment (bringing related attributes together)
* Standardizing keys and timestamps across joined data
* Light derivations needed to support marts (for example: calculating an order-level total from line items)

Not allowed:

* Publishing a “business-facing” fact or dimension as the final output
* Mixing multiple grains in one model
* Hiding complexity in nested subqueries

Your goal is to produce stable, readable building blocks.

## Decide your intermediate models

Use intermediate models to prepare clean inputs for marts built from these raw tables only:

* customers
* orders
* order_items
* payments
* products

A practical approach is to build intermediate models that cover two grains:

* **Order grain**: one row per order
* **Order item grain**: one row per order item

Do not try to solve everything in one model.

### Naming convention

Use `int_` for intermediate models.

Examples:

* `int_orders_enriched` (order grain)
* `int_order_items_enriched` (order item grain)

Name by purpose, not by which tables you joined.

## Build intermediate models with `ref()`

Intermediate models must reference staging models with `ref()`.

Never hard-code schema names or object names.

Good:

* `{{ ref('stg_orders') }}`

Bad:

* `RAW.OLIST.ORDERS`

If you hard-code names, dbt cannot reliably build lineage or manage dependencies.

## CTE discipline in intermediate models

Intermediate models should be written as a CTE chain.

* One logical step per CTE
* Each CTE name describes what it does
* Final select is short and explicit

### Pattern to follow

Use this structure in every intermediate model:

```sql
WITH orders AS (

    SELECT
        ...
    FROM {{ ref('stg_orders') }}

),

customers AS (

    SELECT
        ...
    FROM {{ ref('stg_customers') }}

),

enriched AS (

    SELECT
        ...
    FROM orders
    LEFT JOIN customers
        ON ...

)

SELECT
    ...
FROM enriched
```

## Handling payments and order totals

Payments and order items introduce multiple rows per order. That can break order-grain models if you join directly.

When you need an order-grain intermediate model, you usually want to **aggregate first** and then join.

Example pattern:

```sql
WITH payments AS (

    SELECT
        ...
    FROM {{ ref('stg_payments') }}

),

payments_by_order AS (

    SELECT
        ...
    FROM payments
    GROUP BY
        ...

)

SELECT
    ...
FROM payments_by_order
```

The same pattern applies to order items when you need order-level totals.

Do not guess column names. Use only columns that exist in your staged models.

## Preventing grain mistakes

Before you write a join, state the grain out loud:

* “This model is one row per order.”
* “This model is one row per order item.”

Then enforce it:

* Do not join a many-to-one table without checking that it stays many-to-one.
* Do not join a many-to-many relationship without aggregating or resolving it.

If you see row counts explode, you have a grain problem.

## Run intermediate models

Build intermediate models after staging is stable.

Expected outcome:

* Intermediate models build successfully
* dbt builds staging dependencies automatically via `ref()`
* Row counts make sense for the stated grain

## Sanity checks

Do these checks immediately after a successful run:

1. **Row count** matches the grain

   * Order-grain model row count should be close to the number of orders
   * Order-item-grain model row count should be close to the number of order items

2. **No unexpected duplication**

   * If you join customers onto orders, each order should still be one row

3. **Key columns are populated**

   * Join keys should not be mostly NULL

If a check fails, fix intermediate models before moving on to marts.

## What you need before moving on

Before you proceed to marts, you must have:

* At least one intermediate model at the order grain
* At least one intermediate model at the order item grain (if you plan to use items in your fact)
* Clean CTE chains and readable joins
* Correct `ref()` usage everywhere

Next: `03-build-facts-and-dimensions.md`
