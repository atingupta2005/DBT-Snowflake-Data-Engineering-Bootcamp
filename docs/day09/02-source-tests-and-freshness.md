# Defining Sources in dbt

Up to Day 08, we built staging, intermediate, and mart models successfully.
We used `ref()` correctly.
We got full end-to-end runs.

We also did something that is common early in a project:
we treated the raw Olist tables as “just tables” and built on top of them.

That approach breaks down as soon as upstream data behaves badly.
In real pipelines, upstream data behaves badly all the time.

A **dbt source** is how we stop pretending raw data is stable.
It is how we make the raw boundary explicit.

## What a source is (and what it is not)

A **model** is built by dbt.

* It comes from SQL in your dbt project.
* dbt compiles it.
* dbt materializes it (view or table, depending on configuration).
* You reference it with `ref()`.

A **source** is not built by dbt.

* It already exists in the warehouse.
* It is loaded by an upstream ingestion process.
* dbt only reads it.
* You reference it with `source()`.

This is not just syntax.
It is a signal in the project.

When you open a dbt model and see:

```sql
FROM {{ source('olist', 'orders') }}
```

you immediately know:

* this dependency is upstream
* dbt cannot “fix” it by rerunning
* data quality and freshness issues must be handled at the boundary

When you see:

```sql
FROM {{ ref('stg_orders') }}
```

you know:

* this asset is owned by the dbt project
* failures are usually in your SQL or in prior dbt models

## Why sources exist

Sources solve three practical problems.
These are not theoretical.
They show up in production projects within weeks.

### 1) Make upstream dependencies visible

Without sources, raw tables are referenced directly.
That scatters upstream knowledge across many files.

When dependencies are scattered:

* reviews miss upstream assumptions
* debugging becomes guesswork
* it is hard to answer “what raw tables does this project rely on?”

Sources centralize that answer in one place.
They become a readable inventory of upstream inputs.

### 2) Create a contract for raw data

Raw data is messy.
Even when the schema is stable, the *meaning* of columns changes.

A source definition is a contract that says:

* this table is an official input
* these columns matter
* these keys must behave like keys

Later today, we attach tests and freshness rules to the same contract.
That is the point.
The contract is executable.

### 3) Catch upstream breakage early

Upstream failures often do not crash your dbt run.
They produce silent wrong outputs.

Examples you will see in real jobs:

* `orders` is late today, but yesterday’s data still exists → your marts look “normal” but are stale
* `customer_id` suddenly has NULLs → joins drop rows silently
* `order_items` contains duplicate `order_item_id` due to a reload bug → aggregates inflate silently

Sources give you a single place to attach early warning systems.

## Where sources live in the dbt project

Source definitions live in YAML files inside the dbt project.
Practically, that means: **somewhere under your `models/` directory**.

dbt discovers YAML configuration automatically.
You do not need to import it.

In this course, Day 09 work is grouped together.
In the lab, you will create a `source.yml` for the Olist raw tables.

Keep the file focused.
A single well-organized source YAML is easier to maintain than many tiny YAML files.

## The shape of `source.yml`

The structure is fixed.
If you get the shape right, dbt can parse it and build a source graph.

Minimum definition for the Olist raw tables:

```yaml
version: 2

sources:
  - name: olist
    schema: <RAW_SCHEMA_NAME>
    tables:
      - name: customers
      - name: orders
      - name: order_items
      - name: payments
      - name: products
```

### What each field means

* `version: 2`

  * Required. dbt expects version 2 schema files for sources, tests, and docs.

* `sources:`

  * A list. Each list item defines one source group.

* `name:`

  * The logical name used inside dbt.
  * This is what you will type in `source('name', 'table')`.

* `schema:`

  * Where the raw tables live in the warehouse.
  * This is the raw schema (or dataset) containing the tables.

* `tables:`

  * A list of the tables that belong to this source.

### Naming guidance you will thank yourself for later

Source names should be:

* short
* lowercase
* stable

Good:

* `olist`

Bad:

* `raw_olist_source_tables`
* `olist_data_2025_final`

Remember: you will type this in SQL many times.

## Organizing sources by schema and table

A source is a grouping mechanism.
In many companies, the grouping aligns to the upstream system.

Examples of real-world source groupings:

* `erp` for finance and inventory tables
* `crm` for sales pipeline tables
* `web` for clickstream data

In this course, the upstream system is “Olist raw”.
So we keep it simple: one source named `olist`.

## Adding table and column metadata

Source YAML is also where documentation lives.
This is not about writing essays.
It is about writing the minimum text that prevents confusion.

### Table descriptions

A good table description answers:

* what system the table represents
* what one row represents (the grain)

Example:

```yaml
- name: orders
  description: "Raw orders from the Olist source system. One row per order."
```

That is enough to prevent a common mistake:
people assuming the table is at customer-level or item-level.

### Column descriptions

Column descriptions matter most for:

* join keys (`*_id`)
* timestamps used for time windows
* categorical fields used for filtering

Avoid writing descriptions like:

* “The order id”
* “Timestamp of order”

Those do not help anyone.

Write descriptions you would want during an incident.
If ingestion breaks at 2 AM, you want quick clarity.

Example:

```yaml
- name: orders
  description: "Raw orders from the Olist source system. One row per order."
  columns:
    - name: order_id
      description: "Order primary key from the source system."
    - name: customer_id
      description: "Customer identifier associated with the order."
```

### A more complete Olist example (metadata only)

This example shows the pattern you will follow in the lab.
It documents key join columns without trying to document everything.

```yaml
version: 2

sources:
  - name: olist
    schema: <RAW_SCHEMA_NAME>
    tables:
      - name: customers
        description: "Raw customers from the Olist source system. One row per customer."
        columns:
          - name: customer_id
            description: "Customer primary key from the source system."

      - name: orders
        description: "Raw orders from the Olist source system. One row per order."
        columns:
          - name: order_id
            description: "Order primary key from the source system."
          - name: customer_id
            description: "Customer identifier associated with the order."

      - name: order_items
        description: "Raw line items for orders. One row per order item."
        columns:
          - name: order_id
            description: "Order identifier this line item belongs to."
          - name: product_id
            description: "Product identifier for the purchased item."

      - name: payments
        description: "Raw payment records for orders. One row per payment record."
        columns:
          - name: order_id
            description: "Order identifier this payment belongs to."

      - name: products
        description: "Raw products from the Olist source system. One row per product."
        columns:
          - name: product_id
            description: "Product primary key from the source system."
```

This is not the only valid way to document.
It is a practical baseline.

## Referencing sources in SQL

When you select from raw data inside a dbt model, use `source()`.

Example (staging model pattern):

```sql
SELECT
  o.order_id,
  o.customer_id
FROM {{ source('olist', 'orders') }} AS o
```

Compare with a reference to a dbt model using `ref()`:

```sql
SELECT
  s.order_id,
  s.customer_id
FROM {{ ref('stg_orders') }} AS s
```

### Why this discipline matters

In real projects, the most common debugging question is:
“Did the problem come from upstream raw data, or from our transformation?”

If your code mixes direct raw table references and `ref()` everywhere, the answer is slow.

If your code uses:

* `source()` for raw
* `ref()` for dbt models

the answer is fast.

## Seeing sources in the dbt graph

Once sources are defined, dbt treats them like graph nodes.
You can list them.

Useful commands:

```bash
dbt ls --resource-type source
```

That should print the sources dbt discovered.
If it prints nothing, dbt did not load your YAML.

## Real-world use cases where sources pay off

These are the patterns I see over and over.

### Use case: upstream key stops being stable

You build a staging model and assume `customer_id` is always present.
Then one day ingestion produces NULL customer IDs.

Without source definitions and tests:

* your staging model still runs
* joins drop orders
* your revenue numbers change silently

With sources:

* you define the contract at the boundary
* later today you add a `not_null` on the key
* failures show up before downstream models lie

### Use case: ingestion is delayed

A pipeline runs hourly.
It fails overnight.
At 9 AM your stakeholders open dashboards.

Without freshness checks:

* dbt runs successfully on yesterday’s data
* everyone sees stale dashboards and thinks they are current

With sources:

* you define a loaded-at timestamp
* you set warning/error thresholds
* dbt tells you the data is stale

### Use case: table moves schemas

A warehouse team reorganizes schemas.
`RAW.ORDERS` becomes `RAW_OLIST.ORDERS`.

Without sources:

* dozens of models contain hard-coded references
* fixing the project is painful

With sources:

* the schema mapping is centralized
* you update it once

(You still need to validate the change, but the blast radius is smaller.)

## Common YAML mistakes (and how to avoid them)

YAML is unforgiving.
Most classroom failures on Day 09 are YAML failures.

### Mistake: using tabs or inconsistent indentation

Use spaces.
Pick one indentation level (2 spaces is common) and stick to it.

### Mistake: forgetting that `sources` and `tables` are lists

This is wrong (missing dashes):

```yaml
sources:
  name: olist
```

Correct shape (list item with dash):

```yaml
sources:
  - name: olist
```

### Mistake: typos in table names

`tables:` entries must match the actual raw table names.
If you misspell one, dbt can still parse YAML, but references will fail later.

## Fast validation: make parsing your first checkpoint

After editing your source YAML, run this first:

```bash
dbt parse
```

Do not jump straight to `dbt run`.
If `dbt parse` fails, you have a YAML or project configuration problem.
Fix it before doing anything else.

In the lab, you will:

* define sources for the five Olist raw tables
* confirm dbt loads them cleanly
* then move on to source tests and freshness
