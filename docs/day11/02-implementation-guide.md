# Implementation guide: converting `fct_orders` to incremental

Today you will modify an existing model: `fct_orders.sql`.

Your goal is not just to “make it run.”

Your goal is to make it **safe**:

* Run it twice → second run should do nothing
* Add new data → only the new slice should be processed
* Late data arrives → your lookback window should pick it up

This guide shows the exact patterns to use.

---

## Before you touch anything

First, confirm the current behavior.

Run your model as it exists today:

```bash
dbt run -s fct_orders
```

Expected result:

* The model builds successfully.
* It likely scans all upstream sources because it is a full-refresh build.

If this fails, stop.
Fix Day 9/10 issues before you continue.

---

## Step 1: Find the `fct_orders` model file

Open the model file that builds `fct_orders`.

In many dbt projects, it will be under something like:

* `models/marts/fct_orders.sql`
* `models/marts/core/fct_orders.sql`
* `models/fct_orders.sql`

Do not create a new model.
We are converting the existing one.

---

## Step 2: Add incremental materialization

At the top of the model SQL file, add a `config` block.

Example pattern:

```sql
{{
    config(
        materialized = 'incremental',
        unique_key   = 'order_id'
    )
}}
```

What this does:

* `materialized = 'incremental'` tells dbt to build this model incrementally.
* `unique_key = 'order_id'` tells dbt how to match rows between runs.

### Why we always include `unique_key`

Without a unique key, dbt cannot safely update existing rows.

In most adapters, that means dbt will behave like insert-only.

Insert-only behavior is the fastest way to introduce duplicates.

For `fct_orders` (one row per order), `order_id` is the correct key.

If your `fct_orders` is not one row per order, stop.
You need to fix the grain before you incrementalize.

---

## Step 3: Add an incremental filter using `is_incremental()`

Now we need to teach dbt which rows to process on incremental runs.

This is the standard pattern:

```sql
WITH ...

SELECT ...
FROM ...

{% if is_incremental() %}
WHERE ...
{% endif %}
```

Important:

* The filter only applies on incremental runs.
* On the first run (or a full refresh), dbt builds the full table.

### Where should the filter go?

Put the filter on the **primary driving dataset**.

For `fct_orders`, that is usually the `orders` table.

If your model starts from `orders` and then joins to other tables, your filter usually applies to `orders`.

Do not filter on a joined table unless you understand the impact.

Filtering on payments, for example, might exclude orders without payments.

---

## Step 4: Use a lookback window for late-arriving data

If you only process “new orders since last run,” you will miss updates.

Example:

* An order was created on Monday.
* Payment data arrived on Wednesday.

If your incremental filter only loads Monday’s order once, you never pick up the payment.

We fix this by reprocessing a small recent window every run.

A common lookback is **3 days**.

### A practical filter for `fct_orders`

Most `orders` tables have a timestamp like:

* `order_purchase_timestamp`

Your incremental filter can reprocess the last 3 days of orders.

Example:

```sql
{% if is_incremental() %}
WHERE order_purchase_timestamp >= DATEADD('day', -3, CURRENT_TIMESTAMP)
{% endif %}
```

What this does:

* Every incremental run rebuilds the “last 3 days” slice.
* Older history stays untouched.

Because we configured `unique_key`, dbt can safely update the rows in that slice.

### Choosing the window size

3 days is a starting point.

In the real world, you choose it based on:

* How late data arrives
* How often upstream systems correct past records
* How expensive it is to reprocess

If late data can arrive 14 days late, a 3-day window is not enough.

---

## Step 5: Keep your model SQL clean

Incremental does not mean messy SQL.

Follow these rules:

* Keep your CTE chain
* One logical step per CTE
* Avoid nested subqueries
* No `SELECT *`
* Use explicit names

A good incremental model should be easy to review.

---

## Step 6: Run the model twice to prove idempotency

After you add incremental config and the `is_incremental()` filter:

Run the model:

```bash
dbt run -s fct_orders
```

Then run it again immediately:

```bash
dbt run -s fct_orders
```

Expected result:

* The second run should be fast.
* The second run should not add new rows.

Depending on your dbt version, the logs might show:

* “0 inserted, 0 updated”
* Or “no-op” behavior
* Or simply a run that completes with no visible row-count changes

If your row count increases on the second run, you have a bug.

The most common causes:

* Missing or wrong `unique_key`
* Wrong grain (order_id not unique)
* Filter logic not aligned with your driving dataset

---

## Step 7: Know when to use `--full-refresh`

Incremental runs only rebuild a slice.

If you change the SQL logic that affects historical rows, incremental runs will not fix history.

Example logic changes:

* Changing how you classify order status
* Fixing a bug in a derived column
* Adding a new join that affects historical values

In those cases, run:

```bash
dbt run -s fct_orders --full-refresh
```

A full refresh rebuilds the whole incremental table.

You use it when:

* Your logic changed
* You need to backfill
* Your table is inconsistent

We cover backfills more deeply in the next file.

---

## Brief note: ordering and clustering

In Snowflake, physical ordering is not controlled the same way as traditional warehouses.

But you can still influence performance using ideas like:

* clustering keys
* designing models that filter efficiently

For this course, keep this simple:

* Make sure your incremental filter uses a timestamp column
* Keep joins and aggregations stable

We are focusing on correctness first.

---

## Checklist: your `fct_orders` model is incremental if

You can answer yes to all of these:

* The model has `materialized = 'incremental'`
* The model has `unique_key = 'order_id'`
* The SQL includes an `is_incremental()` filter
* Running twice results in no duplicates
* You know when to run `--full-refresh`

Now move to:

* `docs/day11/03-challenges-and-backfills.md`
