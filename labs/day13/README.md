# Day 13 Lab — Environments, Targets, and Tag-Based Runs

You will do three tasks today.

* Task 1: Update `profiles.yml` to include a `prod` target.
* Task 2: Add tags to existing models.
* Task 3: Execute runs targeting specific tags and environments.

This lab assumes you already have a working dbt project from earlier days.

You are not creating new models today.

You are making the project safer to operate.

## Before you start

### 1) Activate your Python virtual environment

From the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

If your environment is already active, you should see `(.venv)` in your shell prompt.

### 2) Confirm dbt is installed in the venv

```bash
dbt --version
```

Expected outcome:

* dbt prints a version

If the command is not found, install dbt the same way you did earlier in the course.

### 3) Confirm you are in the dbt project directory

Run this from the dbt project root (the folder containing `dbt_project.yml`):

```bash
ls
```

Expected outcome:

* you see `dbt_project.yml` in the output

If you do not, `cd` into the correct directory before continuing.

---

## Task 1 — Add a PROD target to profiles.yml

In this course, “PROD” is simulated by writing to a different schema.

* DEV schema: `OLIST_DEV`
* PROD schema: `OLIST_PROD`

You will configure both in `profiles.yml`.

### Step 1.1 — Locate your profiles.yml

dbt stores profiles outside the project by default.

To see where dbt expects it:

```bash
dbt debug --config-dir
```

Expected outcome:

* dbt prints a directory path

In that directory, there should be a `profiles.yml`.

### Step 1.2 — Open profiles.yml and add a prod output

You will add a second target named `prod` under `outputs:`.

Hard rules:

* The profile name must match the `profile:` value in `dbt_project.yml`.
* `threads` must be a literal integer.
* Use `env_var()` for sensitive fields.

In the next task you will validate this file by running `dbt debug`.

Do not guess.

If `dbt debug` fails, fix the profile.

### Step 1.3 — Ensure dev writes to OLIST_DEV and prod writes to OLIST_PROD

Your two outputs must differ at least by schema.

* `dev` schema must be `OLIST_DEV`
* `prod` schema must be `OLIST_PROD`

If your schemas do not exist, create them in Snowflake before continuing.

(You do not need to create tables. dbt will create tables when it runs.)

### Step 1.4 — Verify both targets connect

First check DEV:

```bash
dbt debug --target dev
```

Expected outcome:

* “All checks passed” (or equivalent)

Then check PROD:

```bash
dbt debug --target prod
```

Expected outcome:

* “All checks passed” (or equivalent)

If PROD fails but DEV works, the most common causes are:

* typo in schema name
* schema does not exist
* role cannot create objects in the PROD schema

Do not continue until both targets pass.

---

## Task 2 — Add tags to existing models

You will tag existing models so you can run subsets of the project.

You will add tags in `dbt_project.yml`.

Tag names for today:

* `hourly`
* `daily`
* `heavy`

### Step 2.1 — Decide which existing models belong to hourly vs daily

Use these rules.

Hourly:

* models that are fast
* models needed for frequent refresh
* lightweight staging models and lightweight marts

Daily:

* most marts
* models that do not need frequent refresh

Heavy:

* slow models
* wide aggregations
* large joins

Do not overthink it.

Pick at least:

* 2 models tagged `hourly`
* 2 models tagged `daily`
* 1 model tagged `heavy`

A model can have multiple tags.

Example:

* a daily mart can also be heavy

### Step 2.2 — Add tags in dbt_project.yml

Edit `dbt_project.yml`.

Your goal is:

* the tags exist
* dbt can see them

Do not put secrets here.

### Step 2.3 — Prove dbt sees your tags

List the models tagged `daily`:

```bash
dbt ls --select tag:daily
```

Expected outcome:

* a list of model names

List the models tagged `hourly`:

```bash
dbt ls --select tag:hourly
```

Expected outcome:

* a list of model names

If either command returns nothing:

* your tag config is not applied
* you likely edited the wrong section of `dbt_project.yml`

Fix it before continuing.

---

## Task 3 — Run by target and tag selection

Now you will run selective slices of the project in different environments.

You will run:

* an hourly slice in DEV
* a daily slice in PROD

You will also practice excluding heavy models.

### Step 3.1 — Run hourly models in DEV

Use `+` to include upstream dependencies.

```bash
dbt run --target dev --select +tag:hourly
```

Expected outcome:

* dbt builds a subset of models
* the output shows only the selected models and their parents

### Step 3.2 — Run daily models in PROD

Treat this like production.

Before running, confirm the target:

```bash
dbt debug --target prod
```

Then run the daily slice:

```bash
dbt run --target prod --select +tag:daily
```

Expected outcome:

* dbt builds into the `OLIST_PROD` schema

If you are unsure where it wrote, stop and check your Snowflake schema.

Do not keep running commands until you confirm.

### Step 3.3 — Run daily but exclude heavy

This simulates a “normal daily run” where heavy work is separated.

```bash
dbt run --target prod --select +tag:daily --exclude tag:heavy
```

Expected outcome:

* daily models run
* heavy-tagged models do not run

To verify selection without running, use `dbt ls`:

```bash
dbt ls --select +tag:daily --exclude tag:heavy
```

Expected outcome:

* list of models that would run

### Step 3.4 — Run heavy models separately (DEV only)

This keeps heavy experimentation away from “prod” even in our simulated setup.

```bash
dbt run --target dev --select tag:heavy
```

Expected outcome:

* only heavy models run

---

## What to do if you get stuck

### If dbt cannot connect

* re-run `dbt debug --target dev`
* confirm your environment variables are set
* confirm your schema names exist

### If selection runs nothing

* run `dbt ls --select tag:daily`
* fix tags in `dbt_project.yml`

### If you accidentally ran against the wrong target

Stop.

Do not try to “undo” with more dbt runs.

First confirm:

* which target you used
* which schema was written

Then clean up:

* drop the wrong tables in the wrong schema
* re-run with the correct target

In a real team, you would report this immediately.

In this lab, treat it as practice for the habit:

* confirm target first

---

## Checkpoint (what must be true before you move on)

You are done when:

* `dbt debug --target dev` passes
* `dbt debug --target prod` passes
* `dbt ls --select tag:daily` returns models
* `dbt ls --select tag:hourly` returns models
* You have run at least one tag-based build in DEV and one in PROD
