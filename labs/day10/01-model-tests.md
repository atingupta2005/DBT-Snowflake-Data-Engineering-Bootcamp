# Day 10 Lab — 01 Model Tests

In this file you will add built-in dbt tests to **models**.

We will test:

* `dim_customers` (dimension)
* `fct_orders` (fact)

We will keep this focused.

You are not trying to “test everything”.

You are adding the minimum tests that protect downstream analytics from obvious breakage.

---

## What model tests protect you from

When model tests fail, it usually means one of these happened:

* Your transformation logic changed and produced duplicates
* A join unexpectedly multiplied rows
* A foreign key stopped matching (data drift)
* A domain/value list changed upstream

These problems do not always show up as query errors.

They show up as **wrong numbers** in dashboards.

That is why we test models.

---

## A) Add tests for two models

### A1) Create the marts tests YAML

Create this file:

```bash
nano models/marts/_marts_tests.yml
```

Paste the full YAML below.

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

---

## B) Parse before you run tests

Parsing catches YAML formatting mistakes and Jinja issues early.

Run:

```bash
dbt parse
```

If parsing fails:

* read the first error line
* fix the YAML indentation first
* run `dbt parse` again

Do not move on until parse succeeds.

---

## C) Run tests only for these two models

Run:

```bash
dbt test --select dim_customers fct_orders
```

This command does **not** run every test in the project.

It runs only the tests for the models you selected.

---

## D) Interpret failures (what to do next)

### D1) If `unique` fails

Example:

* `unique` fails on `fct_orders.order_id`

Meaning:

* your fact table has more than one row per order

Typical causes:

* a join multiplied rows
* a model grain changed without you noticing

First check duplicates directly in Snowflake:

```sql
select
  order_id,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
group by order_id
having count(*) > 1
order by nrows desc
limit 50;
```

If you see duplicate order_ids, stop and inspect the model logic.

---

### D2) If `relationships` fails

Example:

* `relationships` fails on `fct_orders.customer_id → dim_customers.customer_id`

Meaning:

* you have orders that do not match to any customer

The most common impact:

* analysts do an inner join to `dim_customers`
* those orders disappear
* order metrics drop silently

Pull the failing keys from Snowflake:

```sql
select
  f.customer_id,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS f
left join OLIST.ANALYTICS_DEV.DIM_CUSTOMERS d
  on f.customer_id = d.customer_id
where d.customer_id is null
group by f.customer_id
order by nrows desc
limit 50;
```

If you get rows:

* confirm `customer_id` type and formatting match
* check for leading/trailing whitespace in staging

---

### D3) If `accepted_values` fails

Example:

* `accepted_values` fails on `fct_orders.order_status`

Meaning:

* some `order_status` values are not in your allowed list

This often happens when:

* upstream introduces a new status
* you copied the list incorrectly

Check the actual values in Snowflake:

```sql
select
  order_status,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS
group by order_status
order by nrows desc;
```

Compare the results to your YAML list.

If you see a valid new status:

* add it to the YAML list
* rerun `dbt test --select fct_orders`

If you see a typo or unexpected value:

* keep the YAML strict
* fix upstream logic instead

---

## E) Rerun only what you changed

If you update only the YAML:

```bash
dbt parse
```

Then rerun model tests:

```bash
dbt test --select dim_customers fct_orders
```

---

## Done criteria

You are done with this file when:

```bash
dbt test --select dim_customers fct_orders
```

finishes with **0 failing tests**.
