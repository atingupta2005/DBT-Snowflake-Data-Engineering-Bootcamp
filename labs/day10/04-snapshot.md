# Day 10 Lab — 04 Snapshot (Change Tracking)

In this file you will create **one** snapshot.

You will run it multiple times so you can see what it does.

You will also simulate a change in dev so you can confirm you get real history.

---

## What problem snapshots solve

Most dbt models represent the **latest state**.

Example:

* `dim_customers` has one row per customer
* that row represents the current known city/state

In the real world, customer attributes change.

Common examples:

* customer moves to a new city
* customer changes state
* customer zip code updates

If you overwrite the dimension row, you lose history.

That means you cannot answer questions like:

* “What city did this customer live in when they placed an order in 2017?”
* “How many customers moved states this quarter?”
* “What was the customer’s state as of the month-end snapshot date?”

A snapshot creates history by storing **multiple versions**.

---

## The mental model you must keep

A normal dimension table:

* one row per entity (one row per `customer_id`)

A snapshot table:

* many rows per entity
* one row per version of the entity

Each version has:

* when the version became valid
* when the version stopped being valid

This is SCD Type 2 behavior.

---

## What we will snapshot

We will snapshot customer attributes from `dim_customers`.

Specifically:

* `customer_zip_code_prefix`
* `customer_city`
* `customer_state`

Why these columns:

* they are plausible “slowly changing” attributes
* they are easy to reason about in a lab

---

## Important training constraint (read this carefully)

The Olist dataset is historical.

In a real pipeline, source data changes over time.

Here, it will not naturally change while you are running the lab.

So to prove snapshots work, we will do something controlled:

* we will simulate a change in **dev only**
* we will not touch `OLIST.RAW`

You will temporarily edit one row in `OLIST.ANALYTICS_DEV.DIM_CUSTOMERS`.

Then you will reset it by rebuilding the model.

---

## A) Create the `snapshots/` folder

From your project root:

```bash
mkdir -p snapshots
```

---

## B) Create the snapshot definition

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

### What this config means (in real terms)

#### `target_schema='ANALYTICS_DEV'`

The snapshot table will be built in:

* `OLIST.ANALYTICS_DEV`

So you can test safely.

#### `unique_key='customer_id'`

This is the entity identifier.

dbt assumes:

* one customer → one current record

If the unique key is wrong, your snapshot history will be wrong.

#### `strategy='check'`

dbt detects changes by comparing columns.

If any `check_cols` value changes compared to the last snapshot run, dbt:

* closes the old version
* inserts a new version

#### `check_cols=[...]`

These are the columns dbt watches.

If a watched column changes, dbt creates a new history row.

If a column is not in `check_cols`, changes to that column will not create new versions.

---

## C) List snapshots (sanity check)

Before you run it, confirm dbt can see the snapshot.

Run:

```bash
dbt ls --resource-type snapshot
```

Expected output:

* `customers_snapshot`

If you do not see it:

* confirm the file is in `snapshots/`
* confirm the file contains `{% snapshot customers_snapshot %}`

---

## D) Run the snapshot (first run)

Run:

```bash
dbt snapshot --select customers_snapshot
```

What to expect on the first run:

* dbt creates the snapshot table
* dbt inserts one version per customer
* every row is considered the current version

---

## E) Find and inspect the snapshot table in Snowflake

### E1) Confirm the table exists

Run:

```sql
show tables like 'CUSTOMERS_SNAPSHOT' in schema OLIST.ANALYTICS_DEV;
```

You should see a table named `CUSTOMERS_SNAPSHOT`.

### E2) Inspect the snapshot table columns

A snapshot table includes your selected columns plus dbt metadata.

Run:

```sql
desc table OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT;
```

You should see (names may vary by adapter/version, but these are common):

* `DBT_SCD_ID`
* `DBT_UPDATED_AT`
* `DBT_VALID_FROM`
* `DBT_VALID_TO`

These columns are how dbt tracks versions.

### E3) Query current rows

Current rows usually have `dbt_valid_to` as null.

Run:

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

Expected outcome:

* you get rows
* `dbt_valid_to` is null for these rows

---

## F) Run the snapshot again (no changes)

Run:

```bash
dbt snapshot --select customers_snapshot
```

Expected outcome:

* dbt succeeds
* it should not insert new versions because nothing changed

### F1) Quick stability check (row count)

Run:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT;
```

Save this number.

Run the same query after step H.

---

## G) How to think about snapshot changes (before you simulate one)

When a watched column changes for an entity:

1. dbt finds the current row for that `customer_id`
2. it sets `dbt_valid_to` on that row (closing it)
3. it inserts a new row with the new values
4. the new row has `dbt_valid_to` as null

So after a change, you should see **two rows** for that customer:

* old version (closed)
* new version (current)

---

## H) Simulate a change in dev (to prove you get two versions)

We will change the city for one customer in the **dev model table**.

This is only to demonstrate snapshot behavior.

### H1) Pick a customer_id

Pick a customer from your dev dimension:

```sql
select
  customer_id,
  customer_city,
  customer_state
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
limit 1;
```

Copy the `customer_id`.

### H2) Capture the original values (so you can sanity-check later)

Run:

```sql
select
  customer_id,
  customer_city,
  customer_state,
  customer_zip_code_prefix
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
where customer_id = '<PASTE_CUSTOMER_ID_HERE>';
```

Copy the original city/state/zip values.

### H3) Update the model table (dev only)

Run:

```sql
update OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
set customer_city = 'test_city_day10'
where customer_id = '<PASTE_CUSTOMER_ID_HERE>';
```

Verify:

```sql
select
  customer_id,
  customer_city,
  customer_state
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
where customer_id = '<PASTE_CUSTOMER_ID_HERE>';
```

You should see `customer_city = 'test_city_day10'`.

---

## I) Run the snapshot after the change

Run:

```bash
dbt snapshot --select customers_snapshot
```

Expected outcome:

* dbt detects the changed city
* it closes the old version
* it inserts a new current version

---

## J) Verify history in the snapshot table

### J1) Query only the changed customer

Run:

```sql
select
  customer_id,
  customer_city,
  customer_state,
  customer_zip_code_prefix,
  dbt_valid_from,
  dbt_valid_to
from OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT
where customer_id = '<PASTE_CUSTOMER_ID_HERE>'
order by dbt_valid_from;
```

Expected outcome:

* 2 rows for the customer

How to interpret the two rows:

* older row: `dbt_valid_to` is populated (closed)
* newer row: `dbt_valid_to` is null (current)

### J2) Confirm overall row count increased

Run the row count again:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.CUSTOMERS_SNAPSHOT;
```

Expected behavior:

* row count should be higher than the value you saved in step F1

It may increase by 1 (one new version row) for this lab.

---

## K) Restore dev back to normal

You manually edited a dbt-built model table.

That is not normal practice.

Reset by rebuilding the model.

Run:

```bash
dbt run --select dim_customers
```

Optional verification:

```sql
select
  customer_id,
  customer_city
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS
where customer_id = '<PASTE_CUSTOMER_ID_HERE>';
```

You should no longer see `test_city_day10`.

---

## L) Common snapshot misunderstandings (read before you move on)

### L1) “Why didn’t my snapshot create new rows?”

Most common reasons:

* you didn’t actually change any `check_cols`
* you changed a column that is not in `check_cols`
* you updated the wrong table (not `OLIST.ANALYTICS_DEV.DIM_CUSTOMERS`)
* you forgot to rerun `dbt snapshot`

Debug it like this:

1. confirm your snapshot definition includes the column you changed
2. confirm your update changed the value (select the row)
3. run `dbt snapshot --select customers_snapshot` again
4. query the snapshot table for that customer_id

### L2) “Why do I see many metadata columns?”

Those columns are the point.

They are how dbt tracks versions.

You typically query snapshots by:

* `dbt_valid_to is null` (current rows)
* filtering by an as-of time range (advanced; not required today)

### L3) “Why not just rebuild the dimension?”

Rebuilding gives you only the latest state.

Snapshots keep the previous states.

If your business needs “as-of” reporting, you need history.

---

## Done criteria

You are done with this file when:

```bash
dbt snapshot --select customers_snapshot
```

runs successfully, and you verified in Snowflake that:

* `CUSTOMERS_SNAPSHOT` exists in `OLIST.ANALYTICS_DEV`
* your chosen customer_id shows **two versions** after the simulated change

---

## Next step

Go back to `labs/day10/README.md` and run the final checkpoints.
