# Day 10 Lab — 03 Seed (Reference Data)

The seed will turn a small CSV into a real table in Snowflake.

Then you will use it in a model so you can feel why seeds matter.

---

## What a seed is (practical definition)

A seed is a CSV file inside your dbt project.

When you run `dbt seed`, dbt loads that CSV into your target schema as a table.

This is useful when you want small, stable reference data to be:

* versioned in git
* reviewed in pull requests
* deployed along with models

Typical real-world seed use cases:

* status mappings (raw status → reporting bucket)
* small SLA thresholds (team → allowed latency)
* feature flags for reporting (metric_name → include/exclude)

Seeds are **not** for:

* large datasets
* anything that grows daily
* anything that should be ingested as a source table

---

## The scenario we will solve

`fct_orders` contains an `order_status` value.

Analysts do not want to repeat this forever:

```sql
CASE
  WHEN order_status IN ('created','approved','processing','invoiced','shipped') THEN 'in_progress'
  WHEN order_status = 'delivered' THEN 'completed'
  WHEN order_status IN ('canceled','unavailable') THEN 'canceled'
END
```

They want a stable mapping table instead.

We will build:

* `order_status` → `status_group`

---

## A) Confirm which `order_status` values exist (before you seed)

Do this first.

You want your seed to cover the real values in the warehouse.

Run in Snowflake:

```sql
select
  order_status,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
group by order_status
order by nrows desc;
```

Keep this result open.

You will use it to confirm your seed covers every status.

---

## B) Create the seed CSV

### B1) Create the `seeds/` folder

From your project root:

```bash
mkdir -p seeds
```

### B2) Create the CSV

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

### B3) Confirm dbt can see the seed

Run:

```bash
dbt ls --resource-type seed
```

Expected output:

* you should see `order_status_map`

If you do not see it:

* confirm the file is under `seeds/`
* confirm the file ends in `.csv`

---

## C) Add tests for the seed

If a seed is reference data, you want it to be clean.

At minimum:

* the key should be unique
* nothing should be null

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

---

## D) Load the seed into Snowflake

Run:

```bash
dbt seed --select order_status_map
```

What dbt does (important mental model):

* first run: creates the table and inserts rows
* later runs: typically truncates the table and reinserts the CSV rows

You should assume `dbt seed` is the source of truth for this table.

If you manually edit the table in Snowflake, your next `dbt seed` will overwrite it.

---

## E) Verify the seed in Snowflake

### E1) Confirm the table exists

Run:

```sql
show tables like 'ORDER_STATUS_MAP' in schema OLIST.ANALYTICS_DEV;
```

### E2) Inspect the rows

Run:

```sql
select
  order_status,
  status_group
from OLIST.ANALYTICS_DEV.ORDER_STATUS_MAP
order by order_status;
```

---

## F) Test the seed

Run:

```bash
dbt test --select order_status_map
```

If tests fail, these are the usual causes:

* duplicate `order_status` rows in the CSV
* a blank cell in the CSV (treated as null)

Fix the CSV, then rerun:

```bash
dbt seed --select order_status_map
dbt test --select order_status_map
```

---

## G) Use the seed in a dbt model (this is the payoff)

A seed becomes a relation you can reference with `ref()`.

That means:

* you can join it
* you can build models on top of it
* you can treat it like any other dbt-managed table

We will build a small reporting model:

* orders grouped by `status_group`

### G1) Create the model

Create:

```bash
nano models/marts/rpt_orders_by_status_group.sql
```

Paste:

```sql
WITH orders AS (
    SELECT
        order_id,
        order_status
    FROM {{ ref('fct_orders') }}
),
status_map AS (
    SELECT
        order_status,
        status_group
    FROM {{ ref('order_status_map') }}
),
joined AS (
    SELECT
        o.order_id,
        o.order_status,
        m.status_group
    FROM orders o
    LEFT JOIN status_map m
        ON o.order_status = m.order_status
),
final AS (
    SELECT
        status_group,
        COUNT(*) AS order_count
    FROM joined
    GROUP BY status_group
)
SELECT
    status_group,
    order_count
FROM final
ORDER BY order_count DESC;
```

Save and exit.

Why we used a `LEFT JOIN`:

* if a new status appears and your seed does not include it
* you will see `status_group` as null
* that is your warning signal

### G2) Build the model

Run:

```bash
dbt run --select rpt_orders_by_status_group
```

### G3) Query the result in Snowflake

Run:

```sql
select
  status_group,
  order_count
from OLIST.ANALYTICS_DEV.RPT_ORDERS_BY_STATUS_GROUP
order by order_count desc;
```

### G4) Check for unmapped statuses

If your seed is missing any `order_status`, you will see a null group.

Run:

```sql
select
  order_status,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_status not in (
  select order_status from OLIST.ANALYTICS_DEV.ORDER_STATUS_MAP
)
group by order_status
order by nrows desc;
```

Expected outcome:

* 0 rows

If you get rows:

* add those statuses to the CSV
* rerun `dbt seed` and `dbt run`

---

## H) Prove you understand seed reload (controlled change)

This is how reference data actually changes in real projects.

Someone proposes a mapping change.

You review it in git.

You redeploy.

### H1) Change one mapping

Edit the CSV:

```bash
nano seeds/order_status_map.csv
```

Change this row:

```csv
unavailable,canceled
```

To:

```csv
unavailable,in_progress
```

Save and exit.

### H2) Reload the seed

Run:

```bash
dbt seed --select order_status_map
```

### H3) Rerun the reporting model

Run:

```bash
dbt run --select rpt_orders_by_status_group
```

### H4) Verify the change in Snowflake

Run:

```sql
select
  status_group,
  order_count
from OLIST.ANALYTICS_DEV.RPT_ORDERS_BY_STATUS_GROUP
order by order_count desc;
```

You should see counts shift between buckets.

---

## I) Common classroom failure: seed columns changed

If you change the **shape** of the CSV (add a column, rename a column), a normal `dbt seed` can error.

When that happens, force a rebuild:

```bash
dbt seed --select order_status_map --full-refresh
```

Use `--full-refresh` only when you changed columns.

If you only changed rows, a normal `dbt seed` is enough.

---

## Done criteria

You are done with this file when all succeed:

```bash
dbt seed --select order_status_map
```

```bash
dbt test --select order_status_map
```

```bash
dbt run --select rpt_orders_by_status_group
```

And in Snowflake:

* `OLIST.ANALYTICS_DEV.ORDER_STATUS_MAP` exists
* your reporting table has **no null status_group**

