# Day 14 — Automated Quality: CI/CD with GitHub Actions

Day 14 is where we stop trusting our laptops.

Up to now, you ran commands manually and you could “make it work” locally. In a team, that isn’t enough. Every Pull Request needs a clean, repeatable check that runs the same way for everyone.

**If it isn’t tested in CI, it’s broken.**

Today we build a slim CI pipeline using GitHub Actions that runs `dbt deps`, `dbt compile`, and `dbt build` on every Pull Request.

---

## What you will do today

By the end of Day 14, you will:

* Add a GitHub Actions workflow at `.github/workflows/dbt.yml`.
* Configure a secret for Snowflake auth (`SNOWFLAKE_PASSWORD`).
* Trigger a CI run on every Pull Request.
* See a CI run fail, fix the cause, and re-run successfully.
* Generate dbt docs in CI (briefly) so you understand how automation extends beyond tests.

---

## What you will not do today

We keep the pipeline intentionally simple.

* No Jenkins (that comes later).
* No dbt Cloud.
* No Docker-based custom runners.

We will use **`ubuntu-latest`** as the runner.

---

## How today is structured

1. **Theory**

   * `01-ci-concepts.md` — what CI is and why we check code before merge.
   * `02-github-actions-setup.md` — how the workflow YAML works and how we run dbt.

2. **Hands-on lab**

   * `labs/day14/README.md` — you will commit the workflow file, open a PR, and watch the checks.

---

## Prerequisites

You should already have:

* A working dbt project from previous days.
* A Snowflake account and working credentials.
* A GitHub repository with your dbt project pushed.

---

## How to read and work today

* Read the theory first.
* Then do the lab.
* If CI fails, do not “try random fixes.”

  * Read the log.
  * Identify the first error.
  * Fix only that.
  * Push the fix.

That workflow (logs → first error → fix → push) is how you will work in real teams.

---

## What “Slim CI” means in this course

A slim CI pipeline does only what it must do to protect merges.

For our dbt project, that means:

* Install the same tooling every time.
* Resolve dependencies (`dbt deps`).
* Ensure the project compiles (`dbt compile`).
* Build models and run tests (`dbt build`).

If these pass on the PR, we are safe to merge.

---

## Files for today

* `docs/day14/01-ci-concepts.md`
* `docs/day14/02-github-actions-setup.md`
* `labs/day14/README.md`

Proceed to `01-ci-concepts.md`.
