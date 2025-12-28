# 02 — Tags and Model Selection

Once you have multiple environments, the next production problem shows up fast:

* You do not want to run everything every time.

In DEV, it is common to run the whole project while you are learning.

In production, “run everything” is often:

* slow
* expensive
* unnecessary
* risky

Today we introduce a controlled way to run subsets of the project:

* tags
* selection
* exclusion

This is not automation.

This is you running the right slice of dbt intentionally.

## Why selective runs matter

Most dbt projects grow over time.

* A handful of staging models
* A few marts
* Then dozens
* Then hundreds

If your production job rebuilds 200 models every hour, the warehouse bill becomes a surprise.

Even worse: a full run increases the chance that one broken model fails the whole pipeline.

Selective runs let you:

* run fast models frequently
* run heavy models less frequently
* isolate experimental work
* avoid expensive backfills during normal runs

### A real-world pattern

A common operating model is:

* Hourly job

  * builds lightweight models
  * refreshes near-real-time reporting

* Daily job

  * builds heavier marts
  * runs more tests
  * updates slower-changing dimensions

We will simulate this pattern using tags:

* `hourly`
* `daily`

## What a tag is

A tag is just a label you attach to a model (or other resources).

Example tag names:

* `hourly`
* `daily`
* `heavy`
* `finance`
* `marketing`

Tags are not magic.

They only matter when you use them in selection.

## Where tags are configured

You can add tags in two common places:

1. In a model’s `config()` block inside the SQL file
2. In a `dbt_project.yml` model config section

In this course, we prefer keeping model configuration centralized.

That usually means:

* tags in `dbt_project.yml`
* SQL files focused on SQL

You will do that in the lab.

## How selection works

dbt selection is a language.

We will start with the basics you can use in production immediately.

### Select by tag

To run all models tagged `daily`:

```bash
dbt run --select tag:daily
```

Expected outcome:

* dbt builds only the models that have the `daily` tag
* dbt will still build their upstream dependencies if needed (depending on the selection syntax you use)

We will talk about dependencies in a moment.

### Exclude by tag

To run everything except models tagged `heavy`:

```bash
dbt run --exclude tag:heavy
```

This is useful when:

* a model is too expensive for normal runs
* a model is temporarily unstable
* you want to keep a large backfill model out of daily workflows

### Combine select and exclude

To run daily models but skip heavy ones:

```bash
dbt run --select tag:daily --exclude tag:heavy
```

This is a practical production pattern.

* You keep the normal job stable.
* You still have a path to run heavy models separately.

## Tags vs folders

Some teams select by folder instead of tags.

Example:

```bash
dbt run --select marts
```

That works when your project structure is clean.

Tags are more flexible.

You can tag across folders.

Example:

* A staging model might be part of the hourly job.
* A mart model might be daily.

Folder selection cannot express that cleanly without awkward restructuring.

## Understanding dependencies when selecting

This is where people get confused.

When you select a model, dbt can also include its parents or children.

There are operators for this.

We will use a few that matter day-to-day.

### Select only the tagged models

This runs the models that are tagged, but does not automatically include parents.

```bash
dbt run --select tag:daily
```

If the tagged model depends on upstream models that are not built yet, the run may fail.

In DEV, you might not notice this because you recently ran everything.

In PROD, this can cause broken runs if you rely on stale upstream data.

### Include parents (upstream)

If you want to build a model and everything it depends on, you use `+`.

Example:

```bash
dbt run --select +tag:daily
```

This means:

* select models tagged `daily`
* also include their upstream parents

This is often safer for production jobs.

It ensures your daily marts are built on fresh staging outputs.

### Include children (downstream)

If you want the tagged models and everything that depends on them:

```bash
dbt run --select tag:daily+
```

This is less common for production scheduling.

It is more useful for impact testing.

Example:

* “If I rebuild this staging model, what marts does it affect?”

### Include both parents and children

```bash
dbt run --select +tag:daily+
```

Use this carefully.

It can expand the run more than you expect.

## Practical tagging strategy for this project

We want tags that map to real run scenarios.

For Day 13, use two scheduling-style tags:

* `hourly`
* `daily`

Also use one cost-style tag:

* `heavy`

These are enough to simulate common operating patterns.

### Suggested meaning

* `hourly`

  * small incremental-style models
  * lightweight marts
  * fast staging models needed frequently

* `daily`

  * most marts
  * most business reporting models
  * models that do not need hourly freshness

* `heavy`

  * large joins
  * wide aggregations
  * models with slow runtime

In the lab, you will tag existing models.

We are not creating new models today.

## Running tags across environments

Targets and selection work together.

You can run the same selection in DEV and PROD.

Example: build daily models in DEV schema.

```bash
dbt run --target dev --select +tag:daily
```

Example: build daily models in PROD schema.

```bash
dbt run --target prod --select +tag:daily
```

This is the core idea of promotion.

* Same code
* Same selection logic
* Different environment context

## Common classroom failures (and how to avoid them)

### Failure 1: “My tag run built nothing.”

Usually means:

* the models are not actually tagged
* the tag is misspelled
* the tag is configured in the wrong scope

Fix:

* run `dbt ls --select tag:daily` to see what dbt thinks is selected

Example:

```bash
dbt ls --select tag:daily
```

Expected outcome:

* a list of model names

If the list is empty, your tagging config is not being applied.

### Failure 2: “It fails because upstream models don’t exist.”

Usually means:

* you selected a mart but not its staging parents

Fix:

* use `+tag:daily` to include parents

### Failure 3: “I ran daily in prod, but it wrote to dev.”

Usually means:

* you forgot `--target prod`
* your default target is still dev

Fix:

* run `dbt debug --target prod` before `dbt run`

## What you will do in the lab

In the lab you will:

* add tags to existing models via `dbt_project.yml`
* prove selection works using `dbt ls`
* run `hourly` selection in DEV
* run `daily` selection in PROD
* exclude `heavy` models to keep runs predictable

The next file covers security practices for profiles and environment variables.
