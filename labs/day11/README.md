# Day 11 Lab: Make `fct_orders` incremental (and prove it)

Today you will modify an existing model so it runs incrementally.

This lab is about **process**, not just code.

If you do not run the model multiple times and observe behavior, you did not complete the lab.

---

## What you will do

You will:

1. Convert `fct_orders` to `materialized='incremental'`.
2. Set `unique_key` correctly.
3. Add an `is_incremental()` filter with a 3-day lookback window.
4. Run the model twice to prove idempotency.
5. Simulate new data and confirm only the new slice is processed.

---

## Rules for this lab

* Do not create new models.
* Do not create new tables.
* Modify the existing `fct_orders.sql` model.
* Do not add custom macros.
* Do not use snapshots.

---

## Setup

From the repo root, activate your virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Confirm dbt can connect:

```bash
dbt debug
```

Expected result:

* You see a successful connection to Snowflake.

If `dbt debug` fails, stop.
Fix your profile before you continue.

---

## Task 1: Locate the `fct_orders` model

Find the SQL file that defines `fct_orders`.

Common locations:

* `models/marts/fct_orders.sql`
* `models/marts/core/fct_orders.sql`
* `models/fct_orders.sql`

Open the file.

You should see a CTE chain that starts from `orders` and joins to other Olist tables.

---

## Task 2: Add incremental configuration

At the top of `fct_orders.sql`, add a config block.

You must include both values:

* `materialized = 'incremental'`
* `unique_key = 'order_id'`

Do not use `SELECT *`.
Do not change the grain.

---

## Task 3: Add incremental filtering with a lookback window

Add an incremental filter using this pattern:

```sql
{% if is_incremental() %}
...
{% endif %}
```

Your filter must:

* Apply only on incremental runs
* Reprocess the last **3 days** of orders
* Use a timestamp column from `orders`

Use this lookback window (exactly 3 days):

```sql
{% if is_incremental() %}
WHERE order_purchase_timestamp >= DATEADD('day', -3, CURRENT_TIMESTAMP)
{% endif %}
```

Important:

* Put the filter on the `orders` driving dataset.
* Do not filter on a joined table.

---

## Task 4: Run the model (first run)

Run only `fct_orders`:

```bash
dbt run -s fct_orders
```

Expected result:

* The model builds successfully.
* dbt creates or updates the target table.

---

## Task 5: Run the model again (idempotency check)

Immediately run the same command again:

```bash
dbt run -s fct_orders
```

Expected result:

* The second run finishes faster.
* The row count of `fct_orders` does not increase.

If the row count increases, you have duplicates.

Do not continue.
Fix the model first.

### How to check row count

Run this in Snowflake (use your schema):

```sql
SELECT COUNT(*) AS row_count
FROM <your_schema>.fct_orders;
```

Run it after the first and second run.
The counts should match.

---

## Task 6: Simulate new data (so the incremental run has work)

In the real world, new data arrives on its own.

In training, we simulate it.

You will do that by changing a timestamp in `orders` so one row appears “recent” and falls inside your 3-day lookback window.

### Step A: Pick one order_id to use

In Snowflake, pick a single order_id from the raw orders table.

Example query:

```sql
SELECT
    order_id,
    order_purchase_timestamp
FROM <your_schema>.orders
ORDER BY order_purchase_timestamp DESC
LIMIT 5;
```

Choose one `order_id` from the results.

### Step B: Make it look recent

Update that row so `order_purchase_timestamp` is set to the current timestamp.

Example:

```sql
UPDATE <your_schema>.orders
SET order_purchase_timestamp = CURRENT_TIMESTAMP
WHERE order_id = '<your_order_id>';
```

Expected result:

* 1 row updated.

Important:

* This is only a training simulation.
* In production, you do not mutate raw sources like this.

---

## Task 7: Run the model again (incremental should pick up the change)

Run:

```bash
dbt run -s fct_orders
```

Expected result:

* dbt processes the lookback slice.
* The order you updated is included in that slice.
* Your table does **not** duplicate rows.

### Prove there are no duplicates

Run this query in Snowflake:

```sql
SELECT
    order_id,
    COUNT(*) AS row_count
FROM <your_schema>.fct_orders
GROUP BY 1
HAVING COUNT(*) > 1;
```

Expected result:

* No rows returned.

If rows return, stop.
Your incremental merge is not idempotent.

---

## Task 8: (Optional but useful) Force a rebuild with `--full-refresh`

Now that the model is incremental, practice the reset.

Run:

```bash
dbt run -s fct_orders --full-refresh
```

Expected result:

* The table is rebuilt from scratch.

Then run again normally:

```bash
dbt run -s fct_orders
```

Expected result:

* The second run is fast and does not change row counts.

---

## What to submit / show in class

Be ready to show:

* Your updated `fct_orders.sql` config block
* Your `is_incremental()` filter
* Evidence from two consecutive runs that row counts are stable
* The duplicate-check query returning no rows

---

## Common problems (and what to check)

### Problem: row count increases on the second run

Check:

* Did you set `unique_key = 'order_id'`?
* Is `fct_orders` truly one row per order_id?
* Did you accidentally create duplicate rows in the final SELECT?

### Problem: incremental run does nothing even after you update `orders`

Check:

* Is your `WHERE` clause inside the `is_incremental()` block?
* Are you filtering on the correct timestamp column?
* Did you update a row inside your target schema’s `orders` table?

### Problem: model fails with SQL errors in the Jinja block

Check:

* Your `{% if %}` and `{% endif %}` tags are correctly placed
* Your `WHERE` clause is valid SQL in Snowflake

---

## Stop here

Do not start building new models.

Next, your instructor will review solution patterns and debugging techniques.
