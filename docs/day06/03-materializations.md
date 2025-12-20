# Materializations — An Introduction

## Why materializations exist

When dbt runs a model, it must decide **what to create** in the warehouse.

That decision is called a **materialization**.

Materializations exist because:

* Different use cases need different persistence
* Not all data should be stored the same way
* Performance and cost trade‑offs matter

Materialization answers one question:

> “What kind of object should this SQL become?”

---

## Materialization is not SQL logic

SQL defines **how data is transformed**.

Materialization defines **how the result is stored**.

Keeping these separate is intentional:

* SQL stays readable
* Storage decisions stay explicit
* Behavior is predictable

You can change materialization without rewriting SQL.

---

## `table`

### What it represents

A table materialization creates a **physical table**.

The result is fully stored in the warehouse.

---

### When teams use tables

Tables are chosen when:

* Data is reused often
* Query performance matters
* Stability is more important than freshness

Tables trade storage for speed.

---

## `view`

### What it represents

A view materialization creates a **logical view**.

The SQL is stored.
The data is computed when queried.

---

### When teams use views

Views are chosen when:

* Logic is lightweight
* Data should always be current
* Storage should be minimal

Views trade compute for simplicity.

---

## `incremental` (conceptual only)

### What it represents

Incremental models update data **in pieces**.

Instead of rebuilding everything:

* New data is added
* Existing data may be updated selectively

---

### Why teams use incremental models

Incremental materializations are used when:

* Datasets grow large
* Full rebuilds become expensive
* Historical data is mostly stable

Details matter here.
Those details come later.

---

## Choosing a materialization

Materialization choice depends on:

* Data size
* Query patterns
* Freshness requirements
* Cost constraints

---
