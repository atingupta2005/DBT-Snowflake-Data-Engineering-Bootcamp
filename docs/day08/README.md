````markdown
# Day 08 — Hands-on dbt Model Development (Staging → Intermediate → Mart)

Day 08 is where dbt stops being abstract.

Up to now, you’ve learned the project structure, how dbt runs models, and how layers are supposed to work. Today you will write real models and see them build end-to-end in Snowflake.

This day will feel slower than earlier days. That’s normal. You will write more files, make more small mistakes, and spend more time reading error messages. The goal is not speed. The goal is discipline: small steps, clean models, correct dependencies.

## What you will build today

By the end of Day 08, you will have a small but complete warehouse slice built in dbt:

- **Staging models** for raw Olist tables
  - Type casting
  - Column renaming
  - Basic standardization
  - No business logic

- **Intermediate models** that join and enrich staged data
  - Clear CTE chains
  - One logical step per CTE
  - Clean inputs for marts

- **Mart models**
  - At least one **dimension**
  - At least one **fact**
  - Explicit grain
  - Correct `ref()` usage across layers

## Time expectations

Plan for **2–3 focused hours** to complete the lab.

You will move faster if you work in this order:

1. Build staging models and run them
2. Build one intermediate model and run it
3. Build mart models and run the full chain

If you jump straight to marts, you will spend the whole day debugging upstream issues.

## How today connects to Day 07

Day 07 was about responsibilities:

- Staging: make raw data usable
- Intermediate: combine and shape data for downstream use
- Mart: publish models with clear meaning and grain

Today you will implement that separation in dbt using:

- `sources` to point to raw tables
- One model per file
- `ref()` to create explicit dependencies
- `dbt run` to build only what is needed

## Files for Day 08

Read these in order. Do not skip ahead.

1. `01-build-staging-models.md`
2. `02-build-intermediate-models.md`
3. `03-build-facts-and-dimensions.md`

Then complete the lab:

- `labs/day08/README.md`

Instructor-only material exists for this day, but students should not reference it.

## What “done” looks like

You are done when you can run these successfully:

```bash
dbt debug
dbt run
````

And you can answer these questions from your own models:

* What is the grain of your fact table?
* Which models are pure staging and contain no business logic?
* Which intermediate model produces the clean input for marts?
* Where does `ref()` appear, and what would break if you used hard-coded table names?

