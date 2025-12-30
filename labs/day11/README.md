# Day 11 Lab — Incremental Models and Optimization

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## What you will do today

Up to Day 10, you rebuilt tables from scratch.

That works when the dataset is small.

It fails when your fact tables grow:

* long runtimes
* higher warehouse cost
* more surface area for failures

Today you will convert `fct_orders` into an incremental model.

You will prove three things:

1. First build works with `--full-refresh`.
2. Running the model twice is safe (idempotent).
3. The model can update and insert rows using `MERGE`.

---

## 0) Confirm you finished Day 10

From your project root:

```bash
cd ~/dbt_olist_project
source .venv/bin/activate
```

These must work before you start Day 11:

```bash
dbt test
```

```bash
dbt seed --select order_status_map
```

```bash
dbt snapshot --select customers_snapshot
```

If any of these fail, fix Day 10 first.

---

## 1) Save a git checkpoint (do this before you change anything)

We will change `fct_orders.sql` to incremental.

Checkpoint first so your diff is clean and reviewable.

```bash
git status
```

If you see `not a git repository`:

```bash
git init
```

Checkpoint the Day 10 state:

```bash
git add -A
git commit -m "day10 checkpoint before day11 incremental"
```

If git says “nothing to commit”, that is fine.

---

## 2) Understand what you are building (before you edit anything)

### Full refresh behavior (what you had until now)

A full refresh model rebuilds the table from scratch every run.

That means:

* every row is reprocessed
* runtime grows with table size

### Incremental merge behavior (what you will build)

An incremental model with `merge` does this on each run:

* identifies a slice of source data to process
* merges that slice into the existing target table

A merge can:

* insert rows that do not exist yet
* update rows that already exist

To make merge safe, you must define the row identity.

That is `unique_key`.

For `fct_orders`, the identity is:

* `order_id`

---

## 3) Convert `fct_orders` to incremental

### What you will implement

You will add:

* `materialized='incremental'`
* `incremental_strategy='merge'`
* `unique_key='order_id'`
* a lookback window (3 days) to handle late-arriving data
* `on_schema_change='sync_all_columns'` (briefly: don’t error if columns evolve)

### Why we use a lookback window

Late-arriving data is real.

Examples:

* an order status updates a day later
* a delivered timestamp arrives late
* a payment correction arrives after initial load

If you only process “newer than max timestamp”, you can miss updates.

A lookback window intentionally reprocesses a recent slice every run.

---

### 3A) Edit the model file

Open:

```bash
nano models/marts/fct_orders.sql
```

Replace the entire file with this content.

```sql
{{
  config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='order_id',
    on_schema_change='sync_all_columns'
  )
}}

WITH orders_enriched AS (
    SELECT
        order_id,
        customer_id,
        order_status,
        order_purchase_ts,
        order_approved_ts,
        order_delivered_carrier_ts,
        order_delivered_customer_ts,
        order_estimated_delivery_date,

        order_item_row_count,
        distinct_product_count,
        items_total_value,
        freight_total_value,

        payment_total_value,
        max_payment_installments,
        payment_row_count,
        payment_type_any
    FROM {{ ref('int_orders_enriched') }}

    {% if is_incremental() %}
    WHERE order_purchase_ts >= (
        SELECT
            DATEADD(day, -3, MAX(order_purchase_ts))
        FROM {{ this }}
    )
    {% endif %}
)

SELECT
    order_id,
    customer_id,
    order_status,
    order_purchase_ts,
    order_approved_ts,
    order_delivered_carrier_ts,
    order_delivered_customer_ts,
    order_estimated_delivery_date,

    order_item_row_count,
    distinct_product_count,
    items_total_value,
    freight_total_value,

    payment_total_value,
    max_payment_installments,
    payment_row_count,
    payment_type_any
FROM orders_enriched
```

Save and exit.

---

## 4) Parse (fail fast)

This catches syntax/Jinja issues before you run anything.

```bash
dbt parse
```

If this fails:

* confirm the config block is at the top
* confirm `{% if is_incremental() %}` and `{% endif %}` both exist

Do not continue until parse succeeds.

---

## 5) First build: run as a full refresh

The first time you switch to incremental, you want the target table to match the new definition.

Run:

```bash
dbt run --select fct_orders --full-refresh
```

What to expect:

* dbt rebuilds the table from scratch
* the run finishes `OK`

Optional sanity check in Snowflake:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

---

## 6) Prove idempotency (run it twice)

An incremental model must be safe to run repeatedly.

### 6A) Capture row count before

In Snowflake:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

Copy the number.

### 6B) Run incremental without full refresh

```bash
dbt run --select fct_orders
```

What to expect:

* dbt runs a `MERGE`
* the model finishes `OK`

### 6C) Capture row count after

In Snowflake, run the same count again:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

Expected outcome:

* the row count should be the same as before

If the row count changes on the second run, stop.

That means your merge is not idempotent.

Most common cause:

* the `unique_key` is wrong

---

## 7) Prove updates work (merge update case)

We will simulate an upstream correction.

We will do it safely by modifying the **dev target table**.

We are not touching `OLIST.RAW`.

### 7A) Pick a recent `order_id`

Pick the most recent order by purchase timestamp:

```sql
select
  order_id,
  order_purchase_ts,
  payment_total_value
from OLIST.ANALYTICS_DEV.FCT_ORDERS
order by order_purchase_ts desc
limit 1;
```

Copy the `order_id`.

### 7B) Corrupt the row in the dev target table

Add 1 to `payment_total_value`:

```sql
update OLIST.ANALYTICS_DEV.FCT_ORDERS
set payment_total_value = payment_total_value + 1
where order_id = '<PASTE_ORDER_ID_HERE>';
```

Verify it changed:

```sql
select
  order_id,
  payment_total_value
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_id = '<PASTE_ORDER_ID_HERE>';
```

### 7C) Run incremental again

```bash
dbt run --select fct_orders
```

Why this should work:

* your lookback window reprocesses recent orders
* the corrupted order should be included in the merge slice

### 7D) Confirm the merge corrected the value

Compare `fct_orders` to `int_orders_enriched`:

```sql
select
  f.order_id,
  f.payment_total_value as fct_value,
  i.payment_total_value as int_value
from OLIST.ANALYTICS_DEV.FCT_ORDERS f
join OLIST.ANALYTICS_DEV.INT_ORDERS_ENRICHED i
  on f.order_id = i.order_id
where f.order_id = '<PASTE_ORDER_ID_HERE>';
```

Expected outcome:

* `fct_value = int_value`

If they do not match:

* your lookback window may not include that order
* pick a more recent order_id and repeat

---

## 8) Prove inserts work (merge insert case)

Now we simulate a missing row.

We will delete one row from the dev target table and let merge re-insert it.

### 8A) Delete the same `order_id`

```sql
delete from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_id = '<PASTE_ORDER_ID_HERE>';
```

Confirm it is gone:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_id = '<PASTE_ORDER_ID_HERE>';
```

Expected:

* `nrows = 0`

### 8B) Run incremental again

```bash
dbt run --select fct_orders
```

### 8C) Confirm the row came back

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_id = '<PASTE_ORDER_ID_HERE>';
```

Expected:

* `nrows = 1`

If it does not come back:

* your lookback window did not include this order
* repeat with a more recent order_id

---

## 9) Understand the lookback window (read this once)

This is the lookback filter:

```sql
WHERE order_purchase_ts >= (
  SELECT DATEADD(day, -3, MAX(order_purchase_ts))
  FROM {{ this }}
)
```

Meaning:

* dbt finds the newest `order_purchase_ts` already loaded
* it backs up 3 days
* it reprocesses that slice on every incremental run

Trade-offs:

* too small → you miss late updates
* too large → you reprocess too much every run

A 3-day lookback is a training default.

In production, you pick the window based on:

* how late your upstream data arrives
* how expensive the merge slice is

---

## 10) Backfills (when to use `--full-refresh`)

Use `--full-refresh` when your logic changes in a way that affects historical rows.

Common examples:

* you change join logic
* you change aggregation logic
* you add/remove a filter

Command:

```bash
dbt run --select fct_orders --full-refresh
```

Do not use `--full-refresh` for routine daily runs.

It defeats the purpose of incremental.

---

## 11) Schema evolution (brief, practical note)

You set:

* `on_schema_change='sync_all_columns'`

That tells dbt:

* if upstream adds columns, try to keep the target table in sync

This is not a magic fix.

You still need to review schema changes.

But it prevents your pipeline from failing immediately when columns appear.

---

## 12) Compare changes and commit Day 11

Review what you changed:

```bash
git diff
```

Commit:

```bash
git add -A
git commit -m "day11 make fct_orders incremental with merge"
```

---

## Checkpoints

You are done when all of these are true:

1. `dbt run --select fct_orders --full-refresh` succeeds.
2. Running `dbt run --select fct_orders` twice does not change row counts.
3. A corrupted value in dev is corrected by the next incremental run.
4. A deleted row in dev is re-inserted by the next incremental run.
