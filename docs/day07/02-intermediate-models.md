# Intermediate Models

## Purpose of intermediate models

Intermediate models sit **between staging and marts**.

They exist to:

* Combine datasets
* Apply reusable logic
* Prepare clean inputs for final outputs

They are a **workspace**, not a destination.

End users should rarely query intermediate models directly.

---

## Why intermediate models exist

As projects grow, logic becomes shared.

Without an intermediate layer:

* The same joins appear in many marts
* Business rules are copied repeatedly
* Fixes must be applied in multiple places

Intermediate models solve this by:

* Centralizing shared logic
* Reducing duplication
* Making changes safer

---

## What kind of logic belongs here

Intermediate models are where **meaningful combinations** happen.

Typical responsibilities include:

* Joining multiple staging models
* Enriching one dataset with another
* Deriving reusable calculated fields

This logic is more than cleanup.
It is still less than final business output.

---

## Combining datasets (illustrative)

A simplified example:

```
SELECT
    o.order_id,
    o.order_purchased_at,
    c.customer_id,
    c.customer_city
FROM stg_orders o
JOIN stg_customers c
  ON o.customer_id = c.customer_id
```

The goal here is preparation.

This result is easier to reuse than repeating the join everywhere.

---

## What intermediate models should not be

Intermediate models should not:

* Be consumed directly by dashboards
* Encode final business metrics
* Define reporting grain implicitly

If a model answers a business question directly, it is likely a mart.

---

## How intermediate models reduce duplication

When logic lives in one place:

* Fixes are applied once
* Reviews are simpler
* Assumptions are visible

Downstream models become thinner.

Thin models are easier to reason about.

---

## Naming and intent

Intermediate models should be named to reflect **process**, not outcome.

Good names describe:

* What data is being combined
* What preparation is happening

They should not sound like reports.

This keeps the layer honest.

---

## Relationship to staging models

Staging models:

* Standardize raw data
* Avoid interpretation

Intermediate models:

* Combine and enrich
* Begin interpretation

Each layer has a clear step up in responsibility.

---

## Relationship to marts

Intermediate models feed marts.

They exist to make mart logic:

* Shorter
* Clearer
* Focused on business meaning

If mart SQL feels complex, intermediate models are usually missing.

---

## Mental checklist for intermediate models

Before placing logic here, ask:

* Will multiple marts need this?
* Does this reduce repetition?
* Is this still preparatory, not final?

If yes, intermediate is the right layer.

Intermediate models are about leverage.
They make everything above them easier.
