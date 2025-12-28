# Day 13 — Production Readiness: Environments and Deployment Strategies

Day 13 is where we stop treating dbt like a personal sandbox.

Up to Day 12, most of our work was done in a single DEV target. That was fine because the main risk was “you break your own work.”

From Day 13 onward, the risk changes to “you break someone else’s work” or “you break production.” That is not a learning moment. That is an incident.

## Why developing in production is the root of all evil

When people say “just run it in prod,” they usually mean:

* Same database
* Same schema
* Same tables
* Same permissions
* Same data

That means your development experiment can:

* Overwrite production models
* Drop production tables
* Backfill large volumes and spike warehouse cost
* Create inconsistent results for downstream users
* Break dashboards or alerts

You do not need a complex CI/CD system to avoid this.

You need **environment isolation**:

* DEV is where you build and iterate.
* PROD is where business users consume stable outputs.
* TEST is optional, but it is useful when you want to validate changes without touching PROD.

For this course, we simulate environments by writing to different schemas.

* DEV schema: `OLIST_DEV`
* PROD schema: `OLIST_PROD`

In real companies, PROD might be a different database or even a different Snowflake account.

## What you will learn today

By the end of Day 13, you will be able to:

* Define multiple dbt targets in `profiles.yml` (DEV and PROD).
* Choose the target at runtime using `--target`.
* Use tags to run only a specific slice of the project (for example: `hourly` or `daily`).
* Understand the promotion workflow: **code moves, data stays**.

## How today is structured

You will read the theory first.

Then you will do the lab.

You will not touch instructor materials.

### Student theory (read these first)

1. `01-environments-and-targets.md`

   * Why environments exist
   * How schema naming keeps you safe
   * What “promotion” actually means

2. `02-tags-and-selection.md`

   * Tagging models
   * Running selective builds
   * Excluding expensive work

3. `03-security-best-practices.md`

   * Using environment variables for credentials
   * Keeping secrets out of Git
   * What to do when something leaks

### Student lab (hands-on)

* `labs/day13/README.md`

  * Task 1: Add a PROD target to `profiles.yml`
  * Task 2: Tag models as `hourly` and `daily`
  * Task 3: Run dbt using different targets and tag selections

## What “promotion” means in dbt

Promotion is not “copying tables from DEV to PROD.”

Promotion is:

* You change dbt code in Git.
* You run it in DEV until you trust the results.
* You merge the change.
* You run the same code against the PROD target.

The data in PROD is rebuilt from source using the trusted code.

That is why we say:

* **Code moves. Data stays.**

## Rules for Day 13

* PROD in this lab is just another schema.
* You must still treat it like production.

That means:

* Never run a full refresh in “prod” unless explicitly asked.
* Always confirm your target before running.
* Use selective runs whenever possible.

If you accidentally write DEV models into PROD schema, you will spend the rest of your career explaining why you did it.
