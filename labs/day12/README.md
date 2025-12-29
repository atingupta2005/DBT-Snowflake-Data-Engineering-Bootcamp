# Day 12 Lab — Jinja, Macros, dbt-utils, and Hooks

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## 0) Confirm you finished Day 11

From your project root:

```bash
cd ~/dbt_olist_project
source ~/.venv/bin/activate
```

These must work before you start Day 12:

```bash
dbt run --select fct_orders
```

```bash
dbt test
```

If either fails, fix Day 11 first.

---

## 1) Save a git checkpoint (do this before you change anything)

Today you will introduce Jinja and macros. A small syntax mistake can break parsing.

Checkpoint first so you can see exactly what changed.

```bash
git status
```

If you see `not a git repository`:

```bash
git init
```

Commit your Day 11 state:

```bash
git add -A
git commit -m "day11 checkpoint before day12 jinja macros hooks"
```

If git says “nothing to commit”, that is fine.

---

## 2) Add dbt-utils (package install)

We will install `dbt-utils` so you can call standard macros like `generate_surrogate_key`.

### 2A) Create `packages.yml` (copy/paste)

From the dbt project root (same folder as `dbt_project.yml`), create:

```bash
nano packages.yml
```

Paste:

```yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
```

Save and exit.

### 2B) Download packages

```bash
dbt deps
```

Expected output (brief): dbt downloads packages into `dbt_packages/`.

---

## 3) Create a reusable macro: `clean_string`

We already repeat patterns like `LOWER(TRIM(col))` across staging models.

We will make that a macro.

### 3A) Create a macros file (copy/paste)

```bash
mkdir -p macros
nano macros/cleaning.sql
```

Paste:

```sql
{% macro clean_string(col, to_case='lower') -%}
  {%- if to_case == 'upper' -%}
    UPPER(TRIM({{ col }}))
  {%- elif to_case == 'lower' -%}
    LOWER(TRIM({{ col }}))
  {%- else -%}
    TRIM({{ col }})
  {%- endif -%}
{%- endmacro %}
```

Notes you should notice:

* `{%-` and `-%}` remove extra whitespace in compiled SQL.
* `col` is the column expression you pass in.

---

## 4) Refactor staging models to use the macro

You are going to replace only string-cleaning expressions.

Do not change column names. Downstream models expect them.

### 4A) Replace `stg_customers.sql` (copy/paste)

```bash
nano models/staging/stg_customers.sql
```

Replace the entire file with:

```sql
SELECT
  customer_id,
  customer_unique_id,
  CAST(customer_zip_code_prefix AS NUMBER) AS customer_zip_code_prefix,
  {{ clean_string('customer_city') }} AS customer_city,
  {{ clean_string('customer_state', 'upper') }} AS customer_state
FROM {{ source('olist', 'customers') }}
```

### 4B) Replace `stg_orders.sql` (copy/paste)

```bash
nano models/staging/stg_orders.sql
```

Replace the entire file with:

```sql
SELECT
  order_id,
  customer_id,
  {{ clean_string('order_status') }} AS order_status,
  CAST(order_purchase_timestamp AS TIMESTAMP) AS order_purchase_ts,
  CAST(order_approved_at AS TIMESTAMP) AS order_approved_ts,
  CAST(order_delivered_carrier_date AS TIMESTAMP) AS order_delivered_carrier_ts,
  CAST(order_delivered_customer_date AS TIMESTAMP) AS order_delivered_customer_ts,
  CAST(order_estimated_delivery_date AS DATE) AS order_estimated_delivery_date
FROM {{ source('olist', 'orders') }}
```

### 4C) Replace `stg_products.sql` using a Jinja loop (copy/paste)

This is the DRY part.

We will generate repeated `CAST(... AS NUMBER) AS ...` columns using a loop.

```bash
nano models/staging/stg_products.sql
```

Replace the entire file with:

```sql
{% set numeric_cols = [
  ('product_name_lenght', 'product_name_length'),
  ('product_description_lenght', 'product_description_length'),
  ('product_photos_qty', 'product_photos_qty'),
  ('product_weight_g', 'product_weight_g'),
  ('product_length_cm', 'product_length_cm'),
  ('product_height_cm', 'product_height_cm'),
  ('product_width_cm', 'product_width_cm')
] %}

SELECT
  product_id,
  {{ clean_string('product_category_name') }} AS product_category_name,

  {%- for src, alias in numeric_cols %}
  CAST({{ src }} AS NUMBER) AS {{ alias }}{%- if not loop.last %},{% endif %}
  {%- endfor %}
FROM {{ source('olist', 'products') }}
```

What to notice:

* The loop generates each numeric cast.
* The comma logic is controlled by `loop.last`.
* `product_name_lenght` is misspelled in the raw data. We keep reading it as-is, but we alias to `product_name_length`.

---

## 5) Use a dbt-utils macro inside a model

We will add a generated key to `fct_orders` using `dbt_utils.generate_surrogate_key`.

This is a standard pattern when you want a stable key for joins.

### 5A) Edit `fct_orders.sql` (small change)

Open:

```bash
nano models/marts/fct_orders.sql
```

Find the final `SELECT` list (near the bottom) and add this column:

```sql
  {{ dbt_utils.generate_surrogate_key(['order_id', 'customer_id']) }} AS order_customer_key,
```

Place it after `customer_id` so it is easy to find.

Do not remove any existing columns.

---

## 6) Add a post-hook to grant permissions

We will add a post-hook on `fct_orders`.

A post-hook runs after dbt builds the model.

### 6A) Update the config block (copy/paste)

At the very top of `models/marts/fct_orders.sql`, replace your config block with this (keep it as the first thing in the file):

```sql
{{
  config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='order_id',
    on_schema_change='sync_all_columns',
    post_hook=["GRANT SELECT ON {{ this }} TO ROLE OLIST_ROLE"]
  )
}}
```

Notes:

* This assumes your role has permission to grant SELECT on objects it owns in the target schema.
* If this fails, the build can still succeed but the post-hook will error. Read the error message.

---

## 7) Parse, run, and inspect

### 7A) Parse first

```bash
dbt parse
```

If `dbt parse` fails:

* check braces in Jinja blocks
* check you did not delete `{% endif %}` / `{% endfor %}`

### 7B) Run the changed models

```bash
dbt run --select stg_customers stg_orders stg_products fct_orders
```

### 7C) Confirm the generated key exists

```sql
select
  order_id,
  customer_id,
  order_customer_key
from OLIST.ANALYTICS_DEV.FCT_ORDERS
limit 10;
```

### 7D) Confirm the GRANT ran (quick permission check)

If you have another role/user available in class, try selecting from the table using that role.

If you do not, at minimum confirm the table still exists and is queryable:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

---

## 8) Debugging Jinja when something breaks

When Jinja breaks, the fastest path is to inspect compiled SQL.

Compile only the product staging model:

```bash
dbt compile --select stg_products
```

Then open the compiled file under `target/compiled/`.

Find it quickly:

```bash
find target/compiled -type f -name "stg_products.sql" -print
```

Open the printed path and inspect what your loop produced.

---

## 9) Compare changes and commit Day 12

Review the diff:

```bash
git diff
```

Commit:

```bash
git add -A
git commit -m "day12 add clean_string macro jinja loop dbt-utils and post-hook"
```

---

## Checkpoints

You are done when all of these are true:

1. `clean_string` macro exists under `macros/` and is used in at least two staging models
2. `stg_products` uses a Jinja loop to generate repeated casts
3. `dbt_utils.generate_surrogate_key` is used in `fct_orders`
4. `fct_orders` has a post-hook that attempts a GRANT
5. `dbt parse` and `dbt run --select stg_customers stg_orders stg_products fct_orders` both succeed
