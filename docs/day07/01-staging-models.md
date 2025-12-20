# Staging Models

## Purpose of staging models

Staging models are the **first layer** in dbt modeling.

They sit directly on top of raw source data.

Their job is simple:

* Make raw data usable
* Make raw data consistent

Staging models do **not** add business meaning.
They prepare inputs for later layers.

---

## What problems staging models solve

Raw data is rarely clean.

Common issues include:

* Inconsistent column names
* Mixed data types
* Unclear timestamp formats
* Ambiguous flags and codes

If every downstream model fixes these problems again:

* Logic gets duplicated
* Bugs appear in multiple places
* Reviews become harder

Staging models fix issues **once**, at the edge.

---

## Standardizing raw inputs

Staging models apply **mechanical cleanup**.

Typical actions include:

* Renaming columns to clear, consistent names
* Casting data types explicitly
* Normalizing date and timestamp fields
* Making null handling predictable

This work is boring by design.
That is a good sign.

---

## Renaming and casting (illustrative)

A simplified example:

```
SELECT
    order_id            AS order_id,
    customer_id         AS customer_id,
    order_purchase_ts   AS order_purchased_at
FROM raw_orders
```

The intent here is clarity, not insight.

Names become readable.
Types become explicit.

---

## What staging models should contain

Staging models should contain:

* One source table
* One row per source row
* No joins to other datasets

They are thin by design.


---

## What staging models must never do

Staging models must not:

* Join multiple tables
* Apply business rules
* Aggregate data
* Filter for business meaning

Those actions hide assumptions.

Hidden assumptions at the base layer cause confusion later.

---

## Why business logic does not belong here

Business logic changes.

Definitions evolve.
Stakeholders disagree.

If business logic lives in staging:

* Every downstream change becomes risky
* Raw data meaning is lost
* Debugging becomes harder

By keeping staging models neutral:

* Raw meaning is preserved
* Changes stay localized
* Reviews stay focused

---

## How staging supports the rest of the project

Well‑designed staging models:

* Simplify intermediate joins
* Reduce duplication
* Make marts easier to read

They form a stable foundation.

If staging is messy, everything above it suffers.

---

## Mental checklist for staging models

Before moving logic out of staging, ask:

* Am I only standardizing raw data?
* Am I avoiding interpretation?
* Would two teams agree on this logic?

If the answer is yes, it belongs here.

Staging is about discipline.
Insight comes later.
