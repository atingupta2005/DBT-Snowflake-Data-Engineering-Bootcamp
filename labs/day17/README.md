# Day 17 Lab — End‑to‑End Capstone Project

## What this lab is

This is the **final capstone**.

You will design, build, test, document, and automate a complete dbt project.

This lab evaluates whether you can work as an **Analytics Engineer**.

---

## Dataset (fixed)

You must use the **Olist dataset** already loaded.

Available source tables:

* `customers`
* `orders`
* `order_items`
* `payments`
* `products`

Do not introduce new datasets.
Do not rename raw tables.

---

## Business scenario

You are joining an analytics team supporting **Operations and Seller Management**.

Leadership asks for a new analytics mart:

### Mart name

`seller_performance`

### Business questions to support

At minimum, the mart must allow analysts to answer:

* How many orders does each seller fulfill?
* What is total revenue per seller?
* What is average order value per seller?
* How many unique customers does each seller serve?

You may add **additional metrics** if justified.

---

## Technical requirements

Your solution **must** include all of the following.

### 1. Layered dbt structure

Your project must clearly separate:

* `staging` models (raw cleanup / renaming)
* `intermediate` models (joins / logic)
* `marts` models (analytics‑ready outputs)

Naming must be consistent and intentional.

---

### 2. Sources

Define dbt `sources` for all raw tables you use.

Requirements:

* correct database and schema
* freshness or column tests where appropriate

---

### 3. Incremental logic

At least **one model** must be incremental.

Requirements:

* clear unique key
* deterministic incremental filter
* full refresh must still work

Do not guess.
Make the logic explicit.

---

### 4. Tests

You must define tests for:

* primary keys
* critical foreign keys
* non‑null business columns

Use:

* generic tests
* custom tests if required

All tests must pass.

---

### 5. Documentation

You must document:

* models
* columns
* business meaning of metrics

`dbt docs generate` must succeed.

---

### 6. CI / automation hook

Your project must be runnable via automation.

One of the following must exist:

* GitHub Actions workflow
* Jenkins pipeline

Minimum requirements:

* installs dbt
* creates `profiles.yml`
* runs `dbt debug`
* runs `dbt build`

---

## Constraints

These are enforced.

* No hardcoded credentials
* No `SELECT *`
* No circular refs
* No unused models
* No broken lineage

If it would fail in production, it fails this lab.

---

## Deliverables

Your submission is complete only when:

* `dbt build` succeeds
* `dbt test` succeeds
* incremental model runs twice without error
* docs generate without warnings
* pipeline runs end‑to‑end

---

## How this will be evaluated

You will be assessed on:

* correctness of SQL
* model layering discipline
* test coverage
* incremental correctness
* documentation quality
* automation readiness

Partial solutions score poorly.

---

## Time expectation

You are expected to need:

* design time
* iteration time
* debugging time

This is intentional.

---

## Rules of engagement

You may:

* refactor earlier models
* reuse patterns from previous days
