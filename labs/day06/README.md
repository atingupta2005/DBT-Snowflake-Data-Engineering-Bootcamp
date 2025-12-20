# Day 06 Labs — Core dbt Execution Validation

## Lab intent

These labs are your **first controlled interaction with dbt execution**.

The goal is **not** to build analytics models.
The goal is to:

* Validate that dbt is wired correctly
* Observe dbt behavior during execution
* Build confidence reading dbt output

If something fails, that is part of the learning.

---

## What you will practice

By completing these labs, you will:

* Run core dbt commands safely
* Confirm project and profile configuration
* See how dbt discovers and orders work
* Understand what dbt touches during execution

You are not expected to optimize or customize anything.

---

## Lab rules (important)

Follow these rules strictly:

* Do not modify project structure
* Do not write new business logic
* Do not add complex SQL
* Do not explore advanced flags

Stay within the task boundaries.

---

## Lab 01 — Validate dbt setup

### Objective

Confirm that your dbt project and environment are correctly configured.

### Tasks

1. Navigate to the root of your dbt project.
2. Run the command that validates project configuration and connectivity.
3. Observe the output carefully.

### What to look for

* dbt can read the project
* dbt can find the profile
* dbt can connect to the warehouse

If this step fails, stop and ask for help.
Do not proceed blindly.

---

## Lab 02 — Prepare project dependencies

### Objective

Ensure all required project dependencies are available locally.

### Tasks

1. Run the command that installs dbt project dependencies.
2. Verify that dependencies are downloaded successfully.

### What to look for

* No missing package errors
* Clean completion message

This lab prepares the project for execution.

---

## Lab 03 — Observe model execution order

### Objective

Understand how dbt determines execution order.

### Tasks

1. Run the command that executes dbt models.
2. Watch the order in which models run.
3. Note that order is not based on filenames.

### What to look for

* Upstream models run first
* Downstream models wait for dependencies

Focus on behavior, not speed.

---

## Lab 04 — Validate tests conceptually

### Objective

Understand how dbt validates assumptions.

### Tasks

1. Run the command that executes dbt tests.
2. Observe how dbt reports pass and fail results.

### What to look for

* Tests do not create models
* Tests report outcomes clearly

Do not attempt to fix failures yet.

---

## Lab 05 — Combined execution

### Objective

See how dbt performs a full, ordered run.

### Tasks

1. Run the command that builds models and runs tests together.
2. Observe the unified execution flow.

### What to look for

* Models run before tests
* Failures stop downstream work

This lab shows how dbt enforces discipline.

---

## Completion checklist

Before moving on, confirm that you:

* Successfully ran each core dbt command
* Understood what each command did
* Can explain why execution order occurred

If anything feels unclear, revisit Day 06 theory docs.

---

## What comes next

Day 07 focused on modeling design.
Day 08 will combine:

* Execution discipline from today
* Modeling judgment from Day 07

Do not rush ahead.
Understanding behavior now prevents confusion later.
