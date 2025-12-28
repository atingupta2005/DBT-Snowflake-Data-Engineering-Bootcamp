# Incremental concepts

We have a simple problem.

Our `fct_orders` table keeps growing.

When you run a full refresh, dbt rebuilds the whole table from scratch. That was fine when the table was small.

At scale, it becomes a bottleneck.

This file explains what “incremental” means in dbt and when you should use it.

---

## The cost of a full refresh

A full refresh does the same work every run.

Even if only 10 new orders arrived, a full refresh will:

* Read all historical orders again
* Re-join to customers, payments, and items again
* Re-calculate all derived fields again
* Write the entire table again

On Snowflake, that creates two kinds of pain:

* **Time**: your run takes longer as the table grows
* **Cost**: you consume more warehouse credits doing repeat work

In production, the cost curve is not linear.

Your model might look like it “works” in a small dataset.
Then it fails when data volume increases.

### A realistic growth pattern

This is common:

* Month 1: 1 million rows, full refresh takes 3 minutes
* Month 6: 20 million rows, full refresh takes 45 minutes
* Month 12: 100 million rows, full refresh takes hours

At that point, your pipeline becomes fragile.

* Teams start skipping runs.
* People run models at weird times to avoid contention.
* Backfills become scary.

Incremental models are the standard fix.

---

## What an incremental model does

An incremental model changes *how dbt builds the table*.

The SQL still defines the rows you want.
But dbt changes the behavior depending on whether the table already exists.

Typical behavior:

* First run: build the table from scratch
* Later runs: only process new or changed rows

This is why today’s key word is **idempotency**.

If the inputs do not change, the output must not change.

If you run your model twice and it duplicates rows, you do not have an incremental model.
You have a production incident waiting to happen.

---

## Insert vs update: what “Merge strategy” means

Incremental builds come in two common flavors.

### Insert-only

You only ever add new rows.
You never update existing rows.

This works when:

* Your source data never changes after it lands
* Your business rules treat records as immutable

It fails when:

* The same business key can appear again with corrected data
* Late payments arrive after the original order record
* You reclassify a status later

### Merge (insert + update)

A merge strategy means:

* If a row is new, insert it
* If a row already exists, update it

This is the most common pattern for fact tables.

In dbt, you enable this by setting a **unique_key**.

That unique key tells dbt what it means for a row to be “the same row” across runs.

On Snowflake, dbt can implement this as a `MERGE` statement under the hood.
You write model SQL the same way.
The configuration tells dbt how to apply it.

---

## Where incremental fits in a dbt project

Incremental models are not a default.
They are a tool.

Use them when full refresh is expensive.

You will typically incrementalize:

* Large fact tables (`fct_orders` is a good example)
* Event-style tables (clicks, logs, interactions)
* Tables that grow every day and are queried often

You usually do **not** incrementalize:

* Small dimensions
* Reference tables
* Models that are cheap to rebuild

Why not incrementalize everything?

Because incremental logic adds complexity:

* You need a stable unique key
* You need a strategy for late-arriving data
* You need a safe way to change logic and rebuild

If your model runs in 10 seconds as a full refresh, you probably leave it alone.

---

## Incremental thinking with the Olist dataset

In this course, we only use the Olist tables:

* `customers`
* `orders`
* `order_items`
* `payments`
* `products`

A typical `fct_orders` model might join orders to customers, payments, and order items.

That makes it heavier than a single-table model.

Even on a small dataset, you can feel the difference once your SQL includes:

* Multiple joins
* Aggregations (for items and payments)
* Derived columns

Incrementalization is about not repeating that work for historical orders that did not change.

---

## Choosing a good unique key

A unique key must be:

* Stable over time
* Present on every row
* Truly unique for the grain of the table

For a fact table like `fct_orders`, you usually build at the **order grain**.

That means “one row per order.”

In the Olist dataset, `order_id` is commonly the right unique key.

Be careful.

If your model is at a different grain, `order_id` might not be unique.

Example:

* If you build one row per order item, `order_id` repeats
* In that case, you would need a different unique key (often composite)

Today’s lab assumes `fct_orders` is one row per order.

---

## A clean mental model for incremental builds

Think in two layers:

1. **Business logic**: what the final rows should look like
2. **Incremental filter**: which source records to process today

You do not rewrite your business logic.
You add a filter so dbt can limit work on incremental runs.

A common incremental filter looks like this:

* “Only process orders created in the last 3 days”
* “Only process rows with updated_at >= max(updated_at) already loaded”

We will use a lookback window later today.

---

## Common failure modes

When teams first implement incremental models, they usually hit one of these issues.

### 1) Duplicates

You used insert-only logic, but the same order appears again.

Or you forgot to configure a unique key.

Or your unique key does not match your grain.

### 2) Missing updates

Your incremental filter only looks for new records.
But updates happen later.

Example: payment data arrives after the order.

If you only process new orders, you never pick up that payment.

### 3) Late-arriving data breaks correctness

You filtered on “today only.”
A record arrives late.
It never gets processed.

This is why we use a lookback window.

### 4) Logic changes require rebuild

You change the SQL logic.
An incremental run only updates a small slice.
Your historical rows stay in the old logic.

When the logic changes, you usually need `--full-refresh`.

We will cover exactly how and when in the next files.

---

## What you should be able to say out loud

After this section, you should be able to explain:

* Why full refresh becomes expensive and unreliable as tables grow
* What incremental means in dbt
* The difference between insert-only and merge behavior
* Why large fact tables benefit most
* Why incremental models must be idempotent

Now move to:

* `docs/day11/02-implementation-guide.md`
