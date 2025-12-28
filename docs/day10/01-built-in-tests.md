# 01 — Built-in model tests

Today we start with the fastest win in dbt: **built-in tests**.

Built-in tests are simple, readable, and easy to standardize across a team.

They catch the most common failures you will see in production:

* Duplicate keys
* Missing required fields
* Broken foreign keys
* Unexpected values appearing in a column

In this course, you will apply tests to **models** (not just sources).

That matters because models are what analysts and dashboards actually use.

---

## How dbt tests work

A dbt test is just a SQL query.

* If the query returns **zero rows**, the test **passes**.
* If the query returns **one or more rows**, the test **fails**.

Built-in tests generate that SQL for you.

You define tests in YAML, dbt generates the SQL, executes it, and reports failures.

This means:

* You must choose tests that match the business meaning of the model.
* You must pick the right columns.
* You must accept that tests are only as good as the assumptions you encode.

---

## Where to define model tests

Model tests live next to model definitions in YAML.

In a typical dbt project, that means a file like:

* `models/<area>/<model_name>.yml`

In this training repository, you will add tests to the existing model YAML files you already worked with in earlier days.

If your project currently uses a single consolidated model YAML file, that is fine too.

The rule is the same:

* Tests belong to the model they validate.
* Tests should be easy to find and review.

---

## The four built-in tests we will use

We will use these built-in tests today:

* `unique`
* `not_null`
* `relationships`
* `accepted_values`

Each one answers a different question.

---

# 1) `unique`

## What it validates

`unique` asserts that a column contains **no duplicates**.

Most often you use it on a primary key.

If the key is not unique, downstream aggregation can double-count.

### Example

In Olist, `orders` has one row per order.

A typical key is `order_id`.

If two rows share the same `order_id`, then:

* Revenue sums can double-count
* Order counts can inflate
* Joins to `order_items` can produce unexpected duplicates

## What failure looks like

A failing `unique` test means:

* The test query returned the duplicate key values.

In practice, you will see output like:

* `Got 12 results, configured to fail if != 0`

That means dbt found 12 duplicate key values.

## Common real-world causes

* Upstream system replays the same extract twice
* Late-arriving updates create multiple versions of the same key
* A model accidentally changed grain (you joined in another table and duplicated rows)

## When to use it

Use `unique` when the model must have one row per entity:

* One row per order
* One row per customer
* One row per product

If the model is intentionally at a lower grain (for example, order item level), then `unique` must match that grain.

That usually means:

* A composite key
* Or a different uniqueness expectation

We will keep it simple today and apply uniqueness to obvious keys.

## YAML example

```yaml
version: 2

models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
```

---

# 2) `not_null`

## What it validates

`not_null` asserts that a column contains **no NULLs**.

You use it for required fields.

If a required field becomes NULL, downstream logic breaks in subtle ways:

* Joins drop rows
* CASE expressions classify incorrectly
* Metrics undercount

## What failure looks like

A failing `not_null` test means:

* dbt found at least one row where the column is NULL

That is a strong signal.

You should treat it as:

* “This model is not safe to use until we understand why.”

## Common real-world causes

* Source system changed (field stopped being populated)
* A model introduced a `LEFT JOIN` and pulled a NULL from the right side
* A transformation cast failed and produced NULLs

## When to use it

Use `not_null` when the column is required for:

* Uniqueness (a primary key should not be NULL)
* Relationships (foreign keys should be present when required)
* Filtering logic (status columns used in WHERE)

Avoid using `not_null` on columns that are naturally optional.

Example: a delivery timestamp may be NULL for undelivered orders.

## YAML example

```yaml
version: 2

models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - not_null
```

---

# 3) `relationships`

## What it validates

`relationships` asserts that every value in a column exists in a target column.

This is a **foreign key check**.

It protects the integrity of joins.

### Example

If `stg_orders.customer_id` is supposed to match `stg_customers.customer_id`:

* Every order should map to exactly one existing customer.

If the relationship breaks:

* Orders become “orphaned”
* Customer-level reporting becomes incorrect
* Marketing attribution pipelines lose linkages

## What failure looks like

A failing relationships test returns:

* The foreign key values that did not match any parent key

This often points to one of two issues:

* The child model contains unexpected values
* The parent model is missing rows

## Common real-world causes

* Late arriving dimension rows (orders arrive before customers)
* Filters applied inconsistently (you filtered customers but not orders)
* Broken source ingestion (partial extracts)

## When to use it

Use `relationships` when:

* The join is business-critical
* You expect the foreign key to always resolve

Be careful when the relationship is legitimately optional.

Example: an order might not have a payment yet.

In those cases, you should combine relationships with `where` clauses later in the course.

For Day 10, we keep it basic.

## YAML example

```yaml
version: 2

models:
  - name: stg_orders
    columns:
      - name: customer_id
        tests:
          - relationships:
              to: ref('stg_customers')
              field: customer_id
```

### Read this carefully

* `to:` points to the parent model.
* `field:` is the parent key column.

The test checks:

* every non-null `stg_orders.customer_id` exists in `stg_customers.customer_id`

---

# 4) `accepted_values`

## What it validates

`accepted_values` asserts that a column contains **only a defined set of values**.

This protects you from domain drift.

### Example

If you classify orders by `order_status`, you might expect values like:

* `delivered`
* `shipped`
* `canceled`

If a new value appears (for example, `returned`), the pipeline might:

* Misclassify orders
* Drop them from filters
* Produce misleading dashboards

## What failure looks like

A failing accepted values test returns:

* Values present in the data that are not in your list

This is one of the most useful “early warning” tests.

It catches product changes and upstream system changes quickly.

## Common real-world causes

* New business process introduces a new status
* Source system changes naming (e.g., `cancelled` vs `canceled`)
* Encoding issues or unexpected whitespace

## When to use it

Use `accepted_values` when:

* The column represents a controlled domain (status, type, category)
* Downstream logic assumes a known set of values

Avoid it when values are truly open-ended (names, free-text fields).

## YAML example

```yaml
version: 2

models:
  - name: stg_orders
    columns:
      - name: order_status
        tests:
          - accepted_values:
              values:
                - delivered
                - shipped
                - canceled
```

---

## Running tests

When you add tests, you run them with:

```bash
dbt test
```

In a real project, you often narrow it down:

```bash
dbt test --select stg_orders
```

Or test only one specific model and everything downstream:

```bash
dbt test --select stg_orders+
```

We will use targeted test runs in the lab to keep feedback fast on slow laptops.

---

## Interpreting failures like a data engineer

When a test fails, do not immediately “fix the test.”

First, decide what kind of failure it is.

### Type A: Data quality problem

The test is correct, the data is wrong.

Examples:

* Duplicate `order_id`
* NULLs in a required key

Your response is usually:

* investigate upstream
* determine if it is a known anomaly
* decide whether to quarantine, backfill, or accept

### Type B: Model design problem

The model changed grain or logic.

Examples:

* Joining `orders` to `order_items` without aggregating causes duplicates

Your response is usually:

* review the model SQL
* confirm the expected grain
* fix the transformation

### Type C: Assumption drift

The business changed and your test assumptions are outdated.

Examples:

* A new status appears

Your response is:

* confirm the new value is real
* update logic downstream
* update accepted values list if appropriate

In all cases:

* a failed test is a signal
* the worst outcome is ignoring it

---
