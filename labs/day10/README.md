# Lab — Day 10

Today you will add **tests**, a **seed**, and a **snapshot** to your existing dbt project.

You will run each piece as you build it, so you get fast feedback.

Keep your work small and reviewable.

---

## Before you start

### 1) Activate your Python virtual environment

From the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

If your dbt packages are already installed in this venv from earlier days, you do not need to reinstall.

### 2) Find your dbt project directory

In this training repository, the dbt project might be at the repo root or inside a subfolder.

Run this from the repository root:

```bash
find . -maxdepth 4 -name dbt_project.yml
```

* If you see exactly one result, that directory is your dbt project.
* If you see multiple results, use the one you have been using in earlier days.

Now `cd` into that directory.

Example (your path may differ):

```bash
cd path/to/your/dbt/project
```

### 3) Confirm dbt can connect

```bash
dbt debug
```

Expected output (brief):

* dbt prints your adapter (Snowflake)
* connection test ends with `OK`

If `dbt debug` fails, fix that before continuing.

---

## Part A — Built-in model tests

You will add built-in tests to existing models.

You should add tests that match the **grain** of the model.

### A1) List models you can test

```bash
dbt ls --resource-type model
```

Find the models that represent (or stage) these Olist tables:

* orders
* customers
* order_items
* payments
* products

Write down the exact model names you will use.

### A2) Identify the key columns in each model

For each model you chose, open its SQL file and identify:

* the primary key (or expected unique key)
* important foreign keys
* any status-like columns with a controlled domain

If you are not sure where a model’s SQL file lives, search for it.

Example (replace `stg_orders` with your model name):

```bash
grep -R "name: stg_orders" -n models
```

This should point you to the YAML where the model is defined.

### A3) Add `unique` and `not_null` to obvious primary keys

For your **orders** model:

* Add `unique` and `not_null` to the order identifier column.

For your **customers** model:

* Add `unique` and `not_null` to the customer identifier column.

If your model names differ from `stg_orders` / `stg_customers`, use your actual names.

Minimum expectation for this section:

* At least **two** models have a tested primary key.

### A4) Add `relationships` tests for critical joins

Choose at least **two** relationships that should always resolve.

Common Olist relationships (use the matching columns from your models):

* orders → customers (customer identifier)
* order_items → orders (order identifier)
* payments → orders (order identifier)

Add `relationships` tests so the child key values must exist in the parent model.

Run a targeted test after each relationship you add.

Example:

```bash
dbt test --select <your_orders_model>
```

Expected output (brief):

* dbt runs only the tests attached to that model
* you see `PASS` or `FAIL` lines per test

If you see failures, do not delete the test.

Instead:

* inspect what the test is complaining about
* decide whether the data is wrong or your assumption is wrong

### A5) Add `accepted_values` to a controlled domain column

Pick one column where the set of values is expected to be small and controlled.

A common candidate in Olist is an order status column.

Add an `accepted_values` test with a list of expected values.

Do not guess blindly.

Before you write the list:

* inspect the column values by running a query in Snowflake (recommended)
* or review the upstream source documentation you have

Then run:

```bash
dbt test --select <your_orders_model>
```

Minimum expectation for this section:

* At least **one** `accepted_values` test exists and is justified.

---

## Part B — One custom SQL test

You will write a single custom test that validates a cross-column rule.

### B1) Choose one rule

Pick one of these rules (choose one):

1. Delivered orders must have a delivered timestamp
2. Monetary amounts must not be negative

Use columns that exist in your current staging models.

If your model uses different column names than the raw Olist CSVs, adapt the test.

### B2) Create a singular test file

In your dbt project directory, confirm whether a `tests/` directory exists:

```bash
ls -la
```

* If `tests/` exists, use it.
* If it does not exist, create it:

```bash
mkdir -p tests
```

Create a test file with a clear name.

Examples:

* `tests/delivered_orders_require_delivered_date.sql`
* `tests/order_item_amounts_non_negative.sql`

Write a SELECT query that returns **violations**.

Rules for the SQL:

* Use uppercase keywords
* No `SELECT *`
* Return columns that make debugging easy

### B3) Run only singular tests

```bash
dbt test --select test_type:singular
```

Expected output (brief):

* dbt runs your custom test file
* test passes if it returns zero rows

If it fails:

* read the returned rows
* confirm whether the rule is correct
* keep the test (do not delete it)

---

## Part C — Seed for reference data

You will add a small reference dataset as a seed.

### C1) Create the seeds directory (if needed)

In your dbt project directory:

```bash
mkdir -p seeds
```

### C2) Create a seed CSV

Create a CSV named:

* `seeds/order_status_map.csv`

The seed should map a status value to a reporting group.

Minimum columns:

* `order_status`
* `status_group`

Minimum rows:

* At least **5** statuses that you believe exist in your data

Guidance:

* Keep groups simple (for example: `completed`, `in_progress`, `canceled`)
* Do not add personal or sensitive data

### C3) Load the seed

```bash
dbt seed --select order_status_map
```

Expected output (brief):

* dbt creates/loads a table for the seed
* dbt reports `OK` for the seed run

### C4) Add tests to the seed

Add built-in tests to the seed:

* `unique` and `not_null` on `order_status`
* `not_null` on `status_group`

Run:

```bash
dbt test --select order_status_map
```

---

## Part D — Snapshot for change tracking

You will define a basic snapshot and run it once.

### D1) Choose a snapshot target model

Snapshots work best for dimension-like models.

Pick one model where values could plausibly change over time.

A typical candidate:

* a customers model

### D2) Create the snapshots directory (if needed)

```bash
mkdir -p snapshots
```

### D3) Create a snapshot definition file

Create a file with a clear name.

Example:

* `snapshots/customers_snapshot.sql`

Your snapshot must include:

* `unique_key` (the stable identifier)
* a `strategy`
* the SELECT query for the tracked columns

Use the `check` strategy and track a small set of columns.

Minimum expectation:

* track **2–3** columns that represent meaningful customer attributes

Do not include volatile fields that change for reasons unrelated to business meaning.

### D4) Run the snapshot

```bash
dbt snapshot --select customers_snapshot
```

Expected output (brief):

* dbt creates the snapshot table on first run
* dbt reports the snapshot step as `OK`

If dbt cannot find your snapshot:

* confirm the file is under `snapshots/`
* confirm the snapshot name matches the one you selected

### D5) Confirm the snapshot exists

Run:

```bash
dbt ls --resource-type snapshot
```

You should see your snapshot name.

---

## Checkpoints

You are done when all of the following are true:

1. Built-in tests exist on at least **two** models and include:

   * `unique`
   * `not_null`
   * `relationships`
   * `accepted_values`
2. At least **one** custom singular test exists under `tests/` and runs
3. A seed named `order_status_map` loads with `dbt seed` and has tests
4. A snapshot exists and runs with `dbt snapshot`

---

## If you finish early

Pick one more model and add a well-justified test.

A good “extra” test is usually:

* a relationships test on a join your reporting depends on

Do not add random tests.

Add tests you can defend in code review.

When you are ready, proceed back to the Day 10 docs if you need a reminder:

* `../../docs/day10/01-built-in-tests.md`
* `../../docs/day10/02-custom-tests.md`
* `../../docs/day10/03-seeds.md`
* `../../docs/day10/04-snapshots.md`
