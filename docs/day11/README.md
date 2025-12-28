# Day 11: Scale and Performance — Mastering Incremental Models

Up to Day 10, we focused on **functional correctness**.

* Your models were correct.
* Your tests caught obvious data problems.
* But most of your models rebuilt tables from scratch.

Day 11 is where we change the mindset.

Our data grew.
Now the pipeline is slow.

A full refresh that used to finish in minutes can start taking hours. In Snowflake, that also means higher warehouse usage, which turns into real cost.

Today we learn how to **process only what changed**, without breaking correctness.

---

## What we are building today

We will take a heavy model (a fact table) and convert it from a full-refresh build into an **incremental model**.

By the end of today, you will be able to:

* Convert a heavy full-refresh model into an incremental model.
* Configure a **unique_key** so dbt can safely update existing rows.
* Use the `--full-refresh` flag when you change logic.
* Handle late-arriving data using a **lookback window**, without duplicate rows.
* Prove idempotency by running the same model twice.

---

## Why full refresh fails at scale

A full refresh rebuilds the entire table every time.

That means:

* Every row is re-read.
* Every join is re-computed.
* Every aggregate is re-built.

If your fact table becomes 100 million rows, you will pay that cost on every run.

The problem is not correctness.
The problem is *repeat work*.

Incremental models are how we avoid repeat work.

---

## What “incremental” means in dbt

An incremental model changes the build behavior:

* On the first run, dbt creates the table from scratch.
* On later runs, dbt only processes **new or recently changed data**.

In dbt, this is not a different SQL language.
It is the same SQL, but with a different materialization and a small amount of control logic.

The key idea is **idempotency**:

* If you run the model twice with the same input data,
* the second run should make **no changes**.

If your model adds duplicates on the second run, it is not production-safe.

---

## Today’s flow

Work through the day in this order.

1. Read the theory and concepts
2. Follow the implementation guide
3. Learn how to handle edge cases
4. Complete the lab

### Student docs

* `docs/day11/01-incremental-concepts.md`
* `docs/day11/02-implementation-guide.md`
* `docs/day11/03-challenges-and-backfills.md`

### Lab

* `labs/day11/README.md`

---

## What you need before starting

You should already have:

* A working dbt project pointed at Snowflake.
* A `fct_orders` model built previously (full refresh behavior).
* Data loaded from the Olist CSVs under `data/raw/`.

If `fct_orders` is missing, stop.
Go back to Day 9/10 and ensure your fact model exists and builds successfully.

---

## Ground rules for today

Incremental models can be fragile if you rush.
We will follow a safety-first approach.

* We will always define a **unique_key**.
* We will always prove behavior by running twice.
* We will always know when a full refresh is required.

If something looks “fast but wrong,” it is wrong.

---

## Commands you will run today

All commands assume you are in the repo root.

Activate your virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Run a single model:

```bash
dbt run -s fct_orders
```

Run a single model with full refresh:

```bash
dbt run -s fct_orders --full-refresh
```

You will run `dbt run -s fct_orders` multiple times today.
The second run should finish quickly and report **0 inserted / 0 updated** (or a similar “no-op” result depending on your dbt version and adapter output).

---

## What “success” looks like

By the end of Day 11:

* Your `fct_orders` model is materialized as incremental.
* You can run it twice with no duplicates.
* You can simulate new data and see only the new rows processed.
* You can explain when to use `--full-refresh`.

Now move to:

* `docs/day11/01-incremental-concepts.md`
