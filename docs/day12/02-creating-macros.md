# 02 — Creating macros (and actually using them)

Macros are where Day 12 becomes useful.

A macro is a reusable function that returns SQL text.

You write it once.

Then you call it from models instead of copy-pasting logic.

## When you should create a macro

Use a macro when all of these are true:

* The logic is repeated in 2+ places.
* The logic is stable enough that you want one “official” version.
* The logic can be expressed as a SQL expression or small SQL fragment.

Do **not** create a macro when:

* It is used only once.
* It hides simple SQL (making the model harder to read).
* The macro becomes a “mini application” with too many branches.

We will talk more about over-engineering in instructor notes.

## Where macros live

Macros live under the `macros/` directory in a dbt project.

In this course, you will **use** macros and you will **write** one.

Important discipline:

* Macro files contain only macro SQL.
* Keep macro bodies short.
* Name macros after what they do, not how they do it.

## Macro syntax

A macro is defined with Jinja control tags:

```sql
{% macro macro_name(arg1, arg2) %}
  ... SQL text here ...
{% endmacro %}
```

A macro call uses the expression printing tags:

```sql
{{ macro_name('value1', 'value2') }}
```

You can call a macro anywhere SQL expects an expression.

## Macro example 1: `cents_to_dollars`

In the Olist dataset, monetary values are often stored in cents.

If we use them for reporting, we usually want dollars.

If you copy-paste this conversion everywhere:

```sql
payment_value / 100.0
```

You will eventually drift:

* some models will round
* some will not
* some will handle NULL
* some will not

We solve that by writing one macro.

### Macro definition

Example macro:

```sql
{% macro cents_to_dollars(cents_col, decimals=2) %}
  ROUND(
    COALESCE({{ cents_col }}, 0) / 100.0,
    {{ decimals }}
  )
{% endmacro %}
```

Read this carefully:

* `cents_col` is the SQL expression for the cents column.
* `decimals` has a default value of `2`.
* We use `COALESCE(..., 0)` so NULL becomes 0.
* We divide by `100.0` (float division).
* We round to a consistent decimal precision.

### Macro call inside a model

In a model SELECT list:

```sql
SELECT
  order_id,
  {{ cents_to_dollars('payment_value') }} AS payment_value_usd
FROM {{ ref('stg_payments') }}
```

dbt will compile that into a plain SQL expression.

### Passing non-column expressions

You can pass any SQL expression.

Example:

```sql
{{ cents_to_dollars('payment_value + 25') }}
```

This is powerful.

It is also dangerous.

Do not pass complex expressions unless you must.

Prefer passing a simple column reference.

## Macro example 2: `clean_string`

String cleanup is a common staging task.

The Olist dataset has typical quality issues:

* mixed casing
* leading/trailing whitespace
* empty strings that should be NULL

You will see copy-paste patterns like:

```sql
NULLIF(TRIM(UPPER(customer_city)), '')
```

That is a perfect macro candidate.

### Macro definition

```sql
{% macro clean_string(col_expr) %}
  NULLIF(
    TRIM(
      UPPER({{ col_expr }})
    ),
    ''
  )
{% endmacro %}
```

### Macro call

```sql
SELECT
  customer_id,
  {{ clean_string('customer_city') }} AS customer_city,
  {{ clean_string('customer_state') }} AS customer_state
FROM {{ ref('raw_customers') }}
```

Now all string cleanup follows one consistent rule.

If later you decide to also remove double spaces or normalize accents, you change one macro.

## Macro arguments and defaults

Defaults matter because they keep macro calls short.

Good:

```sql
{{ cents_to_dollars('payment_value') }}
```

Also allowed:

```sql
{{ cents_to_dollars('payment_value', 0) }}
```

Your rule:

* Use defaults for the common case.
* Override only when you have a real requirement.

## Calling macros correctly

### Strings vs expressions

This is where students usually get stuck.

If your macro expects a SQL expression, you usually pass it as a string.

Example:

```sql
{{ clean_string('customer_city') }}
```

Because inside the macro, we print it:

```sql
UPPER({{ col_expr }})
```

If you pass a column without quotes:

```sql
{{ clean_string(customer_city) }}
```

Jinja will treat `customer_city` as a Jinja variable.

Unless you defined it with `{% set customer_city = ... %}`, compilation fails.

### Practical rule

When you are passing a column name into a macro:

* pass it as a quoted string

Example:

```sql
{{ cents_to_dollars('payment_value') }}
```

Later, you can learn more advanced techniques like `adapter.quote()`.

We are not doing that today.

## Macro naming and review discipline

Macros can make projects clean.

Macros can also make projects unreadable.

Follow these rules:

* Keep macro names descriptive and specific.
* Avoid “generic helper” macros that do too much.
* Prefer a few small macros over one huge macro.
* Always verify the compiled SQL is readable.

In code review, the macro should make your intent clearer.

If reviewers have to open the macro file to understand the model, the macro is probably too clever.

## How to test your macro quickly

You do not need to run the full project to validate macro syntax.

Use compilation.

1. Add your macro.
2. Call it in one model.
3. Compile:

```bash
dbt compile
```

If compilation succeeds, open the compiled SQL and confirm:

* the macro expanded correctly
* parentheses are balanced
* commas are correct

Then you can run:

```bash
dbt run -s <model_name>
```

This is the fastest “edit → verify” loop.
