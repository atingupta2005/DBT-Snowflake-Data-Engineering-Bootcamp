# profiles.yml and Environment Boundaries

## Why configuration is split in dbt

dbt deliberately separates **project configuration** from **connection configuration**.

This separation is not accidental.
It enforces clear boundaries.

There are two different questions:

* How is this project structured?
* Where and as whom does it run?

Mixing these leads to fragile projects.

---

## What belongs in `dbt_project.yml`

`dbt_project.yml` answers **project-level questions**.

It describes:

* Project identity
* Folder layout
* Default behaviors
* How dbt should interpret files

This file is:

* Shared by the team
* Committed to Git
* Stable across environments

Every developer should have the same `dbt_project.yml`.

---

## What belongs in `profiles.yml`

`profiles.yml` answers **connection-level questions**.

It describes:

* Which warehouse to connect to
* Which database and schema to use
* Which credentials are required

This file is:

* User-specific or environment-specific
* Never committed to Git
* Different across machines

`profiles.yml` exists **outside** the project directory.

That location matters.

---

## Why credentials never live in the project

Credentials change.
Projects should not.

If credentials are committed:

* They leak
* They get copied
* They become hard to rotate

By keeping credentials outside the repository:

* The same project runs everywhere
* Access can be controlled independently
* Rotation does not require code changes

This boundary protects both security and sanity.

---

## Environment variables as the glue

Environment variables provide a safe bridge.

They allow `profiles.yml` to reference values without storing them.

Conceptually:

* The project stays static
* The environment supplies secrets

This keeps:

* Repositories clean
* Configuration flexible
* Behavior predictable

Details of setting variables come later.
Today, understand **why** they exist.

---

## Understanding DEV, TEST, and PROD

dbt does not have built-in environments.

DEV, TEST, and PROD exist only as **execution targets** defined in `profiles.yml`.
An execution target determines **where dbt writes data** and **which credentials are used**.

In dbt, an environment boundary is created by:

* Database
* Schema
* Credentials

Names alone do not create isolation.
Isolation comes from configuration.

---

## Execution Targets in `profiles.yml`

Each profile defines one or more **targets** under `outputs`.

Example:

```yaml
analytics:
  target: dev
  outputs:
    dev:
      database: analytics_dev
      schema: alice
      user: alice
      role: developer

    test:
      database: analytics_test
      schema: dbt_test
      user: ci_user
      role: dbt_test

    prod:
      database: analytics
      schema: analytics
      user: dbt_prod
      role: dbt_prod
```

Each target represents a distinct execution context.

---

## What Happens at Runtime

### Running in DEV

```bash
dbt run --target dev
```

dbt will:

* Build models in `analytics_dev.alice`
* Use developer credentials
* Affect only personal objects

DEV is safe because it is isolated.

---

### Running in TEST

```bash
dbt run --target test
```

dbt will:

* Build models in `analytics_test.dbt_test`
* Use shared test credentials
* Validate changes in a production-like structure

Errors here are visible but contained.

---

### Running in PROD

```bash
dbt run --target prod
```

dbt will:

* Build models in `analytics.analytics`
* Use production credentials
* Overwrite business-facing tables

Every run affects downstream users.

---

## The Default Target

Each profile defines a **default target**:

```yaml
analytics:
  target: dev
```

If no target is specified:

```bash
dbt run
```

dbt will automatically use the default target.

This is equivalent to:

```bash
dbt run --target dev
```

---

## Why the Default Target Matters

The default target determines dbt’s behavior when no explicit choice is made.

Best practice:

* Default target → DEV
* TEST and PROD → require explicit selection

This prevents accidental writes to production.

---

## Dangerous Configuration Example

```yaml
analytics:
  target: prod
```

This makes:

```bash
dbt run
```

execute against production.

This is unsafe on developer machines.

---

## Where Boundaries Really Come From

The real boundary is **where data is written**.

| Target | Database       | Schema    | Risk   |
| ------ | -------------- | --------- | ------ |
| dev    | analytics_dev  | alice     | Low    |
| test   | analytics_test | dbt_test  | Medium |
| prod   | analytics      | analytics | High   |

If two targets write to the same database and schema, they are the same environment, regardless of name.

---

## Key Takeaway

> In dbt, environments are not conceptual.
> They are enforced only through execution targets and write locations.

Clear boundaries come from configuration, not intent or naming.
