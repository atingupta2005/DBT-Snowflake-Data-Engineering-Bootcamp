# Source-Level Tests and Freshness

Today we make raw data **explicitly accountable**.

Up to now, your models probably did something like:

```sql
FROM {{ ref('stg_orders') }}
```

…and you trusted that whatever sits upstream is correct.

Sources change that.

A dbt **source** is your formal contract with upstream data:

* **This table should exist.**
* **These columns should look like this.**
* **This table should be updated on time.**

When those expectations break, you want the failure to happen **as early as possible**, before your marts and dashboards become misleading.

This document covers two things you will do on Day 09:

1. Add **source-level tests** to raw Olist tables.
2. Configure **freshness checks** to detect ingestion delays.

---

## Why source-level tests exist

When a downstream model fails, it is often **too late**.

Examples:

* A report shows fewer orders than yesterday.
* A KPI drops for no business reason.
* A join suddenly explodes because keys are duplicated.

If you only test models, you might catch these problems **after** transformations.

Source tests push detection to the boundary:

* Raw data is incomplete → fail at the source.
* Raw keys are duplicated → fail at the source.
* A dimension table is missing values → fail at the source.

In practice, teams rely on source tests to answer one question:

**“Can we trust today’s raw data enough to build downstream models?”**

---

## Where source tests live

Source tests belong in your `sources.yml` (or `source.yml`) file under:

* `sources:`

  * `tables:`

    * `columns:`

Example shape (not complete):

```yaml
version: 2

sources:
  - name: olist
    schema: raw
    tables:
      - name: orders
        columns:
          - name: order_id
            tests:
              - not_null
              - unique
```

On Day 09 you will define sources for these raw tables only:

* `customers`
* `orders`
* `order_items`
* `payments`
* `products`

---

## How dbt runs tests

When you run:

```bash
dbt test
```

dbt turns each configured test into a SQL query.

Most dbt tests follow a simple rule:

* **If the query returns 0 rows → the test passes.**
* **If the query returns 1+ rows → the test fails.**

So a failed test is not abstract. It always means:

* “Here are the rows that violate the rule.”

That is why tests are valuable.

They fail with evidence.

---

## Test 1: not_null

### What it checks

`not_null` ensures a column has **no NULL values**.

Typical use:

* Primary keys
* Required foreign keys
* Business-critical fields

### What a failure means

A `not_null` failure indicates one of these is happening:

* Upstream ingestion is dropping values.
* A parsing step is producing NULLs.
* The raw table contains partial rows.

### Practical example

If `orders.order_id` is NULL even once:

* You cannot safely join to `order_items`.
* Counts may be off.
* Deduping becomes impossible.

In a production pipeline, this is usually a **stop-the-line** issue.

---

## Test 2: unique

### What it checks

`unique` ensures a column has **no duplicates**.

Typical use:

* Natural primary keys (like `order_id`, `customer_id`, `product_id`)

### What a failure means

A `unique` failure means:

* Upstream is sending duplicate rows.
* A merge job is replaying data.
* A batch ran twice.

Duplicates at the source are dangerous because they can:

* Inflate revenue
* Inflate order counts
* Inflate item quantities

### Practical example

If `orders.order_id` is duplicated:

* Any join from `orders` to `order_items` may multiply rows.
* A dashboard can show *more* orders than actually exist.

---

## Test 3: relationships

### What it checks

`relationships` ensures values in one table **exist in another table**.

This is how you enforce referential integrity in dbt.

Typical use:

* `orders.customer_id` must exist in `customers.customer_id`
* `order_items.order_id` must exist in `orders.order_id`
* `order_items.product_id` must exist in `products.product_id`

### What a failure means

A relationships failure means:

* The child table arrived, but the parent table did not.
* The parent table is incomplete.
* A foreign key was corrupted.
* Ingestion is out of order.

This is one of the most useful signals for upstream health.

### Practical example

If `order_items.order_id` contains IDs not present in `orders`:

* Your order item facts cannot roll up correctly.
* Revenue per order becomes wrong.
* Models may silently drop items when joining.

In production, this is often caused by:

* Late-arriving orders
* Partial loads
* Missing partitions

---

## Test 4: accepted_values

### What it checks

`accepted_values` ensures a column value is in a **known allowed set**.

It is appropriate when:

* The domain is small and stable.
* The values represent a categorical code.
* You want to detect a new/unexpected category quickly.

Do not use it when:

* The value space is huge.
* New values are expected daily.
* You cannot confidently list all valid values.

### What a failure means

An `accepted_values` failure usually means:

* Upstream introduced a new code.
* A parsing step changed the format.
* Dirty values slipped in (leading/trailing spaces, invalid codes).

### Practical example

If you apply `accepted_values` to `orders.order_status`:

* It helps catch a new status like `returned` or `chargeback`.
* It helps catch typos like `delivered ` (trailing space).

You will decide case-by-case in the lab.

---

## Choosing tests for Olist raw tables

You are not trying to test everything.

You are trying to protect your pipeline from the **most damaging upstream failures**.

Here is a practical starting point for Olist.

### customers

Common protections:

* `customer_id` should be `not_null` and `unique`.

Optional:

* If `customer_state` exists and is stable, `accepted_values` can be useful.

### orders

Common protections:

* `order_id` should be `not_null` and `unique`.
* `customer_id` should be `not_null`.
* `customer_id` should have a `relationships` test to `customers`.

Optional:

* `order_status` can use `accepted_values`.

### order_items

Common protections:

* Composite keys are common here, but source tests still help.
* `order_id` should be `not_null`.
* `product_id` should be `not_null`.
* `order_id` should relate to `orders`.
* `product_id` should relate to `products`.

### payments

Common protections:

* `order_id` should be `not_null`.
* `order_id` should relate to `orders`.

Optional:

* Payment type fields can be good candidates for `accepted_values`.

### products

Common protections:

* `product_id` should be `not_null` and `unique`.

---

## Running source tests safely

When you add tests, keep the workflow predictable.

### Step 1: validate your project loads

```bash
dbt parse
```

If YAML indentation is wrong, `dbt parse` usually fails fast.

### Step 2: run only tests first

```bash
dbt test
```

You should see output that includes:

* test names
* PASS / FAIL status
* how long each test took

If a test fails, do not delete it.

Instead:

1. Read the failure output.
2. Decide if the data really violates the rule.
3. Decide if your assumption is wrong.

In real teams, tests often drive a conversation with upstream owners.

---

## Freshness in dbt

Tests answer:

* “Is the data shaped correctly?”

Freshness answers:

* “Is the data recent enough to use?”

Freshness checks are especially valuable when:

* Data loads are scheduled.
* Dashboards depend on daily updates.
* Late arrivals cause misleading metrics.

A table can be perfectly valid and still be useless if it is **stale**.

---

## loaded_at_field

Freshness requires a timestamp column that tells you **when the row was loaded**.

In dbt this is called:

* `loaded_at_field`

This field should represent:

* The ingestion or load time

Not:

* A business event time (like order purchase time)

Why?

If you use a business event timestamp, freshness will confuse “late orders” with “late ingestion”.

In a production warehouse, teams often add a dedicated ingestion timestamp.

For this training repository:

* Use a timestamp column that exists in the raw table and best represents load time.
* If you are unsure, you will verify by inspecting the data.

---

## Freshness thresholds

Freshness thresholds let you define two levels:

* `warn_after` → not ideal, but still usable
* `error_after` → too old to trust

Example shape:

```yaml
freshness:
  warn_after:
    count: 24
    period: hour
  error_after:
    count: 48
    period: hour
```

Interpretation:

* If the newest loaded row is older than 24 hours → dbt warns.
* If it is older than 48 hours → dbt errors.

You pick thresholds based on business expectations.

A daily load pipeline might use:

* warn after 24 hours
* error after 36–48 hours

A near-real-time pipeline would be much tighter.

---

## Running freshness checks

Freshness checks run with:

```bash
dbt source freshness
```

This command:

* checks each configured source/table freshness
* reports status: PASS, WARN, or ERROR

What you should do with the output:

* If it WARNs, you investigate but may continue.
* If it ERRORs, you treat upstream ingestion as broken.

In production, freshness checks are often the earliest alert you get that a pipeline is delayed.

---

## Common freshness failure patterns

### 1) Scheduled job did not run

Symptoms:

* Freshness errors suddenly appear across multiple tables.

Typical cause:

* The ingestion scheduler failed.

### 2) Partial ingestion

Symptoms:

* Some tables are fresh, others stale.

Typical cause:

* One source system failed.
* A dependency table did not load.

### 3) loaded_at_field is wrong

Symptoms:

* Freshness always fails even though the table has new rows.

Typical cause:

* You picked a business timestamp that does not update on load.

Fix:

* Choose a better column for `loaded_at_field`.

---

## YAML mistakes that break tests and freshness

YAML is strict.

Most failures on this day are not SQL problems.

They are indentation or structure problems.

Common issues:

* `tests:` is placed under `tables:` instead of under a column.
* `columns:` is mis-indented.
* Using tabs instead of spaces.
* Forgetting `version: 2` at the top.
* Typo in `loaded_at_field`.

Your workflow should always include:

```bash
dbt parse
```

before you run `dbt test` or `dbt source freshness`.

---

## How to reason about failures

When a source test fails, it is telling you:

* Your raw data violates an assumption.

That assumption might still be correct.

Example:

* `orders.order_id` should be unique.

If it fails, you do not “fix dbt”.

You investigate:

1. Is the raw table duplicated?
2. Did ingestion replay data?
3. Did you test the wrong column?

The goal is not to make tests green.

The goal is to make data trustworthy.

---

## What comes next

Day 10 goes deeper on testing.

Today you are learning the boundaries:

* sources define the raw contract
* source tests prevent bad raw data from flowing downstream
* freshness catches ingestion delays early

For the lab, you will implement these checks for Olist raw tables and learn how to interpret failures.
