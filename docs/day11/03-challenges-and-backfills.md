# Challenges and backfills

Incremental models are where dbt stops being “just SQL.”

The SQL is still SQL.
But the *runtime behavior* now depends on history.

That makes incremental models powerful.
It also makes them easier to get wrong.

This file covers three problems you will hit in real projects:

* Late arriving data
* Backfills when logic changes
* Schema evolution

---

## Late arriving data

Late arriving data means a record shows up later than you expect.

In analytics, this is normal.

Common causes:

* Upstream systems retry on failure
* Payment providers settle transactions later
* Data is delivered in batches
* Someone fixes records after the fact

If your incremental filter only looks for “new rows since last run,” you miss late arrivals.

That creates silent correctness bugs.

### Example: late payment

Imagine this timeline:

* Monday: order is created
* Monday: your pipeline runs and loads the order
* Wednesday: payment record arrives

If your incremental model only loads “orders created today,” you will never update Monday’s order with the payment.

Your fact table stays wrong.

### The standard fix: a lookback window

A lookback window means:

* Every run, you reprocess a small slice of recent history
* You accept redoing a little work to protect correctness

Typical lookback windows:

* 1 day (very strict pipelines)
* 3 days (common default)
* 7 days (more tolerance for late systems)

### A lookback window in dbt

The usual pattern is:

* Filter the driving dataset by a timestamp
* Use `is_incremental()` so the filter only applies on incremental runs

Example pattern:

```sql
{% if is_incremental() %}
WHERE order_purchase_timestamp >= DATEADD('day', -3, CURRENT_TIMESTAMP)
{% endif %}
```

This is not “perfect.”

It is a trade-off.

You choose the window size based on how late you expect records to arrive.

### How to know your lookback window works

You test the process.

1. Run the model once
2. Run it again → row count should not grow
3. Modify some data that falls inside the lookback window
4. Run again → the row should update, not duplicate

The lab will guide you through this.

---

## Backfills and the `--full-refresh` flag

Incremental models protect you from repeat work.

But they create a new risk:

Historical data can get stuck in an old version of your logic.

### When you need a full refresh

You usually need `--full-refresh` when you change logic that affects historical rows.

Examples:

* You change how a derived column is calculated
* You add a new join that affects existing rows
* You change the grain of the model
* You fix a bug that existed for months

If you only run incremental after those changes, you will fix only the lookback slice.

Everything older stays wrong.

### What `--full-refresh` does

When you run:

```bash
dbt run -s fct_orders --full-refresh
```

dbt will:

* Drop and rebuild the incremental table
* Recompute all rows using the new logic

This is your “reset button.”

### A safe backfill workflow

When you change logic in an incremental model, use this sequence.

1. Make the SQL change

2. Run a full refresh for the model

```bash
dbt run -s fct_orders --full-refresh
```

3. Run the model again normally

```bash
dbt run -s fct_orders
```

That second run is your idempotency check.

If the second run changes row counts, something is off.

### Be honest about backfill cost

A full refresh is expensive.

That is fine.

You do it intentionally and rarely.

Incremental models are about making the *daily* run cheap.

Backfills are planned events.

---

## Schema evolution (brief but important)

Schema evolution means the input columns change over time.

Examples:

* A new column is added to `orders`
* A column changes type (string to timestamp)
* A column disappears

Incremental tables can be sensitive to schema changes.

Why?

Because dbt is not rebuilding the table every time.

If the table already exists, a new column might not appear unless you handle it.

### dbt’s `on_schema_change`

dbt provides a config option called `on_schema_change`.

You can set it in the model config.

Common options include behaviors like:

* Ignore schema changes
* Append new columns
* Fail the run

We are keeping this brief today.

In most training projects, schema drift is not frequent.

In production, it is.

If your organization has upstream teams that frequently change schemas, you should prefer:

* A safe default (fail fast or append)
* Clear coordination

We will not implement `on_schema_change` in today’s lab unless it is already part of your project standards.

---

## A practical debugging mindset

Incremental issues feel confusing because they depend on state.

When something looks wrong, debug in this order.

### 1) Confirm the unique key

Ask:

* Is this model truly one row per `order_id`?
* Does the final SELECT contain duplicate order_ids?

A fast check is to run a query in Snowflake:

```sql
SELECT
    order_id,
    COUNT(*) AS row_count
FROM <your_schema>.fct_orders
GROUP BY 1
HAVING COUNT(*) > 1;
```

If this returns rows, your grain is wrong or your logic duplicates.

### 2) Confirm the incremental filter

Ask:

* Is the filter applied to the right dataset?
* Does the timestamp column exist and behave as expected?

A common mistake is filtering after a join.
That can change which rows are included.

### 3) Prove idempotency again

Run twice.

```bash
dbt run -s fct_orders
dbt run -s fct_orders
```

If the second run changes row counts, you still have a correctness problem.

### 4) Use full refresh to reset state

If you suspect state drift, reset it:

```bash
dbt run -s fct_orders --full-refresh
```

Then run once more normally.

---

## What you should be able to do after this

You should be able to:

* Explain why we need a lookback window
* Use `--full-refresh` when logic changes
* Recognize why schema changes can break incrementals
* Debug duplicates using the unique key and grain

Now go to the lab:

* `labs/day11/README.md`
