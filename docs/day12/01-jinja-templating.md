# 01 — Jinja basics in dbt

Today we start using Jinja inside dbt models.

Jinja is a templating language.

In dbt, it runs **before** your SQL runs.

That means:

* Jinja generates SQL text.
* dbt compiles that SQL.
* Snowflake executes the compiled SQL.

If Jinja fails, dbt never reaches Snowflake.

## Where you will see Jinja

You will see Jinja in three places in a dbt project:

1. **Models** (`models/…/*.sql`)

   * Used to generate repeated SQL fragments.

2. **Macros** (`macros/*.sql`)

   * Used to define reusable SQL snippets.

3. **Project config** (`dbt_project.yml`)

   * Used to attach hooks (pre/post).

Today we focus on Jinja inside models and macros.

## Jinja delimiters you must recognize

dbt uses standard Jinja syntax.

### 1) Expression printing: `{{ ... }}`

Use this when you want Jinja to output text into your SQL.

Example:

```sql
SELECT
  {{ 1 + 1 }} AS two
```

dbt compiles that to:

```sql
SELECT
  2 AS two
```

In real dbt work, we do not print arithmetic.

We print:

* column names
* SQL expressions
* macro calls

Example (macro call):

```sql
SELECT
  {{ cents_to_dollars('payment_value') }} AS payment_value_usd
FROM {{ ref('stg_payments') }}
```

### 2) Control blocks: `{% ... %}`

Use this for logic:

* loops
* if/else
* set variables

These blocks do not output text directly.

They control what gets output.

Example:

```sql
{% if true %}
SELECT 1 AS value
{% endif %}
```

### 3) Comments: `{# ... #}`

This is a Jinja comment.

It is removed during compilation.

Example:

```sql
{# this will not appear in compiled SQL #}
SELECT 1
```

Do not use Jinja comments as “documentation.”

If the SQL needs explanation, put it in `docs/`.

## Variables with `{% set %}`

You can define a Jinja variable.

This is useful when:

* the same string appears multiple times
* the same list of columns is reused

Example:

```sql
{% set cents_columns = ['payment_value'] %}

SELECT
  {% for col in cents_columns %}
  {{ col }}
  {% endfor %}
FROM {{ ref('stg_payments') }}
```

That example is not very helpful yet.

But it shows the shape:

* set a list
* loop over it

### Realistic variable use case: standardized column list

Imagine you have a model that selects a known set of columns.

You want to reuse that set across two staging models.

You can store the column list in a variable and loop.

Example:

```sql
{% set customer_cols = [
  'customer_id',
  'customer_unique_id',
  'customer_zip_code_prefix',
  'customer_city',
  'customer_state'
] %}

SELECT
  {% for col in customer_cols %}
  {{ col }}{% if not loop.last %},{% endif %}
  {% endfor %}
FROM {{ ref('raw_customers') }}
```

Important details:

* `loop.last` tells you whether you are on the final element.
* We use it to control commas.

If you forget this, you will generate invalid SQL.

## For loops

A loop is the main tool for removing repeated column expressions.

### Common case: generating repeated expressions

Suppose you want to apply the same function to multiple columns.

Example goal:

* trim whitespace
* convert to uppercase
* replace empty strings with NULL

You do not want to copy-paste the expression five times.

Instead, you loop.

Example:

```sql
{% set cols_to_clean = ['customer_city', 'customer_state'] %}

SELECT
  customer_id,
  {% for col in cols_to_clean %}
  NULLIF(TRIM(UPPER({{ col }})), '') AS {{ col }}{% if not loop.last %},{% endif %}
  {% endfor %}
FROM {{ ref('raw_customers') }}
```

The compiled SQL becomes:

```sql
SELECT
  customer_id,
  NULLIF(TRIM(UPPER(customer_city)), '') AS customer_city,
  NULLIF(TRIM(UPPER(customer_state)), '') AS customer_state
FROM ...
```

This is readable and predictable.

### Loop discipline

When you use loops, follow these rules:

* Keep the loop short.
* Keep the generated SQL human-readable.
* Avoid generating 50+ columns with complex logic.

If the compiled SQL is unreadable, you have overdone it.

## If / else logic

Use if/else when you want optional behavior.

### Example: optional filtering

Sometimes you want a model to filter only during development.

dbt provides the `target` variable.

In a real team project you might do something like:

```sql
SELECT
  order_id,
  customer_id,
  order_purchase_timestamp
FROM {{ ref('stg_orders') }}

{% if target.name != 'prod' %}
WHERE order_purchase_timestamp >= DATEADD(day, -7, CURRENT_DATE)
{% endif %}
```

What this means:

* In non-prod targets, you only query recent data.
* In prod, you query everything.

We are not going to use this pattern heavily in class.

But you must recognize it when you see it.

## Whitespace control: `{{-` and `-}}`

Jinja normally preserves whitespace around blocks.

That can lead to awkward formatting in compiled SQL.

Whitespace control lets you strip whitespace.

### The squiggles

* `{{-` trims whitespace to the left
* `-}}` trims whitespace to the right
* `{%-` and `-%}` do the same for control blocks

### Why you care

If you generate SQL inside loops, whitespace can create:

* blank lines
* indentation drift
* extra spaces before commas

Snowflake usually ignores whitespace.

Humans do not.

When you debug compiled SQL, messy whitespace slows you down.

### Example: trimming whitespace around a loop

Compare these.

#### Without trimming

```sql
SELECT
  {% for col in ['customer_city', 'customer_state'] %}
  {{ col }}{% if not loop.last %},{% endif %}
  {% endfor %}
FROM {{ ref('raw_customers') }}
```

This often compiles with extra newlines and indentation.

#### With trimming

```sql
SELECT
  {%- for col in ['customer_city', 'customer_state'] -%}
  {{ col }}{% if not loop.last %},{% endif %}
  {%- endfor %}
FROM {{ ref('raw_customers') }}
```

Now Jinja strips whitespace around the loop tags.

The compiled SQL is typically cleaner.

### Practical rule

Use whitespace trimming when:

* you are looping inside a SELECT list
* you are generating comma-separated output

Do not obsess over it.

The goal is:

* compiled SQL is readable
* the model is easy to review

## How you debug Jinja problems

When Jinja breaks, do this in order:

1. Run compilation only:

```bash
dbt compile
```

2. Find the compiled SQL.

Compiled files are written under:

* `target/compiled/…`

3. Open the compiled file for your model and read it.

You are checking:

* Did the loop generate valid SQL?
* Did you accidentally generate a trailing comma?
* Did you reference a column name incorrectly?

4. Fix the Jinja.

Do not guess.

Small syntax errors are common:

* missing `{% endif %}`
* missing quotes around a string
* using `{{ }}` when you meant `{% %}`

If you build the habit of compiling early, you avoid long debugging sessions.
