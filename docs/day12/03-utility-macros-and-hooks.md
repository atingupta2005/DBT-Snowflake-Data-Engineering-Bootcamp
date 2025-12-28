# 03 — Utility macros, packages, and hooks

On Day 12 you learn two patterns that show up in real dbt repos:

* **utility macros** that standardize common transformations
* **hooks** that attach operational behavior to a model

We will keep both practical and small.

## Utility macros: why teams standardize

In production, teams want consistency.

Two people should not implement “date standardization” or “string cleanup” in two different ways.

Utility macros create a single rule.

This matters when:

* reporting definitions must be stable
* code review is strict
* many models repeat the same small transformations

A good utility macro is:

* short
* obvious
* easy to inspect in compiled SQL

## Using a package: `dbt-utils` (briefly)

Most dbt projects depend on at least one package.

`dbt-utils` is one of the most common.

It provides standard macros like:

* `surrogate_key` style utilities (package macros vary by version)
* `date_spine`
* tests and helpers that save time

We are not going to explore the whole package.

Today you only need to understand:

* how packages are installed
* how to call a macro from a package

### Installing packages

Packages are defined in `packages.yml`.

A minimal example:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.2.0
```

Then you install them with:

```bash
dbt deps
```

That downloads the package into `dbt_packages/`.

In many corporate environments:

* internet access is restricted
* package installs may be pre-baked

If `dbt deps` fails due to network restrictions, your instructor will provide the prepared environment.

You still need to understand how the call works.

### Calling a package macro

Package macros are namespaced by package name.

Example:

```sql
{{ dbt_utils.generate_surrogate_key(['order_id', 'product_id']) }}
```

That expands into SQL that generates a stable key.

Why teams like this:

* everyone uses the same definition of “surrogate key”
* less chance of subtle differences across models

Important note:

* Macro names and behavior can change across versions.
* In real work, always check your installed package version.

## Standardizing dates and column names

The biggest value of utility macros is removing repeated cleanup logic.

### Standardizing dates

You often want to enforce a consistent pattern:

* date is a DATE (not timestamp)
* time zone conversions happen upstream
* naming uses `_date` suffix

Example staging rule:

* `order_purchase_timestamp` is a timestamp
* we also want `order_purchase_date` as a date

In a model, you might do:

```sql
SELECT
  order_id,
  CAST(order_purchase_timestamp AS DATE) AS order_purchase_date,
  order_purchase_timestamp
FROM {{ ref('raw_orders') }}
```

If that rule repeats everywhere, it becomes a macro candidate.

But be careful.

A macro like `to_date()` can be too generic and hide logic.

A better approach:

* keep date casting in the model unless it repeats heavily
* if you macro it, keep it very small

Example:

```sql
{% macro cast_to_date(ts_expr) %}
  CAST({{ ts_expr }} AS DATE)
{% endmacro %}
```

Then in a model:

```sql
{{ cast_to_date('order_purchase_timestamp') }} AS order_purchase_date
```

Whether you macro this depends on repetition and team preferences.

Today’s lab focuses on the string and money conversion examples because they are clearer.

### Standardizing column names

Teams often standardize:

* casing
* trimming
* mapping empty strings to NULL

That is why `clean_string` is a common macro.

If you standardize column naming conventions, do it via:

* consistent SELECT aliases
* or dbt model conventions

Do not try to “auto-rename everything” using loops.

That makes compiled SQL hard to review.

## Generating SQL keys dynamically

You often need a key that represents a unique grain.

Example grains:

* `orders`: one row per order (`order_id`)
* `order_items`: one row per (order_id, product_id, order_item_id)

If a downstream model needs a single-column key, you might generate a surrogate key.

### Using dbt-utils for keys

A common approach is a utility macro.

Example:

```sql
SELECT
  {{ dbt_utils.generate_surrogate_key(['order_id', 'product_id']) }} AS order_product_key,
  order_id,
  product_id
FROM {{ ref('stg_order_items') }}
```

This keeps the key generation consistent.

If the team changes the key logic, they do it once (package or wrapper macro).

### Wrapper macro pattern

In many teams, you wrap package macros.

Why?

* you control the interface
* you can swap packages later
* you standardize behavior

Example wrapper macro:

```sql
{% macro order_item_key() %}
  {{ dbt_utils.generate_surrogate_key(['order_id', 'product_id', 'order_item_id']) }}
{% endmacro %}
```

Then in models:

```sql
{{ order_item_key() }} AS order_item_key
```

This can be useful.

But again: do not overdo it.

If every key becomes a wrapper macro, people stop understanding the data grain.

## Hooks: pre-hook and post-hook

Hooks are dbt configuration that runs SQL **before** or **after** a model.

* **pre-hook**: runs before the model builds
* **post-hook**: runs after the model builds

Hooks are useful for operational tasks.

They can also create invisible behavior.

So we use them carefully.

## Post-hook example: granting permissions

A real operational task is granting access to a reporting role.

In Snowflake, that might look like:

```sql
GRANT SELECT ON TABLE <schema>.<table> TO ROLE <role_name>;
```

In dbt, you can attach that as a post-hook so it always runs after the model is built.

### Where the hook lives

Hooks can be defined:

* on a model (in the model config)
* in `dbt_project.yml` for a group of models

Today we focus on a simple model-level hook.

### Model config example

At the top of a model:

```sql
{{
  config(
    post_hook=[
      "GRANT SELECT ON TABLE {{ this }} TO ROLE ANALYST"
    ]
  )
}}

SELECT ...
```

Read the hook string carefully:

* It is a string, so the outer quotes matter.
* Inside the string, `{{ this }}` resolves to the fully qualified relation.

In Snowflake, `{{ this }}` typically becomes:

* `DATABASE.SCHEMA.MODEL_NAME`

So you do not hardcode the schema or database.

That is why hooks are useful.

### Practical constraints for class

* The role name must exist.
* In training environments, you may not have privileges to grant.

If grants fail in your environment, you still learn the pattern.

The instructor will explain the permission setup.

## Pre-hook example: audit logging

Pre-hooks are often used for:

* deleting temporary data
* inserting audit records
* setting session variables

A simple audit pre-hook might insert a record into an audit table.

We will not implement full audit tables today.

But you need to understand the shape.

Example:

```sql
{{
  config(
    pre_hook=[
      "INSERT INTO audit_log(model_name, run_ts) VALUES ('{{ this.name }}', CURRENT_TIMESTAMP)"
    ]
  )
}}

SELECT ...
```

In a real project:

* `audit_log` must exist
* the dbt role must have INSERT privileges

Because of those requirements, audit logging is usually handled by platform teams.

Today we treat it as a pattern, not a full implementation.

## Hook debugging discipline

Hooks fail like normal SQL.

If a hook fails:

* dbt marks the model run as failed
* downstream models may not run

When a hook breaks, you debug it like this:

1. Compile the project:

```bash
dbt compile
```

2. Inspect the compiled SQL.

3. Look for the hook SQL in the run logs.

4. If needed, copy the hook SQL and run it directly in Snowflake.

Hooks are “just SQL.”

Treat them like production SQL.

## How to decide: macro, package, or plain SQL

Use plain SQL when:

* the logic is short
* it appears once
* the model remains readable

Use a macro when:

* logic repeats
* consistency matters
* you can keep the macro short

Use a package macro when:

* the problem is common (keys, date spine)
* you do not want to maintain your own implementation

And always remember:

* compiled SQL is the truth
* if compiled SQL is messy, reduce the cleverness
