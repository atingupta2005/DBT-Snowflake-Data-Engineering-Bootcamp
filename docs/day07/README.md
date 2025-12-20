# Day 07 — dbt Modeling Layers and Design Principles

## Why modeling starts now

Up to Day 06, dbt was about **execution mechanics**.

You learned:

* How dbt discovers models
* How dependencies define order
* How commands behave
* How lineage is created

What you did **not** do yet is decide *what models to build*.

That decision comes first.

Before writing SQL, you must know **where that SQL belongs**.

---

## What Day 07 focuses on

Day 07 introduces **modeling design**, not heavy implementation.

Today is about:

* Why dbt models are layered
* What responsibility each layer owns
* How layering prevents duplication and confusion

You will not build large transformations today.
You will build **judgment**.

---

## Why layered modeling exists

Analytics projects grow over time.

Without structure:

* Logic gets duplicated
* Fixes require changes in many places
* Reviews become difficult

Layered modeling solves this by:

* Separating concerns
* Making intent explicit
* Allowing reuse without copying

Each layer answers a different question.

---

## How Day 06 connects to Day 07

Execution mechanics make layering possible.

Because dbt:

* Knows dependencies
* Enforces order
* Tracks lineage

You can safely split logic across layers.

Without understanding execution and lineage, layers feel arbitrary.
Now they are intentional.

---

## What you should understand by the end of today

By the end of Day 07, you should be able to:

* Explain why models are split into layers
* Decide whether logic belongs in staging, intermediate, or mart
* Identify common modeling anti‑patterns early
* Read a dbt project and understand its design

This prepares you for hands‑on modeling.

---

## Documents for Day 07

Read these in order:

1. **01-staging-models.md**
   Standardizing raw data and setting clean foundations

2. **02-intermediate-models.md**
   Combining and enriching data safely

3. **03-mart-models.md**
   Designing fact and dimension outputs

Each document builds on the previous one.

---

## Looking ahead to Day 08

Day 08 is hands‑on.

You will:

* Build your first real dbt models
* Apply staging and intermediate patterns
* Begin creating marts

Day 07 ensures those decisions feel obvious, not forced.

Design first.
Implementation next.
