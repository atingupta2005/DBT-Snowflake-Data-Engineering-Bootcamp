# 03 — Seeds

Some data does not belong in the raw ingestion pipeline.

It is not coming from an operational system.

It is small, stable, and owned by the analytics team.

Examples:

* a mapping table from codes to readable names
* a list of allowed values with display ordering
* a small set of “business categories” used for reporting

In dbt, the simplest way to manage that kind of data is a **seed**.

A seed is:

* a CSV file stored inside your dbt project
* loaded into your warehouse by dbt

Seeds are not a replacement for proper source tables.

They are a tool for small reference datasets that you want version-controlled.

---

## Why seeds matter

In production, reference data causes constant friction:

* Someone keeps a mapping table in a spreadsheet.
* Another person copy/pastes it into SQL.
* Two dashboards use different versions of the mapping.

Seeds fix the workflow:

* the data lives in the repo
* changes go through code review
* environments stay consistent
* your project becomes reproducible

This is exactly the goal of a training repository:

* learners can run the same steps
* results are predictable

---

## When seeds are appropriate

Use seeds when the dataset is:

* small (typically tens to a few thousand rows)
* relatively stable
* safe to store as plaintext in the repo
* required by transformations or reporting

Good seed examples:

* status display order
* region normalization mapping
* category grouping mapping

In this course, we will use a small reference dataset that helps reporting without introducing new raw data.

---

## When seeds are NOT appropriate

Do not use a seed when:

* the data is large
* it changes frequently
* it is sensitive
* it must come from a source system

Bad seed examples:

* full customer lists
* transactional facts
* anything with personal data

In other words:

* seeds are not an ingestion tool

---

## Seeds vs sources

It is easy to confuse seeds with sources.

They solve different problems.

### Sources

* come from upstream systems
* live in a raw schema
* can be large
* change continuously
* are validated with freshness and source tests

### Seeds

* are owned by the analytics/dbt project
* live in your repository as CSV
* are loaded on demand
* are usually small and controlled

A good mental model:

* sources are “data we receive”
* seeds are “data we define”

---

## How dbt loads seeds

Seeds are CSV files stored under the project `seeds/` directory.

When you run:

```bash
dbt seed
```

dbt will:

1. Create or replace a table in your target schema (depending on config)
2. Load the CSV contents into that table

Then you can reference the seed in models using:

* `ref()`

Example:

```sql
SELECT
  status,
  status_group
FROM {{ ref('order_status_map') }}
```

The seed name is the CSV filename (without `.csv`).

---

## Seed configuration basics

Seeds can be configured in `dbt_project.yml`.

Common configuration points:

* schema
* quoting
* column types

We will keep configuration minimal today.

On slow laptops, the biggest thing you care about is:

* making sure the seed loads cleanly
* making sure joins to the seed work

If types are wrong, you will see it quickly when you join.

---

## A realistic seed for Olist: order status grouping

Olist has an `order_status` field.

Analysts often want “grouped” statuses:

* completed
* in_progress
* canceled

This grouping is not raw data.

It is a reporting decision.

That makes it a good seed.

### Example mapping

Imagine a small mapping like:

| order_status | status_group |
| -----------: | -----------: |
|    delivered |    completed |
|      shipped |  in_progress |
|     canceled |     canceled |

That mapping lets you:

* standardize dashboards
* keep logic out of ad-hoc SQL
* avoid repeating CASE statements in every model

### Why not just use a CASE statement?

You can.

But CASE statements scattered across models are hard to maintain.

A seed makes this explicit:

* the mapping lives in one place
* changes are reviewed
* downstream models stay clean

---

## How seeds show up in the warehouse

After you run `dbt seed`, you will see a table in your target schema.

The exact name depends on your project configuration, but the key behavior is:

* the table exists like any other table

You can query it directly.

That is useful for debugging.

Example:

```sql
SELECT
  order_status,
  status_group
FROM <your_schema>.order_status_map
ORDER BY order_status
```

---

## Testing seeds

Seeds can be tested like any other model.

Two common tests:

* `unique` on the seed key
* `not_null` on required columns

For a status mapping, you typically want:

* `order_status` unique
* `order_status` not null
* `status_group` not null

If the mapping is wrong, you want to know immediately.

---

## Common classroom mistakes

You will see these problems when people first use seeds.

### Mistake 1: Forgetting to run `dbt seed`

They create the CSV and add joins, then the model fails because the seed table does not exist.

Fix:

* run `dbt seed`

### Mistake 2: Mismatched values

They map `canceled` but the data contains `cancelled`.

Fix:

* confirm actual values in the warehouse
* update the mapping
* consider adding an `accepted_values` test on the raw/model column

### Mistake 3: Incorrect grain

They try to store something that needs many-to-many relationships.

Fix:

* seeds should typically map one key to one attribute

---

## What you will do in the lab

In `labs/day10/README.md`, you will:

* add a small seed CSV for reference data
* load it with `dbt seed`
* use it in a model join (or validate it exists)
* add tests to the seed

Proceed to `04-snapshots.md` when you are ready.
