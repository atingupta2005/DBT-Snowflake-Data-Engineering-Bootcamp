# Day 13 Lab — Environments, Targets, and Tag-Based Runs

Today you will make your dbt project safer to operate.

You are **not** creating new models today.

You will do three tasks:

* Task 1: Update `profiles.yml` to include a `prod` target
* Task 2: Add tags to existing models
* Task 3: Execute runs targeting specific tags and environments

This lab assumes you already have a working dbt project from earlier days.

---

## Before you start

### 1) Activate your Python virtual environment

From the repository root:

```bash
source ~/.venv/bin/activate
```

Expected outcome:

* your shell prompt shows `(.venv)`

### 2) Confirm dbt is installed in the venv

```bash
dbt --version
```

Expected outcome:

* dbt prints a version

### 3) Confirm you are in the dbt project directory
```
cd ~/<project-dir>
```

Run this from the folder containing `dbt_project.yml`:

```bash
ls
```

Expected outcome:

* you see `dbt_project.yml` in the output

---

## Task 1 — Add a PROD target to `profiles.yml`

In this course, “PROD” is simulated by writing to a different schema.

* DEV schema: `OLIST_DEV`
* PROD schema: `OLIST_PROD`

Same Snowflake account. Same database. Same warehouse.

The isolation comes from writing dbt outputs into different schemas.

### Step 1.1 — Locate your `profiles.yml`

### Step 1.2 — Confirm your project’s profile name

Your profile name must match the `profile:` value in `dbt_project.yml`.

Open `dbt_project.yml` and find:

```yaml
profile: dev
```

That exact value must be the top-level key in `profiles.yml`.

If the names do not match, dbt will load the wrong profile or fail to connect.

### Step 1.3 — Add a second output named `prod`

Open `profiles.yml` and find your profile block (the top-level key from Step 1.2).

Inside it, you should see:

* `outputs:`
* `target:`

Add a second output under `outputs:` named `prod`.

Hard rules:

* `threads` must be a literal integer (example: `threads: 4`)
* Use `env_var()` for sensitive fields (do **not** hardcode passwords/tokens)

Practical approach:

* Copy your existing `dev` output
* Paste it as `prod`
* Change only what must change for PROD

What should differ (minimum):

* `schema: OLIST_DEV` for `dev`
* `schema: OLIST_PROD` for `prod`

Example of the *shape* you are aiming for (do not copy blindly; match your project and adapter):

```yaml
outputs:
  dev:
    schema: OLIST_DEV
    threads: 4
  prod:
    schema: OLIST_PROD
    threads: 4
```

### Step 1.4 — Snowflake cross-check: confirm schemas exist

In a Snowflake worksheet, run:

```sql
SHOW SCHEMAS LIKE 'OLIST_%';
```

Expected outcome:

* you see `OLIST_DEV` and `OLIST_PROD`

If one is missing and you have permission, create it:

```sql
CREATE SCHEMA IF NOT EXISTS OLIST_DEV;
CREATE SCHEMA IF NOT EXISTS OLIST_PROD;
```

If schema creation fails, you need a role with the right permissions.

### Step 1.5 — Set environment variables required by your profile

If your `profiles.yml` uses `env_var()`, your terminal session must have those variables set.

Set variables in your current terminal session using `export`.

Example pattern (names are examples — use the exact names referenced in your `profiles.yml`):

```bash
export SNOWFLAKE_ACCOUNT="<your_account>"
export SNOWFLAKE_USER="<your_user>"
export SNOWFLAKE_PASSWORD="<your_password>"
export SNOWFLAKE_ROLE="<your_role>"
export SNOWFLAKE_WAREHOUSE="<your_warehouse>"
export SNOWFLAKE_DATABASE="<your_database>"
```

Quick check that you actually set them:

```bash
env | grep -E "SNOWFLAKE|DBT" || true
```

Expected outcome:

* you see the variable names you rely on

### Step 1.6 — Verify both targets connect

First check DEV:

```bash
dbt debug --target dev
```

Then check PROD:

```bash
dbt debug --target prod
```

Expected outcome for both:

* dbt prints a connection check and ends with “All checks passed” (or equivalent)

If PROD fails but DEV works, the most common causes are:

* typo in the schema name
* schema does not exist
* role can’t create objects in the PROD schema
* missing environment variable

Do not continue until both targets pass.

---

## Task 2 — Add tags to existing models

You will tag existing models so you can run subsets of the project.

You will add tags in `dbt_project.yml`.

Tag names for today:

* `hourly`
* `daily`
* `heavy`

Why tags matter (real-world use cases):

* **Operations:** run different slices on different schedules without branching the project.
* **Cost control:** exclude heavy models during business hours.
* **Debugging:** rerun only the slice you changed.

### Step 2.1 — Choose models for each tag

Use these rules.

Hourly:

* fast models
* models needed for frequent refresh
* lightweight staging models and lightweight marts

Daily:

* most marts
* models that do not need frequent refresh

Heavy:

* slow models
* wide aggregations
* large joins

Pick at least:

* 2 models tagged `hourly`
* 2 models tagged `daily`
* 1 model tagged `heavy`

A model can have multiple tags.

Example:

* a daily mart can also be heavy

### Step 2.2 — Add tags in `dbt_project.yml`

You can tag in two common ways.

#### Option A: tag by folder

This is what teams do when a directory has a clear operational meaning.

Example patterns (adapt to your folder names):

```yaml
models:
  <your_project_name>:
    staging:
      +tags: ["hourly"]
    marts:
      +tags: ["daily"]
```

#### Option B: tag by model name

This is what teams do when a few specific models need special treatment.

Example pattern:

```yaml
models:
  <your_project_name>:
    marts:
      some_mart_model:
        +tags: ["daily", "heavy"]
      another_mart_model:
        +tags: ["daily"]
```

Important:

* YAML indentation matters
* Do not put secrets in `dbt_project.yml`

### Step 2.3 — Prove dbt sees your tags

List models tagged `daily`:

```bash
dbt ls --select tag:daily
```

List models tagged `hourly`:

```bash
dbt ls --select tag:hourly
```

List models tagged `heavy`:

```bash
dbt ls --select tag:heavy
```

Expected outcome:

* each command prints one or more model names

If a command returns nothing:

* you tagged the wrong place in `dbt_project.yml`, or
* indentation broke the config

Fix it before continuing.

---

## Task 3 — Run by target and tag selection

Now you will run selective slices of the project in different environments.

You will run:

* an hourly slice in DEV
* a daily slice in PROD

You will also practice excluding heavy models.

Core habit:

* **preview the selection with `dbt ls` before you run**

That habit prevents accidental full rebuilds.

### Step 3.1 — Preview and run hourly models in DEV

Preview (use `+` to include upstream dependencies):

```bash
dbt ls --select +tag:hourly
```

If the list looks too large, stop and fix tags.

Run:

```bash
dbt run --target dev --select +tag:hourly
```

Expected outcome:

* dbt builds a subset of models

### Step 3.2 — Snowflake cross-check: confirm DEV writes into `OLIST_DEV`

In Snowflake, run:

```sql
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_ROLE(), CURRENT_WAREHOUSE();
```

Then list objects in the DEV schema (you may need to set your database first):

```sql
-- If needed
-- USE DATABASE <your_database>;

SHOW TABLES IN SCHEMA OLIST_DEV;
SHOW VIEWS IN SCHEMA OLIST_DEV;
```

Expected outcome:

* you see tables/views created by dbt in `OLIST_DEV`

Pick one object name from the output and check row count:

```sql
SELECT COUNT(*) AS row_count
FROM OLIST_DEV.<one_object_from_show_tables_or_views>;
```

### Step 3.3 — Preview and run daily models in PROD

Treat this like production. Confirm the target first:

```bash
dbt debug --target prod
```

Preview the run set:

```bash
dbt ls --select +tag:daily
```

Run:

```bash
dbt run --target prod --select +tag:daily
```

Expected outcome:

* dbt builds into the `OLIST_PROD` schema

### Step 3.4 — Snowflake cross-check: confirm PROD writes into `OLIST_PROD`

List objects in the PROD schema:

```sql
SHOW TABLES IN SCHEMA OLIST_PROD;
SHOW VIEWS IN SCHEMA OLIST_PROD;
```

Pick one object name and check row count:

```sql
SELECT COUNT(*) AS row_count
FROM OLIST_PROD.<one_object_from_show_tables_or_views>;
```

Optional comparison (use the *same* object name if it exists in both schemas):

```sql
SELECT
  (SELECT COUNT(*) FROM OLIST_DEV.<object_name>)  AS dev_count,
  (SELECT COUNT(*) FROM OLIST_PROD.<object_name>) AS prod_count;
```

If counts differ, do not panic.

Most common reasons:

* you ran different slices (hourly vs daily)
* dependencies differ because of `+`
* your tagging decisions differ

### Step 3.5 — Run daily but exclude heavy (PROD)

Preview first:

```bash
dbt ls --select +tag:daily --exclude tag:heavy
```

Run:

```bash
dbt run --target prod --select +tag:daily --exclude tag:heavy
```

Expected outcome:

* daily models run
* heavy-tagged models do not run

Operational example:

* a team runs `daily` every morning
* `heavy` runs after-hours

Same project. Same code. Different selection.

### Step 3.6 — Run heavy models separately (DEV only)

Preview:

```bash
dbt ls --select tag:heavy
```

Run:

```bash
dbt run --target dev --select tag:heavy
```

Expected outcome:

* only heavy models run

---

## What to do if you get stuck

### If dbt cannot connect

1. Re-run the debug step:

```bash
dbt debug --target dev
```

2. Confirm environment variables exist:

```bash
env | grep -E "SNOWFLAKE|DBT" || true
```

3. Confirm schemas exist in Snowflake:

```sql
SHOW SCHEMAS LIKE 'OLIST_%';
```

### If selection runs nothing

Start here:

```bash
dbt ls --select tag:daily
```

If it’s empty, fix tags in `dbt_project.yml`.

Most common mistakes:

* tags configured under the wrong project key
* indentation error
* you edited a different dbt project than the one you are running

### If you accidentally ran against the wrong target

Stop.

Do not try to “undo” with more dbt runs.

First confirm:

* which target you used
* which schema was written

Snowflake cross-check:

```sql
SHOW TABLES IN SCHEMA OLIST_DEV;
SHOW TABLES IN SCHEMA OLIST_PROD;
```

Then clean up:

* drop the wrong objects in the wrong schema
* re-run with the correct target

In a real team, you would report this immediately.

---

## Checkpoint (what must be true before you move on)

You are done when:

* `dbt debug --target dev` passes
* `dbt debug --target prod` passes
* `dbt ls --select tag:daily` returns models
* `dbt ls --select tag:hourly` returns models
* `dbt ls --select tag:heavy` returns models
* you have run at least one tag-based build in DEV and one in PROD
* you verified in Snowflake that objects exist in both `OLIST_DEV` and `OLIST_PROD`
