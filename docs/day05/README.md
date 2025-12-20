# Day 05 — DBT Project Setup, Configuration Boundaries, and Git Discipline

## Why dbt is introduced now

Until Day 04, all work has been **SQL-first**.

You wrote readable queries.
You used CTE chains to break down logic.
You learned to think in steps that can be reviewed.

That foundation matters.

dbt does not replace SQL.
dbt **organizes** SQL.

If dbt is introduced too early, learners focus on commands instead of thinking.
That leads to confusion, rework, and bad habits.

Day 05 exists to prevent that.

---

## What Day 05 is (and is not)

Today is about **orientation**, not execution.

Today we focus on:

* What a dbt project represents
* How a dbt project is structured
* Where configuration belongs
* Where credentials must never live
* Why version control is mandatory

Today we explicitly do **not**:

* Run dbt
* Build models
* Write transformations
* Set up environments

You will not type dbt commands today.
That starts next.

---

## How this day fits in the course

Think of the course in three phases:

1. **Days 01–04** — Analytical SQL discipline
2. **Day 05** — Project structure and boundaries
3. **Day 06 onward** — Executing dbt with confidence

Day 05 is the bridge.

Without this bridge:

* Configuration feels magical
* Errors feel random
* Git history becomes messy

With this bridge:

* Files have clear purpose
* Changes are intentional
* Teams can review work safely

---

## What you should understand by the end of today

By the end of Day 05, you should be able to answer:

* What lives **inside** a dbt project
* What lives **outside** the project
* Why credentials are never committed
* How dbt projects evolve over time
* What discipline is expected before writing the first model

If these are clear, Day 06 becomes straightforward.

---

## Documents for Day 05

Read these in order:

1. **01-dbt-project-structure.md**
   What a dbt project is and why its structure matters

2. **02-profiles-and-environments.md**
   Configuration boundaries and environment mental models

3. **03-git-integration.md**
   Why dbt work must be version-controlled

Do not skip ahead.
Each document builds context for the next.

---

## How to approach today

Read slowly.
Do not rush to implementation.

If something feels abstract, that is expected.
The purpose is to give names and boundaries to ideas you will use tomorrow.

Execution starts next day.
Understanding starts now.
