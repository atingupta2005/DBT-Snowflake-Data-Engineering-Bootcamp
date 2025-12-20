# Mart Models — Facts and Dimensions

> **Mart models are the contract between data teams and the business.**
> They must be correct, understandable, and stable.

---

## 1. Purpose of mart models

Mart models are the **final outputs** of a dbt project.

They exist to support:

* Analytics
* Reporting
* Decision-making

This is where data becomes **business-ready**.

All upstream layers exist to **support, simplify, and protect** mart models.

If upstream models prepare data, **marts explain it**.

---

## 2. Why marts are separated from other layers

Mart models directly answer business questions.

Because of that:

* They change with business needs
* They are consumed by many users
* Errors are immediately visible

By isolating marts:

* Business logic stays contained
* Changes do not ripple unnecessarily
* Reviews focus on **meaning**, not mechanics

**Example**

If the definition of revenue changes from:

> “order total” → “captured payment amount”

Only the mart should change — not every dashboard or analysis.

---

## 3. Facts and dimensions: the core pattern

Mart models typically fall into two categories:

* **Facts** — things that happened
* **Dimensions** — context about those things

A simple mental model:

| Type      | Answers                        |
| --------- | ------------------------------ |
| Fact      | How many? How much? How often? |
| Dimension | Who? What? Where? When?        |

This separation makes models easier to reason about and reuse.

---

## 4. Fact models

### 4.1 What fact models represent

Fact models represent **events or measurements**.

Examples:

* Orders placed
* Payments completed
* Items shipped
* Sessions started

Facts usually:

* Have many rows
* Grow over time
* Represent activity at a defined level of detail

---

## 4.2 Grain

Grain defines the **level of detail** of a table.

It answers one precise question:

> **“What real-world thing does one row represent?”**

This is not theoretical.
Grain is a **contract** between the data model and every downstream user.

---

### Grain is about reality, not columns

Grain is **not**:

* The primary key
* The number of columns
* The GROUP BY clause

Grain **is**:

* The real-world event or entity each row corresponds to

You should always be able to say:

> “Each row in this table represents exactly one ______.”

If that sentence is unclear or debatable, the model is broken.

---

## Example 1: One domain, multiple valid grains

Consider this raw data:

| order_id | item_id | product | price |
| -------- | ------- | ------- | ----- |
| 101      | 1       | A       | 60    |
| 101      | 2       | B       | 40    |

### Option A: Order-level grain

**Grain:** one row per order

| order_id | order_total |
| -------- | ----------- |
| 101      | 100         |

Meaning:

* Row = one completed order
* `SUM(order_total)` = total revenue
* `COUNT(*)` = number of orders

---

### Option B: Item-level grain

**Grain:** one row per order item

| order_id | item_id | price |
| -------- | ------- | ----- |
| 101      | 1       | 60    |
| 101      | 2       | 40    |

Meaning:

* Row = one item sold
* `COUNT(*)` = number of items
* Revenue requires aggregation

Both are correct.
They answer **different questions**.

---

## Example 2: Why unclear grain breaks metrics

Consider this table:

| order_id | product | order_total |
| -------- | ------- | ----------- |
| 101      | A       | 100         |
| 101      | B       | 100         |

What is the grain?

* One row per order? ❌ (order repeats)
* One row per product? ❌ (order_total repeats)
* One row per order-product? ❌ (metric doesn’t match)

### What goes wrong

If an analyst runs:

```sql
SELECT SUM(order_total) FROM table;
```

Result = **200**
Correct value = **100**

The data looks valid, but the **meaning is undefined**.

This is the most dangerous failure mode in analytics.

---

## Grain determines what is safe to do

Once grain is known, rules become obvious.

### If grain = one row per order:

| Operation           | Safe? | Why                    |
| ------------------- | ----- | ---------------------- |
| COUNT(*)            | ✅     | counts orders          |
| SUM(order_total)    | ✅     | one value per order    |
| Join to customers   | ✅     | one customer per order |
| Join to order_items | ❌     | multiplies rows        |

---

## Example 3: Grain explosion through joins

### Fact: one row per order

| order_id | order_total |
| -------- | ----------- |
| 101      | 100         |

### Dimension: order items (many rows per order)

| order_id | item_id |
| -------- | ------- |
| 101      | 1       |
| 101      | 2       |

After join:

| order_id | order_total | item_id |
| -------- | ----------- | ------- |
| 101      | 100         | 1       |
| 101      | 100         | 2       |

Now:

```sql
SUM(order_total) = 200 ❌
```

The join violated the grain contract.

---

## Grain must be declared, not implied

Good mart models **declare grain explicitly** in documentation or model descriptions:

> “This table contains one row per completed order.”

Bad models rely on readers to infer grain from columns.

Inference fails under pressure.

---

## Example 4: Grain vs aggregation confusion

If you see this pattern:

```sql
SELECT
  order_id,
  customer_id,
  SUM(price) AS total
FROM ...
GROUP BY order_id, customer_id, product_id
```

This is a warning sign.

Ask:

* Is the grain order?
* Product?
* Order-product?

If aggregation and grouping disagree, grain is unstable.

---

## Grain is more important than performance

Teams often optimize queries before clarifying grain.

This is backwards.

A slow query with correct grain can be optimized.
A fast query with unclear grain cannot be trusted.

---

## Key mental model

> **Grain defines truth.
> Aggregation only summarizes it.
> Joins must respect it.**

If everyone agrees on the grain:

* Metrics align
* Joins behave
* Trust increases

If grain is unclear:

* Every number is suspect

---

## Final takeaway on grain

Before writing SQL, answer this out loud:

> “One row equals one ______.”

If the answer changes depending on context,
the model is not ready to be a mart.

---

## 5. Dimension models

### 5.1 What dimension models represent

Dimension models provide **descriptive context**.

Examples:

* Customers
* Products
* Locations
* Dates

They answer **who or what** was involved in a fact.

---

### 5.2 Dimension grain

Dimensions also have grain.

Examples:

* One row per customer
* One row per product
* One row per date
* One row per customer per attribute version

Dimensions must clearly define their grain — especially when time is involved.

---

### 5.3 Example: Current-state dimension

**Grain:** one row per customer (current values)

| customer_id | customer_name | country |
| ----------- | ------------- | ------- |
| 1           | Alice         | US      |
| 2           | Bob           | CA      |

Safe to join by `customer_id` without additional logic.

---

### 5.4 Example: Valid SCD Type 2 dimension

**Grain:** one row per customer per attribute version

| customer_id | country | valid_from | valid_to   |
| ----------- | ------- | ---------- | ---------- |
| 1           | US      | 2023-01-01 | 2024-01-31 |
| 1           | UK      | 2024-02-01 | NULL       |

This is **correct and intentional**.

Facts must join using time logic (e.g. order date).

---

### 5.5 Example: Broken dimension grain

| customer_id | country | updated_at |
| ----------- | ------- | ---------- |
| 1           | US      | 2024-01-01 |
| 1           | UK      | 2024-02-01 |

Problems:

* No definition of which row applies
* `updated_at` is informational, not semantic
* No safe join rule exists

Multiple rows are not the problem.
**Undefined meaning is.**

---

## 6. Keys and joins

Facts and dimensions are connected using keys.

Typical pattern:

* Fact tables store foreign keys
* Dimensions store one row per key (per grain)

If a join increases row count:

* The join is incorrect, or
* The dimension grain is wrong, or
* Required SCD logic is missing

---

## 7. Surrogate keys (conceptual)

Surrogate keys are system-generated identifiers that:

* Represent a business entity
* Remain stable over time
* Support SCDs and multi-source joins

They improve reliability and flexibility.

Implementation details belong elsewhere.

---

## 8. What mart models should contain

Mart models should contain:

* Explicit grain (documented)
* Clear business definitions
* Stakeholder-relevant metrics
* Clean, predictable joins

A good mart can be explained in one sentence.

---

## 9. What mart models should not contain

Mart models should not:

* Fix raw data issues
* Perform complex preparation logic
* Hide assumptions
* Rely on defensive `DISTINCT`s

Those belong upstream.

---

## 10. Relationship to intermediate models

Intermediate models **prepare data**.
Mart models **interpret data**.

If a mart is long or hard to explain, logic belongs upstream.

Thin marts are a sign of good design.

---

## 11. Final checklist

Before publishing a mart:

* Is the grain explicit and unambiguous?
* Would two analysts calculate the same metric?
* Do joins preserve row counts?
* Are assumptions documented?

If yes, the model is ready.
