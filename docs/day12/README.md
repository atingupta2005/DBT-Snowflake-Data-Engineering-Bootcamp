# Day 12: Advanced Engineering — Jinja, Macros, and Utilities

Day 12 is where we stop treating dbt models like “SQL files with a header” and start treating them like **maintainable code**.

Up to Day 11, we focused on writing clear, explicit SQL. That never goes away.

Today we add two tools that help you scale your project:

* **Jinja templating**: lets you write dynamic SQL safely.
* **Macros**: lets you wrap repeated SQL logic into reusable functions.

We will also touch two operational features:

* **Utility packages** like `dbt-utils` for standard macros.
* **Hooks** (pre/post) for simple operational tasks like granting permissions.

## What “DRY” means in dbt

**DRY** stands for **Don’t Repeat Yourself**.

In analytics engineering, repetition usually shows up as:

* The same `CASE` expression repeated across multiple staging models
* The same string cleanup logic repeated on many columns
* The same “convert cents to dollars” math repeated in every model
* The same surrogate key logic repeated for joins

Repetition is expensive.

If you change the logic later, you have to remember to update it everywhere.

That’s how small code changes turn into slow bugs.

## Copy-paste SQL vs macro-based SQL

Here’s a typical “copy-paste SQL” pattern.

You see the same logic repeated in multiple models:

```sql
SELECT
  payment_value / 100.0 AS payment_value_usd
FROM some_table
```

Then later, someone realizes:

* We should **round to 2 decimals**.
* We should handle **NULL** values explicitly.

Now you have to update 10 models.

### The macro approach

Instead, we put the logic in one place:

```sql
{{ cents_to_dollars('payment_value') }}
```

And the macro contains the real conversion rules.

When requirements change, you update the macro once.

Every model benefits.

## What you will build today

By the end of Day 12, you will be able to:

* Write a custom macro to encapsulate repeated SQL logic.
* Use Jinja loops to generate repeated column expressions.
* Use a `post-hook` to grant permissions on a model.
* Use a utility package (`dbt-utils`) for common macros.

You will do this in a controlled way.

We are not trying to “make everything dynamic.”

We are learning the engineering patterns that help you keep a growing dbt project readable.

## What we are **not** doing today

To keep the scope tight, we are explicitly not covering:

* Custom materializations
* Python models

If you see those in the real world, treat them as “later topics.”

## How today’s materials are organized

* `01-jinja-basics.md`

  * Jinja variables, loops, conditions
  * whitespace control (`{{-` and `-}}`)

* `02-creating-macros.md`

  * How to define macros
  * Arguments and defaults
  * Calling macros inside models

* `03-utility-macros-and-hooks.md`

  * Using `dbt-utils` briefly
  * Standard patterns for dates and strings
  * Post-hooks for permissions
  * Pre-hooks for audit logging

## How to work today

1. Read the theory docs in order.
2. Do the lab tasks.
3. Use `dbt compile` whenever you get stuck.

   * Compilation errors are common when Jinja syntax is wrong.
4. Only run `dbt run` after compilation succeeds.

## Classroom realism rules for today

* If your Jinja fails to compile, dbt will not run anything.
* Small syntax mistakes (missing braces, missing quotes) cause large errors.
* When a loop generates SQL, you must still verify the compiled SQL looks correct.

You will be faster today if you adopt this habit:

* Write a small change
* Compile
* Inspect the compiled SQL
* Then run

That is how we avoid “mystery dbt failures.”
