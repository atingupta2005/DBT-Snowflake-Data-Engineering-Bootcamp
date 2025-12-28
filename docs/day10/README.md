# Day 10 — Data Quality, Reference Data, and Change Tracking in dbt

Today we move from **trusting sources** to **proving our models are correct**.

Up to Day 09, we focused on upstream confidence:

* You defined dbt sources
* You added source-level tests
* You configured freshness checks
* You learned what it means to trust (or not trust) the raw data

That work answers: **“Is the raw data usable?”**

Day 10 answers a different question: **“Are our models safe to use downstream?”**

Downstream analytics breaks in predictable ways:

* A “unique” key stops being unique and dashboards double-count.
* A required field becomes NULL and joins silently drop rows.
* A relationship breaks and facts stop matching dimensions.
* A field’s domain drifts (new unexpected statuses appear) and reports mislabel data.

The goal today is to make those failures loud and early.

---

## What you will learn today

By the end of Day 10, you will be able to:

* Add **built-in dbt tests** to models:

  * `unique`
  * `not_null`
  * `relationships`
  * `accepted_values`
* Recognize when built-in tests are **not enough**, and how to create a **simple custom test**
* Use **seeds** for small, static reference data that belongs *inside* the project
* Understand **snapshots** as a way to track changes over time (SCD Type 2), and when they are worth the cost

You will write and run actual tests and see failure output.

---

## How today fits into the course

Testing and controlled state management only makes sense once you have models worth protecting.

You already have Olist models that produce business-ready tables.

Now we add guardrails:

* **Tests** protect correctness.
* **Seeds** provide stable reference data where a database table would be overkill.
* **Snapshots** track history when the “current value” is not enough.

Tomorrow (Day 11), we will introduce **incremental models**.

Incremental builds are fast, but they also raise the stakes:

* If a model is wrong, it can keep being wrong silently.
* If you load only new rows, you might miss corrections.

That is why today happens now.

---

## Files you will use today

Read the theory in this order:

1. `01-built-in-tests.md`
2. `02-custom-tests.md`
3. `03-seeds.md`
4. `04-snapshots.md`

Then complete the hands-on work:

* `../../labs/day10/README.md`

