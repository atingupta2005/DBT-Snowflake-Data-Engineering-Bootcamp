# dbt Project Structure

## What a dbt project represents

a dbt project is a **container for analytics logic**.

It is not a database.
It is not an execution engine.
It is not a replacement for SQL.

A dbt project:

* Organizes SQL transformations
* Defines how those transformations relate to each other
* Stores configuration that controls behavior

Think of the project as the **source of truth** for analytical logic.
The warehouse executes the SQL.
dbt provides structure and rules around it.

---

## Why dbt is configuration-driven

dbt relies on configuration files to answer questions like:

* Where are my models stored?
* How should files be interpreted?
* What defaults apply to this project?

This approach keeps SQL files focused.

SQL should describe **what** the transformation does.
Configuration should describe **how** dbt treats that SQL.

When configuration is centralized:

* Behavior is predictable
* Reviews are easier
* Changes are intentional

---

## The role of `dbt_project.yml`

`dbt_project.yml` is the **control file** of a dbt project.

It defines:

* The project name
* Folder locations
* Default behaviors
* How dbt should interpret files in this repository

This file answers the question:

> “How should dbt treat the contents of this project?”

It does **not** contain:

* Credentials
* Passwords
* User-specific settings

Those belong elsewhere.

---

## Core folders in a dbt project

A dbt project is organized by **intent**, not by execution order.
Each top-level folder has a clear responsibility.

Understanding these boundaries early prevents confusion later.

---

### `models/`

This folder holds **SQL transformation logic**.

What belongs here:

* Select-based SQL files
* Stepwise transformations
* Business logic expressed as queries

What does not belong here:

* Credentials
* Connection details
* One-off exploratory queries

This folder grows the most over time.
Structure inside `models/` is intentional and reviewed.

---

### `macros/`

This folder holds **reusable logic written once and reused many times**.

Macros are used to:

* Avoid repeating patterns
* Standardize logic
* Centralize small pieces of behavior

Think of macros as helpers.
They support models but do not replace them.

If logic feels generic and repeated, it likely belongs here.

---

### `tests/`

This folder defines **data expectations**.

Tests describe assumptions such as:

* A column should not be null
* A key should be unique

Tests do not transform data.
They **validate outcomes**.

Keeping tests separate makes intent clear:

* Models build
* Tests verify

---

### `seeds/`

This folder contains **small, static datasets**.

Seeds are typically:

* Reference data
* Mappings
* Lookups that change rarely

Seeds are versioned with the project.
They are treated as inputs, not transformations.

---

### `snapshots/`

This folder captures **changes over time**.

Snapshots are used when:

* Source data mutates
* Historical states matter

They preserve history that would otherwise be lost.

Snapshots are purposeful.
They are not a default choice.

---

## Why structure matters before execution

Without structure:

* Files drift
* Logic becomes hard to find
* Reviews slow down

With structure:

* Intent is visible
* Teams collaborate safely
* Projects scale without chaos

This is why structure is introduced **before** running dbt.

Execution comes next.
Discipline comes first.
