# Day 10 Lab — Model Tests, One Custom Test, One Seed, One Snapshot

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## 0) Confirm you finished Day 09

From your project root:

```bash
cd ~/dbt_olist_project
source ~/.venv/bin/activate
```

These must work before you start Day 10:

```bash
dbt test --select source:olist
```

```bash
dbt source freshness --select source:olist
```

If either fails, fix Day 09 first.

---

## 1) Save a git checkpoint (do this before you change anything)

Today you will add YAML files, SQL tests, a seed CSV, and a snapshot.

You want a clean diff so you can review exactly what you changed.

```bash
git status
```

If you see `not a git repository`:

```bash
git init
```

Checkpoint the Day 09 state:

```bash
git add -A
git commit -m "day09 checkpoint before day10 tests seeds snapshots"
```

If git says “nothing to commit”, that is fine.

---

## Part A — Built-in model tests

We are going to test **models**, not sources.

These tests protect the transformed tables people actually query.

### A1) Create a model test YAML file (copy/paste)

Create this file:

```bash
nano models/marts/_marts_tests.yml
```

Paste this content exactly.

This assumes you built these models on Day 08:

* `dim_customers`
* `fct_orders`

If your model names differ, stop and fix the names in the YAML before running tests.

```yml
version: 2

models:
  - name: dim_customers
    description: "Customer dimension built from stg_customers."
    columns:
      - name: customer_id
        description: "Customer primary key."
        tests:
          - not_null
          - unique

  - name: fct_orders
    description: "Orders fact table built from int_orders_enriched. One row per order_id."
    columns:
      - name: order_id
        description: "Order primary key."
        tests:
          - not_null
          - unique

      - name: customer_id
        description: "Customer foreign key. Must exist in dim_customers.customer_id."
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id

      - name: order_status
        description: "Order lifecycle status."
        tests:
          - not_null
          - accepted_values:
              values:
                - created
                - approved
                - processing
                - invoiced
                - shipped
                - delivered
                - canceled
                - unavailable
```

Save and exit.

### A2) Parse (fail fast)

```bash
dbt parse
```

If this fails, do not run tests.
Fix indentation first.

### A3) Run only the tests for these two models

This runs the tests attached to `dim_customers` and `fct_orders`.

```bash
dbt test --select dim_customers fct_orders
```

What a failure means:

* `unique` fails on `order_id`: your fact table is no longer one row per order
* `relationships` fails on `customer_id`: you have orders without a matching customer
* `accepted_values` fails: raw status values changed or your list does not match actual data

If `accepted_values` fails, do not guess.
Check the actual values in your **model**, not the raw source:

```sql
select
  order_status,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
group by order_status
order by nrows desc;
```

Update the list in `_marts_tests.yml` to match exactly.

---

## Part B — One custom SQL test (singular test)

Built-in tests cannot express every rule.

A singular test is just a SQL query.

* If it returns **zero rows**, the test passes.
* If it returns **any rows**, the test fails and the rows are your debugging output.

### B1) Create the tests folder

```bash
mkdir -p tests
```

### B2) Create one test file (copy/paste)

Create this file:

```bash
nano tests/delivered_orders_require_delivered_date.sql
```

Paste this SQL.

This checks a real rule:

* If `order_status = 'delivered'`, then `order_delivered_customer_ts` must not be null.

```sql
SELECT
  order_id,
  order_status,
  order_delivered_customer_ts,
  order_purchase_ts
FROM {{ ref('fct_orders') }}
WHERE order_status = 'delivered'
  AND order_delivered_customer_ts IS NULL
```

Save and exit.

### B3) Run only singular tests

```bash
dbt test --select test_type:singular
```

If it fails, do not delete the test.

Instead:

* Look at the returned rows.
* Decide whether the rule is wrong, or the data is wrong.

---

## Part C — Seed for reference data

A seed is a small CSV under version control.

Use seeds when the data is:

* small
* stable
* meaningful in business logic

### C1) Create the seeds folder

```bash
mkdir -p seeds
```

### C2) Create the seed CSV (copy/paste)

Create this file:

```bash
nano seeds/order_status_map.csv
```

Paste this content exactly.

```csv
order_status,status_group
created,in_progress
approved,in_progress
processing,in_progress
invoiced,in_progress
shipped,in_progress
delivered,completed
canceled,canceled
unavailable,canceled
```

Save and exit.

### C3) Create seed tests YAML (copy/paste)

Create this file:

```bash
nano seeds/_seeds.yml
```

Paste this content exactly.

```yml
version: 2

seeds:
  - name: order_status_map
    description: "Reference mapping from order_status to a reporting-friendly status_group."
    columns:
      - name: order_status
        description: "Raw order status value."
        tests:
          - not_null
          - unique

      - name: status_group
        description: "High-level grouping used for reporting."
        tests:
          - not_null
```

Save and exit.

### C4) Load the seed

```bash
dbt seed --select order_status_map
```

### C5) Test the seed

```bash
dbt test --select order_status_map
```

---

## Part D — Snapshot for change tracking

A snapshot keeps history when rows change over time.

For training, we will snapshot customer attributes.

We are not trying to build a complex historical warehouse.

We are proving the mechanics:

* first run creates the snapshot table
* later runs insert new versions when attributes change

### D1) Create the snapshots folder

```bash
mkdir -p snapshots
```

### D2) Create the snapshot file (copy/paste)

Create this file:

```bash
nano snapshots/customers_snapshot.sql
```

Paste this content exactly.

This uses the `check` strategy.

* `unique_key` identifies the row
* `check_cols` are the attributes we want to track

```sql
{% snapshot customers_snapshot %}

{{
  config(
    target_schema='ANALYTICS_DEV',
    unique_key='customer_id',
    strategy='check',
    check_cols=['customer_city', 'customer_state', 'customer_zip_code_prefix']
  )
}}

SELECT
  customer_id,
  customer_zip_code_prefix,
  customer_city,
  customer_state
FROM {{ ref('dim_customers') }}

{% endsnapshot %}
```

Save and exit.

### D3) Run the snapshot

Run in dev:

```bash
dbt snapshot --select customers_snapshot
```

What to expect on first run:

* dbt creates a snapshot table
* dbt inserts one “current” version of each customer

### D4) Confirm dbt sees the snapshot

```bash
dbt ls --resource-type snapshot
```

You should see `customers_snapshot`.

---

## Checkpoints

You are done when all of these succeed:

```bash
dbt test --select dim_customers fct_orders
```

```bash
dbt test --select test_type:singular
```

```bash
dbt seed --select order_status_map
```

```bash
dbt test --select order_status_map
```

```bash
dbt snapshot --select customers_snapshot
```

---

## Compare changes and commit Day 10

Review your changes:

```bash
git diff
```

Commit:

```bash
git add -A
git commit -m "day10 add model tests singular test seed snapshot"
```

---

## If you finish early

Add one more relationship test that matches how you join models.

Good candidate:

* `fct_orders.order_id` should exist in `int_orders_enriched.order_id`

Only add tests you can defend in code review.
