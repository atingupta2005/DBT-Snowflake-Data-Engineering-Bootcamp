# Day 10 Lab — Model Tests, One Custom Test, One Seed, One Snapshot

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## 0) Confirm you finished Day 09

Day 10 builds on Day 09.

If sources are failing or freshness is broken, your model tests and snapshots will be noisy.

From your project root:

```bash
cd ~/dbt_olist_project
source ~/.venv/bin/activate
```

Run:

```bash
dbt test --select source:olist
```

```bash
dbt source freshness --select source:olist
```

Both must succeed before continuing.

---

## 1) Save a git checkpoint (do this before you change anything)

Today you will add YAML tests, a custom SQL test, a seed CSV, and a snapshot.

Checkpoint first so you can compare Day 09 vs Day 10 changes.

```bash
git status
```

If you see `not a git repository`:

```bash
git init
```

Commit the Day 09 state:

```bash
git add -A
git commit -m "day09 checkpoint before day10 tests seeds snapshots"
```

If git says “nothing to commit”, that is fine.

---

## Part A — Built-in model tests

### What model tests are (simple example)

A model test is a rule applied to a *transformed* table.

Example:

* `dim_customers.customer_id` must be unique

If it is not unique, your dimension is no longer one row per customer.

That causes duplicates when you join facts to dimensions.

### A1) Create a marts tests file (copy/paste)

Create:

```bash
nano models/marts/_marts_tests.yml
```

Paste:

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

### A2) Parse first

Parsing catches YAML/Jinja mistakes early.

```bash
dbt parse
```

### A3) Run tests for only these two models

```bash
dbt test --select dim_customers fct_orders
```

How to read failures (practical meaning):

* `unique` fails on `fct_orders.order_id`

  * Your fact is no longer one row per order.
  * Downstream metrics like “number of orders” become wrong.

* `relationships` fails on `fct_orders.customer_id`

  * You have orders that do not match a customer.
  * Joins to `dim_customers` will drop those orders.

* `accepted_values` fails on `order_status`

  * The upstream status domain changed, or your list is wrong.

If `accepted_values` fails, validate the actual values in Snowflake.

```sql
select
  order_status,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
group by order_status
order by nrows desc;
```

Update the YAML list to match exactly.

---

## Part B — One custom SQL test (singular test)

### What a custom (singular) test is

A singular test is just a SQL query that returns **violations**.

* 0 rows returned → PASS
* 1+ rows returned → FAIL (and the rows help you debug)

This is useful for rules that are not simple key/relationship checks.

### Example of a real rule

Business rule:

* “If an order is delivered, it should have a delivered timestamp.”

This is not a `not_null` on the column, because non-delivered orders can legitimately have null delivery timestamps.

### B1) Create the tests folder

```bash
mkdir -p tests
```

### B2) Create the test file (copy/paste)

Create:

```bash
nano tests/delivered_orders_require_delivered_date.sql
```

Paste:

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

If it fails, pull the violating rows from Snowflake too:

```sql
select
  order_id,
  order_status,
  order_delivered_customer_ts,
  order_purchase_ts
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_status = 'delivered'
  and order_delivered_customer_ts is null
limit 50;
```

---

## Part C — Seed for reference data

### What a seed is (simple example)

A seed is a small CSV committed to git.

dbt loads it into your warehouse as a table.

You use it when:

* the data is small (tens/hundreds of rows)
* the mapping is stable
* you want the mapping versioned with the code

Real example:

* map `order_status` → `status_group` so dashboards don’t need long CASE statements

### C1) Create the seeds folder

```bash
mkdir -p seeds
```

### C2) Create the seed CSV (copy/paste)

Create:

```bash
nano seeds/order_status_map.csv
```

Paste:

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

### C3) Add seed tests YAML (copy/paste)

Create:

```bash
nano seeds/_seeds.yml
```

Paste:

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

### C6) Use the seed in Snowflake (example)

This is what you would do in a reporting query.

```sql
select
  m.status_group,
  count(*) as order_count
from OLIST.ANALYTICS_DEV.FCT_ORDERS f
join OLIST.ANALYTICS_DEV.ORDER_STATUS_MAP m
  on f.order_status = m.order_status
group by m.status_group
order by order_count desc;
```

---

## Part D — Snapshot for change tracking

### What a snapshot is (plain English)

A snapshot is how you keep **history of changes**.

Without a snapshot, a table only shows the latest state.

With a snapshot, you can answer questions like:

* “What was the customer’s city last month?”
* “When did this customer attribute change?”

### The common mental model

Think of a snapshot like this:

* A normal dimension table is a single row per customer.
* A snapshot table is **many rows per customer**, one row per version.

Each version has:

* when it became valid
* when it stopped being valid

That is SCD Type 2 behavior.

### What we snapshot today

We will snapshot customer attributes from `dim_customers`.

This is realistic:

* customers can change city/state over time
* analysts often want to understand behavior “as-of” a time period

### Important training detail

The Olist dataset is historical.

There will be no natural changes flowing in.

So we will simulate a change in **dev** to prove snapshots work.

We are not touching `OLIST.RAW`.

### D1) Create the snapshots folder

```bash
mkdir -p snapshots
```

### D2) Create the snapshot definition (copy/paste)

Create:

```bash
nano snapshots/customers_snapshot.sql
```

Paste:

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

What the config means:

* `unique_key`: identifies the entity we track (one customer)
* `strategy='check'`: dbt compares columns in `check_cols`
* `check_cols`: if any of these change, dbt creates a new version row

### D3) Run the snapshot (first run)

```bash
dbt snapshot --select customers_snapshot
```

What to expect on first run:

* dbt creates the snapshot table
* dbt inserts one current version per customer

### D4) Find the snapshot table in Snowflake

Most of the time the table will be:

* `OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT`

Confirm with:

```sql
show tables like 'CUSTOMERS_SNAPSHOT' in schema OLIST.ANALYTICS_DEV;
```

If you do not see it, list snapshots in dbt:

```bash
dbt ls --resource-type snapshot
```

### D5) Inspect the snapshot schema (Snowflake)

A dbt snapshot table includes metadata columns.

Common ones are:

* `DBT_SCD_ID`
* `DBT_UPDATED_AT`
* `DBT_VALID_FROM`
* `DBT_VALID_TO`

Check what exists in your table:

```sql
desc table OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT;
```

### D6) Query “current” customer records

Current rows are usually where `DBT_VALID_TO` is null.

```sql
select
  customer_id,
  customer_city,
  customer_state,
  dbt_valid_from,
  dbt_valid_to
from OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT
where dbt_valid_to is null
limit 20;
```

### D7) Prove it is stable (second run)

Run the snapshot again without changing anything:

```bash
dbt snapshot --select customers_snapshot
```

Expected outcome:

* dbt reports success
* it should not insert new versions, because nothing changed

Quick check (row count should stay the same):

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT;
```

### D8) Simulate a change (dev only)

We will force one customer to “move cities” in dev so the snapshot detects a change.

#### Step 1: pick a customer_id

Pick any customer from `dim_customers`:

```sql
select
  customer_id,
  customer_city,
  customer_state
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
limit 1;
```

Copy the `customer_id`.

#### Step 2: update `dim_customers` directly (dev only)

We are deliberately introducing a change in the *model table*.

This is only to demonstrate snapshot behavior.

```sql
update OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
set customer_city = 'test_city_day10'
where customer_id = '<PASTE_CUSTOMER_ID_HERE>';
```

Verify the change:

```sql
select
  customer_id,
  customer_city,
  customer_state
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
where customer_id = '<PASTE_CUSTOMER_ID_HERE>';
```

#### Step 3: run the snapshot again

```bash
dbt snapshot --select customers_snapshot
```

Expected outcome:

* dbt detects the changed city
* it closes the old version (`dbt_valid_to` becomes non-null)
* it inserts a new current version (`dbt_valid_to` is null)

#### Step 4: view the history for that customer

```sql
select
  customer_id,
  customer_city,
  customer_state,
  dbt_valid_from,
  dbt_valid_to
from OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT
where customer_id = '<PASTE_CUSTOMER_ID_HERE>'
order by dbt_valid_from;
```

You should see two rows:

* older row with `dbt_valid_to` filled
* current row with `dbt_valid_to` null

### D9) Restore `dim_customers` back to normal

Because you edited a model table manually, reset it by re-running the model.

```bash
dbt run --select dim_customers
```

If you want to verify the reset:

```sql
select
  customer_id,
  customer_city
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
where customer_id = '<PASTE_CUSTOMER_ID_HERE>';
```

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

And you verified in Snowflake that:

* `CUSTOMERS_SNAPSHOT` exists in `OLIST.ANALYTICS_DEV`
* one customer shows two versions after your simulated change

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

Good candidates:

* `fct_orders.order_id` should exist in `int_orders_enriched.order_id`
* `dim_customers.customer_unique_id` should be `not_null` if you rely on it

Only add tests you can defend in code review.
