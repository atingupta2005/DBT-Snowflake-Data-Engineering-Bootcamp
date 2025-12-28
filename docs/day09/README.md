# Day 09 — Defining Sources, Source-Level Tests, and Freshness in dbt

Up to Day 08, we built models on top of the Olist raw tables and ran dbt end-to-end.
That proves you can write transformations and wire dependencies with `ref()`.

Now we handle the part that makes dbt projects reliable in real environments:
**making raw inputs explicit, testable, and monitorable**.

When raw tables are treated as “just tables”, upstream problems show up as:

* silent join drops (missing keys)
* join explosions (duplicate keys)
* stale results (ingestion delays)

All three can produce “successful” dbt runs with wrong outputs.
Day 09 is how you catch those problems early.

## Why source management matters now

So far, we have been transforming data.
Today we formalize the boundary between:

* **raw ingestion** (what dbt does not control)
* **transformations** (what dbt does control)

In a production data stack:

* ingestion pipelines fail independently of your dbt project
* upstream schemas change without notice
* key constraints drift over time

Source definitions, source tests, and freshness checks are how dbt helps you detect those issues before they infect downstream models.

## What you will do today

By the end of Day 09, you will be able to:

* Define raw tables as dbt sources in a `source.yml`
* Attach metadata to sources (table and column descriptions)
* Apply source-level tests:

  * `not_null`
  * `unique`
  * `relationships`
  * `accepted_values` (only when appropriate)
* Configure and interpret freshness:

  * `loaded_at_field`
  * warn vs error thresholds
  * `dbt source freshness`

This day is about correctness and trust.
You are not building new marts today.

## What you are not doing today

Day 09 is intentionally narrow.
Do not introduce:

* custom tests
* macros
* snapshots
* incremental models
* CI/CD or dbt Cloud concepts

If you find yourself drifting into those topics, stop.
We keep Day 09 focused so you can execute it cleanly.

## Documents for today

Read these in order:

1. `01-defining-sources.md`

   * why sources exist
   * how to structure `source.yml`
   * how to reference raw tables with `source()`

2. `02-source-tests-and-freshness.md`

   * how source tests work and what failures mean
   * how freshness works and what warn/error means
   * the commands you will run in the lab

Then complete the lab:

* `../../labs/day09/README.md`

## How this connects to downstream model stability

Everything downstream depends on the raw inputs.
If the raw inputs drift, downstream models can still run but become wrong.

Today’s configuration protects you from the most common failure modes:

* **Missing keys** → caught by `not_null`
* **Duplicate keys** → caught by `unique`
* **Broken foreign keys** → caught by `relationships`
* **Unexpected categories** → caught by `accepted_values`
* **Stale data** → caught by freshness checks

These are the checks that prevent “quiet failures”.

## What to expect in the lab

You will spend most of your time in YAML.
That is normal.

Your workflow today should be:

1. Validate YAML structure

```bash
dbt parse
```

2. Run only source tests

```bash
dbt test --select source:*
```

3. Run freshness

```bash
dbt source freshness
```

Do not aim for “all green”.
Aim for:

* “I can run the checks.”
* “I can explain what a failure means.”

## Setting expectations for Day 10

Day 09 is about the raw boundary.
Day 10 goes deeper on testing discipline across the project.

The key distinction to carry forward:

* **Source tests** protect your upstream inputs.
* **Model tests** protect your transformations.

If you leave Day 09 understanding that distinction, Day 10 will feel natural.
