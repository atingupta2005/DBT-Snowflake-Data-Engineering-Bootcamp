# 02 — Custom tests

Built-in tests cover a lot.

In most projects, `unique`, `not_null`, `relationships`, and `accepted_values` will catch the majority of issues.

But sooner or later, you will hit a validation rule that does not fit a built-in test.

That is when you write a **custom test**.

A custom test is still the same thing:

* a SQL query
* returning zero rows means pass
* returning one or more rows means fail

The difference is:

* you write the SQL yourself

---

## When built-in tests are not enough

Here are common patterns where built-in tests are insufficient.

### 1) Row-level business rules

Built-in tests validate a single column (or a relationship).

They do not express “if X then Y” rules.

Examples you will see in production:

* If an order is `delivered`, it must have a delivery timestamp.
* If payment type is `credit_card`, installments must be >= 1.
* If an order has items, total payment should be > 0.

These rules are not about one column in isolation.

They are about consistency across columns.

### 2) Multi-column uniqueness

Built-in `unique` checks one column.

Sometimes the real key is a combination.

Example:

* `order_id` + `order_item_id` is unique in `order_items` grain.

You can work around this with surrogate keys later, but custom tests are the simplest way to validate this concept right now.

### 3) Numeric constraints

Built-in tests do not directly express “must be positive” or “must be within range.”

Examples:

* `price` must be >= 0
* `freight_value` must be >= 0

### 4) Drift in aggregated logic

Sometimes you need to validate something like:

* “model totals should reconcile with another model”

That can be useful, but it can also be expensive.

We will keep today’s examples small.

---

## The structure of a SQL-based custom test

In dbt, a test is just a SELECT statement.

The query should return the **rows that violate the rule**.

That is the best practice.

Why?

* When the test fails, you immediately get examples of the bad data.
* You can copy the failing rows into a debugging query.

A good custom test:

* is readable
* is scoped (doesn’t scan unnecessary models)
* produces output that helps you debug

---

## Custom test example 1: delivered orders must have a delivered timestamp

This is a classic “if X then Y” rule.

We use Olist `orders` status fields.

**Rule**:

* If `order_status = 'delivered'`, then `order_delivered_customer_date` must not be NULL.

### Why this matters

Downstream teams use delivered date for:

* delivery time metrics
* cohort reporting
* customer satisfaction analysis

If delivered orders are missing a delivered date:

* averages become wrong
* metrics undercount
* analysis breaks

### Test SQL concept (not file path yet)

A custom test would return rows like:

* order_id
* order_status
* order_delivered_customer_date

where the rule is violated.

Example query:

```sql
SELECT
  order_id,
  order_status,
  order_delivered_customer_date
FROM {{ ref('stg_orders') }}
WHERE order_status = 'delivered'
  AND order_delivered_customer_date IS NULL
```

If the query returns any rows, the test fails.

---

## Custom test example 2: price and freight must not be negative

**Rule**:

* `price` >= 0
* `freight_value` >= 0

This sounds obvious, but you will see negative values in real systems:

* refunds stored as negative amounts
* corrections applied without a separate adjustment table
* bad parsing of strings to numbers

### Why this matters

Negative amounts can:

* break revenue calculations
* produce negative shipping costs in dashboards
* confuse finance teams

### Test SQL concept

```sql
SELECT
  order_id,
  order_item_id,
  price,
  freight_value
FROM {{ ref('stg_order_items') }}
WHERE price < 0
   OR freight_value < 0
```

If this fails, you need to decide what negative values mean.

Do not automatically “fix” by filtering them out.

First confirm:

* are these refunds?
* are these data errors?

---

## Where custom tests live in a dbt project

Custom tests live under a `tests/` directory.

Each `.sql` file defines one test.

The file contains only SQL.

No prose.

A typical structure is:

* `tests/<test_name>.sql`

When dbt runs tests, it compiles and runs those SQL files.

You do not need to register them in YAML.

If the file exists under `tests/`, dbt will run it.

That said, in many teams, tests are also documented in model YAML for discoverability.

For this course:

* you will create one or two custom test SQL files
* you will keep them small
* you will run them with `dbt test`

---

## Writing custom tests that are reviewable

You will be tempted to write one huge test that checks many rules.

Do not do that.

Write one test per rule.

### Why

* Failing output is clearer
* Maintenance is easier
* You can disable a single test if needed

### Recommended shape

Use clean SQL formatting:

* uppercase keywords
* explicit columns
* no SELECT *

Keep the output columns useful for debugging.

Do not return dozens of columns.

Return the ones that explain the violation.

---

## Parameterizing tests conceptually

In real projects, you often want the same rule applied to many models.

Example:

* “all monetary amounts must be non-negative”

You could write the same test SQL repeatedly.

But that is hard to maintain.

This is where parameterized tests help.

### What “parameterized” means

Instead of hardcoding:

* the model name
* the column names

You pass them as parameters.

Conceptually, you want something like:

* model: `stg_order_items`
* columns: `price`, `freight_value`

and one reusable test definition.

We are not going to build a macro-heavy testing framework today.

But you should understand the design goal:

* reduce copy/paste
* centralize logic
* enforce consistent rules

In later work, you will see custom test macros that accept:

* `model`
* `column_name`
* `values`
* thresholds

For Day 10, keep it simple:

* write explicit test SQL
* make it readable

---

## How to run and debug custom tests

Run all tests:

```bash
dbt test
```

Run only custom tests (common workflow in teams):

```bash
dbt test --select test_type:singular
```

Run one specific custom test by name:

```bash
dbt test --select <test_name>
```

When a custom test fails:

1. Open the compiled SQL in `target/compiled/` (optional)
2. Copy the failing query and run it in your warehouse
3. Inspect the failing rows

In the lab, you will follow a lighter workflow:

* run the test
* read the failure output
* use the failing rows as your debugging starting point

