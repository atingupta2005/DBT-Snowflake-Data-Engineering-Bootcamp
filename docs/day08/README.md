# Day 08 — Hands-on dbt Model Development (Staging → Intermediate → Mart)

## What you will build today

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
