# Core dbt Commands

This document explains the **primary dbt commands** you will use during development.

The goal is not memorization.
The goal is understanding **what each command does** and **when it is appropriate**.

---

## How to think about dbt commands

dbt commands fall into three broad categories:

* **Validation** — Is the project wired correctly?
* **Preparation** — Is everything the project depends on available?
* **Execution** — Build and validate data assets

Each command has a clear purpose.
Using the wrong command at the wrong time creates confusion.

---

## `dbt debug`

### When it is used

`dbt debug` is used **before doing any real work**.

You run it when:

* A project is newly set up
* You are unsure about connectivity
* Something feels misconfigured

---

### What it does

At a high level, `dbt debug` checks:

* The project configuration is readable
* The profile configuration can be found
* dbt can connect to the warehouse

It answers a simple question:

> “Is dbt able to run here at all?”

---

### What it touches

* Reads `dbt_project.yml`
* Reads `profiles.yml`
* Attempts a connection to the warehouse

It does **not**:

* Build models
* Create tables or views
* Modify data

---

## `dbt deps`

### When it is used

`dbt deps` is used when a project relies on **external packages**.

You typically run it:

* After cloning a project
* After package versions change

---

### What it does

At a high level, `dbt deps`:

* Reads package definitions
* Downloads required dependencies
* Makes them available to the project

It prepares the project.

---

### What it touches

* Reads project configuration
* Writes downloaded packages locally

It does **not**:

* Execute models
* Touch warehouse data

---

## `dbt run`

### When it is used

`dbt run` is used to **build data models**.

You use it when:

* You want SQL transformations to execute
* You want objects created or updated in the warehouse

---

### What it does

At a high level, `dbt run`:

* Discovers models in the project
* Resolves dependencies between them
* Executes SQL in the correct order

Each model is run exactly once per invocation.

---

### What it touches

* Reads SQL files in `models/`
* Creates or updates tables and views
* Writes results to the warehouse

---

## `dbt test`

### When it is used

`dbt test` is used **after models exist**.

You run it when:

* You want to verify assumptions
* You want to catch data quality issues

---

### What it does

At a high level, `dbt test`:

* Executes test queries
* Checks expected conditions
* Reports pass or fail

Tests do not transform data.
They only validate it.

---

### What it touches

* Reads test definitions
* Runs validation queries in the warehouse

It does **not**:

* Create new models
* Modify existing data

---

## `dbt build`

### When it is used

`dbt build` is used when you want a **complete, ordered run**.

It is typically used when:

* You want models and tests together
* You want consistent execution behavior

---

### What it does

At a high level, `dbt build`:

* Builds models
* Runs tests

It is a **composed command**.

---

### What it touches

* Models
* Tests
* Warehouse objects created by models

It follows lineage strictly.

---

## Why command choice matters

Each command has a narrow responsibility.

Using the right command:

* Reduces noise
* Speeds up feedback
* Makes intent clear

Using the wrong command:

* Hides real issues
* Slows iteration
* Confuses results

Understanding purpose comes before automation.

Execution discipline starts here.
