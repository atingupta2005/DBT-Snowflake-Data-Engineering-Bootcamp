# 01 — Environments and Targets

Today we stop pretending “one environment” is enough.

If you only ever develop in a single schema, you eventually run a command that hurts you.

If you develop in the same schema that other people rely on, you eventually run a command that hurts everyone.

This file is about avoiding that mistake with boring, reliable mechanics:

* Separate environments
* Clear schema naming
* dbt targets
* Running the same project in different contexts

## What an “environment” actually is

In most companies, “environment” is a bundle of decisions:

* Which Snowflake account or region
* Which database
* Which warehouse
* Which schema
* Which role and permissions
* Which data retention rules

In this course, we simulate environments using **different schemas**.

That is enough to learn the operational discipline:

* DEV is safe to break.
* PROD is not.

We will use:

* DEV schema: `OLIST_DEV`
* PROD schema: `OLIST_PROD`

The raw Olist tables stay in the same place.

* Sources stay where they are.
* Only your modeled outputs move between schemas.

## Why isolation exists

Isolation exists for safety.

When you separate DEV from PROD, you get three practical benefits.

### 1) You can iterate without fear

In DEV you will:

* Re-run models repeatedly
* Drop and recreate tables
* Change logic and compare results
* Rename columns
* Add and remove models

That is normal development.

If you do that in PROD, you create churn that breaks downstream consumers.

### 2) You reduce blast radius

If a model is wrong in DEV, the impact is limited.

If a model is wrong in PROD, you can:

* Break dashboards
* Trigger incorrect alerts
* Ship incorrect numbers to leadership
* Cause expensive backfills and reruns

Most real production incidents are not “complex bugs.”

They are “someone ran the wrong command in the wrong place.”

### 3) You can control cost

DEV is where queries are messy.

People will:

* Run large joins while exploring
* Materialize models too early
* Forget to filter
* Re-run expensive models repeatedly

If DEV runs on PROD warehouses, costs spike.

Even when you simulate environments with schemas, keep the mindset:

* DEV work should be cheaper and more flexible.
* PROD work should be predictable and measurable.

## Schema naming conventions that prevent mistakes

You want schema names that make it hard to do the wrong thing.

Good:

* `OLIST_DEV`
* `OLIST_PROD`

Bad:

* `OLIST`
* `OLIST2`
* `OLIST_NEW`

Bad naming makes it easy to forget what the schema is for.

When you type commands all day, you rely on pattern recognition.

If the pattern is clear, you catch mistakes before they happen.

### A practical safety check

Before you run any dbt command, you should be able to answer this without thinking:

* “Which schema will this write to?”

If you cannot answer instantly, stop.

Fix your naming or your target.

## The promotion lifecycle

Promotion is how changes get into PROD without chaos.

The key mental model is:

* **Code moves. Data stays.**

That sounds abstract, so let’s make it concrete.

### What “code moves” means

Your dbt project is code:

* SQL models
* YAML configs
* macros and tests
* documentation

That code is versioned in Git.

When you are ready, you merge it.

That merged code is what you run for PROD.

### What “data stays” means

You do not “copy DEV tables into PROD.”

Instead:

* DEV tables are disposable.
* PROD tables are built from sources using trusted code.

Even in this training setup (where PROD is just another schema), we stick to the habit:

* A PROD run builds PROD outputs.
* A DEV run builds DEV outputs.

### A realistic example

You change the definition of a revenue metric.

In DEV:

* You adjust the SQL.
* You run the model.
* You validate row counts and spot-check results.

Once you trust it:

* You promote the code.
* You run the same model against PROD.

The number changes in PROD only because the code changed, not because you copied data around.

## How dbt targets map to environments

dbt uses a **profile** to know where to connect.

Inside the profile you define:

* Outputs (connection settings)
* Targets (which output to use by default)

dbt calls each named output a **target**.

In this course we will create at least two targets:

* `dev`
* `prod`

Both targets connect to Snowflake.

The big difference is the schema they write into.

* `dev` writes to `OLIST_DEV`
* `prod` writes to `OLIST_PROD`

## How profiles.yml is structured

`profiles.yml` lives in your dbt profiles directory.

In many corporate setups, this file is not committed to Git.

That is fine.

You should treat it as machine-specific configuration.

The important part is the structure.

Here is what you need to recognize when you see a real `profiles.yml`.

### Profile name

The top-level key is the **profile name**.

This must match the `profile:` value in `dbt_project.yml`.

If these don’t match, dbt will not find your profile.

### Outputs and target

Inside the profile:

* `outputs:` contains named targets (like `dev`, `prod`).
* `target:` chooses which one is the default.

A simplified shape looks like this:

```yaml
my_dbt_profile:
  target: dev
  outputs:
    dev:
      ...
    prod:
      ...
```

The rest is connection configuration.

You will implement a working version in the lab.

## Running dbt against different targets

You do not need a separate project for each environment.

You use the same project.

You switch targets at runtime.

### Default target

If you run dbt without a target flag, it uses whatever `target:` is set to in `profiles.yml`.

That is why the default should usually be `dev`.

In real teams, people sometimes set the default to `dev` and make `prod` explicit.

That forces a conscious choice.

### Using the --target flag

You can override the default like this:

```bash
dbt run --target prod
```

dbt will connect using the `prod` output in your profile.

If that output points at `OLIST_PROD`, your models will be built there.

### The safety habit: print the target before running

When you are learning, it is easy to forget what target you are on.

Build a habit:

1. Run `dbt debug --target dev`
2. Confirm the schema in the debug output
3. Then run models

Example:

```bash
dbt debug --target dev
```

Expected outcome:

* dbt connects successfully
* The connection context is for DEV

Then:

```bash
dbt debug --target prod
```

Expected outcome:

* dbt connects successfully
* The connection context is for PROD

If `dbt debug` fails, do not “try random fixes.”

Check:

* Your environment variables
* Your Snowflake account/user
* Your role and permissions
* The schema name

## Environment variables: the minimum security bar

Credentials do not belong in Git.

That is not a preference.

That is how leaks happen.

The safer baseline is:

* `profiles.yml` references environment variables
* Your shell provides the values at runtime

That way:

* The file can be shared safely
* Secrets are not visible in diffs
* You can rotate credentials without editing code

### What we mean by “secrets”

In Snowflake, treat these as secrets:

* password
* private key
* OAuth token

Also treat these as sensitive in many companies:

* account identifier
* username
* role

Different teams draw that line differently.

The safe habit is to keep anything connection-related out of Git when possible.

We will implement this cleanly in `03-security-best-practices.md`.

For now, the key idea is:

* Targets define context.
* Context should not expose secrets.

## What you will do in the lab

In the lab you will:

* Add a `prod` target to your `profiles.yml`
* Make sure `dev` writes to `OLIST_DEV`
* Make sure `prod` writes to `OLIST_PROD`
* Run dbt against both targets to prove they work

You will also start using selective runs with tags.

That is in the next file.
