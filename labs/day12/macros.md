# Day 12 Lab — Macros

In this file you will create a reusable macro and refactor staging models to use it.

You will create:

* `clean_string(col, to_case='lower')`

Then you will apply it to:

* `stg_customers`
* `stg_orders`

You will keep column names unchanged so downstream models keep working.

---

## Why macros exist

A macro is a reusable SQL snippet.

Use a macro when:

* you repeat the same logic across multiple models
* the logic must be consistent
* you want one place to update it later

In real projects, these are classic macro targets:

* string cleanup (`trim`, `lower`, `upper`)
* safe casting patterns
* standard timestamp parsing

Macros are not magic.

A macro should stay small and predictable.

If a macro becomes a mini-language, you created a maintenance problem.

---

## A) Create the macro

### A1) Create the macros folder

From your project root:

```bash
mkdir -p macros
```

### A2) Create the macro file

Create:

```bash
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

Save and exit.

---

## B) Understand how to call the macro

In a model, you call it like a function.

Examples:

* lower + trim (default):

```sql
{{ clean_string('customer_city') }}
```

* upper + trim:

```sql
{{ clean_string('customer_state', 'upper') }}
```

Important detail:

* we pass a string like `'customer_city'`
* the macro injects it into the SQL expression

This works because the macro wraps the expression in `TRIM(...)` / `LOWER(...)`.

---

## C) Refactor `stg_customers` to use the macro

### C1) Open the file

```bash
nano models/staging/stg_customers.sql
```

### C2) Replace the entire file

Paste:

```sql
SELECT
  customer_id,
  customer_unique_id,
  CAST(customer_zip_code_prefix AS NUMBER) AS customer_zip_code_prefix,
  {{ clean_string('customer_city') }} AS customer_city,
  {{ clean_string('customer_state', 'upper') }} AS customer_state
FROM {{ source('olist', 'customers') }}
```

Save and exit.

What you changed:

* `customer_city` now uses `LOWER(TRIM(...))`
* `customer_state` now uses `UPPER(TRIM(...))`

What you did not change:

* column names
* source reference
* numeric casting

---

## D) Refactor `stg_orders` to use the macro

### D1) Open the file

```bash
nano models/staging/stg_orders.sql
```

### D2) Replace the entire file

Paste:

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

Save and exit.

---

## E) Parse (fail fast)

Run:

```bash
dbt parse
```

If this fails:

* confirm `macros/cleaning.sql` exists
* confirm the macro name is `clean_string`
* confirm your braces are balanced (`{{` and `}}`)

---

## F) Compile one staging model to see the macro expansion

Compile only `stg_orders`:

```bash
dbt compile --select stg_orders
```

Find the compiled file:

```bash
find target/compiled -type f -name "stg_orders.sql" -print
```

Open the file and confirm you see plain SQL:

* `LOWER(TRIM(order_status)) AS order_status`

There should be no `{{ clean_string(...) }}` left.

---

## G) Run only the changed staging models

Run:

```bash
dbt run --select stg_customers stg_orders
```

---

## H) Validate in Snowflake

### H1) Confirm state/city are normalized

Pick a few rows:

```sql
select
  customer_city,
  customer_state
from OLIST.ANALYTICS_DEV.STG_CUSTOMERS
limit 20;
```

What you expect:

* `customer_state` values should be uppercase (example: `SP`)
* `customer_city` values should be lowercase

### H2) Confirm order_status is normalized

```sql
select
  order_status,
  count(*) as nrows
from OLIST.ANALYTICS_DEV.STG_ORDERS
group by order_status
order by nrows desc;
```

You should not see whitespace differences or mixed casing.

---

## I) Common macro mistakes (classroom failures)

### I1) “My macro is not found”

Causes:

* the file is not under `macros/`
* the macro name in the file does not match what you called
* you have a syntax error in the macro, and dbt cannot load it

Fix:

* run `dbt parse`
* read the first macro error line carefully

### I2) “My compiled SQL has quotes around the column”

If you accidentally pass `customer_city` without quotes:

```sql
{{ clean_string(customer_city) }}
```

Jinja will treat it as a variable (not a string).

In this lab, pass column names as strings:

```sql
{{ clean_string('customer_city') }}
```

### I3) “Why are we using whitespace control?”

The `{%-` and `-%}` squiggles remove extra whitespace.

They do not change query meaning.

If you remove them, your SQL still works.

They just keep compiled SQL cleaner.

---

## Done criteria

You are done with this file when all succeed:

```bash
dbt parse
```

```bash
dbt run --select stg_customers stg_orders
```

And you confirmed in Snowflake:

* city/state values look normalized

---

## Next file

Next you will install `dbt-utils` and use a package macro to generate a surrogate key.

Go to `dbt-utils.md`.
