# Day 12 Lab — Jinja, macros, and DRY patterns

Today’s lab has three tasks.

You will:

1. Refactor repeated logic into a reusable macro.
2. Use a Jinja loop to generate repeated column cleanup expressions.
3. Add a post-hook to grant permissions on a model.

## Before you start

You will be editing dbt code.

Do not start by running the full project.

Use this workflow:

1. Make a small change
2. Compile
3. Inspect compiled SQL if needed
4. Run a single model

### Commands you will use

From the dbt project directory:

```bash
dbt deps
```

```bash
dbt compile
```

```bash
dbt run -s <model_name>
```

If you do not know model names yet, list them:

```bash
dbt ls
```

## Task 1 — Refactor repeated logic into a macro

### Goal

You will create a macro named **either**:

* `cents_to_dollars`

**or**

* `clean_string`

Then you will refactor one staging model to call the macro.

### What you are refactoring

Pick one pattern you have already used in earlier days.

Common options:

1. Money conversion

* Values stored in cents
* You convert to dollars with `/ 100.0`

2. String cleanup

* You trim
* you uppercase
* you convert empty strings to NULL

### Steps

1. Create a macro definition.

You will place it under the existing `macros/` directory.

Your macro must:

* accept a column expression as an argument
* return a SQL expression

2. Refactor one staging model.

Replace the repeated logic with a macro call.

Example pattern (do not copy this blindly):

```sql
{{ clean_string('customer_city') }} AS customer_city
```

3. Compile to validate syntax.

```bash
dbt compile
```

Expected result:

* compilation succeeds

If compilation fails:

* fix the Jinja syntax first
* do not run `dbt run` yet

4. Run only the model you changed.

```bash
dbt run -s <model_name>
```

Expected result:

* model runs successfully

### Checks

* The compiled SQL should contain plain SQL, not Jinja.
* The macro expansion should be readable.

If your macro call generates confusing SQL, simplify the macro.

## Task 2 — Use a Jinja loop to select columns dynamically

### Goal

You will use a Jinja loop to generate repeated expressions for cleaning multiple columns.

You will not use `SELECT *`.

You will explicitly list columns.

### Suggested target

A good candidate is a staging model that selects columns from one of the raw Olist tables.

If you want the simplest option, use a model that stages `customers`.

That table has multiple string columns that benefit from cleanup.

### Steps

1. Define a list of column names with `{% set %}`.

Example shape:

```sql
{% set cols_to_clean = ['customer_city', 'customer_state'] %}
```

2. In your SELECT list, loop over the columns.

Your loop must:

* generate one cleaned column expression per input column
* avoid a trailing comma

3. Compile.

```bash
dbt compile
```

4. Inspect compiled SQL.

Find the compiled SQL file under:

* `target/compiled/`

Open the compiled file for your model.

Expected result:

* you see the full SELECT list
* columns are comma-separated correctly
* no trailing comma

5. Run only the model you changed.

```bash
dbt run -s <model_name>
```

### Checks

Your model should remain readable.

If the model becomes harder to read than the copy-paste version, reduce the loop.

Jinja is not a goal.

Maintainability is the goal.

## Task 3 — Add a post-hook to grant permissions

### Goal

You will attach a post-hook to one model that grants SELECT permissions.

### Requirements

* The hook must be a `post_hook`.
* The hook must use `{{ this }}`.
* The hook must grant SELECT on the built relation.

### Steps

1. Pick a model to attach the hook to.

Choose a model that builds a table or view that would be consumed by analysts.

2. Add a config block at the top of the model.

Example shape:

```sql
{{
  config(
    post_hook=[
      "GRANT SELECT ON TABLE {{ this }} TO ROLE ANALYST"
    ]
  )
}}
```

Do not change this into pseudo-code.

You must use real SQL.

3. Compile.

```bash
dbt compile
```

Expected result:

* compilation succeeds

4. Run only the model.

```bash
dbt run -s <model_name>
```

### What might happen

In some training environments:

* the role `ANALYST` might not exist
* your dbt user might not have grant privileges

If the run fails due to permissions:

* do not keep retrying
* capture the exact error
* tell the instructor

You still did the right implementation.

The failure is environment setup, not your Jinja.

## Submission checklist

You are done when:

* You can show the macro definition you created.
* You can show one model calling the macro.
* You can show one model using a Jinja loop for repeated expressions.
* You can show one model with a post-hook using `{{ this }}`.

All three tasks must compile.
