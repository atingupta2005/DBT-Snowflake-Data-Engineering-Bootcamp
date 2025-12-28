# Day 09 Lab — dbt Sources, Source Tests, and Freshness

Today you will formalize the boundary between raw data and transformations.

By the end of this lab, your dbt project will:

* explicitly declare the five raw Olist tables as **sources**
* attach **source-level tests** to key columns
* configure **freshness** on raw orders data

This lab is about correctness and trust.
Do not add new transformations.

## Before you start

You need:

* your dbt project from previous days
* access to the Snowflake database/schema that contains the raw Olist tables

You will run only a few commands repeatedly.
Do them in order.

## Lab workflow rules

Follow this loop whenever you change YAML:

1. Parse

```bash
dbt parse
```

2. Run source tests

```bash
dbt test --select source:*
```

3. Run freshness

```bash
dbt source freshness
```

If `dbt parse` fails, stop.
Fix YAML before running tests or freshness.

## Task 0 — Confirm your environment

From the dbt project root, run:

```bash
dbt debug
```

Expected output:

* dbt connects successfully
* your profile is found

If `dbt debug` fails, fix that before continuing.

## Task 1 — Create a source YAML file for Day 09

You will define your raw Olist tables in YAML.

### 1.1 Locate where Day 09 YAML should live

Your source YAML must be inside the dbt project under `models/`.

Create a Day 09 location that matches how your project is organized.
Use the same structure you have been using for course work.

Example pattern (your folder names may differ):

```bash
mkdir -p models/day09
```

### 1.2 Create a YAML file

Create a new file named `source.yml` inside that folder.

Example:

```bash
touch models/day09/source.yml
```

Do not create files outside the dbt project.

## Task 2 — Define the Olist source and tables

Open your `models/day09/source.yml` and add a valid YAML skeleton.

### 2.1 Add the required version and sources block

Your file must start with:

* `version: 2`
* `sources:`

Write a source named `olist`.

### 2.2 Set the correct raw schema

You must set the `schema:` field to the schema where the raw Olist tables exist.

Use the same schema you have been querying in earlier days.

Do not guess.
If you are unsure, confirm by running a simple SQL query in Snowflake.

### 2.3 Declare the five raw tables

Under your `olist` source, declare exactly these tables:

* `customers`
* `orders`
* `order_items`
* `payments`
* `products`

Do not add new tables.

### Checkpoint A — YAML parses

Run:

```bash
dbt parse
```

Expected output:

* parse completes successfully

If it fails:

* read the YAML error carefully
* fix indentation, missing dashes, or invalid structure
* rerun `dbt parse`

## Task 3 — Add lightweight table and column metadata

You will now add descriptions.
Keep them short and useful.

### 3.1 Add a table description for each raw table

For each of the five tables, add:

* `description:`

Write one sentence that answers:

* what system this is from (Olist)
* what one row represents

### 3.2 Add column descriptions for key columns only

For each table, add `columns:` and document the join keys you already used earlier in the course.

Rules:

* do not document every column
* focus on IDs and timestamps
* keep each description to one sentence

### Checkpoint B — YAML still parses

Run:

```bash
dbt parse
```

Expected output:

* parse completes successfully

## Task 4 — Add key column tests (not_null, unique)

Now you will protect the raw boundary.

### 4.1 Identify primary keys you relied on in earlier days

You have already been joining these tables.
Use the same key columns you used in your staging models.

For each raw table:

* pick the primary key column you treated as the identifier
* add `not_null`
* add `unique`

Do not invent new keys.
If you are unsure, check your staging SQL from earlier days.

### 4.2 Add not_null on foreign keys you join on

Foreign keys used in joins must not be NULL if you expect stable joins.

Add `not_null` to the foreign key columns you used as join keys.

### Checkpoint C — Source tests run

Run:

```bash
dbt test --select source:*
```

Expected output:

* tests run
* some may fail depending on raw data quality

If a test fails:

* note which test failed
* do not delete the test immediately
* move to the “Failure investigation” section later in this lab

## Task 5 — Add relationship tests between raw tables

Now you will assert that foreign keys match parent keys.

### 5.1 Add relationships for the join paths you already used

Add relationship tests for these common Olist join paths (only if you used them earlier):

* `orders` → `customers`
* `order_items` → `orders`
* `order_items` → `products`
* `payments` → `orders`

Your relationship tests must reference:

* the parent table using `source('olist', '<parent_table>')`
* the parent key column you expect to match

Do not create relationships that you cannot justify from prior joins.

### Checkpoint D — Relationship tests run

Run:

```bash
dbt test --select source:*
```

Expected output:

* tests run
* relationship tests may fail in messy data

If relationship tests fail:

* capture the failure message
* move to “Failure investigation”

## Task 6 — Add freshness configuration on raw orders

Freshness detects ingestion delay.
It is separate from data quality tests.

### 6.1 Choose a loaded-at field on raw orders

Pick a timestamp column in the raw `orders` table that represents recency.

Use a timestamp column you have already used successfully in earlier queries.

Rules:

* do not guess column names
* do not pick a nullable timestamp if you can avoid it

### 6.2 Add freshness thresholds

Add:

* `loaded_at_field`
* `freshness` with `warn_after` and `error_after`

Pick thresholds that are easy to reason about:

* warn threshold should be smaller than error threshold
* periods must be valid (for example: `hour`, `day`)

### Checkpoint E — Freshness runs

Run:

```bash
dbt source freshness
```

Expected output:

* dbt reports freshness status for the orders source
* pass/warn/error depending on data recency

If freshness errors out:

* confirm your loaded-at field column exists
* confirm it is a timestamp (or behaves like one)

## Failure investigation (required)

You are expected to see at least one failure at some point.
That is the point of today.

When something fails, do not panic.
Use this checklist.

### A) `dbt parse` fails

This is a YAML structure problem.

Common causes:

* indentation
* missing dash (`-`) for a list
* mixing tabs and spaces

Fix:

* correct the YAML
* rerun `dbt parse`

### B) `not_null` fails

Meaning:

* upstream is emitting NULLs for a column you expected to be populated

Actions:

* confirm you picked the correct key column
* inspect failing rows in Snowflake
* decide whether the test is correct for this dataset

In real work, you typically keep the test and fix ingestion.

### C) `unique` fails

Meaning:

* duplicates exist for a key you expected to be unique

Actions:

* confirm the column is really intended as a key
* inspect duplicates and count how many

In real work, this is often caused by ingestion append bugs.

### D) `relationships` fails

Meaning:

* child keys do not have matching parent rows

Actions:

* confirm your parent table and key are correct
* inspect a few failing IDs

In real work, this may indicate:

* ingestion loaded tables out of order
* late-arriving dimensions
* partial loads

### E) Freshness warns/errors

Meaning:

* data is older than your threshold

Actions:

* confirm your loaded-at field is correct
* confirm the latest timestamp in the table

In real work, freshness failures drive operational alerts.

## What to submit (for classroom check)

When you are done, you should be able to run these successfully (even if some tests warn/fail due to data):

```bash
dbt parse
```

```bash
dbt test --select source:*
```

```bash
dbt source freshness
```

Be ready to explain:

* which tests you added
* what failures you saw
* what those failures indicate

Do not remove tests just to make output green.
The goal is to learn the failure signals.
