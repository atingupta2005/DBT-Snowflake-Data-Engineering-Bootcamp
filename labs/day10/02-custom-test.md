# Day 10 Lab — 02 Custom Test (Singular Test)

In this file you will create **one** custom dbt test.

This is a **singular test**.

A singular test is a SQL query.

It must return **violations**.

* 0 rows returned → PASS
* 1+ rows returned → FAIL

We will keep the rule simple and realistic.

---

## Why you need custom tests

Built-in tests are great for:

* keys (`unique`, `not_null`)
* foreign keys (`relationships`)
* domains (`accepted_values`)

But real data quality rules are often conditional.

Example:

* “If an order is delivered, it must have a delivered timestamp.”

That rule is not a plain `not_null`.

Because:

* non-delivered orders can legitimately have null delivery timestamps

So we write a test that checks the condition.

---

## A) Create the tests folder

From your project root:

```bash
mkdir -p tests
```

---

## B) Create the singular test SQL

Create this file:

```bash
nano tests/delivered_orders_require_delivered_date.sql
```

Paste:

```sql
SELECT
  order_id,
  order_status,
  order_delivered_customer_ts,
  order_purchase_ts
FROM {{ ref('fct_orders') }}
WHERE order_status = 'delivered'
  AND order_delivered_customer_ts IS NULL
```

Save and exit.

What this query is doing:

* It looks only at `delivered` orders.
* It returns the orders that violate the rule.

If this returns rows, dbt will fail the test.

---

## C) Run only singular tests

Run:

```bash
dbt test --select test_type:singular
```

If it passes, you are done.

If it fails, do not guess.

Pull the failing rows from Snowflake.

---

## D) Debug failures in Snowflake

Run:

```sql
select
  order_id,
  order_status,
  order_delivered_customer_ts,
  order_purchase_ts
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_status = 'delivered'
  and order_delivered_customer_ts is null
limit 50;
```

Now make a decision:

* Are these rows legitimate?
* Or is something wrong upstream?

### Common reasons it fails (in real projects)

1. Upstream started sending a new “delivered-like” status

* Your transformation marks it as delivered
* But it does not populate delivered timestamp

Fix:

* update the status mapping logic
* or adjust the test to match the real meaning of “delivered”

2. Column is populated, but formatting made it null

Examples:

* string-to-timestamp conversion failed
* invalid timestamp values

Fix:

* find conversion logic in your staging/intermediate model
* correct the cast/parse

3. Data is genuinely broken

Fix:

* keep the test
* escalate to upstream data owner
* keep the failing rows as evidence

---

## E) Rerun only the failing test (optional)

If you want to run only this one test file:

```bash
dbt test --select delivered_orders_require_delivered_date
```

This is useful when you are iterating.

---

## Done criteria

You are done with this file when:

```bash
dbt test --select test_type:singular
```

finishes with **0 failing tests**.
