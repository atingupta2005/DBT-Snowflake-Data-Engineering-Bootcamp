# Day 12 Lab — Jinja

In this file you will use Jinja to generate repetitive SQL safely.

You will do this in one place:

* `models/staging/stg_products.sql`

This is the fastest way to understand Jinja because you can:

* compile it
* inspect the compiled SQL
* confirm it matches what you intended

---

## Why we use Jinja in dbt

Jinja is not here to be clever.

It is here to reduce copy-paste and make SQL easier to maintain.

In real projects, copy-paste causes:

* inconsistent casting
* inconsistent naming
* missed updates (you change one file and forget the other)

For staging models, a very common pattern is:

* many columns need the same type casting
* a few columns need the same cleaning logic

Jinja helps you standardize that.

---

## Rules for Day 12

Follow these rules while you work:

* Keep the generated SQL readable.
* Use Jinja only where it removes repetition.
* If a loop saves you 3 lines, it is not worth it.
* If a loop saves you 30 lines, it is worth it.

---

## A) The Jinja tools you will use

### A1) `set`

You can define a variable.

In this lab, we will define a list of tuples.

Each tuple is:

* raw column name
* alias column name

Example:

```jinja
{% set cols = [
  ('raw_col', 'clean_col'),
  ('raw_col_2', 'clean_col_2')
] %}
```

### A2) `for` loop

You can loop over the tuples.

Example:

```jinja
{%- for src, alias in cols %}
CAST({{ src }} AS NUMBER) AS {{ alias }}
{%- endfor %}
```

### A3) `loop.last`

When you generate comma-separated SQL, you must handle commas correctly.

The clean pattern is:

* emit a comma for every row except the last

Example:

```jinja
{%- if not loop.last %},{% endif %}
```

### A4) Whitespace control (`{%-` and `-%}`)

Jinja normally preserves newlines and spaces.

That can create ugly compiled SQL.

Using the hyphen strips whitespace.

You will use it only around loops.

Do not obsess over whitespace.

Just make the compiled SQL readable.

---

## B) Edit `stg_products.sql` to use a loop

We will generate repeated numeric casts.

We will also fix a known raw typo using aliasing.

Raw data contains:

* `product_name_lenght` (misspelled)
* `product_description_lenght` (misspelled)

We will keep reading the raw columns as-is, but alias them correctly.

### B1) Open the file

```bash
nano models/staging/stg_products.sql
```

### B2) Replace the entire file

Paste this content.

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
  product_category_name,

  {%- for src, alias in numeric_cols %}
  CAST({{ src }} AS NUMBER) AS {{ alias }}{%- if not loop.last %},{% endif %}
  {%- endfor %}
FROM {{ source('olist', 'products') }}
```

Save and exit.

### B3) What you should notice

* The loop generates **seven** cast expressions.
* The comma logic prevents a trailing comma.
* The aliases fix the two misspelled column names.

---

## C) Parse (fail fast)

Run:

```bash
dbt parse
```

If parsing fails, it is almost always one of these:

* missing `{% endfor %}`
* missing `%}`
* a stray `{` or `}`

Fix parse before you run anything.

---

## D) Compile and inspect the generated SQL

This is the most important habit for Day 12.

When Jinja breaks, the fastest path is:

* compile
* inspect compiled SQL

### D1) Compile only this model

```bash
dbt compile --select stg_products
```

### D2) Find the compiled file

```bash
find target/compiled -type f -name "stg_products.sql" -print
```

You will get a path like:

* `target/compiled/<project_name>/models/staging/stg_products.sql`

### D3) Open the compiled file

Open it with `nano` using the printed path.

You should see real SQL with seven `CAST(... AS NUMBER)` lines.

You should **not** see any Jinja blocks.

---

## E) Run the model and validate it exists

Run:

```bash
dbt run --select stg_products
```

Now check the table in Snowflake.

Quick row count:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.STG_PRODUCTS;
```

Optional: check the corrected alias columns exist:

```sql
select
  product_name_length,
  product_description_length
from OLIST.ANALYTICS_DEV.STG_PRODUCTS
limit 10;
```

---

## F) A common mistake (and how to spot it)

### Mistake: forgetting the comma logic

If your loop emits commas incorrectly, compiled SQL may look like:

* missing commas (SQL syntax error)
* trailing comma before `FROM` (SQL syntax error)

That will fail at runtime.

The fix is always:

* use `loop.last` to control the comma

---

## Done criteria

You are done with this file when all succeed:

```bash
dbt parse
```

```bash
dbt compile --select stg_products
```

```bash
dbt run --select stg_products
```

And you confirmed in Snowflake:

* `OLIST.ANALYTICS_DEV.STG_PRODUCTS` exists

---

## Next file

Next you will create your first custom macro and refactor staging models to use it.

Go to `macros.md`.
