# 01 — Build Staging Models

Today we start modeling by building **staging models**.

Staging models sit directly on top of raw sources. Their job is to make raw tables usable and consistent.

In this file, we will do three things:

1. Define dbt sources for the raw Olist tables
2. Create staging models that clean up names and types
3. Run staging models and confirm they build correctly

## Rules for staging models

Staging models must stay boring.

Allowed:

* Renaming columns to consistent names
* Casting types (timestamps, numbers)
* Standardizing simple formats (trim, lower/upper if needed)
* Selecting only the columns you need downstream

Not allowed:

* Business logic
* Filtering to “valid” rows
* Aggregations
* Joins across tables

If you catch yourself answering “why” questions in staging, stop. That belongs later.

## 1) Define raw sources

We will point dbt at the existing raw Olist tables.

You will add sources for these tables only:

* customers
* orders
* order_items
* payments
* products

Your project should already have a `models/` folder. Under that folder, you should have a place for staging models. If the folder already exists, use it.

You will create a YAML file that defines the sources. Keep it minimal and readable.

Minimum requirement:

* One source name for the raw dataset
* One table entry per raw table

You will use these sources later via `source()`.

## 2) Create staging models

Each staging model should be one file.

Naming convention:

* `stg_customers`
* `stg_orders`
* `stg_order_items`
* `stg_payments`
* `stg_products`

Each model should follow this pattern:

* A single CTE that selects from `source()`
* A final `SELECT` with renamed and casted columns

Keep SQL clean and explicit:

* Uppercase SQL keywords
* No `SELECT *`
* Clear column aliases
* Minimal inline comments

### Example staging pattern

Use this structure for each staging model:

```sql
WITH source AS (

    SELECT
        ...
    FROM {{ source('...', '...') }}

),

final AS (

    SELECT
        ...
    FROM source

)

SELECT
    ...
FROM final
```

You do not need both CTEs if it feels redundant, but the flow should be readable.

### Type casting guidance

Raw CSV loads often produce loose types.

Use casts deliberately:

* IDs should be consistent (string vs numeric depends on raw schema)
* Dates/timestamps should be timestamps
* Numeric values should be numeric

Do not guess. Inspect your raw tables in Snowflake if you are unsure.

Common casting patterns you will use:

* `CAST(col AS VARCHAR)`
* `CAST(col AS NUMBER(38, 0))`

### Column naming guidance

Staging is where you make names consistent.

Use a predictable style:

* lowercase
* snake_case
* prefixes only if needed to avoid collisions later

Examples:

* `customer_id`
* `order_id`
* `product_id`
* `order_purchase_timestamp`

If raw columns contain mixed casing or awkward names, fix them here.

## 3) Run staging models

You should build staging models before touching intermediate or marts.

Run dbt for staging only

Expected outcome:

* dbt builds 5 models successfully
* The models appear in your target schema in Snowflake
* You can query them directly

If dbt fails, fix staging before moving on. Do not continue with broken upstream models.

## Sanity checks you should do

After your staging models run, pick one or two and validate quickly in Snowflake.

Things to check:

* Row counts are reasonable (not zero, not exploding)
* Key columns look populated
* Timestamps cast cleanly (not all NULL)

Keep this quick. We will do deeper validation later.

## What you need before moving on

Before you proceed to intermediate models, you must have:

* All five staging models building successfully
* No joins or business logic inside staging
* Clean naming and explicit columns

Once staging is stable, intermediate models become much easier.

Next: `02-build-intermediate-models.md`
