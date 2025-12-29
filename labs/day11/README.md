# Day 11 Lab — Incremental Models and Optimization

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## 0) Confirm you finished Day 10

From your project root:

```bash
cd ~/dbt_olist_project
source ~/.venv/bin/activate
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

If you get stuck, you want a clean diff that shows exactly what changed.

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

## 2) Convert `fct_orders` to an incremental model

### What we are doing

Right now, `fct_orders` is a table built as a full refresh.

On large fact tables, rebuilding from scratch is slow and expensive.

We will:

* switch `fct_orders` to `materialized='incremental'`
* use `incremental_strategy='merge'` so dbt can **insert new rows** and **update changed rows**
* set `unique_key='order_id'` so merges are idempotent
* add a small **lookback window** so late-arriving changes still get reprocessed

### File to edit

Open:

```bash
nano models/marts/fct_orders.sql
```

Replace the *entire file* with this content (copy/paste).

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

## 3) Parse (fail fast)

This catches syntax/Jinja issues before you run anything.

```bash
dbt parse
```

If this fails:

* confirm the config block is at the top
* confirm `{% if is_incremental() %}` and `{% endif %}` are both present

---

## 4) Run `fct_orders` as a full refresh (first build)

The first time you switch to incremental, do a full refresh so the target table matches the new definition.

```bash
dbt run --select fct_orders --full-refresh
```

What to expect:

* dbt rebuilds the table from scratch
* you should see `OK` for `fct_orders`

---

## 5) Prove idempotency (run it twice)

An incremental model must be safe to run repeatedly.

Run again without `--full-refresh`:

```bash
dbt run --select fct_orders
```

What to expect:

* dbt runs a `MERGE`
* the model finishes `OK`
* row counts should not change

Quick count check in Snowflake:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

Run that count query before and after the second run. The number should be the same.

---

## 6) Prove the model can correct changes (update case)

We will simulate an “upstream change” safely by modifying the **target table** in dev.

We are not touching `OLIST.RAW`.

### Step 6A) Pick a recent order_id from the fact table

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

Copy the `order_id` value you get.

### Step 6B) Corrupt the row in the target table (dev only)

Update that order’s `payment_total_value` to a wrong value:

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

### Step 6C) Run incremental again

```bash
dbt run --select fct_orders
```

Because we reprocess a 3-day lookback window based on `order_purchase_ts`, dbt should include that recent order in the merge.

Verify the value is corrected back to the value coming from `int_orders_enriched`:

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

You want `fct_value = int_value`.

---

## 7) Prove inserts work (missing row case)

Now we simulate “a new row needs to be inserted” by deleting a single row from the target table.

Again: dev only. Do not touch raw.

Delete the same order_id from Step 6:

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

Now run incremental again:

```bash
dbt run --select fct_orders
```

Confirm the row is back:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_id = '<PASTE_ORDER_ID_HERE>';
```

You want `nrows = 1`.

---

## 8) What the lookback window is doing

This line is the lookback window:

```sql
WHERE order_purchase_ts >= (
  SELECT DATEADD(day, -3, MAX(order_purchase_ts))
  FROM {{ this }}
)
```

Meaning:

* dbt looks at the current target table (`{{ this }}`)
* finds the latest `order_purchase_ts` already loaded
* backs up 3 days
* reprocesses that slice on every incremental run

Why this matters:

* If late-arriving data changes something within the last 3 days, it gets picked up.
* If you set the window too small, you miss late updates.
* If you set it too large, you reprocess too much data each run.

---

## 9) Backfill behavior (when to use `--full-refresh`)

Use `--full-refresh` when you change logic in a way that affects historical rows.

Examples:

* you change join logic
* you change aggregation logic
* you add a filter that changes which rows should exist

Command:

```bash
dbt run --select fct_orders --full-refresh
```

---

## 10) Compare changes and commit Day 11

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

1. `fct_orders` is incremental and builds with `--full-refresh`
2. Running `dbt run --select fct_orders` twice does not change row counts
3. A corrupted value in dev is corrected by the next incremental run
4. A deleted row in dev is re-inserted by the next incremental run
