# Day 05 Labs — dbt Structure, Boundaries, and Discipline (Conceptual)

## Lab intent

These labs are **non‑executable by design**.

You will not run dbt.
You will not write SQL.
You will not connect to Snowflake.

The goal is to practice **decision‑making** before execution begins.

If you get these decisions wrong, dbt execution later becomes confusing and risky.

---

## How to approach these labs

* Read each scenario carefully
* Decide **where something belongs**
* Be able to explain **why**

There is no typing required.
Your answers should be reasoned, not guessed.

---

## Lab 01 — dbt Project Structure Reasoning

### Objective

Understand the responsibility of core dbt folders.

### Scenario

You are reviewing a dbt project with the following requirements:

* Clean raw order data
* Reuse common logic across models
* Validate assumptions about keys
* Track historical changes in mutable data

### Tasks

For each requirement below, decide **which dbt folder is responsible**:

1. Standardizing column names from raw source tables
2. Reusing a commonly repeated SQL expression
3. Ensuring a primary key is never null
4. Preserving historical versions of changing records

Choose from:

* `models/`
* `macros/`
* `tests/`
* `snapshots/`

Explain your reasoning for each choice.

---

## Lab 02 — Configuration Boundary Classification

### Objective

Practice separating **project configuration** from **environment configuration**.

### Scenario

You are setting up dbt on a new machine.
You encounter the following pieces of information:

* Project name
* Database username
* Default materialization behavior
* Warehouse connection details
* Folder locations for models

### Tasks

For each item, decide:

* Does this belong in `dbt_project.yml`?
* Does this belong in `profiles.yml`?
* Should this be committed to Git?

Be prepared to justify your decisions.

---

## Lab 03 — Environment Mental Model Check

### Objective

Validate your understanding of DEV, TEST, and PROD.

### Scenario

A teammate says:

> “Let’s create a `prod/` folder so production models don’t mix with dev models.”

### Tasks

Answer the following:

1. Is this statement correct or incorrect?
2. What misunderstanding does it reveal?
3. How should environments be thought of instead?

Keep your explanation short and precise.

---

## Lab 04 — Git Hygiene for dbt Projects

### Objective

Identify what belongs in version control.

### Scenario

You see the following files in a dbt repository:

* `dbt_project.yml`
* `profiles.yml`
* SQL model files
* A file containing database passwords
* Documentation markdown files

### Tasks

For each file type, decide:

* Commit to Git
* Never commit to Git

Explain **why** committing the wrong file is risky.

---

## Completion criteria

You are done when you can:

* Clearly explain why dbt has strict boundaries
* Confidently say where something belongs
* Spot common beginner mistakes immediately

There is no execution today.

Correct judgment today prevents failures tomorrow.

---

## What comes next

Day 06 moves from judgment to execution.

Because you understand structure and boundaries:

* dbt commands will feel predictable
* Errors will be easier to diagnose
* Execution will feel controlled

Do not skip this thinking step.
It is intentional.
